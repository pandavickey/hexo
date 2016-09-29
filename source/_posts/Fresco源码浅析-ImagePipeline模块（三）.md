---
title: Fresco源码浅析-ImagePipeline模块（三）
date: 2016-09-29 15:02:07
tags:
---
ImagePipeline是Fresco读取数据的整个调度系统，作为一个图片加载组件，主要工作流程为：
- 检查内存缓存
- 检查磁盘缓存
- 文件读取或网络请求，并存储到各个缓存。
官方流程图如下：
![imagepipeline.png](/img/imagepipeline.png)

这和主要的图片加载逻辑基本类似，既然如此，那我们就从图片加载组件最主要的两个方面入手分析源码：
1） 如何自定义缓存线程和加载线程的配置；
2） 缓存设计算法。

首先看第一个问题，要看ImagePipeline的配置，我们来分析一下ImagePipelineConfig的源码：
```
  @Nullable private final AnimatedImageFactory mAnimatedImageFactory;
  private final Bitmap.Config mBitmapConfig;
  private final Supplier<MemoryCacheParams> mBitmapMemoryCacheParamsSupplier; //内存缓存数据的策略
  private final CacheKeyFactory mCacheKeyFactory; //缓存键值对的获取
  private final Context mContext;
  private final boolean mDownsampleEnabled;
  private final boolean mDecodeMemoryFileEnabled;
  private final FileCacheFactory mFileCacheFactory; // 文件缓存键值对
  private final Supplier<MemoryCacheParams> mEncodedMemoryCacheParamsSupplier;  //原码内存缓存参数
  private final ExecutorSupplier mExecutorSupplier;  //获取线程池
  private final ImageCacheStatsTracker mImageCacheStatsTracker;//Cache埋点工具
  @Nullable private final ImageDecoder mImageDecoder; //解码器
  private final Supplier<Boolean> mIsPrefetchEnabledSupplier;
  private final DiskCacheConfig mMainDiskCacheConfig;//磁盘缓存配置
  private final MemoryTrimmableRegistry mMemoryTrimmableRegistry;
  private final NetworkFetcher mNetworkFetcher; //网络获取器
  @Nullable private final PlatformBitmapFactory mPlatformBitmapFactory;
  private final PoolFactory mPoolFactory;
  private final ProgressiveJpegConfig mProgressiveJpegConfig;  //渐进图片配置
  private final Set<RequestListener> mRequestListeners;
  private final boolean mResizeAndRotateEnabledForNetwork;
  private final DiskCacheConfig mSmallImageDiskCacheConfig;//小图缓存配置
  private final ImagePipelineExperiments mImagePipelineExperiments;
  ...
```
ImagePipeline的可配置项如下：
```
ImagePipelineConfig config = ImagePipelineConfig.newBuilder()
    .setBitmapMemoryCacheParamsSupplier(bitmapCacheParamsSupplier)  //bitmap缓存配置
    .setCacheKeyFactory(cacheKeyFactory) //设置缓存键值对
    .setEncodedMemoryCacheParamsSupplier(encodedCacheParamsSupplier)//设置原码内存缓存配置
    .setExecutorSupplier(executorSupplier) //各种线程池
    .setImageCacheStatsTracker(imageCacheStatsTracker)//缓存打点
    .setMainDiskCacheConfig(mainDiskCacheConfig) //主磁盘缓存
    .setMemoryTrimmableRegistry(memoryTrimmableRegistry) 
    .setNetworkFetchProducer(networkFetchProducer)//网络请求配置
    .setPoolFactory(poolFactory)
    .setProgressiveJpegConfig(progressiveJpegConfig)//渐进图片配置
    .setRequestListeners(requestListeners)//请求监听
    .setSmallImageDiskCacheConfig(smallImageDiskCacheConfig)//小图缓存
    .build();
Fresco.initialize(context, config);
```
ImagePipeline用到了三个缓存，首先是DiskCache，然后还有两个MemoryCache，分别是保存已解码Bitmap的和保存EncodedImage的缓存。Fresco将未解码的原始数据也进行了内存缓存，然后根据是否旋转或者缩放以及解码质量进行解码成bitmap存放内存空间，其实在我所接触的应用场景中这部分内容其实是不太需要的，因为一张图片基本上只在一个地方使用，即使多处使用也不太需要这么复杂的变换，可能Fresco想的比较周到吧。
内存缓存使用的是通用的lru算法（最近最少使用原则），内存缓存的设计代码在CountingMemoryCache，CountingMemoryCache是一个基于LRU策略来管理缓存中元素的一个类，它实现的trim()方法可以根据Type的不同来采取不同策略的回收为：
```
/**
 * Layer of memory cache stack responsible for managing eviction of the the cached items.
 *
 * <p> This layer is responsible for LRU eviction strategy and for maintaining the size boundaries
 * of the cached items.
 *
 * <p> Only the exclusively owned elements, i.e. the elements not referenced by any client, can be
 * evicted.
 *
 * @param <K> the key type
 * @param <V> the value type
 */
@ThreadSafe
public class CountingMemoryCache<K, V> implements MemoryCache<K, V>, MemoryTrimmable {
   ...//省略代码
    /** Trims the cache according to the specified trimming strategy and the given trim type. */
  @Override
  public void trim(MemoryTrimType trimType) {
    ArrayList<Entry<K, V>> oldEntries;
    final double trimRatio = mCacheTrimStrategy.getTrimRatio(trimType);
    synchronized (this) {
      int targetCacheSize = (int) (mCachedEntries.getSizeInBytes() * (1 - trimRatio));
      int targetEvictionQueueSize = Math.max(0, targetCacheSize - getInUseSizeInBytes());
      oldEntries = trimExclusivelyOwnedEntries(Integer.MAX_VALUE, targetEvictionQueueSize);
      makeOrphans(oldEntries);
    }
    maybeClose(oldEntries);
    maybeNotifyExclusiveEntryRemoval(oldEntries);
    maybeUpdateCacheParams();
    maybeEvictEntries();
  }
  ...//省略代码
}
```
Fresco使用的黑科技还有很多，它是一份巨大的宝藏等着挖掘，我只是粗浅的总结了部分我get到的点，以后进一步深入学习中再和大家分享。