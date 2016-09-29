---
title: Fresco源码浅析-Drawee模块（二）
date: 2016-09-29 15:01:07
tags:
---
Drawee模块负责图片的展示，主要涉及到的概念包括DraweeView（V）、DraweeHierarchy(M)、DraweeController(C),这是一个典型的MVC的结构。

# DraweeView（V）
使用Fresco的时候，我们首先直接使用SimpleDraweeView这个官方自定义的控件，它的继承结构是这样的：
![SimpleDraweeView类继承关系.png](/img/SimpleDraweeView类继承关系.png)

SimpleDraweeView主要实现了setImageURI方法，设置了一个controller。
```
  public void setImageURI(Uri uri, @Nullable Object callerContext) {
    DraweeController controller = mSimpleDraweeControllerBuilder
        .setCallerContext(callerContext)
        .setUri(uri)
        .setOldController(getController())
        .build();
    setController(controller);
  }
```
GenericDraweeView继承自DraweeView，实现了一个GenericDraweeHierarchy的泛型。Fresco支持高度定制，你可以通过重写DraweeHierarchy定制自己的数据处理。
DraweeView控制图片展示的核心业务逻辑，控件初始化的时候初始化了一个DraweeHolder帮助类，并将得到的Hierarchy以及Controller交给DraweeHolder处理。
# DraweeHierarchy(M)
Fresco实现了各种数据处理场景下界面的展示，包括：
- 占位图
- 自身图片
- 进度
- 错误图
- 重试
- ...
DraweeHierarchy作为model主要组织这些数据。GenericDraweeHierarchy实现了一个默认的DraweeHierarchy，因为展示的图片数据都是Drawable，针对这些场景，DraweeHierarchy实现了一个FadeDrawable类，继承自ArrayDrawable，从名字可以看出这是一个层级的Drawable，然后通过Fade某一层Alpha值，控制显示。成员变量如下：
```
private final RootDrawable mTopLevelDrawable; //Imageview最终绘制需要的Drawable。
private final FadeDrawable mFadeDrawable; //所有数据整合后存放的ArrayDrawable。
private final ForwardingDrawable mActualImageWrapper;  //加载成功展示的图片
private final int mPlaceholderImageIndex;//占位图index
private final int mProgressBarImageIndex;//进度条index
private final int mActualImageIndex;//图片index
private final int mRetryImageIndex;//重新加载index
private final int mFailureImageIndex;//失败图index
```

在DraweeHierarchy构造的时候填充mFadeDrawable：
```
    mActualImageWrapper = new ForwardingDrawable(mEmptyActualImageDrawable);

    int numBackgrounds = (builder.getBackgrounds() != null) ? builder.getBackgrounds().size() : 0;
    int numOverlays = (builder.getOverlays() != null) ? builder.getOverlays().size() : 0;
    numOverlays += (builder.getPressedStateOverlay() != null) ? 1 : 0;

    // layer indices and count
    int numLayers = 0;
    int backgroundsIndex = numLayers;
    numLayers += numBackgrounds;
    mPlaceholderImageIndex = numLayers++;
    mActualImageIndex = numLayers++;
    mProgressBarImageIndex = numLayers++;
    mRetryImageIndex = numLayers++;
    mFailureImageIndex = numLayers++;
    int overlaysIndex = numLayers;
    numLayers += numOverlays;

    // array of layers
    Drawable[] layers = new Drawable[numLayers];
    if (numBackgrounds > 0) {
      int index = 0;
      for (Drawable background : builder.getBackgrounds()) {
        layers[backgroundsIndex + index++] = buildBranch(background, null);
      }
    }
    layers[mPlaceholderImageIndex] = buildBranch(
        builder.getPlaceholderImage(),
        builder.getPlaceholderImageScaleType());
    layers[mActualImageIndex] = buildActualImageBranch(
        mActualImageWrapper,
        builder.getActualImageScaleType(),
        builder.getActualImageFocusPoint(),
        builder.getActualImageMatrix(),
        builder.getActualImageColorFilter());
    layers[mProgressBarImageIndex] = buildBranch(
        builder.getProgressBarImage(),
        builder.getProgressBarImageScaleType());
    layers[mRetryImageIndex] = buildBranch(
        builder.getRetryImage(),
        builder.getRetryImageScaleType());
    layers[mFailureImageIndex] = buildBranch(
        builder.getFailureImage(),
        builder.getFailureImageScaleType());
    if (numOverlays > 0) {
      int index = 0;
      if (builder.getOverlays() != null) {
        for (Drawable overlay : builder.getOverlays()) {
          layers[overlaysIndex + index++] = buildBranch(overlay, null);
        }
      }
      if (builder.getPressedStateOverlay() != null) {
        layers[overlaysIndex + index] = buildBranch(builder.getPressedStateOverlay(), null);
      }
    }
```
DraweeHierarchy 对于属性的设置利用了构造者模式，定义了一个GenericDraweeHierarchyBuilder。GenericDraweeHierarchyBuilder可以直接在代码里初始化，也可以读取XML的配置信息。
# DraweeController(C)
DraweeController的功能包括：
- ###### 发起数据请求，得到请求结果，刷新界面；
- ###### 针对界面行为事件控制数据流的处理逻辑。
controller 的类结构：
DraweeController
--| AbstractDraweeController
----| PipelineDraweeController

在纷繁的代码中追踪数据加载的逻辑确实好头疼，我发现单纯的从drawee来查看数据的加载过程还是太复杂，我将在imagepipeline模块整理完之后再回头整理drawee与image pipeline的交互过程，今天我们只是简单的看一下流程。
我们只能从最开始的view初始化看起，DraweeView的onAttachedToWindow函数中调用了attachController方法：
```
  private void attachController() {
    if (mIsControllerAttached) {
      return;
    }
    mEventTracker.recordEvent(Event.ON_ATTACH_CONTROLLER);
    mIsControllerAttached = true;
    if (mController != null &&
        mController.getHierarchy() != null) {
      mController.onAttach();
    }
  }
```
这里就是view调用controller数据加载的入口。controller的onAttach方法又做了什么呢：
```
  @Override
  public void onAttach() {
    mEventTracker.recordEvent(Event.ON_ATTACH_CONTROLLER);
    Preconditions.checkNotNull(mSettableDraweeHierarchy);
    mDeferredReleaser.cancelDeferredRelease(this);
    mIsAttached = true;
    if (!mIsRequestSubmitted) {
      submitRequest(); //发起请求
    }
  }
```
看字面含义我们已经猜到，他在最后发起了数据的请求，我们再看看这个请求是怎么做的：
```
  protected void submitRequest() {
    ...........//省略代码
    mDataSource = getDataSource();
   ...........//省略代码
    final DataSubscriber<T> dataSubscriber =
        new BaseDataSubscriber<T>() {
          @Override
          public void onNewResultImpl(DataSource<T> dataSource) {
            ..............//获取图片成功
          }
          @Override
          public void onFailureImpl(DataSource<T> dataSource) {
            ...............//获取图片失败
          }
          @Override
          public void onProgressUpdate(DataSource<T> dataSource) {
            ..............//图片获取过程中
          }
        };
    mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
  }
```
我们看到这里利用了观察者的设计模式，得到了一个数据流，并向这个数据流注册了一个观察者。getDataSource是个虚方法，真正的实现在PipelineDraweeController和VolleyDraweeController中，默认调用Fresco的initialize方法，创建的是PipelineDraweeController，他的getDataSource实现如下：
```
  @Override
  protected DataSource<CloseableReference<CloseableImage>> getDataSource() {
    return mDataSourceSupplier.get();
  }
```
Supplier的用法我还是第一次看到，他有一个官方的名字叫做“惰性求值”，我们传递Supplier对象，直到调用get方法时，运算才会执行。我们查看get方法的重载在AbstractDraweeControllerBuilder看到了这样一块代码：
```
  protected Supplier<DataSource<IMAGE>> getDataSourceSupplierForRequest(
      final REQUEST imageRequest,
      final boolean bitmapCacheOnly) {
    final Object callerContext = getCallerContext();
    return new Supplier<DataSource<IMAGE>>() {
      @Override
      public DataSource<IMAGE> get() {
        return getDataSourceForRequest(imageRequest, callerContext, bitmapCacheOnly);
      }
      @Override
      public String toString() {
        return Objects.toStringHelper(this)
            .add("request", imageRequest.toString())
            .toString();
      }
    };
  }
```
没错，就是getDataSourceForRequest实现了数据的加载，我们在PipelineDraweeControllerBuilder中看到了具体实现：
```
  @Override
  protected DataSource<CloseableReference<CloseableImage>> getDataSourceForRequest(
      ImageRequest imageRequest,
      Object callerContext,
      boolean bitmapCacheOnly) {
    if (bitmapCacheOnly) {
      return mImagePipeline.fetchImageFromBitmapCache(imageRequest, callerContext);
    } else {
      return mImagePipeline.fetchDecodedImage(imageRequest, callerContext);
    }
  }
```
至此我们终于调用了ImagePipeline的fetch方法，实现了数据的请求，我们回过头来总结一下这个流程：
ImageView在setImageUri的过程过程中创建了一个controller，controller的创建基于builder模式，最后调用build函数，接下来的函数链是：buildController  -> obtainController -> obtainDataSourceSupplier -> getDataSourceSupplierForRequest完成了整个controller的创建，接下来在View的onAttachedToWindow调用controller中依次调用 onAttach -> submitRequest -> getDataSource 然后执行get中的请求操作。

Controller中同时定义了一个GestureDetector，用于对View事件的拦截，我看到主要还是在onClick事件中处理是否重新加载。

至此整个图片展示的部分我们已经大致的梳理了一遍，核心还是在于model的处理，ImageView设置了一个drawable之后，control直接控制这个drawable，利用drawable的invalidateSelf函数直接刷新View，MVC模式中model是可以操作View的，这也真是MVC与MVP模式的最大区别。controller的逻辑比较简单，就是先初始化构建，然后View显示的时候执行他的数据请求操作，但是代码比较复杂，梳理起来比较麻烦，还好在梳理的过程中看到了很多优秀的设计。

# [下一页 ImagePipeline介绍](http://www.jianshu.com/p/116639f920b6)