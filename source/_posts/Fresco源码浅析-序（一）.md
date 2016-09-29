
---
title: Fresco源码浅析-序（一）
date: 2016-09-29 15:00:07
tags:
---
大名鼎鼎的Fresco开源一年多了，已经有越来越多的尝鲜者开始使用。目前我们的项目对于图片的加载没有过多的需求，但对于Fresco还是充满了好。我接下来几篇博文讲围绕着他的架构设计、图片处理、缓存设计、线程管理等议题深入源码，尽可能的给出自己的一些理解。（水平有限，第一次尝试写博客，有不对的地方欢迎指正）
# Fresco优势
- ###### 内存管理
bitmap是Android中内存占用的大块头，Fresco用了一个略显“流氓”的方案，将bitmap存储在共享内存区域，这样就不会计算在app分配的堆内存大小了，有效的减少了OOM，但这毕竟是钻了Android的一个空子，5.0以上的版本他及时更改了过来。不过这也可以作为一种思路，学习一下如何高效利用共享内存，以备我们随时“流氓”一下^_^
- ###### 支持更多的图片格式
Fresco支持本地图片和网络图片，支持GIF和WebP格式的图片，
- ###### 渐进式图片加载
渐进式图片能带来更好的用户体验，Fresco支持这个格式，而且使用的时候只需要像普通图片一样提供Url即可，不过这还需要后端哥哥们的配合。。。
- ###### 封装了各种场景下的图片展示
圆角，自定义居中，占位图，错误图，loading图，再次加载图，你能想到的场景Fresco都帮你想到了，使用起来及其方便。
# 源码分析要点
github上down下来的Fresco代码结构是这样的：
![Fresco官方代码结构.png](/img/Fresco官方代码结构.png)
看到这么多module是不是有点爆炸的感觉，反正我第一眼是炸了。。
然后我在另外一个工程引入了一下Fresco包，给我下载的代码结构是这样的：
![Fresco引入包代码结构.png](/img/Fresco引入包代码结构.png)
对比看起来就比较清晰了，主要是Facebook的core包，drawee和imagepipeline。core包是Facebook的基础包，这里面有很多工具类可以借用，drawee主要是用来图片的展示，imagepipeline则设计了一个图片加载框架，接下来讲主要围绕这两块深入到Fresco的源码中去（一直在尝试深入，其实只是在门口游荡，惭愧）。

- [ drawee](http://www.jianshu.com/p/edc36431fede)
- [imagepipeline](http://www.jianshu.com/p/116639f920b6)

# 简单使用
Fresco使用起来特别简单，如果你是已经在代码中使用了大量网络图片，那改动可能会有点大，因为view需要改用Fresco默认提供的一个叫SimpleDraweeView的ImageView，这要涉及到xml和Java代码的改动。不过使用起来确实超级方便，直接调用setimageuri方法就完事了，更多的关于初始化设置占位图、错误图、loading图、重复加载图详见官方说明，这里不再赘述了（有时间我准备改造一下Fresco的使用方式，还是用我们熟悉的传入ImageView和Uri的方式，这样集成和替换图片加载库的代价就会很低，如果觉得有价值请顶我，让我更有动力）。

# 文献引用
> [Fresco 中文文档](http://www.fresco-cn.org)
> [Fresco 源码解析](http://www.cnblogs.com/pandapan/p/4634454.html)

# [下一页 Drawee模块介绍](http://www.jianshu.com/p/edc36431fede)