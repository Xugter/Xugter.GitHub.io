---
layout: post
title: CoordinatorLayout系列(一):基本使用
tags: [CoordinatorLayout, Android]
---

CoordinatorLayout简单理解就是增强版的FrameLayout,特点是协调子View.

## 准备

这是Google官方的文档(请自备梯子)

[CoordinatorLayout](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout)

>CoordinatorLayout is a super-powered FrameLayout.
CoordinatorLayout is intended for two primary use cases:
1.As a top-level application decor or chrome layout
2.As a container for a specific interaction with one or more child views


总结一下就是，CoordinatorLayout是加强版的FrameLayout,但并不是直接继承，基本的布局方式跟FrameLayout是一样的。

加强的点是CoordinatorLayout可以给它的child之间提供各种交互特性，简单来说就是一个child可以根据另一个child的状态来相应的更新自己的状态，大部分情况都是位置的更新。所以应该叫协调布局？？？

先看几个Demo，自己感受一下，看一下这个疗效怎么样
![1.gif](https://ae01.alicdn.com/kf/HTB1bo48XAH0gK0jSZPi5javapXaI.gif)
![3.gif](https://ae01.alicdn.com/kf/HTB11O87XuL2gK0jSZFm5jc7iXXaq.gif)

这里所有的代码都已经放在[Github](https://github.com/Xugter/CooridnatorLayoutStudy)

## 开始

CoordinatorLayout主要提供了三种方式来实现child之间的互动:
1. 通过anchor实现
2. 通过insetEdge实现
3. 通过Behaviors实现

那么我们依次来看一下这三种方式是怎么使用的
提示:TouchView是一个可以自由拖动的View

### 1. anchor

关键词：layout_anchor和layout_anchorGravity

简单说明：child B通过layout_anchor 设置child A为anchor，再通过layout_anchorGravity来根据需要设置属性，这样B就可以A的位移相应的位移了。

使用步骤:

1 . 先设置一个被观察的child A的id

```
<com.xugter.cooridnatorlayoutstudy.other.TouchView
    android:id="@+id/view_host"
    android:layout_width="80dp"
    android:layout_height="80dp"
    android:layout_gravity="center"
    android:background="@color/colorPrimary"
    app:layout_insetEdge="top" />
```
2 . 然后在另一个观察的child B的设置两个参数，layout_anchor和layout_anchorGravity

```
<View
    android:layout_width="80dp"
    android:layout_height="80dp"
    android:background="@color/colorAccent"
    app:layout_anchor="@id/view_host"
    app:layout_anchorGravity="bottom|end" />
```

效果如下，B就会随着A的移动而跟着移动
![anchor.gif](https://ae01.alicdn.com/kf/HTB1PTd8Xrj1gK0jSZFu5jcrHpXaI.gif)


### 2. insetEdge

关键词：layout_insetEdge和layout_dodgeInsetEdges

简单说明：child A 通过layout_insetEdge来设置插入CoordinatorLayout的方向，child B通过设置layout_dodgeInsetEdges来躲避来自相同方向的A，这样就可以避免产生重叠。

使用步骤:
1 . 先在被观察的child A中设置参数layout_insetEdge

```
<com.xugter.cooridnatorlayoutstudy.other.TouchView
    android:id="@+id/view_host"
    android:layout_width="80dp"
    android:layout_height="80dp"
    android:layout_gravity="center"
    android:background="@color/colorPrimary"
    app:layout_insetEdge="top" />
```

2 . 在观察的child B中设置参数layout_dodgeInsetEdges

```
<View
    android:layout_width="100dp"
    android:layout_height="50dp"
    android:layout_margin="20dp"
    android:background="@android:color/black"
    app:layout_dodgeInsetEdges="top" />
```
效果如下如上面的那个黑块

另外介绍两个特殊的控件跟insetEdge有关
[FloatingActionButton](https://developer.android.com/reference/android/support/design/widget/FloatingActionButton)和[Snackbar](https://developer.android.com/reference/android/support/design/widget/Snackbar)

这两个控件可以产生如下的交互
![insetEdge.gif](https://ae01.alicdn.com/kf/HTB1T4t7XuL2gK0jSZPh5jahvXXaL.gif)


原因是FloatingActionButton自带app:layout_dodgeInsetEdges="bottom"，
而Snackbar自带app:layout_insetEdge="bottom"，所以当Snackbar出现的时候FloatingActionButton能发生躲避行为

我们还可以自己加一个View来验证一下Snackbar是自带app:layout_insetEdge="bottom"
```
<View
    android:layout_width="40dp"
    android:layout_height="40dp"
    android:layout_gravity="bottom"
    android:background="@android:color/holo_blue_bright"
    app:layout_dodgeInsetEdges="bottom" />
```

这个view加入了app:layout_dodgeInsetEdges="bottom就可以产生和FloatingActionButton一样的行为，效果参考上面的蓝块

### 3. Behaviors

[Behaviors](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html)的功能就强大很多，后面会重点介绍。这里先简单介绍几个Google官方提供的几个使用了Behavior的控件。

便于理解我采用了最简单的结构，AppBarLayout+NestedScrollView,其中NestedScrollView可以由其他实现了NestedScrollingChild的控件代替

如下:![几个已经实现了NestedScrollingChild的控件](https://ae01.alicdn.com/kf/HTB1kQd8Xq61gK0jSZFl760DKFXaB.png)

其中我们最常用的大概就是下面这两货了，注意ScrollView是不可以的
- NestedScrollView
- RecyclerView

**注意**:一定要在NestedScrollView设置
``` 
app:layout_behavior="@string/appbar_scrolling_view_behavior"
```
不然达不到效果，具体原因会在以后分析Behaviors里在说明。

下面方便起见，称NestedScrollView为滚动View吧

#### AppBarLayout
[AppBarLayout](https://developer.android.com/reference/android/support/design/widget/AppBarLayout)继承自LinearLayout，所以基本布局方式跟LinearLayout是一样的

主要代码:
```
<android.support.design.widget.CoordinatorLayout ...>
    <android.support.design.widget.AppBarLayout
        ...
        >
        <ImageView
        ...
            app:layout_scrollFlags="scroll|enterAlways" />
    </android.support.design.widget.AppBarLayout>

    <android.support.v4.widget.NestedScrollView
        ...
        app:layout_behavior="@string/appbar_scrolling_view_behavior">
    </android.support.v4.widget.NestedScrollView>
</android.support.design.widget.CoordinatorLayout>
```
使用AppBarLayout的关键点是，在Child里面设置属性layout_scrollFlags

layout_scrollFlags可以设置下面这5个参数

**scroll**

**snap**

**enterAlways**

**enterAlwaysCollapsed**

**exitUntilCollapsed**

- **scroll**
这参数是其他四个参数的基础，即只有设置了这个参数，其他参数才会生效。相应的Child才会对滚动view的滚动才会响应。
```
app:layout_scrollFlags="scroll"
```

- **snap**
这个参数的作用是让Child具有吸附效果，抬手后会根据距离向上或向下滑动
```
app:layout_scrollFlags="scroll|snap"
```

- **enterAlways**
这个参数的作用是滑动Child在任何位置向下滑动时都触发该Child的向下滑动
```
app:layout_scrollFlags="scroll|enterAlways"
```

- **enterAlwaysCollapsed**
这个参数是enterAlways的附加值，滚动View向下滚动(未在顶部)，先显示最小高度，然后是enterAlways的滚动效果，最后显示最大高度。除了设置
```
app:layout_scrollFlags="scroll|enterAlways|enterAlwaysCollapsed"
```
为了有最大高度需要另外设置
```
android:minHeight="100dp"
```

- **exitUntilCollapsed**
这个参数表示滚动view向上滚动时，保留最小高度
```
app:layout_scrollFlags="scroll|exitUntilCollapsed"
```

#### CollapsingToolbarLayout
[CollapsingToolbarLayout](https://developer.android.com/reference/android/support/design/widget/CollapsingToolbarLayout)继承自FrameLayout所以布局特性和FrameLayout一样。谷歌官方的解释总结一下就是，带有视差动效的toolbar，是放在AppBarLayout中的一个直接子View。

主要代码：
```
<android.support.design.widget.CoordinatorLayout
    ...>
    <android.support.design.widget.AppBarLayout
        ...>
        <android.support.design.widget.CollapsingToolbarLayout
            ...
            app:collapsedTitleTextAppearance="@style/AppTheme.TextAppearance"
            app:contentScrim="@color/colorPrimary"
            app:layout_scrollFlags="scroll|exitUntilCollapsed">
            <ImageView
                ...
                app:layout_collapseParallaxMultiplier="0"
                app:layout_collapseMode="parallax" />
            <android.support.v7.widget.Toolbar
                ...
                app:layout_collapseMode="pin"
                app:popupTheme="@style/ThemeOverlay.AppCompat.Light" />
        </android.support.design.widget.CollapsingToolbarLayout>
        <android.support.design.widget.TabLayout
            .../>
    </android.support.design.widget.AppBarLayout>
    <android.support.v4.view.ViewPager
        ...
        app:layout_behavior="@string/appbar_scrolling_view_behavior" />
</android.support.design.widget.CoordinatorLayout>
```

下面简单介绍一下CollapsingToolbarLayout的几个属性

- **collapsedTitleTextAppearance**，**expandedTitleTextAppearance**
    这两个可以定制标题收起和展开的字体样式
- **contentScrim**
    可以设置Content scrim，参数可以是color和drawable，简单来说就是收起状态Toolbar的背景
- **layout_collapseMode**
说明CollapsingToolbarLayout的child可以设置为两种模式parallax和pin
  - **pin**
    就是简单的固定模式
  - **parallax**
    表示视差模式，可以根据需要
    **layout_collapseParallaxMultiplier**食用，**layout_collapseParallaxMultiplier**的数值是0~1
 0表示滚动没有视差，意思就是完全跟着下面的滚动view，1表示不动
![0.gif](https://ae01.alicdn.com/kf/HTB1MVB9XAL0gK0jSZFA5jcA9pXan.gif)
![1.gif](https://ae01.alicdn.com/kf/HTB1W_N7XEY1gK0jSZFM5jaWcVXaE.gif)


**哦~~~谷歌文档里还提醒，不要在toolbar运行时手动添加view到toolbar里面**

## 总结

这篇文章主要讲了CoordinatorLayout的特色，以及使用的几个方法。
介绍了一下跟CoordinatorLayout配合使用的几个控件的基本使用方法。但是这样的基本搭配很有可能无法达到开发者的需求，很多细节还是需要自己定制。要达到这样的要求，就需要去掌握Behaviors的使用了。

代码:[Github](https://github.com/Xugter/CooridnatorLayoutStudy)