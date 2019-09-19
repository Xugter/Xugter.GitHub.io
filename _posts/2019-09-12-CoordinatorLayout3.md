---
layout: post
title: CoordinatorLayout系列(三):Behavior的实践
catalog:  true
tags:
    - CoordinatorLayout
    - Android
---

在知道Behavior的使用方法了之后，再用一个模仿虾米音乐的首页来实践一下Behavior的使用。

![1.gif](https://ae01.alicdn.com/kf/H36b00bc66e45416bb0acf17723ef96f9f.gif)

为了实现这个效果，我们我可以分为下面四步走

# 第一步  观察一下可以分为几个部分

这个页面从上到下，大致可以分为searchbar header banner list四个部分。

还有一个部分bottom是一直在底部没有互动的，我们就先暂时不管。
```
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
    
        <ImageView
            android:id="@+id/img_search"
            android:layout_width="match_parent"
            android:layout_height="45dp"
            android:scaleType="fitXY"
            android:src="@drawable/xiami_search"
            app:layout_behavior="com.xugter.cooridnatorlayoutstudy.part3.SearchBarBehavior" />
    
        <ImageView
            android:id="@+id/img_header"
            android:layout_width="match_parent"
            android:layout_height="48dp"
            android:scaleType="fitXY"
            android:src="@drawable/xiami_header"
            app:layout_behavior="com.xugter.cooridnatorlayoutstudy.part3.HeaderBehavior" />
    
        <ImageView
            android:id="@+id/img_banner"
            android:layout_width="match_parent"
            android:layout_height="127dp"
            android:scaleType="fitXY"
            android:src="@drawable/xiami_banner"
            app:layout_behavior="com.xugter.cooridnatorlayoutstudy.part3.BannerBehavior" />
    
        <android.support.v4.widget.NestedScrollView
            android:id="@+id/view_content"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:layout_marginBottom="43dp"
            app:layout_behavior="com.xugter.cooridnatorlayoutstudy.part3.ListBehavior">
    
            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:orientation="vertical">
    
                <ImageView
                    android:layout_width="match_parent"
                    android:layout_height="81dp"
                    android:onClick="useless"
                    android:scaleType="fitXY"
                    android:src="@drawable/xiami_tab" />
    
                ......
    
            </LinearLayout>
        </android.support.v4.widget.NestedScrollView>
    
        <ImageView
            android:layout_width="match_parent"
            android:layout_height="43dp"
            android:layout_gravity="bottom"
            android:scaleType="fitXY"
            android:src="@drawable/xiami_bottom" />
    
</android.support.design.widget.CoordinatorLayout>
```

# 第二步  确定这几个部分的依赖关系

这里我定的是

searchbar→header→banner→list

# 第三步 确定可以愿意接受滑动事件的部分

这里我定的是list部分

# 第四步 开始写代码

## 1.List
这里我们先从依赖的根源开始写相应的behavior，即ListBehavior。

写ListBehavior第一步先确定List的初始布局

从上面的gif图来看，初始位置是在searchbar header banner的下面的。当画到最上面的时候在header的下面。

所以可以在onlayout那里，先确定能显示最大的时候位置，用child.layout来确定他的宽和高。再通过sety把他放到初始位置。
```
    @Override
    public boolean onLayoutChild(@NonNull CoordinatorLayout parent, @NonNull View child, int layoutDirection) {
        //来定位list的初始位置，并进行第一次的位移
        child.layout(0, headerHeight, child.getMeasuredWidth(), parent.getMeasuredHeight() - bottomHeight);
        child.setY(searchBarHeight + headerHeight + bannerHeight);
        return true;
    }
```

这几个height都会在behavior构造的时候初始化。

**然后**就是处理list的滑动事件了。

这里有个好玩的事情发生了，在上一篇介绍behavior的时候，我们说的是能接收滑动事件的childA把事件发给Coordinatorlayout，然后coordinatorlayout发给愿意接收滑动事件的childB，再通过**onNestedPreScroll**和**onNestedScroll**交互。

但是这里情况有点不一样，这里childA是list，childB也是list，childA和childB就变成同一个人了。其实这是不用管的，虽然是同一个对象，但是可以当成两个对象，直接把list当成childB，至于childA是谁你就不用管了。

首先在onStartNestedScroll直接返回true，表示愿意接收滑动事件。

然后再onNestedPreScroll开始处理滑动事件，这里就需要根据list当前的位置和滑动的方向来判断是否需要消费这个滑动事件。这里可以根据代码里面的注释，大致看一下思路。
```
    @Override
    public void onNestedPreScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull View child, @NonNull View target, int dx, int dy, @NonNull int[] consumed, int type) {
        if (startPos) {
            if (scrollMode == 0) {
                if (child.getY() == headerHeight) {
                    //当list处于页面的顶部，需要根据滑动的方向来确定 是滚动模式还是嵌套滑动模式
                    if (dy > 0) {
                        //向上滑动 进入滚动模式
                        scrollMode = 1;
                    } else {
                        //向下滑动  进入嵌套滑动模式
                        scrollMode = -1;
                    }
                } else {
                    //当list不处于页面顶部，进入嵌套滑动模式
                    scrollMode = -1;
                }
            }
            //是滚动模式，不触发嵌套滑动 直接返回
            if (scrollMode > 0) return;
            if (dy > 0) {
                if (child.getY() > headerHeight) {
                    //只有未到达顶部  list才能嵌套滑动  不然滑不动界面
                    float targetPos = child.getY() - dy;
                    if (targetPos > headerHeight) {
                        child.setY(targetPos);
                    } else {
                        child.setY(headerHeight);
                    }
                }
            } else {
                if (child.getY() < searchBarHeight + headerHeight + bannerHeight) {
                    //只有未到达底部  list才能嵌套滑动  不然滑不动界面
                    float targetPos = child.getY() - dy;
                    if (targetPos < searchBarHeight + headerHeight + bannerHeight) {
                        child.setY(targetPos);
                    } else {
                        child.setY(searchBarHeight + headerHeight + bannerHeight);
                    }
                }
            }
            //嵌套滑动 无论什么情况 都要消费掉所有事件
            consumed[1] = dy;
        }
    }
```

其中为了每次滑动动作开始的时候，重置一下状态我在onInterceptTouchEvent加了一些动作，当接受到DOWN这个事件的时候，回重置一些状态。这个部分主要是为了实现，当虾米list滑到顶部的时候，需要断掉滑动。然后需要再次按下滑动，才能继续滑动这个效果。
```
    @Override
    public boolean onInterceptTouchEvent(@NonNull CoordinatorLayout parent, @NonNull View child, @NonNull MotionEvent ev) {
        if (ev.getAction() == 0) {
            if (child.getScrollY() > 0) {
                //List没有滚到顶部，先进行自己滚动，不触发嵌套滑动
                startPos = false;
            } else {
                //List滚动到顶部，待确定触发嵌套滑动
                startPos = true;
            }
            //重置一下 是否滚动模式
            scrollMode = 0;
        }
        return super.onInterceptTouchEvent(parent, child, ev);
    }
```

ListBehaivor的思路大致就是这样了，完成ListBehaivor基本上就完成了70%了。

## 2.Banner
接下来我们在看依赖于list的BannerBehaivor。

这个部分思路就很简单，首先在layoutDependsOn里确定依赖关系。
```
    @Override
    public boolean layoutDependsOn(@NonNull CoordinatorLayout parent, @NonNull View child, @NonNull View dependency) {
        return dependency.getId() == R.id.view_content;
    }
```

然后再onDependentViewChanged里，根据dependency的位置来更新自己的位置
```
    @Override
    public boolean onDependentViewChanged(@NonNull CoordinatorLayout parent, @NonNull View child, @NonNull View dependency) {
        if (dependency.getY() >= headerHeight && dependency.getY() <= headerHeight + bannerHeight) {
            child.setAlpha((dependency.getY() - headerHeight) / bannerHeight);
            child.setY(headerHeight);
        } else if (dependency.getY() > headerHeight + bannerHeight) {
            child.setY(dependency.getY() - bannerHeight);
        }
        return true;
    }
```
## 3.Header
同理HeaderBehavior也先确定依赖关系
```
    @Override
    public boolean layoutDependsOn(@NonNull CoordinatorLayout parent, @NonNull View child, @NonNull View dependency) {
        return dependency.getId() == R.id.img_banner;
    }
```

然后再根据dependency的位置来更新自己的位置
```
    @Override
    public boolean onDependentViewChanged(@NonNull CoordinatorLayout parent, @NonNull View child, @NonNull View dependency) {
        if (dependency.getY() - child.getMeasuredHeight() > 0) {
            child.setY(dependency.getY() - child.getMeasuredHeight());
        } else {
            child.setY(0);
        }
        return true;
    }
```
## 4.SearchBar
再同理来写SearchBarBehavior
```
    @Override
    public boolean layoutDependsOn(@NonNull CoordinatorLayout parent, @NonNull View child, @NonNull View dependency) {
        return dependency.getId() == R.id.img_header;
    }
```
```
    @Override
    public boolean onDependentViewChanged(@NonNull CoordinatorLayout parent, @NonNull View child, @NonNull View dependency) {
        float progress = dependency.getY() / getSearchBarHeight();
        child.setAlpha(progress * progress);
        return true;
    }
```
这样大致完成了虾米首页的效果，还有一些细节没有完善，比如自动开合的效果等，这个就等有闲心再完善一下吧。

效果如下:

![1.gif](https://ae01.alicdn.com/kf/H2c439dcde99841afa657b51ad65ef974I.gif)


其实这个方案也不是唯一的，比如可以把header和banner当成一个child，然后根据位置信息控制banner的透明

再比如 接收嵌套滑动事件的也可以不是list，可以是header或者其他，思路也大同小异，可以自己尝试实现一下。

# 总结
实现类似页面效果的步骤大致分为四步

**1.观察，确定分为几个部分**

**2.确定依赖关系**

**3.确定接收滑动事件的部分**

**4.写代码**，写代码都一般思路是先从根view(也就是最后被依赖的部分)开始，一般根view也是愿意接收滑动事件的view。然后再根据依赖路径，一直写behavior。

[Github](https://github.com/Xugter/CooridnatorLayoutStudy)