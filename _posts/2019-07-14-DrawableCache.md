---
layout: post
title: DrawableCache遇坑小记
catalog:  true
tags:
    - DrawableCache
    - Android
    - 小记
---
# 一.问题背景描述

测试妹子:我发现我一打开X界面，Y界面的分隔线消失了！！！

我:这么神奇？？？

# 二.初找原因
经过一顿操作后发现，这两个界面用到同一个颜色值，而X界面对用了这个颜色值的当背景的view改变了透明度，所以再打开Y界面的时候，用同一个颜色的分割线就消失了。

# 三.临时解决
考虑到项目进度，我先用这个颜色写了一个shape来当背景，五分钟解决问题。

# 四.寻找真相
对于真相的好奇，打算看看究竟咋回事，之前对于资源加载这块也没好好了解过，趁这次正好看一下。

## 猜测
一定是哪里对这颜色做了缓存，如果这个猜测是对的，那么打印这两个drawable应该是同一个对象

```
I bbbb    : ====background=android.graphics.drawable.ColorDrawable@f74fed8
I bbbb    : ====getDrawable=android.graphics.drawable.ColorDrawable@d868231
```

凉凉,不是同一个对象，猜想错误

## 分析
那么接下来只能看源码了。通过setbackground不断的深入，最后来到ResourcesImpl.loadDrawable。
仔细看一下这个drawable的加载流程。

```
Drawable loadDrawable(@NonNull Resources wrapper, @NonNull TypedValue value, int id,
            int density, @Nullable Resources.Theme theme)
            throws NotFoundException {
                ...
    }
```

**1.** 先判断是不是ColorDrawable,然后去取相应类型的缓存。
```
            final boolean isColorDrawable;
            final DrawableCache caches;
            final long key;
            if (value.type >= TypedValue.TYPE_FIRST_COLOR_INT
                    && value.type <= TypedValue.TYPE_LAST_COLOR_INT) {
                isColorDrawable = true;
                caches = mColorDrawableCache;
                key = value.data;
            } else {
                isColorDrawable = false;
                caches = mDrawableCache;
                key = (((long) value.assetCookie) << 32) | value.data;
            }
```
ColorDrawable和其他drawable分别是两个缓存
```
    private final DrawableCache mDrawableCache = new DrawableCache();
    private final DrawableCache mColorDrawableCache = new DrawableCache();
```
至于其他drawable都有些啥，这就有点多了，如下，

![1.jpg](https://ae01.alicdn.com/kf/HTB1UyqFarY1gK0jSZTE760DQVXa5.png)

接下来其他Drawable都以BitmapDrawable为代表。

**2.** 缓存里是否能取到Drawable，如果有就直接返回。这里代码请先忽略!mPreloading && useCache这个条件
```
if (!mPreloading && useCache) {
    //注意这里getInstance方法，等一下会分析到
    final Drawable cachedDrawable = caches.getInstance(key, wrapper, theme);
    if (cachedDrawable != null) {
        return cachedDrawable;
    }
}
```

**3.** 如果缓存里没有，就加载新的drawable。(中间略过了跟预加载相关的部分)
```
Drawable dr;
if (cs != null) {
    //这里是对预加载来的判断，暂时先跳过
    dr = cs.newDrawable(wrapper);
} else if (isColorDrawable) {
    dr = new ColorDrawable(value.data);
} else {
    dr = loadDrawableForCookie(wrapper, value, id, null);
}
```

**4.** 缓存这次生成的drawable，并返回，然后看一下缓存的代码
```
if (dr != null && useCache) {
    dr.setChangingConfigurations(value.changingConfigurations);
    cacheDrawable(value, isColorDrawable, caches, theme, canApplyTheme, key, dr);
}
return dr;
```
然后看一下缓存的代码,就是把当前的ConstantState缓存起来，ConstantState是从Drawable获取的，具体ConstantState是啥，接着往下。
```
private void cacheDrawable(TypedValue value, boolean isColorDrawable, DrawableCache caches,
            Resources.Theme theme, boolean usesTheme, long key, Drawable dr) {
        final Drawable.ConstantState cs = dr.getConstantState();
        if (cs == null) {
            return;
        }
        ...
            synchronized (mAccessLock) {
                caches.put(key, theme, cs, usesTheme);
            }
        }
    }
```


**目前的结论**，加载drawable这个过程，针对ColorDrawable和非ColorDrawable分别加了两个缓存mColorDrawableCache和mDrawableCache，这两个cache都是DrawableCache，都继承于ThemedResourceCache。里面都存放着Drawable.ConstantState。

```
class DrawableCache extends ThemedResourceCache<Drawable.ConstantState> {
    ...
    public Drawable getInstance(long key, Resources resources, Resources.Theme theme) {
        final Drawable.ConstantState entry = get(key, theme);
        if (entry != null) {
            return entry.newDrawable(resources, theme);
        }

        return null;
    }
    ...
}
```

其中ThemedResourceCache有mThemedEntries，mUnthemedEntries，mNullThemedEntries存放着各种资源，主要是对不同主题的支持。这里的存储数据结构是ArrayMap和LongSparseArray。

```
abstract class ThemedResourceCache<T> {
    private ArrayMap<ThemeKey, LongSparseArray<WeakReference<T>>> mThemedEntries;
    private LongSparseArray<WeakReference<T>> mUnthemedEntries;
    private LongSparseArray<WeakReference<T>> mNullThemedEntries;
    ...
}
```

mColorDrawableCache和ColorDrawable具体缓存的东西不一样，mColorDrawableCache是存放ColorDrawable.ColorState,mDrawableCache是存放BItmapDrawable.BItmapState。ColorDrawable.ColorState和BItmapDrawable.BItmapState都继承于Drawable.ConstantState。

**5.** 接着分析，从刚才的第2步里面看到，如果缓存里面取到Drawable.ConstantState，就会调用getInstance，然后返回Drawable。

那么这个getInstance是怎么得到相应的Drawable的呢？
在刚才的DrawableCache的代码看到是通过Drawable.ConstantState.newDrawable得到的。再贴一下这段代码
```
class DrawableCache extends ThemedResourceCache<Drawable.ConstantState> {
    ...
    public Drawable getInstance(long key, Resources resources, Resources.Theme theme) {
        final Drawable.ConstantState entry = get(key, theme);
        if (entry != null) {
            return entry.newDrawable(resources, theme);
        }

        return null;
    }
    ...
}
```

那么接下来看看Drawable.ConstantState.newDrawable，这里我们选取ColorDrawable.ColorState作为代表，比较简单，易于理解。

```
public class ColorDrawable extends Drawable {

    private ColorState mColorState;
    private boolean mMutated;
    public ColorDrawable() {
        mColorState = new ColorState();
    }

    public ColorDrawable(@ColorInt int color) {
        mColorState = new ColorState();
        setColor(color);
    }

    @Override
    public Drawable mutate() {
        if (!mMutated && super.mutate() == this) {
            mColorState = new ColorState(mColorState);
            mMutated = true;
        }
        return this;
    }

    @ColorInt
    public int getColor() {
        return mColorState.mUseColor;
    }

    @Override
    public ConstantState getConstantState() {
        return mColorState;
    }

    final static class ColorState extends ConstantState {
        int[] mThemeAttrs;
        int mBaseColor; // base color, independent of setAlpha()
        @ViewDebug.ExportedProperty
        int mUseColor;  // basecolor modulated by setAlpha()
        @Config int mChangingConfigurations;
        ColorStateList mTint = null;
        Mode mTintMode = DEFAULT_TINT_MODE;

        ColorState() {
            // Empty constructor.
        }

        ColorState(ColorState state) {
            mThemeAttrs = state.mThemeAttrs;
            mBaseColor = state.mBaseColor;
            mUseColor = state.mUseColor;
            mChangingConfigurations = state.mChangingConfigurations;
            mTint = state.mTint;
            mTintMode = state.mTintMode;
        }

        @Override
        public Drawable newDrawable() {
            return new ColorDrawable(this, null);
        }

        @Override
        public Drawable newDrawable(Resources res) {
            return new ColorDrawable(this, res);
        }

    }

    private ColorDrawable(ColorState state, Resources res) {
        mColorState = state;
        updateLocalState(res);
    }
}
```

这里我删除了很多于这里无关的代码。从newDrawable的方法看出，即使刚才我们从缓存里面取到了缓存的ColorState，但还是会new ColorDrawable，所以就出现了，刚开始打脸的一幕，Drawable并不是同一个对象。但是，new drawable这个过程中，会把ColorState也传到新的Drawable中，所以ColorState中的属性都会传到新的Drawable。

那么现在可以大胆的猜想:

**1.** 刚开始的猜想，再进一步获取ConstantState,这两个的ConstantState肯定就是同一个对象了。验证如下

```
I bbbb    : ====background cs=android.graphics.drawable.ColorDrawable$ColorState@dfe7bd1
I bbbb    : ====getDrawable cs=android.graphics.drawable.ColorDrawable$ColorState@dfe7bd1
```

**2.** ConstantState里面一定存储着改变后的Drawable，要不然也不会获取到，透明度改变后的颜色了。
这个可以从ColorState里面的属性可以容易的发现mUseColor，这个就是改变后的颜色。

同理BitmapDrawable等一些其他的Drawable也是类似的，只是相应的ConstantState存储东西不一样而已。

至此我们，大概缕清楚了这个缓存的过程。

# 五.正经的解决办法

那么这个getDrawable互相影响的问题，到底怎么解决呢？难道只能想我刚开始用的方法一样，新建另外一个资源吗。那可能有点低估了Google爸爸了。

在刚才ColorDrawable里面的代码中有个mutate方法注意到没，这个方法里面有new ColorState(mColorState)，会把原来的ColorState传进去，再new一个ColorState。

这样我们只要在设置独有的属性前，调用mutate，就不会影响到其他的Drawable了。

# 六.总结
**问题:** drawable在不同的地方，修改属性会相互影响，
**原因:** android缓存了调用过的drawable的ConstantState，ConstantState里面保存了drawable的一些属性
**解决方法:** 设置独有的属性前，调用mutate，就不会影响到其他的Drawable了
