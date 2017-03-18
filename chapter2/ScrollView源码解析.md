

ScrollView 源码解析
==

> 本文分析版本: **Android API 23**

### 1.简介
`ScrollView`是我们在开发中经常使用的控件。当我们需要展示的内容比较多但并不是重复的`item`时，我们就会使用`ScrollView`使内容可以在垂直方向滚动显示防止显示不全。`ScrollView`使用起来非常简单，大多数情况下你甚至都不用写一行`Java`代码就能使用`ScrollView`了。但是要注意的是`ScrollView`中只能添加一个子`View`。今天我们就来看看`ScrollView`到底是如何实现的。以及最后会教大家一行代码实现类似`IOS`上的弹性`ScrollView`。

### 2.源码分析

#### 2.1 继承关系

![extend_relation.png](http://upload-images.jianshu.io/upload_images/1485091-1e3254ea271dfa6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 2.2 主要辅助类

```java
//用来计算滑动位置
private OverScroller mScroller;
//用来绘制边缘阴影
private EdgeEffect mEdgeGlowTop;
private EdgeEffect mEdgeGlowBottom;
//用于计算滑动时的加速度
private VelocityTracker mVelocityTracker;

```
#### 2.3 构造方法
`ScrollView`的构造方法如下：

```java

public ScrollView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context, attrs, defStyleAttr, defStyleRes);
    initScrollView();

    final TypedArray a = context.obtainStyledAttributes(
            attrs, com.android.internal.R.styleable.ScrollView, defStyleAttr, defStyleRes);
    setFillViewport(a.getBoolean(R.styleable.ScrollView_fillViewport, false));
    a.recycle();
}

```
在构造方法中分别调用了`initScrollView()`与`setFillViewport()`方法，代码如下：

```java
private void initScrollView() {
    //初始化OverScroller
    mScroller = new OverScroller(getContext());
    setFocusable(true);
    setDescendantFocusability(FOCUS_AFTER_DESCENDANTS);
    setWillNotDraw(false);
    final ViewConfiguration configuration = ViewConfiguration.get(mContext);
    //被认为是滑动操作的最小距离
    mTouchSlop = configuration.getScaledTouchSlop();
    //最小加速度
    mMinimumVelocity = configuration.getScaledMinimumFlingVelocity();
    //最大加速度
    mMaximumVelocity = configuration.getScaledMaximumFlingVelocity();
    //用手指拖动超过边缘的最大距离
    mOverscrollDistance = configuration.getScaledOverscrollDistance();
    //滑动超过边缘的最大距离
    mOverflingDistance = configuration.getScaledOverflingDistance();
}
```
可以看到是初始化了一些类与参数，继续看看`setFillViewport()`：

```java
public void setFillViewport(boolean fillViewport) {
    if (fillViewport != mFillViewport) {
        mFillViewport = fillViewport;
        requestLayout();
    }
}
```
只是根据布局文件中的`fillViewport`属性来给`mFillViewport`赋值并调用`requestLayout()`方法。`mFillViewport`如果为`true`则表示：将子`View`的高度延伸到和视图高度一致，即充满整个视图。初始化结束之后，会进入到绘制流程。下面我们按照`Measure` -> `Layout` -> `Draw`的绘制流程来分析`ScrollView`中的实现。

#### 2.4 Measure、Layout与Draw
##### 2.4.1 onMeasure方法的实现
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    if (!mFillViewport) {
        return;
    }

    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    if (heightMode == MeasureSpec.UNSPECIFIED) {
        return;
    }

    if (getChildCount() > 0) {
        // 获取子View
        final View child = getChildAt(0);
        // 获取ScrollView的高度
        final int height = getMeasuredHeight();
        if (child.getMeasuredHeight() < height) {
            final int widthPadding;
            final int heightPadding;
            final FrameLayout.LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final int targetSdkVersion = getContext().getApplicationInfo().targetSdkVersion;
            // 获取ScrollView的padding
            if (targetSdkVersion >= VERSION_CODES.M) {
                widthPadding = mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin;
                heightPadding = mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin;
            } else {
                widthPadding = mPaddingLeft + mPaddingRight;
                heightPadding = mPaddingTop + mPaddingBottom;
            }

            final int childWidthMeasureSpec = getChildMeasureSpec(
                    widthMeasureSpec, widthPadding, lp.width);
            final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                    height - heightPadding, MeasureSpec.EXACTLY);
            //根据新的高度重新measure子View
            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
```
从代码中可以看到首先调用了`super.onMeasure(widthMeasureSpec, heightMeasureSpec);`即父类`FrameLayout`的`onMeasure()`方法。如果我们将`mFillViewport`设置为`false`的话将会直接`return`。当为`true`时才会继续执行，会根据子`View`的高度和`ScrollView`本身的高度决定是否重新`measure`子`View`使其充满`ScrollView`。`ScrollView`的`onMeasure()`其实就是处理了`mFillViewport`。

##### 2.4.1 onLayout方法的实现
```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    super.onLayout(changed, l, t, r, b);
    mIsLayoutDirty = false;
    // Give a child focus if it needs it
    if (mChildToScrollTo != null && isViewDescendantOf(mChildToScrollTo, this)) {
        scrollToChild(mChildToScrollTo);
    }
    mChildToScrollTo = null;
    //是否还未添加过window中去
    if (!isLaidOut()) {
        if (mSavedState != null) {
            mScrollY = mSavedState.scrollPosition;
            mSavedState = null;
        } // mScrollY default value is "0"

        final int childHeight = (getChildCount() > 0) ? getChildAt(0).getMeasuredHeight() : 0;
        final int scrollRange = Math.max(0,
                childHeight - (b - t - mPaddingBottom - mPaddingTop));

        // Don't forget to clamp
        if (mScrollY > scrollRange) {
            mScrollY = scrollRange;
        } else if (mScrollY < 0) {
            mScrollY = 0;
        }
    }

    // Calling this with the present values causes it to re-claim them
    scrollTo(mScrollX, mScrollY);
}
```
首先也是调用了父类的`onLayout`方法。接下来处理了是否有需要滚动到的`View`，以及根据保存的滚动状态来决定是否需要滚动。如果需要则调用`scrollTo()`方法。

##### 2.4.1 draw方法的实现

```java
@Override
public void draw(Canvas canvas) {
    super.draw(canvas);
    if (mEdgeGlowTop != null) {
        final int scrollY = mScrollY;
        final boolean clipToPadding = getClipToPadding();
        if (!mEdgeGlowTop.isFinished()) {
            ......
            if (mEdgeGlowTop.draw(canvas)) {
                postInvalidateOnAnimation();
            }
            canvas.restoreToCount(restoreCount);
        }
        if (!mEdgeGlowBottom.isFinished()) {
            ......
            if (mEdgeGlowBottom.draw(canvas)) {
                postInvalidateOnAnimation();
            }
            canvas.restoreToCount(restoreCount);
        }
    }
}
```
依然是调用了父类的`draw`方法。之后则是根据是否需要绘制边缘阴影来绘制阴影。`ScrollView`的边缘阴影就是在这里绘制的。值得一提的是包括`ListView`以及`RecycleView`的边缘阴影都是用这种方法来绘制的。以上就是`ScrollView`的整个绘制流程。可以看出都是调用了父类的对应方法。自身只处理了一些与`ScrollView`相关的属性。分析完绘制流程我们就来看看`ScrollView`中的触摸事件处理机制，来看看`ScrollView`中的滑动滚动到底是如何做到的：

#### 2.5 触摸事件处理
说到触摸事件的分发与消费机制这算是一个比较基础的知识。但是要是完全掌握也并不是那么容易的，这里推荐一篇文章[Android：View的事件分发与消费机制](http://www.jianshu.com/p/1528eb2ee54b)。对事件处理机制还不了解的同学可以先看看这边文章。`ScrollView`因为是继承自`ViewGroup`的，所以触摸事件会依次调用`dispatchTouchEvent()` -> `onInterceptTouchEvent()` 若返回`true`-> `onTouchEvent()`处理触摸事件。`ScrollView`并没有重写`dispatchTouchEvent()`方法，所以我们从`onInterceptTouchEvent()`方法来看。

##### 2.5.1 onInterceptTouchEvent方法的实现

```java
//这个方法只决定我们是否拦截这个手势，如果返回true，则onMotionEvent会被调用，并处理滑动事件。
//此方法并不处理事件
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {

    //如果是移动手势并在处于拖拽阶段，直接返回true
    final int action = ev.getAction();
    if ((action == MotionEvent.ACTION_MOVE) && (mIsBeingDragged)) {
        return true;
    }

    //如果并不能滑动则返回false
    if (getScrollY() == 0 && !canScrollVertically(1)) {
        return false;
    }

    switch (action & MotionEvent.ACTION_MASK) {
        case MotionEvent.ACTION_MOVE: {
            //检测用户是否移动了足够远的距离。

            final int activePointerId = mActivePointerId;
            if (activePointerId == INVALID_POINTER) {
                // If we don't have a valid id, the touch down wasn't on content.
                break;
            }

            final int pointerIndex = ev.findPointerIndex(activePointerId);
            if (pointerIndex == -1) {
                Log.e(TAG, "Invalid pointerId=" + activePointerId
                        + " in onInterceptTouchEvent");
                break;
            }
            //得到当前触摸的y左边
            final int y = (int) ev.getY(pointerIndex);
            //计算移动的插值
            final int yDiff = Math.abs(y - mLastMotionY);
            //如果yDiff大于最小滑动距离，并且是垂直滑动则认为触发了滑动手势。
            if (yDiff > mTouchSlop && (getNestedScrollAxes() & SCROLL_AXIS_VERTICAL) == 0) {
                //标记拖动状态为true
                mIsBeingDragged = true;
                //赋值mLastMotionY
                mLastMotionY = y;
                //初始化mVelocityTracker并添加
                initVelocityTrackerIfNotExists();
                mVelocityTracker.addMovement(ev);
                mNestedYOffset = 0;
                if (mScrollStrictSpan == null) {
                    mScrollStrictSpan = StrictMode.enterCriticalSpan("ScrollView-scroll");
                }
                final ViewParent parent = getParent();
                if (parent != null) {
                    //通知父布局不再拦截触摸事件
                    parent.requestDisallowInterceptTouchEvent(true);
                }
            }
            break;
        }

        case MotionEvent.ACTION_DOWN: {
            final int y = (int) ev.getY();
            //触摸点不在子View内
            if (!inChild((int) ev.getX(), (int) y)) {
                mIsBeingDragged = false;
                recycleVelocityTracker();
                break;
            }

            //记录当前位置
            mLastMotionY = y;
            //记录pointer的ID,ACTION_DOWN总会在index 0
            mActivePointerId = ev.getPointerId(0);
            //初始化mVelocityTracker
            initOrResetVelocityTracker();
            mVelocityTracker.addMovement(ev);
            //如果在滑动过程中则mIsBeingDragged = true
            mIsBeingDragged = !mScroller.isFinished();
            if (mIsBeingDragged && mScrollStrictSpan == null) {
                mScrollStrictSpan = StrictMode.enterCriticalSpan("ScrollView-scroll");
            }
            //回调NestedScroll相关接口
            startNestedScroll(SCROLL_AXIS_VERTICAL);
            break;
        }

        case MotionEvent.ACTION_CANCEL:
        case MotionEvent.ACTION_UP:
            //清除Drag状态
            mIsBeingDragged = false;
            mActivePointerId = INVALID_POINTER;
            recycleVelocityTracker();
            if (mScroller.springBack(mScrollX, mScrollY, 0, 0, 0, getScrollRange())) {
                postInvalidateOnAnimation();
            }
            //回调NestedScroll相关接口
            stopNestedScroll();
            break;
        case MotionEvent.ACTION_POINTER_UP:
            //当多个手指触摸中有一个手指抬起时，判断是不是当前active的点，如果是则寻找新的
            //mActivePointerId
            onSecondaryPointerUp(ev);
            break;
    }
    //最终根据是否开始拖拽的状态返回
    return mIsBeingDragged;
}

```

以上就是`onInterceptTouchEvent()`的整体实现。`onInterceptTouchEvent()`只决定是否拦截触摸事件并交给`onTouchEvent()`处理。内部并不处理触摸逻辑。`ScrollView`中根据`mIsBeingDragged`来决定是否拦截事件。当手指按下发生`MotionEvent.ACTION_DOWN`时，会记录当前位置并检测是否在快速滚动过程中如果是则返回`true`。当手指移动发生`MotionEvent.ACTION_MOVE`时，会判断是否是垂直方向上的滑动事件，如果是则返回`true`。当手指抬起发生`MotionEvent.ACTION_UP`时，则清除状态并返回`false`。在返回`true`的情况中，`onTouchEvent()`方法就会被调用来处理触摸事件。我们继续来看`onTouchEvent()`方法的实现。

##### 2.5.2 onTouchEvent方法的实现

在看`onTouchEvent()`的实现之前，我们知道在`ScrollView`中手指无论怎么移动，只会有垂直方向上的滑动发生。而触摸事件的大致流程是：

        ACTION_DOWN -> ACTION_MOVE -> ... -> ACTION_MOVE -> ACTION_UP

我们根据事件的类型分别来分析：

- ACTION_DOWN:
  `ACTION_DOWN`代表手指按下时第一个发生的事件，在`onTouchEvent()`中实现如下：

```java

    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        //初始化VelocityTracker
        initVelocityTrackerIfNotExists();
        //复制当前的MotionEvent赋值给vtev
        MotionEvent vtev = MotionEvent.obtain(ev);

        final int actionMasked = ev.getActionMasked();

        if (actionMasked == MotionEvent.ACTION_DOWN) {
            mNestedYOffset = 0;
        }
        //调整vtev的偏移量
        vtev.offsetLocation(0, mNestedYOffset);

        switch (actionMasked) {
            case MotionEvent.ACTION_DOWN: {
                if (getChildCount() == 0) {
                    return false;
                }
                // 将!mScroller.isFinished()赋值给mIsBeingDragged
                if ((mIsBeingDragged = !mScroller.isFinished())) {
                    final ViewParent parent = getParent();
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }
                }

                //如果正在fling状态并且用户触摸。则停止fling。
                //当处于fling过程中isFinished为false。
                //fling ：即快速滑动。
                if (!mScroller.isFinished()) {
                    //停止
                    mScroller.abortAnimation();
                    if (mFlingStrictSpan != null) {
                        mFlingStrictSpan.finish();
                        mFlingStrictSpan = null;
                    }
                }

                //记录触摸事件的初始值
                mLastMotionY = (int) ev.getY();
                mActivePointerId = ev.getPointerId(0);
                startNestedScroll(SCROLL_AXIS_VERTICAL);
                break;
            }
            ......
        }

        if (mVelocityTracker != null) {
            mVelocityTracker.addMovement(vtev);
        }
        vtev.recycle();
        return true;
    }
```
在处理各种事件之前，首先初始化了`VelocityTracker`。并且复制一个新的`MotionEvent`对象用于计算加速度。接着开始处理`ACTION_DOWN`：首先是给`mIsBeingDragged`赋值，接着检查是否在`fling`动画执行过程中，如果正在执行则停止，这也是为什么我们在`ScrollView`滑动过程中手指触摸时会终止`ScrollView`的滑动。最后记录了`mLastMotionY`与`mActivePointerId`。

- ACTION_MOVE:
  当手指移动时，会产生`ACTION_MOVE`事件：

```java

 @Override
    public boolean onTouchEvent(MotionEvent ev) {
        ......
        switch (actionMasked) {
            ......
            //如果为ACTION_MOVE事件时
            case MotionEvent.ACTION_MOVE:
                final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
                if (activePointerIndex == -1) {
                    Log.e(TAG, "Invalid pointerId=" + mActivePointerId + " in onTouchEvent");
                    break;
                }
                //得到当前y值
                final int y = (int) ev.getY(activePointerIndex);
                //计算偏移量deltaY
                int deltaY = mLastMotionY - y;
                //如果dispatchNestedPreScroll返回true，即有NestedScroll存在
                if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset)) {
                    deltaY -= mScrollConsumed[1];
                    vtev.offsetLocation(0, mScrollOffset[1]);
                    mNestedYOffset += mScrollOffset[1];
                }
                //如果还未处于drag状态，并且deltaY大于最小滑动距离，
                //则赋值mIsBeingDragged为true
                if (!mIsBeingDragged && Math.abs(deltaY) > mTouchSlop) {
                    final ViewParent parent = getParent();
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }
                    mIsBeingDragged = true;
                    if (deltaY > 0) {
                        deltaY -= mTouchSlop;
                    } else {
                        deltaY += mTouchSlop;
                    }
                }
                //如果在拖拽状态
                if (mIsBeingDragged) {
                    //记录当前的y值
                    mLastMotionY = y - mScrollOffset[1];

                    final int oldY = mScrollY;
                    final int range = getScrollRange();
                    final int overscrollMode = getOverScrollMode();
                    boolean canOverscroll = overscrollMode == OVER_SCROLL_ALWAYS ||
                            (overscrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);

                    // 调用overScrollBy()方法处理滑动事件。
                    if (overScrollBy(0, deltaY, 0, mScrollY, 0, range, 0, mOverscrollDistance, true)
                            && !hasNestedScrollingParent()) {
                        // Break our velocity if we hit a scroll barrier.
                        mVelocityTracker.clear();
                    }

                    final int scrolledDeltaY = mScrollY - oldY;
                    final int unconsumedY = deltaY - scrolledDeltaY;
                    //处理NestedScroll
                    if (dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset)) {
                        mLastMotionY -= mScrollOffset[1];
                        vtev.offsetLocation(0, mScrollOffset[1]);
                        mNestedYOffset += mScrollOffset[1];
                    } else if (canOverscroll) {
                        //如果canOverscroll，即可以越过边缘滑动。
                        final int pulledToY = oldY + deltaY;
                        //初始化边缘阴影
                        if (pulledToY < 0) {
                            mEdgeGlowTop.onPull((float) deltaY / getHeight(),
                                    ev.getX(activePointerIndex) / getWidth());
                            if (!mEdgeGlowBottom.isFinished()) {
                                mEdgeGlowBottom.onRelease();
                            }
                        } else if (pulledToY > range) {
                            mEdgeGlowBottom.onPull((float) deltaY / getHeight(),
                                    1.f - ev.getX(activePointerIndex) / getWidth());
                            if (!mEdgeGlowTop.isFinished()) {
                                mEdgeGlowTop.onRelease();
                            }
                        }
                        if (mEdgeGlowTop != null
                                && (!mEdgeGlowTop.isFinished() || !mEdgeGlowBottom.isFinished())) {
                            postInvalidateOnAnimation();
                        }
                    }
                }
                break;
        }
        ......
    }

```
`ACTION_MOVE`事件中，首先计算当前的垂直偏移量`deltaY`。然后判断是否大于最小滑动距离，并且给`mIsBeingDragged`赋值。接着如果`mIsBeingDragged`为`true`。就取得处理滑动需要的各种参数，并调用`overScrollBy()`方法来处理触摸事件，`overScrollBy()`是在`View`里实现的方法，大致实现如下：

```java
    protected boolean overScrollBy(int deltaX, int deltaY,
            int scrollX, int scrollY,
            int scrollRangeX, int scrollRangeY,
            int maxOverScrollX, int maxOverScrollY,
            boolean isTouchEvent) {
        .....

        int newScrollX = scrollX + deltaX;
        if (!overScrollHorizontal) {
            maxOverScrollX = 0;
        }

        int newScrollY = scrollY + deltaY;
        if (!overScrollVertical) {
            maxOverScrollY = 0;
        }

        // Clamp values if at the limits and record
        final int left = -maxOverScrollX;
        final int right = maxOverScrollX + scrollRangeX;
        final int top = -maxOverScrollY;
        final int bottom = maxOverScrollY + scrollRangeY;

        boolean clampedX = false;
        if (newScrollX > right) {
            newScrollX = right;
            clampedX = true;
        } else if (newScrollX < left) {
            newScrollX = left;
            clampedX = true;
        }

        boolean clampedY = false;
        if (newScrollY > bottom) {
            newScrollY = bottom;
            clampedY = true;
        } else if (newScrollY < top) {
            newScrollY = top;
            clampedY = true;
        }

        onOverScrolled(newScrollX, newScrollY, clampedX, clampedY);

        return clampedX || clampedY;
    }

```
方法的参数很清楚了应该不难理解，可以看到在`overScrollBy()`方法中根据我们传入的参数以及`View`本身是否可以滑动的设定，等等来最终决定了新的`newScrollX`与`newScrollY`。接着调用了`onOverScrolled()`方法来处理滑动，`onOverScrolled()`方法在`View`中是空实现，所以再回到`ScrollView`中可以看到重写了`onOverScrolled()`方法：

```java
    @Override
    protected void onOverScrolled(int scrollX, int scrollY,
            boolean clampedX, boolean clampedY) {
        // Treat animating scrolls differently; see #computeScroll() for why.
        if (!mScroller.isFinished()) {
            final int oldX = mScrollX;
            final int oldY = mScrollY;
            mScrollX = scrollX;
            mScrollY = scrollY;
            invalidateParentIfNeeded();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (clampedY) {
                mScroller.springBack(mScrollX, mScrollY, 0, 0, 0, getScrollRange());
            }
        } else {
            super.scrollTo(scrollX, scrollY);
        }

        awakenScrollBars();
    }
```
首先是有一句注释是说：不同的对待滑动动画，查看`computeScroll()`方法看原因。说到滑动动画一定是和`Scroller`相关了，目前我们还没涉及到，下面我们再谈。回到这里看到根据`!mScroller.isFinished()`来判断，根据前面的判断得知，要么是滑动动画并不存在，要么就已经被终止，所以在这里`!mScroller.isFinished()`为`false`。所以会调用`super.scrollTo(scrollX, scrollY);`最终产生滑动。到这里手指触摸产生的滑动就分析完了。

- ACTION_UP:
  `ACTION_UP`是当我们手指离开时产生的事件，在`ScrollView`中当我们手指离开时，会根据当前的加速度再滑动一段距离。具体的实现我们来看看是如何实现的：

```java

 @Override
    public boolean onTouchEvent(MotionEvent ev) {
        ......
        switch (actionMasked) {
            ......
            case MotionEvent.ACTION_UP:
                //如果实在drag状态中
                if (mIsBeingDragged) {
                    //计算加速度
                    final VelocityTracker velocityTracker = mVelocityTracker;
                    velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
                    int initialVelocity = (int) velocityTracker.getYVelocity(mActivePointerId);
                    //如果有有效的加速度
                    if ((Math.abs(initialVelocity) > mMinimumVelocity)) {
                        //处理带有加速度的滑动事件
                        flingWithNestedDispatch(-initialVelocity);
                    } else if (mScroller.springBack(mScrollX, mScrollY, 0, 0, 0,
                            getScrollRange())) {
                        postInvalidateOnAnimation();
                    }

                    mActivePointerId = INVALID_POINTER;
                    //清除drag状态
                    endDrag();
                }
                break;
        }
        ......
    }

```
可以看到代码并不复杂，在计算了加速度后，调用了`flingWithNestedDispatch(-initialVelocity);`：
​    
```java
    private void flingWithNestedDispatch(int velocityY) {
        final boolean canFling = (mScrollY > 0 || velocityY > 0) &&
                (mScrollY < getScrollRange() || velocityY < 0);
        if (!dispatchNestedPreFling(0, velocityY)) {
            dispatchNestedFling(0, velocityY, canFling);
            if (canFling) {
                fling(velocityY);
            }
        }
    }
```
代码如上，我们这里不考虑`NestedFling`的方式，所以`dispatchNestedPreFling(0, velocityY)`默认会返回`false`，所以最终会执行`fling(velocityY);`：

```java

    public void fling(int velocityY) {
        if (getChildCount() > 0) {
            int height = getHeight() - mPaddingBottom - mPaddingTop;
            int bottom = getChildAt(0).getHeight();

            mScroller.fling(mScrollX, mScrollY, 0, velocityY, 0, 0, 0,
                    Math.max(0, bottom - height), 0, height/2);

            if (mFlingStrictSpan == null) {
                mFlingStrictSpan = StrictMode.enterCriticalSpan("ScrollView-fling");
            }

            postInvalidateOnAnimation();
        }
    }

```
可以看到是调用了`mScroller`的`fling`方法，在上一篇[Scroller源码分析](http://www.jianshu.com/p/d74f6badb164)中，我们已经详细解释了`Scroller`的原理，`ScrollView`中虽然使用的是`OverScroller`但是使用方法也是类似的。所以在调用了`mScroller`的`fling`方法后。我们需要在`computeScroll()`处理`mScroller`计算出的值。`ScrollView`中的`computeScroll()`方法实现如下：

```java

    @Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
            int oldX = mScrollX;
            int oldY = mScrollY;
            int x = mScroller.getCurrX();
            int y = mScroller.getCurrY();

            if (oldX != x || oldY != y) {
                final int range = getScrollRange();
                final int overscrollMode = getOverScrollMode();
                final boolean canOverscroll = overscrollMode == OVER_SCROLL_ALWAYS ||
                        (overscrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);

                overScrollBy(x - oldX, y - oldY, oldX, oldY, 0, range,
                        0, mOverflingDistance, false);
                onScrollChanged(mScrollX, mScrollY, oldX, oldY);

                if (canOverscroll) {
                    if (y < 0 && oldY >= 0) {
                        mEdgeGlowTop.onAbsorb((int) mScroller.getCurrVelocity());
                    } else if (y > range && oldY <= range) {
                        mEdgeGlowBottom.onAbsorb((int) mScroller.getCurrVelocity());
                    }
                }
            }

            if (!awakenScrollBars()) {
                // Keep on drawing until the animation has finished.
                postInvalidateOnAnimation();
            }
        } else {
            if (mFlingStrictSpan != null) {
                mFlingStrictSpan.finish();
                mFlingStrictSpan = null;
            }
        }
    }

```
我省略了一些注释，意思是说：`computeScroll()`会在绘制的过程中调用，为了不重复的显示滚动条。这里重复做了`scrollTo()`方法中的代码。但并没有调用`scrollTo()`，因为`scrollTo()`中也有滚动条相关的处理。所以`computeScroll()`中也调用了`overScrollBy()`方法处理滑动。所以最终仍然会调用`onOverScrolled()`方法：
```java
    @Override
    protected void onOverScrolled(int scrollX, int scrollY,
            boolean clampedX, boolean clampedY) {
        // Treat animating scrolls differently; see #computeScroll() for why.
        if (!mScroller.isFinished()) {
            final int oldX = mScrollX;
            final int oldY = mScrollY;
            mScrollX = scrollX;
            mScrollY = scrollY;
            invalidateParentIfNeeded();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (clampedY) {
                mScroller.springBack(mScrollX, mScrollY, 0, 0, 0, getScrollRange());
            }
        } else {
            super.scrollTo(scrollX, scrollY);
        }

        awakenScrollBars();
    }
```

这次就会进入到第一个`if`语句里，可以看到是给`mScrollX`与`mScrollY`赋值后调用了`invalidateParentIfNeeded();`方法来完成最终的滑动处理。

- ACTION_POINTER_DOWN:

```java

 @Override
    public boolean onTouchEvent(MotionEvent ev) {
        ......
        switch (actionMasked) {
            ......
            case MotionEvent.ACTION_POINTER_DOWN: {
                //更新状态，即新的触摸手势决定是否滑动。
                final int index = ev.getActionIndex();
                mLastMotionY = (int) ev.getY(index);
                mActivePointerId = ev.getPointerId(index);
                break;
            }
        }
        ......
    }
```
`ACTION_POINTER_DOWN`是指有另外一个手指发生了触摸。这里的处理是将`mActivePointerId`赋值给新的点了。所以在`ScrollView`中当有一个手指按下，我们再按下另一个手指时，第二个按下的手指能决定`ScrollView`的滑动。
- ACTION_POINTER_UP:

```java

 @Override
    public boolean onTouchEvent(MotionEvent ev) {
        ......
        switch (actionMasked) {
            ......
            case MotionEvent.ACTION_POINTER_UP:
                onSecondaryPointerUp(ev);
                mLastMotionY = (int) ev.getY(ev.findPointerIndex(mActivePointerId));
                break;
        }
        ......
    }

```
`ACTION_POINTER_UP`是指多个手指中的一个手指离开屏幕。所以这里会检测是否是当前`active`的手指离开了，并做相应的处理，具体逻辑在`onSecondaryPointerUp(ev);`方法中，我们就不多解释了。至此整个`ScrollView`我们应该有了一个清晰完整的理解了。最后再分享一个小`trick`。一行代码实现仿`ios`的弹性`ScrollView`。

### 3.一行代码实现弹性ScrollView

我们都知道`ios`上的弹性滑动做的相当顺滑。我们`Android`系统一直都没有。看完`ScrollView`的代码发现，其实通过变量`mOverflingDistance`就能决定弹性滑动的最大值。但是`ScrollView`并没有暴露出方法给我们设置。但是我们只需要通过反射来设定`mOverflingDistance`的值即可。。[JOOR]( https://github.com/jOOQ/jOOR)对反射做了封装，可以使我们非常简洁的来写反射，所以这里我们只需要一行代码即可：

```java
Reflect.on(scrollView).set("mOverflingDistance", 100);
```
这样就实现了弹性`ScrollView`。以上`demo`的代码在[SkyScrollViewDemo](https://github.com/Skykai521/SkyScrollViewDemo)。