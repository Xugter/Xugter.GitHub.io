---
layout: post
title: CoordinatorLayout系列(二):Behavior的使用
catalog:  true
tags:
    - CoordinatorLayout
    - Android
---

# 准备

上一篇文章介绍了CoordinatorLayout的基本使用方法，网上有很多demo，是基于这些基本使用的官方控件。但是基于这些的方法能实现的效果很有限。要想非常自由地实现各种炫酷效果，还是需要学习Behaivor的使用。

先立个小目标，脱离官方提供的Behavior，自己去实现这些交互。

要实现这个目标，我准备分两步走。
1.需要学习Behavior。Behavior主要功能有三块.
1跟布局有关的，来控制view初始显示的位置大小
onLayoutChild和onMeasureChild

2跟另外的子view的位置互动
layoutDependsOn和onDependentViewChanged
3跟嵌套滑动有关
onStartNestedScroll
onNestedPreScroll
onNestedScroll
onStopNestedScroll
onNestedPreFling
onNestedFling

接下来看看都是怎么使用的

1onMeasureChild和onLayoutChild   

public boolean onMeasureChild(@NonNull CoordinatorLayout parent, @NonNull V child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed) {
    return false;
}
这个方法用来算出child的宽高(getMeasuredWidth和getMeasuredHeight)
int newHeightMeasureSpec = View.MeasureSpec.makeMeasureSpec(300, View.MeasureSpec.EXACTLY);
int newWidthMeasureSpec = View.MeasureSpec.makeMeasureSpec(500, View.MeasureSpec.EXACTLY);
如上，我们重新设定parent的宽高是500*300,然后用新的参数测量child
parent.onMeasureChild(child, newWidthMeasureSpec, widthUsed, newHeightMeasureSpec, heightUsed);
最后看日志输出
com.xugter.cooridnatorlayoutstudy I/MeasureBehavior: onMeasureChild==========w=500   h=300

PIC

就像上面的蓝色的方块，即使我在xml文件设定的宽高是match_parent
<View
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/holo_blue_light"
    app:layout_behavior=".part2.layout.MeasureBehavior" />

public boolean onLayoutChild(@NonNull CoordinatorLayout parent, @NonNull V child, int layoutDirection) {
    return false;
}
这个方法是用来放置child的位置的，
@Override
public boolean onLayoutChild(@NonNull CoordinatorLayout parent, @NonNull View child, int layoutDirection) {
    child.layout(0, 500, child.getMeasuredWidth(), 500 + child.getMeasuredHeight());
    return true;
}
只要这样设置了后，就会像上图的红丝方块，往下移动500

另外简单介绍onInterceptTouchEvent和onTouchEvent这两个功能和view的onInterceptTouchEvent和onTouchEvent是一样的
在这两个方法里面进行一些事件的分发，返回true就可以代替view自己的这两个方法

2layoutDependsOn和onDependentViewChanged
Behavior可以让一个child根据另外的一个child的位移，来改变自己的状态，无论是位置还是透明度等一些属性
public boolean layoutDependsOn(@NonNull CoordinatorLayout parent, @NonNull V child, @NonNull View dependency) {
    return false;
}
这个方法主要是决定当前child要根据哪个child(即dependency)来改变自己，可以用dependency的id,tag,类型等一些办法来判定是否是dependency
这里直接用id，也可以用类似ScrollingViewBehavior来判定
public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
    return dependency instanceof AppBarLayout;
}

public boolean onDependentViewChanged(@NonNull CoordinatorLayout parent, @NonNull V child, @NonNull View dependency) {
    return false;
}
当上面确定好dependency，就可以用这个方法根据dependency位置变化来改变自己的状态了
@Override
public boolean onDependentViewChanged(@NonNull CoordinatorLayout parent, @NonNull View child, @NonNull View dependency) {
    int dependBottom = dependency.getBottom();
    child.setY(dependBottom + 50);
    child.setX(dependency.getLeft());
    return true;
}
这里只是简单让红色方块跟住绿色方块
如下图
GIF
https://www.aconvert.com/cn/video/mp4-to-gif/

3onXXXXXXXScroll
关于这部分的原理解释起来会涉及到嵌套滑动，在以后会详细讨论
这里先简单介绍一下嵌套滑动的概念，然后直接讨论怎么使用。
嵌套滑动简单来说就是关于两个可滑动控件打架的故事，即一个可滑动的控件包含另一个可滑动的控件，当用户滑动里面的控件，怎么处理触摸事件。
CoordinatorLayout实现了NestedScrollingParent2就能接收实现了NestedScrollingChild2接口的child的滑动事件，至于这是为啥。。。那就等到以后讨论到嵌套滑动再说吧。现在要假装强行可以接收到就可以了

CoordinatorLayout接收到滑动事件后就会把事件发送给每个愿意接收这个事件的child
至于怎么确定是否愿意是在这个方法
onStartNestedScroll
只要返回true就表示这个child愿意接收这个滑动事件

就会在接下来的这两个方法接收到事件，处理自己想要处理的事情,那么为啥有两个方法来接收这个事件呢，这个又跟嵌套滑动的机制有关，再次强行绕过嵌套滑动这个坑。如果有兴趣了解的话，可以去看这两篇文章，个人感觉这两篇文章写的非常好，基本把事件的分发和嵌套滑动讲的很透彻。
这里先简单介绍一下这两个方法的流程，当一个NestedScrollingChild2的childA发生滑动的时候，会先询问有没有childB愿意接收这个滑动事件的onStartNestedScroll
有的话(返回true)就把这个事件发给onNestedPreScroll，在onNestedPreScroll里面，childB有可能会消费这个滑动事件，也有可能一点都不消费，或者消费部分，这个都是通过consumed来传回的。当onNestedPreScroll结束后，childA就会处理自己的滑动，当自己还没有消费完滑动事件(比如滑到底部了)，会再次把事件通过onNestedScroll发给childB，如果childB还没消费完这个事件，会在返回到childA，自生自灭

流程PIC

由于onNestedScroll接收的不是第一个接收到的，所以一般情况下onNestedPreScroll这个用的比较多一点。

demo里面，有个nestscrollview和一个方块
运行规则是这个方块会根据nestscrollview的反方向移动，当触顶或者触底的时候，才会轮到nestscrollview自己滚动。
consumed控制着方块的消费情况
if (child.getY() + dy < 0) {
    //方块到达顶部，滑动距离消费不完
    ViewCompat.offsetTopAndBottom(child, (int) (0 - child.getY()));
    consumed[1] = 0 - (int) child.getY();
} else if (child.getY() + dy > coordinatorLayout.getHeight() - child.getHeight()) {
    //方块到达底部，滑动距离消费不完
    ViewCompat.offsetTopAndBottom(child, (int) (coordinatorLayout.getHeight() - child.getHeight() - child.getY()));
    consumed[1] = coordinatorLayout.getHeight() - child.getHeight() - (int) child.getY();
} else {
    //方块消费完全部事件
    ViewCompat.offsetTopAndBottom(child, dy);
    consumed[1] = dy;
}

从日志情况也可以看出onNestedPreScroll和onNestedScroll的运行关系。
当nestscrollview可以滑动的时候，是不会触发onNestedScroll的，日志如下

2019-09-07 16:15:15.169 15774-15774/com.xugter.cooridnatorlayoutstudy D/====ScrollBehavior====: onStartNestedScroll
2019-09-07 16:15:15.170 15774-15774/com.xugter.cooridnatorlayoutstudy D/====ScrollBehavior====: onNestedScrollAccepted
2019-09-07 16:15:15.313 15774-15774/com.xugter.cooridnatorlayoutstudy D/====ScrollBehavior====: onNestedPreScroll
..............
2019-09-07 16:15:16.094 15774-15774/com.xugter.cooridnatorlayoutstudy D/====ScrollBehavior====: onNestedPreScroll
2019-09-07 16:15:16.387 15774-15774/com.xugter.cooridnatorlayoutstudy D/====ScrollBehavior====: onStopNestedScroll

当nestscrollview不可滑动了，才会触发onNestedScroll，日志如下
2019-09-07 16:17:50.257 15774-15774/com.xugter.cooridnatorlayoutstudy D/====ScrollBehavior====: onStartNestedScroll
2019-09-07 16:17:50.257 15774-15774/com.xugter.cooridnatorlayoutstudy D/====ScrollBehavior====: onNestedScrollAccepted
2019-09-07 16:17:50.475 15774-15774/com.xugter.cooridnatorlayoutstudy D/====ScrollBehavior====: onNestedPreScroll
2019-09-07 16:17:50.494 15774-15774/com.xugter.cooridnatorlayoutstudy D/====ScrollBehavior====: onNestedPreScroll
.............................
2019-09-07 16:17:53.919 15774-15774/com.xugter.cooridnatorlayoutstudy D/====ScrollBehavior====: onNestedScroll
2019-09-07 16:17:54.017 15774-15774/com.xugter.cooridnatorlayoutstudy D/====ScrollBehavior====: onNestedPreScroll
2019-09-07 16:17:54.017 15774-15774/com.xugter.cooridnatorlayoutstudy D/====ScrollBehavior====: onNestedScroll
2019-09-07 16:17:54.018 15774-15774/com.xugter.cooridnatorlayoutstudy D/====ScrollBehavior====: onStopNestedScroll

另外 不知道有没有注意到，demo里面把onNestedScroll里面的每个view都设置了onclick事件，如果不设置onclick事件，这个demo就有问题了。
至于为什么，你猜对了，又需要理解嵌套机制。这个小坑，我也懵逼了好久，以后再拎出来讨论吧。