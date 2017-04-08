> 本文分析版本: **Android API 22**

## 1. 简介
`Android`开发中，如果我们希望使一个`View`滑动的话，除了使用属性动画外。我们还可以使用系统提供给我们的两个类`Scroller`和`OverScroller`用来实现弹性滑动。在我以前的一篇[ViewDragHelper源码分析](http://www.jianshu.com/p/07d717ef0b28)中我们有讲到过`Scroller`的作用。那么我们今天就来仔细分析一下`Scroller`的使用方法以及实现方式。

## 2. 使用方法
在看`Scroller`的使用方法之前我们需要先了解一下`View`中的`scrollBy()`和`scrollTo()`方法，`scrollTo()`方法的实现如下：

```java

    public void scrollTo(int x, int y) {
    	//如果当前偏移量变化
        if (mScrollX != x || mScrollY != y) {
            int oldX = mScrollX;
            int oldY = mScrollY;
			//赋值偏移量
            mScrollX = x;
            mScrollY = y;
            invalidateParentCaches();
            //回调onScrollChanged方法
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (!awakenScrollBars()) {
                postInvalidateOnAnimation();
            }
        }
    }

```
`scrollTo()`是指将当前视图内容横向偏移`x`距离，纵向偏移`y`距离。注意这里是`View`的内容的偏移，而不是`View`本身。而`scrollBy()`方法如下：

```java
    public void scrollBy(int x, int y) {
        scrollTo(mScrollX + x, mScrollY + y);
    }
```
`scrollBy()`方法里直接调用了`scrollTo()`方法，表示在当前偏移量的基础上继续偏移`(x,y)`。现在我们来看看`Scroller`的用法。[SkyScrollerDemo](https://github.com/Skykai521/SkyScrollerDemo)是我写的一个`Scroller`和`OverScroller`的使用`demo`。下面的用法都是来自于这个`demo`里，大家可以`clone`下来配合本文一起阅读。本文我们主要研究`Scroller`。对于`OverScroller`我在`demo`里也写了相关的使用方法，在本文的最后我们再做讨论。

`Scroller`一般需要配合重写`computeScroll()`一起使用，代码如下：

```java
public class ScrollTextView extends TextView {
    private Context mContext;
    private Scroller mScroller;

    public ScrollTextView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        this.mContext = context;
        init();
    }

    private void init() {
        mScroller = new Scroller(mContext);
    }

    @Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
            offsetLeftAndRight(mScroller.getCurrX() - mLeft);
            offsetTopAndBottom(mScroller.getCurrY() - mTop);
            invalidate();
        }
    }
    //以mLeft,mTop为初始点，在DEFAULT_DURATION的时间内，在Y轴上滑动-400的偏移量
    public void startScrollerScroll() {
        mScroller.startScroll(mLeft, mTop, 0, -400, DEFAULT_DURATION);
        invalidate();
    }
    //以mLeft,mTop为初始点，并以Y方向上-5000的加速度滑动，最小Y坐标为200，最大Y坐标为1200
    public void startScrollerFling() {
        mScroller.fling(mLeft, mTop, 0, -5000, mLeft, mLeft, 200, 1200);
        invalidate();
    }
}
```

在上面的代码里，当我们调用`startScrollerScroll()`与`startScrollerFling()`方法时我们就发现`View`滑动了。如果以前没了解过`Scroller`的同学可能会不理解。这里大致分析一下调用流程，首先我们要知道`Scroller`其实只负责计算，它并不负责滑动`View`，当我们调用了`Scroller`的`startScrollerScroll()`方法时，我们紧接着调用了`invalidate()`方法。`invalidate()`方法会使`View`重新绘制。因此会调用`View`的`draw()`方法，在`View`的`draw()`方法中又会去调用`computeScroll()`方法，`computeScroll()`方法在`View`中是一个空实现，所以需要我们自己实现`computeScroll()`方法。在上面的`computeScroll()`方法中，我们调用了`mScroller.computeScrollOffset()`方法来计算当前滑动的偏移量。如果还在滑动过程中就会返回`true`。所以我们就能在`if`中通过`Scroller`拿到当前的滑动坐标从而做任何我们想做的处理。在`demo`里我们根据滑动的偏移量来改变了`View`的坐标偏移量。从而形成了滑动动画。下面我们解释一下`Scroller`的两个方法的具体作用：

### 1.startScroll(int startX, int startY, int dx, int dy, int duration):
通过起始点、偏移的距离和滑动的时间来开始滑动。
- startX 起始滑动点的X坐标
- startY 起始滑动点的Y坐标
- dx 滑动的水平偏移量。>0 则表示往左滑动。
- dy 滑动的垂直偏移量。>0 则表示往上滑动。
- duration 滑动执行的时间

### 2.fling(int startX, int startY, int velocityX, int velocityY, int minX, int maxX, int minY, int maxY) :
基于一个快速滑动手势下的滑动。滑动的距离与这个手势最初的加速度有关。
- startX 起始滑动点的X坐标
- startY 起始滑动点的Y坐标
- velocityX X方向上的加速度
- velocityY Y方向上的加速度
- minX X方向上滑动的最小值，不会滑动超过这个点
- maxX X方向上滑动的最大值，不会滑动超过这个点
- minY Y方向上滑动的最小值，不会滑动超过这个点
- maxY Y方向上滑动的最大值，不会滑动超过这个点

## 3. 源码分析
我们依然通过调用流程来分析`Scroller`的实现：

### 1.构造方法

```java
public Scroller(Context context) {
    this(context, null);
}

public Scroller(Context context, Interpolator interpolator) {
    this(context, interpolator,
            context.getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.HONEYCOMB);
}

public Scroller(Context context, Interpolator interpolator, boolean flywheel) {
    mFinished = true;
    if (interpolator == null) {
        mInterpolator = new ViscousFluidInterpolator();
    } else {
        mInterpolator = interpolator;
    }
    mPpi = context.getResources().getDisplayMetrics().density * 160.0f;
    mDeceleration = computeDeceleration(ViewConfiguration.getScrollFriction());
    mFlywheel = flywheel;

    mPhysicalCoeff = computeDeceleration(0.84f); // look and feel tuning
}
```
最终都会调用最后一个构造方法。必须传入`Context`对象。可以传入自定义的`interpolator`和是否支持飞轮`flywheel`的功能，当然这两个并不是必须的。如果不传入`interpolator`会默认创建一个`ViscousFluidInterpolator`，从字面意义上看是一个粘性流体插值器。对于`flywheel`是指是否支持在滑动过程中，如果有新的`fling()`方法调用是否累加加速度。如果不传默认在2.3以上都会支持。剩下就是初始化了一些用于计算的参数。这样就完成了`Scroller`的初始化了。下面我们来看看`startScroll()`方法的实现：

### 2.startScroll()方法的实现

```java
public void startScroll(int startX, int startY, int dx, int dy, int duration) {
  // mMode 分两种方式 1.滑动:SCROLL_MODE 2. 加速度滑动:FLING_MODE
  mMode = SCROLL_MODE;
  // 是否滑动结束 这里是开始所以设置为false
  mFinished = false;
  // 滑动的时间
  mDuration = duration;
  // 开始的时间
  mStartTime = AnimationUtils.currentAnimationTimeMillis();
  // 开始滑动点的X坐标
  mStartX = startX;
  // 开始滑动点的Y坐标
  mStartY = startY;
  // 最终滑动到位置的X坐标
  mFinalX = startX + dx;
  // 最终滑动到位置的Y坐标
  mFinalY = startY + dy;
  // X方向上滑动的偏移量
  mDeltaX = dx;
  // Y方向上滑动的偏移量
  mDeltaY = dy;
  // 持续时间的倒数 最终用来计算得到插值器返回的值
  mDurationReciprocal = 1.0f / (float) mDuration;
}
```
很简单只是一些变量的赋值。根据我们前面使用方法里的分析，最终会调用`computeScrollOffset()`方法：

### 3.computeScrollOffset() 方法中 SCROLL_MODE 的实现

```java
// 当你需要知道新的位置的时候调用这个方法，如果动画还未结束则返回true
public boolean computeScrollOffset() {
    //如果已经结束 则直接返回false
    if (mFinished) {
        return false;
    }
    //得到以及度过的时间
    int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);

    //如果还在动画时间内
    if (timePassed < mDuration) {
        switch (mMode) {
            case SCROLL_MODE:
                // 根据timePassed * mDurationReciprocal,从mInterpolator中取出当前需要偏移量的比例
                final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
                // 赋值给 mCurrX，mCurrY
                mCurrX = mStartX + Math.round(x * mDeltaX);
                mCurrY = mStartY + Math.round(x * mDeltaY);
                break;
            case FLING_MODE:
                ...
                break;
        }
    }
    else {
        mCurrX = mFinalX;
        mCurrY = mFinalY;
        mFinished = true;
    }
    return true;
}
```
首先的到当前时间与滑动开始时间的时间差，如果还在滑动时间内则通过插值器获得当前的进度并乘以总偏移量并赋值给`mCurrX`，`mCurrY`。如果已经结束则直接将`mFinalX`和`mFinalY`赋值并将`mFinished`设置为`true`。所以这样我们就能通过`getCurrX()`和`getCurrY()`来得到对应的`mCurrX`和`mCurrY`来做相应的处理了。整个`Scroll`的过程就是这样了。

### 4.fling()方法的实现

```java
    public void fling(int startX, int startY, int velocityX, int velocityY,
                      int minX, int maxX, int minY, int maxY) {
        // 如果前一次滑动还未结束，又调用了新的fling()方法时，
        // 则累加相同方向上加速度
        if (mFlywheel && !mFinished) {
            float oldVel = getCurrVelocity();

            float dx = (float) (mFinalX - mStartX);
            float dy = (float) (mFinalY - mStartY);
            float hyp = FloatMath.sqrt(dx * dx + dy * dy);

            float ndx = dx / hyp;
            float ndy = dy / hyp;

            float oldVelocityX = ndx * oldVel;
            float oldVelocityY = ndy * oldVel;
            if (Math.signum(velocityX) == Math.signum(oldVelocityX) &&
                    Math.signum(velocityY) == Math.signum(oldVelocityY)) {
                velocityX += oldVelocityX;
                velocityY += oldVelocityY;
            }
        }

        //设置为FLING_MODE
        mMode = FLING_MODE;
        mFinished = false;
        //根据勾股定理获得总加速度
        float velocity = FloatMath.sqrt(velocityX * velocityX + velocityY * velocityY);

        mVelocity = velocity;
        // 通过加速度得到滑动持续时间
        mDuration = getSplineFlingDuration(velocity);
        mStartTime = AnimationUtils.currentAnimationTimeMillis();
        mStartX = startX;
        mStartY = startY;

        float coeffX = velocity == 0 ? 1.0f : velocityX / velocity;
        float coeffY = velocity == 0 ? 1.0f : velocityY / velocity;

        double totalDistance = getSplineFlingDistance(velocity);
        mDistance = (int) (totalDistance * Math.signum(velocity));

        mMinX = minX;
        mMaxX = maxX;
        mMinY = minY;
        mMaxY = maxY;

        mFinalX = startX + (int) Math.round(totalDistance * coeffX);
        // Pin to mMinX <= mFinalX <= mMaxX
        mFinalX = Math.min(mFinalX, mMaxX);
        mFinalX = Math.max(mFinalX, mMinX);

        mFinalY = startY + (int) Math.round(totalDistance * coeffY);
        // Pin to mMinY <= mFinalY <= mMaxY
        mFinalY = Math.min(mFinalY, mMaxY);
        mFinalY = Math.max(mFinalY, mMinY);
    }
```
依然是为计算需要的各种变量赋值。因为引入了加速度的概念所以变得相对复杂，首先先判断了如果一次滑动未结束又触发另一次滑动时，是否需要累加加速度。然后是设置`mMode`为`FLING_MODE`。然后根据`velocityX`和`velocityY`算出总的加速度`velocity`，紧接着算出这个加速度下可以滑动的距离`mDistance`。最后再通过`x`或`y`方向上的加速度比值以及我们设定的最大值和最小值来给`mFinalX`或`mFinalY`赋值。赋值结束后，通过调用`invalidate()`，最终依然会调用`computeScrollOffset()`方法：

### 5.computeScrollOffset() 方法中 FLING_MODE 的实现

```java

    public boolean computeScrollOffset() {
        if (mFinished) {
            return false;
        }

        int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);

        if (timePassed < mDuration) {
            switch (mMode) {
                case SCROLL_MODE:
                    ...
                    break;
                case FLING_MODE:
                    // 当前已滑动的时间与总滑动时间的比值
                    final float t = (float) timePassed / mDuration;
                    final int index = (int) (NB_SAMPLES * t);
                    // 距离系数
                    float distanceCoef = 1.f;
                    // 加速度系数
                    float velocityCoef = 0.f;
                    if (index < NB_SAMPLES) {
                        final float t_inf = (float) index / NB_SAMPLES;
                        final float t_sup = (float) (index + 1) / NB_SAMPLES;
                        final float d_inf = SPLINE_POSITION[index];
                        final float d_sup = SPLINE_POSITION[index + 1];
                        velocityCoef = (d_sup - d_inf) / (t_sup - t_inf);
                        distanceCoef = d_inf + (t - t_inf) * velocityCoef;
                    }

                    // 计算出当前的加速度
                    mCurrVelocity = velocityCoef * mDistance / mDuration * 1000.0f;
                    // 计算出当前的mCurrX 与mCurrY
                    mCurrX = mStartX + Math.round(distanceCoef * (mFinalX - mStartX));
                    // Pin to mMinX <= mCurrX <= mMaxX
                    mCurrX = Math.min(mCurrX, mMaxX);
                    mCurrX = Math.max(mCurrX, mMinX);

                    mCurrY = mStartY + Math.round(distanceCoef * (mFinalY - mStartY));
                    // Pin to mMinY <= mCurrY <= mMaxY
                    mCurrY = Math.min(mCurrY, mMaxY);
                    mCurrY = Math.max(mCurrY, mMinY);

                    // 如果到达了终点 则结束
                    if (mCurrX == mFinalX && mCurrY == mFinalY) {
                        mFinished = true;
                    }

                    break;
            }
        }
        else {
            mCurrX = mFinalX;
            mCurrY = mFinalY;
            mFinished = true;
        }
        return true;
    }
```
由于`fling()`方法中将`mMode`赋值为`FLING_MODE`。所以我们直接来看`FLING_MODE`中的代码。可以看出根据当前滑动时间与总滑动时间的比例。再根据一个`SPLINE_POSITION`数组计算出了距离系数`distanceCoef`与加速度系数`velocityCoef`。再根据这两个系数计算出当前加速度与当前的`mCurrX`与`mCurrY`。关于`SPLINE_POSITION`的初始化是在下面的静态代码块里赋值的：

```java
    static {
        float x_min = 0.0f;
        float y_min = 0.0f;
        for (int i = 0; i < NB_SAMPLES; i++) {
            final float alpha = (float) i / NB_SAMPLES;

            float x_max = 1.0f;
            float x, tx, coef;
            while (true) {
                x = x_min + (x_max - x_min) / 2.0f;
                coef = 3.0f * x * (1.0f - x);
                tx = coef * ((1.0f - x) * P1 + x * P2) + x * x * x;
                if (Math.abs(tx - alpha) < 1E-5) break;
                if (tx > alpha) x_max = x;
                else x_min = x;
            }
            SPLINE_POSITION[i] = coef * ((1.0f - x) * START_TENSION + x) + x * x * x;

            float y_max = 1.0f;
            float y, dy;
            while (true) {
                y = y_min + (y_max - y_min) / 2.0f;
                coef = 3.0f * y * (1.0f - y);
                dy = coef * ((1.0f - y) * START_TENSION + y) + y * y * y;
                if (Math.abs(dy - alpha) < 1E-5) break;
                if (dy > alpha) y_max = y;
                else y_min = y;
            }
            SPLINE_TIME[i] = coef * ((1.0f - y) * P1 + y * P2) + y * y * y;
        }
        SPLINE_POSITION[NB_SAMPLES] = SPLINE_TIME[NB_SAMPLES] = 1.0f;
    }
```
我并没有看懂这段代码的实际意义。网上也没有找到比较清晰的解释。通过`debug`得知`SPLINE_POSITION`是一个长度为`101`并且从`0-1`递增数组。猜想这应该是一个函数模型并且最终用于计算出滑动过程中的加速度与位置。至此`Scroller`的两个主要方法的实现我们就分析完了。

## 4. OverScroller解析

`OverScroller`是对`Scroller`的拓展，它在`Scroller`的基础上拓展出了更多的方法。`OverScroller`的`fling`方法支持滑动到终点之后并超出一段距离并返回，类似于弹性效果。另外一个`springBack()`方法是指将指定的点平滑滚动到指定的终点上。这个终点由设置的参数决定。原理我们就不再探究了，大家可以自行研究这两个类的差别。最后具体的使用方法在文章最上面的`demo`里都有提供。可以`clone`下来帮助理解。