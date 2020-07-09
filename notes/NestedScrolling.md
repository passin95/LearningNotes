

# 一、NestedScrolling

## 1.1 嵌套滑动方法

compileSdk Lollipop（21） 及以上版本的 View 都支持了 NestedScrolling（对应 NestedScrollingChild 和 NestedScrollingParent 的接口实现）。

其中 View 多了以下这些方法：

```java
/**
 * 启用或禁用此视图的嵌套滚动。
 *
 * @param enabled true 支持嵌套, 反之不支持。
 */
public void setNestedScrollingEnabled(boolean enabled);

/**
 * 此 View 是否支持嵌套滑动。
 *
 * @return true 表示此子 View 支持嵌套滑动。
 */
public boolean isNestedScrollingEnabled();

/**
 * 开始嵌套滑动。
 * 该方法是整个嵌套滑动的开始。
 *
 * @param axes 支持什么方向上的嵌套滑动。
 *             SCROLL_AXIS_NONE：无视轴滚动。
 *             SCROLL_AXIS_HORIZONTAL：沿水平轴滚动。
 *             SCROLL_AXIS_VERTICAL：沿垂直轴滚动。
 * @return true：找到嵌套滑动的父视图并且嵌套滚动成功开始。
 */
public boolean startNestedScroll(int axes);

/**
 * 停止正在进行的嵌套滚动。
 */
public void stopNestedScroll();

/**
 * 检查此视图是否具有嵌套滚动父视图。只有在嵌套滑动过程中才返回 true。
 */
public boolean hasNestedScrollingParent();

/**
 * 子 View 在滑动前都需要将滑动细节传递给嵌套滚动父 View，一般是在 ACTION_MOVE 中调用。
 *
 * @param dx 本次滑动 x 轴的距离。
 * @param dy 本次滑动 y 轴的距离。
 * @param consumed consumed 由子 View 创建并传递给父 View ，父 View 对 consumed 赋值表示父 View 消耗的滑动距离。其中 consumed[0] 代表 x 轴，consumed[1] 代表 y 轴。
 * @param offsetInWindow 子 View 创建给父 View 使用的数组,保存了子 View 滑动前后的坐标偏移量。
 * @return 如果父级使用了任何嵌套滚动，则为 true。
 */
public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow);

/**
 * 在 dispatchNestedPreScroll() 之后，view 将消费剩余的滑动距离，并告知父 View 消费了多少。
 *
 * @param dxConsumed 子 View x 轴消费的滑动距离。
 * @param dyConsumed 子 View y 轴消费的滑动距离。
 * @param dxUnconsumed x 轴未消费的滑动距离。
 * @param dyUnconsumed y 轴未消费的滑动距离。
 * @return 如果父级使用了任何嵌套滚动，则为 true。
 */
public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow);

/**
 * 如果当子 View ACTION_UP 时伴随着 fling 的产生,就需要子 View 在 stopNestedScroll 前调用。
 *
 * @param velocityX x 轴的滑动速度。
 * @param velocityY y 轴的滑动速度。
 * @return 如果父 View 消费了嵌套的 fling 操作，则为 true。若返回 true，则不再调 dispatchNestedFling()。
 */
public boolean dispatchNestedPreFling(float velocityX, float velocityY);

/**
 * @param dxConsumed x 轴的滑动速度。
 * @param dyConsumed y 轴的滑动速度。
 * @param consumed consumed 代表子 View 是否消费掉了 fling，fling 不存在部分消费，一旦被消费就是指全部。
 * @return 如果消费了嵌套的 fling 操作，则为 true。
 */ 
public boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed);
```

而 ViewGroup 多了以下方法，其中以 on 打头的方法皆是相应子 View 的 dispatchXXX() 中进行调用。

```java
/**
 * 有子 View 有嵌套滑动的需求时被调用以确认是否进行嵌套滑动。
 *
 * @param child child 是确定进行嵌套滑动视图的 子 View。一般是 target 本身或 target 的父 View。
 * @param target 需要联合嵌套滑动的子 View。
 * @param nestedScrollAxes 嵌套滑动的方向。
                           SCROLL_AXIS_NONE：无视轴滚动。 SCROLL_AXIS_HORIZONTAL：沿水平轴滚动。 SCROLL_AXIS_VERTICAL：沿垂直轴滚动。
 * @return 如果父 View 需要进行嵌套滑动，则返回 true。
 */ 
public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes);
public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes);
public void onStopNestedScroll(View target);
/**
 * 在 dispatchNestedScroll() 调用，dxConsumed、dyConsumed 表示子视图消费的滑动记录。
 */
public void onNestedScroll(View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed);
/**
 * 在 dispatchNestedPreScroll() 中调用，dx、dy 表示子 View 滑动的距离，consumed 用于保存嵌套滑动的父视图滑动的距离。
 */ 
public void onNestedPreScroll(View target, int dx, int dy, int[] consumed);
/**
 * 在 dispatchNestedFling() 中调用，consumed 用于保存是否已经消费过 fling。
 */ 
public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed);
public boolean onNestedPreFling(View target, float velocityX, float velocityY);
public int getNestedScrollAxes();
```

若想支持 NestedScrollingParent2 和 NestedScrollingChild2 新增的方法（主要是增加一个 type 以支持非手动触摸的情况），则可使用 NestedScrollingParentHelper 和 NestedScrollingChildHelper，这 2 个类包含了 Google 提供的默认嵌套实现以及版本兼容处理。

## 1.2 NestedScrolling 流程

我们先看一下 [传统的事件分发机制](./触摸反馈#%E4%BA%8C%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91%E6%9C%BA%E5%88%B6)，它是由 Activity 发起事件再传到父 View，一旦父 View 需要自己处理触摸事件就要拦截掉事件，则子 View 再没有机会接手此事件流。而 NestedScrolling 不一样，它是由子 View 发起的，一般的处理流程图如下所示：

 <img src="../pictures//NestedScrolling%20流程图.webp" /> </div>

下文以 View、ViewGroup 的默认实现源码去具体分析一般情况下嵌套滑动的流程：

**（1）** 当子 View 接受触摸事件后，一般在 ACTION_DOWN 事件中调 startNestedScroll()
开始确认嵌套滑动的父 View。

```java
public boolean startNestedScroll(int axes) {
    // 是否处于嵌套滑动的过程中
    if (hasNestedScrollingParent()) {
        // 处于嵌套滑动的过程，则直接返回 true。
        return true;
    }
    if (isNestedScrollingEnabled()) {
        // 该子 View 支持嵌套滑动。
        ViewParent p = getParent();
        View child = this;
        while (p != null) {
            try {
                // 调用父 View 的 onStartNestedScroll() 以确定联合嵌套滑动的父 View。
                if (p.onStartNestedScroll(child, this, axes)) {
                    // mNestedScrollingParent 也是 hasNestedScrollingParent() 方法的判断依据。
                    mNestedScrollingParent = p;
                    // 调用父 View 的 onNestedScrollAccepted()，表示已开启嵌套滑动。
                    p.onNestedScrollAccepted(child, this, axes);
                    return true;
                }
            } catch (AbstractMethodError e) {
                Log.e(VIEW_LOG_TAG, "ViewParent " + p + " does not implement interface " +
                        "method onStartNestedScroll", e);
            }
            // 循环不断获取更上一级的父 View，直到有父 View 接受嵌套滑动。
            if (p instanceof View) {
                child = (View) p;
            }
            p = p.getParent();
        }
    }
    return false;
}

public boolean hasNestedScrollingParent() {
    return mNestedScrollingParent != null;
}
```

ViewGroup 的默认实现：

```java
@Override
public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
    return false;
}

@Override
public void onNestedScrollAccepted(View child, View target, int axes) {
    mNestedScrollAxes = axes;
}
```

**（2）** 每次子 View 在滑动前都需要将滑动细节传递给父 View，一般情况下在 ACTION_MOVE 事件中调用 dispatchNestedPreScroll()。如果父 View 没有消费完所有距离，view 将自己接着消费并调用 dispatchNestedScroll() 告知父 View 消费了多少距离。

```java
public boolean dispatchNestedPreScroll(int dx, int dy, @Nullable int[] consumed, @Nullable int[] offsetInWindow) {
    if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
        if (dx != 0 || dy != 0) {
            int startX = 0;
            int startY = 0;
            // startX 和 startY 为当前 View 的滑动坐标。
            if (offsetInWindow != null) {
                getLocationInWindow(offsetInWindow);
                startX = offsetInWindow[0];
                startY = offsetInWindow[1];
            }

            if (consumed == null) {
                // mTempNestedScrollConsumed 用于内存优化，避免重复创建。
                if (mTempNestedScrollConsumed == null) {
                    mTempNestedScrollConsumed = new int[2];
                }
                consumed = mTempNestedScrollConsumed;
            }
            // 重置 consumed 数组，传给父 View 来获取父 View 消耗的滑动距离。
            consumed[0] = 0;
            consumed[1] = 0;
            // 调用父 View 的 onNestedPreScroll()。
            mNestedScrollingParent.onNestedPreScroll(this, dx, dy, consumed);

            if (offsetInWindow != null) {
                // 在父 View 消耗滑动距离后（例如滑动了），计算当前子 View 的滑动坐标。
                getLocationInWindow(offsetInWindow);
                offsetInWindow[0] -= startX;
                offsetInWindow[1] -= startY;
            }
            // 父 View 消费了滑动距离则返回 true。
            return consumed[0] != 0 || consumed[1] != 0;
        } else if (offsetInWindow != null) {
            offsetInWindow[0] = 0;
            offsetInWindow[1] = 0;
        }
    }
    return false;
}

public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, @Nullable @Size(2) int[] offsetInWindow) {
    if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
        if (dxConsumed != 0 || dyConsumed != 0 || dxUnconsumed != 0 || dyUnconsumed != 0) {
            int startX = 0;
            int startY = 0;
            if (offsetInWindow != null) {
                getLocationInWindow(offsetInWindow);
                startX = offsetInWindow[0];
                startY = offsetInWindow[1];
            }
            // 调用父 View 的 onNestedScroll()，将自己的滑动结果再次传递给父 View。
            mNestedScrollingParent.onNestedScroll(this, dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed);

            if (offsetInWindow != null) {
                getLocationInWindow(offsetInWindow);
                offsetInWindow[0] -= startX;
                offsetInWindow[1] -= startY;
            }
            return true;
        } else if (offsetInWindow != null) {
            // No motion, no dispatch. Keep offsetInWindow up to date.
            offsetInWindow[0] = 0;
            offsetInWindow[1] = 0;
        }
    }
    return false;
}
```

**（3）** 当触发 ACTION_UP 的时候，如果余下的速度达到 Fling 的标准，将调用 dispatchNestedPreFling() 让父 View 继续消费 velocity，如果返回 false（父 View 不消费），则继续调用 dispatchNestedFling()。在这些方法调用结束后最终都需要 stopNestedScroll() 来告知父 View 本次嵌套滑动结束。

```java
public boolean dispatchNestedPreFling(float velocityX, float velocityY) {
    if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
        // 回调父 View 的 onNestedPreFling()。
        return mNestedScrollingParent.onNestedPreFling(this, velocityX, velocityY);
    }
    return false;
}

public boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed) {
    if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
        // 回调父 View 的 onNestedFling()。
        return mNestedScrollingParent.onNestedFling(this, velocityX, velocityY, consumed);
    }
    return false;
}

public void stopNestedScroll() {
    if (mNestedScrollingParent != null) {
        // 回调父 View 的 onStopNestedScroll()。
        mNestedScrollingParent.onStopNestedScroll(this);
        mNestedScrollingParent = null;
    }
}
```