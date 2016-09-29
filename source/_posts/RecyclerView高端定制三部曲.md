---
title: RecyclerView高端定制三部曲
date: 2016-09-29 15:22:18
tags:
---
RecyclerView随V7拓展包发布以来，因其高效和使用便利，基本取代了listview和gridview，成为了使用频率最高的控件之一。默认的设置基本能满足大部分场景，如果需要更好的体验，需要自定义以下三个部分的内容：
- ###### Animator
- ###### ItemDecoration
- ###### LayoutManager


# 自定义Animator
自定义Animator可以实现各种绚丽的动画效果，RecyclerView动画相关的类主要有三个：
- ###### RecyclerView.ItemAnimator
- ###### SimpleItemAnimator
- ###### DefaultItemAnimator

RecyclerView.ItemAnimator是自定义RecyclerView动画效果的核心类，当继承一个ItemAnimator时，有如下几个方法需要被实现：
```
//当一个ViewHolder从RecyclerView里面消失时调用,不一定是移除，也有可能是move操作
@Override  
    public boolean animateDisappearance(RecyclerView.ViewHolder viewHolder, ItemHolderInfo preLayoutInfo, ItemHolderInfo postLayoutInfo) {  
        return false;  
    }  
  //当一个ViewHolder从RecyclerView里面显示时调用,不一定是新增，也有可能是move操作
    @Override  
    public boolean animateAppearance(RecyclerView.ViewHolder viewHolder, ItemHolderInfo preLayoutInfo, ItemHolderInfo postLayoutInfo) {  
        return false;  
    }  
  //没有调用notify而引起布局的改变，比如滑动
    @Override  
    public boolean animatePersistence(RecyclerView.ViewHolder viewHolder, ItemHolderInfo preLayoutInfo, ItemHolderInfo postLayoutInfo) {  
        return false;  
    }  
  //item发生改变的时候调用
    @Override  
    public boolean animateChange(RecyclerView.ViewHolder oldHolder, RecyclerView.ViewHolder newHolder, ItemHolderInfo preLayoutInfo, ItemHolderInfo postLayoutInfo) {  
        return false;  
    }  
  //统筹RecyclerView中所有的动画，统一启动执行,一般的思路是在前面几个函数调用中放入一个动画列表，在这个函数中统一执行
    @Override  
    public void runPendingAnimations() {  
  
    }  
  //结束某一个item的动画
    @Override  
    public void endAnimation(RecyclerView.ViewHolder item) {  
    }  
  //结束所有的动画
    @Override  
    public void endAnimations() { 
    }  
  //动画是否执行过程中
    @Override  
    public boolean isRunning() {  
        return false;  
}  
```

SimpleItemAnimator对RecyclerView.ItemAnimator实现了简单的封装，将基本的ItemAnimator不同场景拆分成我们熟悉的四种场景：add、remove、move、change。所以我们如果实现自定义的动画，继承自SimpleItemAnimator会更容易实现，这也是较为普遍的做法。

DefaultItemAnimator是RecyclerView默认的动画效果，只有一个fadein和fadeout的渐变动画，开始看代码的时候一直有一个疑问，默认的动画效果明明是一个先展开然后插入的动画啊，哪里是渐变的效果。对比了各种动画效果之后发现，这里实现的动画效果只是针对item出现或者消失时的动画，位置展开是RecylerView固定的效果，具体代码没有去源码中跟踪，调用notifyItemInserted函数刷新界面时，先执行对应位置的展开再执行item的动画效果，我们自定义动画就是实现这个item出现的方式，位置展开和收缩是固定的。V7拓展包23.0版本DefaultItemAnimator继承自RecyclerView.ItemAnimator，23.1版本就直接继承自SimpleItemAnimator。

我们参照DefaultItemAnimator的方式实现我们自定义的动画效果。其中最主要的删除和新增实现如下：
```
    private void animateRemoveImpl(final RecyclerView.ViewHolder holder) {
        final View view = holder.itemView;
        final ViewPropertyAnimatorCompat animation = ViewCompat.animate(view);
        mRemoveAnimations.add(holder);
        animation.setDuration(getRemoveDuration())
                .alpha(0).setListener(new VpaListenerAdapter() {
            @Override
            public void onAnimationStart(View view) {
                dispatchRemoveStarting(holder);
            }

            @Override
            public void onAnimationEnd(View view) {
                animation.setListener(null);
                ViewCompat.setAlpha(view, 1);
                dispatchRemoveFinished(holder);
                mRemoveAnimations.remove(holder);
                dispatchFinishedWhenDone();
            }
        }).start();
    }

    private void animateAddImpl(final RecyclerView.ViewHolder holder) {
        final View view = holder.itemView;
        final ViewPropertyAnimatorCompat animation = ViewCompat.animate(view);
        mAddAnimations.add(holder);
        animation.alpha(1).setDuration(getAddDuration()).
                setListener(new VpaListenerAdapter() {
                    @Override
                    public void onAnimationStart(View view) {
                        dispatchAddStarting(holder);
                    }
                    @Override
                    public void onAnimationCancel(View view) {
                        ViewCompat.setAlpha(view, 1);
                    }

                    @Override
                    public void onAnimationEnd(View view) {
                        animation.setListener(null);
                        dispatchAddFinished(holder);
                        mAddAnimations.remove(holder);
                        dispatchFinishedWhenDone();
                    }
                }).start();
    }
```
其中写死了animation的效果，而又无法重载，我们将DefaultItemAnimator的代码完全拷出，实现一个可配置动画的BaseItemAnimator类。主要替换的代码如下：
```
    private void animateRemoveImpl(final RecyclerView.ViewHolder holder) {
        mRemoveAnimations.add(holder);
        final ViewPropertyAnimatorCompat animation = getRemoveAnimator(holder);
        animation.setDuration(getRemoveDuration()).setListener(new VpaListenerAdapter() {
                    @Override
                    public void onAnimationStart(View view) {
                        dispatchRemoveStarting(holder);
                    }
                    @Override
                    public void onAnimationEnd(View view) {
                        animation.setListener(null);
                        clear(view);
                        dispatchRemoveFinished(holder);
                        mRemoveAnimations.remove(holder);
                        dispatchFinishedWhenDone();
                    }
                });
        animation.start();
    }

    private void animateAddImpl(final RecyclerView.ViewHolder holder) {
        mAddAnimations.add(holder);
        final ViewPropertyAnimatorCompat animation = getAddAnimator(holder);
        animation.setDuration(getAddDuration()).
                setListener(new VpaListenerAdapter() {
                    @Override
                    public void onAnimationStart(View view) {
                        ViewCompat.setAlpha(view,1);
                        dispatchAddStarting(holder);
                    }
                    @Override
                    public void onAnimationCancel(View view) {
                        clear(view);
                    }

                    @Override
                    public void onAnimationEnd(View view) {
                        animation.setListener(null);
                        dispatchAddFinished(holder);
                        mAddAnimations.remove(holder);
                        dispatchFinishedWhenDone();
                    }
                }).start();
    }

    protected abstract ViewPropertyAnimatorCompat getAddAnimator(RecyclerView.ViewHolder viewHolder);
    protected abstract ViewPropertyAnimatorCompat getRemoveAnimator(RecyclerView.ViewHolder viewHolder);

    public void clear(View v) {
        ViewCompat.setAlpha(v, 1);
        ViewCompat.setScaleY(v, 1);
        ViewCompat.setScaleX(v, 1);
        ViewCompat.setTranslationY(v, 0);
        ViewCompat.setTranslationX(v, 0);
        ViewCompat.setRotation(v, 0);
        ViewCompat.setRotationY(v, 0);
        ViewCompat.setRotationX(v, 0);
        ViewCompat.setPivotY(v, v.getMeasuredHeight() / 2);
        ViewCompat.setPivotX(v, v.getMeasuredWidth() / 2);
        ViewCompat.animate(v).setInterpolator(null).setStartDelay(0);
    }
```
接下来我们就可以继承BaseItemAnimator类实现自己的动画效果，下面给出一个示例：
```
    @Override
    protected ViewPropertyAnimatorCompat getAddAnimator(RecyclerView.ViewHolder item) {
        ViewCompat.setTranslationX(item.itemView, -item.itemView.getWidth());
        return ViewCompat.animate(item.itemView).translationX(0);
    }
    @Override
    protected ViewPropertyAnimatorCompat getRemoveAnimator(RecyclerView.ViewHolder item) {
        return ViewCompat.animate(item.itemView).translationX(item.itemView.getWidth());
    }
```
详细代码见[github](https://github.com/pandavickey/CustomFeature-RecyclerView)
# 自定义ItemDecoration
自定义ItemDecoration需要继承RecyclerView.ItemDecoration抽象类，源码很简单：
```
    public static abstract class ItemDecoration {
        public void onDraw(Canvas c, RecyclerView parent, State state) {
            onDraw(c, parent);
        }
        @Deprecated
        public void onDraw(Canvas c, RecyclerView parent) {
        }
        public void onDrawOver(Canvas c, RecyclerView parent, State state) {
            onDrawOver(c, parent);
        }
        @Deprecated
        public void onDrawOver(Canvas c, RecyclerView parent) {
        }
        @Deprecated
        public void getItemOffsets(Rect outRect, int itemPosition, RecyclerView parent) {
            outRect.set(0, 0, 0, 0);
        }
        public void getItemOffsets(Rect outRect, View view, RecyclerView parent, State state) {
            getItemOffsets(outRect, ((LayoutParams) view.getLayoutParams()).getViewLayoutPosition(),
                    parent);
        }
    }
```
官方推荐使用含有state参数的方法，所以主要就是重载三个方法：
-  onDraw(Canvas c, RecyclerView parent, State state)
-  onDrawOver(Canvas c, RecyclerView parent, State state)
-  getItemOffsets(Rect outRect, View view, RecyclerView parent, State state)

onDraw用于绘制divider，它是绘制在item下面的，所以中间部分会被item遮挡住；
onDrawOver绘制在item上面，所以不受位置的限制；
getItemOffsets实际上就是给每个item一个padding，实现item之间的间隙。

实际业务中我们经常会遇到这样的需求，一个可选择的gridview，选中与否有不同的边框，效果图如下：

![设计图](/img/设计图.png)

以我有限的界面开发经验，不管如何控制，要实现每个分割线都是相同的宽度，选中界面的边框正好压盖周围的边框还是很有难度的，看到recyclerview的自定义ItemDecoration才终于发现一道曙光。

我的做法是在getItemOffsets函数中，我判断一个item上下左右是否有临近的item，没有的话给两倍的dividerwidth，否则一倍的dividerwidth，这样就能控制所有的item四周都是有同样宽度的间隙。然后在onDrawOver中实现每个item分割线的绘制，同时判断是否选中，再绘制选中的边框，整体效果如下：

![效果图](/img/效果图.png)

还是完美的实现了UI妹子要求的效果(^_^)。

详细代码见[github](https://github.com/pandavickey/CustomFeature-RecyclerView)

# 自定义LayoutManager
LayoutManager可以说是整个RecyclerView的精髓，整个RecyclerView的Recycler也是在LayoutManager做的，官方目前提供了LinearLayoutManager、GridLayoutManager和StaggeredGridLayoutManager三种LayoutManager，分别使用在线性、方格以及不规则瀑布流的场景，基本上实现了日常的大部分需求。目前精力有限，后续再做分解。