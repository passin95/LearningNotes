
<!-- TOC -->

- [一、自定义 ItemDecoration](#%E4%B8%80%E8%87%AA%E5%AE%9A%E4%B9%89-itemdecoration)
- [二、自定义 LayoutManager](#%E4%BA%8C%E8%87%AA%E5%AE%9A%E4%B9%89-layoutmanager)
  - [2.1 常用方法](#21-%E5%B8%B8%E7%94%A8%E6%96%B9%E6%B3%95)
    - [2.1.1 测量](#211-%E6%B5%8B%E9%87%8F)
    - [2.1.2 布局](#212-%E5%B8%83%E5%B1%80)
    - [2.1.3 回收复用](#213-%E5%9B%9E%E6%94%B6%E5%A4%8D%E7%94%A8)
      - [2.1.3.1 detachAndScrapAttachedViews()](#2131-detachandscrapattachedviews)
      - [2.1.3.2 removeAndRecycleView()](#2132-removeandrecycleview)
      - [2.1.3.3 getViewForPosition()](#2133-getviewforposition)
      - [2.1.3.4 缓存池小结](#2134-%E7%BC%93%E5%AD%98%E6%B1%A0%E5%B0%8F%E7%BB%93)
  - [2.2 自定义 LayoutManager 流程](#22-%E8%87%AA%E5%AE%9A%E4%B9%89-layoutmanager-%E6%B5%81%E7%A8%8B)
  - [2.3 回收复用的实现思路](#23-%E5%9B%9E%E6%94%B6%E5%A4%8D%E7%94%A8%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%80%9D%E8%B7%AF)
  - [2.4 技巧](#24-%E6%8A%80%E5%B7%A7)
    - [2.4.1 getChildDrawingOrder()](#241-getchilddrawingorder)
    - [2.4.2 滑动时回收](#242-%E6%BB%91%E5%8A%A8%E6%97%B6%E5%9B%9E%E6%94%B6)
- [三、RecyclerView 性能优化](#%E4%B8%89recyclerview-%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96)
  - [3.1 RecyclerView.setHasFixdSize()](#31-recyclerviewsethasfixdsize)
  - [3.2 RecyclerView.setRecycledViewPool()](#32-recyclerviewsetrecycledviewpool)
  - [3.3 DiffUtil](#33-diffutil)

<!-- /TOC -->

# 一、自定义 ItemDecoration

ItemDecoration 用于给 Item 添加装饰，从本质上讲，就是在绘制 Item 的前后允许我们插入自己的绘制需求（onDraw 打头的方法）和添加 Item 的间隔（getItemOffsets）。

```java
public abstract static class ItemDecoration {
    public ItemDecoration() {
    }

    /**
     * @param c：c 为画布，一般情况下只在 getItemOffsets 所撑出来的区域任意绘图，否则可能会被 Item 所覆盖（即该方法先于 Item 的绘制）
     * @param parent：设置当前装饰的 RecyclerView 对象。
     * @param state：设置当前装饰的 RecyclerView 对象，用在 LayoutManager、 Adapter 等组件之间共享 RecyclerView 状态的。
     */
    public void onDraw(@NonNull Canvas c, @NonNull RecyclerView parent, @NonNull RecyclerView.State state) {
        this.onDraw(c, parent); // 空实现
        
        RecyclerView.LayoutManager manager = parent.getLayoutManager();

        // 通过遍历拿到子 View 从而拿到 View 的属性。
        int childCount = parent.getChildCount();
        for (int i = 0; i < childCount; i++) {
            View child = parent.getChildAt(i);
            // layoutParams
            RecyclerView.LayoutParams layoutParams = ((RecyclerView.LayoutParams) child.getLayoutParams());
            // 拿到 getItemOffsets() 参数 outRect 的 left、top、right、bottom。
            int left = manager.getLeftDecorationWidth(child);
            int top = manager.getTopDecorationHeight(child);
            int right = manager.getRightDecorationWidth(child);
            int bottom = manager.getBottomDecorationHeight(child);
        }
    }

    /**
     * 该方法与 onDraw() 唯一不同在于该方法执行于 Item 绘制完成之后。
     */
    public void onDrawOver(@NonNull Canvas c, @NonNull RecyclerView parent, @NonNull RecyclerView.State state) {
        this.onDrawOver(c, parent);
    }

    /**
     * @param outRect：item 的上下左右所撑开的距离，默认情况下都为 0，本质上是控制 View 的宽高.在 RecyclerView.onMesure() 时便将其加入了测量。
     * @param view：当前 Item 的视图对象。
     * @param parent：设置当前装饰的 RecyclerView 对象。
     * @param state：RecyclerView 的绘制状态，用在 LayoutManager、Adapter 等组件之间共享 RecyclerView 状态的。
     */
    public void getItemOffsets(@NonNull Rect outRect, @NonNull View view, @NonNull RecyclerView parent, @NonNull RecyclerView.State state) {
        this.getItemOffsets(outRect, ((RecyclerView.LayoutParams)view.getLayoutParams()).getViewLayoutPosition(), parent);
        // 可以通过 ((RecyclerView.LayoutParams)view.getLayoutParams()).getViewLayoutPosition() 拿到 view 在数据源中的位置。
    }

}
```

# 二、自定义 LayoutManager

## 2.1 常用方法

在测量和布局上，LayoutManager 和 ViewGroup 的方法基本上都是一一对应的，不同在于 LayoutManager 需要考虑 ItemDecoration 的偏移。

### 2.1.1 测量

在自定义 ViewGroup 的时候，子 View 在 onMeasure() 中统一测量的，而在自定义 LayoutManager 中子 View 是在需要 layout 之前才测量。

与 ViewGroup 测量方法不同之处在于会将 ItemDecoration 中的偏移加入一同计算。

```java
public void measureChild(@NonNull View child, int widthUsed, int heightUsed) {
    RecyclerView.LayoutParams lp = (RecyclerView.LayoutParams)child.getLayoutParams();
    Rect insets = this.mRecyclerView.getItemDecorInsetsForChild(child);
    widthUsed += insets.left + insets.right;
    heightUsed += insets.top + insets.bottom;
    int widthSpec = getChildMeasureSpec(this.getWidth(), this.getWidthMode(), this.getPaddingLeft() + this.getPaddingRight() + widthUsed, lp.width, this.canScrollHorizontally());
    int heightSpec = getChildMeasureSpec(this.getHeight(), this.getHeightMode(), this.getPaddingTop() + this.getPaddingBottom() + heightUsed, lp.height, this.canScrollVertically());
    if (this.shouldMeasureChild(child, widthSpec, heightSpec, lp)) {
        child.measure(widthSpec, heightSpec);
    }
}

public void measureChildWithMargins(@NonNull View child, int widthUsed, int heightUsed) {
    RecyclerView.LayoutParams lp = (RecyclerView.LayoutParams)child.getLayoutParams();
    Rect insets = this.mRecyclerView.getItemDecorInsetsForChild(child);
    widthUsed += insets.left + insets.right;
    heightUsed += insets.top + insets.bottom;
    // 与 measureChild 不同在于会加入 LayoutParams 中的 margin 值一同计算。
    int widthSpec = getChildMeasureSpec(this.getWidth(), this.getWidthMode(), this.getPaddingLeft() + this.getPaddingRight() + lp.leftMargin + lp.rightMargin + widthUsed, lp.width, this.canScrollHorizontally());
    int heightSpec = getChildMeasureSpec(this.getHeight(), this.getHeightMode(), this.getPaddingTop() + this.getPaddingBottom() + lp.topMargin + lp.bottomMargin + heightUsed, lp.height, this.canScrollVertically());
    if (this.shouldMeasureChild(child, widthSpec, heightSpec, lp)) {
        child.measure(widthSpec, heightSpec);
    }
}
```

在 ViewGroup 中测量完毕后就可以通过 View.getMeasureWidth() or View.getMeasureHeight() 拿到测量的宽高，但在 LayoutManager 中，应该通过 LayoutManager.getDecoratedMeasuredWidth(View child) 拿到测量结果，它会将 ItemDecoration 的偏移加入一起计算，从而适配 ItemDecoration.getItemOffsets()。

```java
public int getDecoratedMeasuredWidth(@NonNull View child) {
    final Rect insets = ((LayoutParams) child.getLayoutParams()).mDecorInsets;
    return child.getMeasuredWidth() + insets.left + insets.right;
}

public int getDecoratedMeasuredHeight(@NonNull View child) {
    final Rect insets = ((LayoutParams) child.getLayoutParams()).mDecorInsets;
    return child.getMeasuredHeight() + insets.top + insets.bottom;
}
```

### 2.1.2 布局

在自定义 ViewGroup 的时候，我们会重写 onLayout()，并在里面去遍历子 View，然后调用子 View 的 layout 方法来进行布局，但在 LayoutManager 里对 Item 进行布局时，不推荐直接使用 layout()，而是使用以下 2 个方法，同样是为了适配 ItemDecoration 的偏移。

```java
public void layoutDecorated(@NonNull View child, int left, int top, int right, int bottom) {
        final Rect insets = ((LayoutParams) child.getLayoutParams()).mDecorInsets;
        child.layout(left + insets.left, top + insets.top, right - insets.right,
                bottom - insets.bottom);
    }

public void layoutDecoratedWithMargins(@NonNull View child, int left, int top, int right,
        int bottom) {
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
    final Rect insets = lp.mDecorInsets;
    child.layout(left + insets.left + lp.leftMargin, top + insets.top + lp.topMargin,
            right - insets.right - lp.rightMargin,
            bottom - insets.bottom - lp.bottomMargin);
}
```

### 2.1.3 回收复用

#### 2.1.3.1 detachAndScrapAttachedViews()

detachAndScrapXXX() 用于分离子视图，并将其添加到 Recycler 缓存堆中。一般只用于 onLayoutChildren() 中。

```java
public void detachAndScrapView(View child, Recycler recycler) {
    int index = mChildHelper.indexOfChild(child);
    scrapOrRecycleView(recycler, index, child);
}

public void detachAndScrapViewAt(int index, Recycler recycler) {
    final View child = getChildAt(index);
    scrapOrRecycleView(recycler, index, child);
}

public void detachAndScrapAttachedViews(Recycler recycler) {
    final int childCount = getChildCount();
    for (int i = childCount - 1; i >= 0; i--) {
        final View v = getChildAt(i);
        scrapOrRecycleView(recycler, i, v);
    }
}

private void scrapOrRecycleView(Recycler recycler, int index, View view) {
    final ViewHolder viewHolder = getChildViewHolderInt(view);
    // shouldIgnore() 为 true 表示此 ViewHolder 由 LayoutManager 完全管理，除非更换了 LayoutManager，否则不会报废、回收或删除它。
    if (viewHolder.shouldIgnore()) {
        if (DEBUG) {
            Log.d(TAG, "ignoring view " + viewHolder);
        }
        return;
    }
    
    if (viewHolder.isInvalid() && !viewHolder.isRemoved()
            && !mRecyclerView.mAdapter.hasStableIds()) {
        // 如果 viewholder 被标记为无效，并且还没有被移除，并且没有设置 StableId，就将其从界面上移除并回收到 mRecyclerPool 中。
        // 这也说明若 Adapter 开启了 StableId，在调用 detachAndScrapXXX() 时，ViewHolder 不会被放到 mRecyclerPool.mScrap 中。
        removeViewAt(index);
        recycler.recycleViewHolderInternal(viewHolder);
    } else {
        // 将 View 从 RecyclerView 上暂时分离，然后放进临时缓存 (mAttachedScrap 或 mChangedScrap)，以便稍后直接重用。
        detachViewAt(index);
        recycler.scrapView(view);
        mRecyclerView.mViewInfoStore.onViewDetached(viewHolder);
    }
}
```

#### 2.1.3.2 removeAndRecycleView()

从 RecyclerView 移除子视图并对其进行缓存或回收。

```java
public void removeAndRecycleView(@NonNull View child, @NonNull Recycler recycler) {
    removeView(child);
    recycler.recycleView(child);
}

public void removeAndRecycleViewAt(int index, @NonNull Recycler recycler) {
    final View view = getChildAt(index);
    removeViewAt(index);
    recycler.recycleView(view);
}

public void removeAndRecycleAllViews(@NonNull Recycler recycler) {
    for (int i = getChildCount() - 1; i >= 0; i--) {
        final View view = getChildAt(i);
        if (!getChildViewHolderInt(view).shouldIgnore()) {
            removeAndRecycleViewAt(i, recycler);
        }
    }
}
```

```java
public void recycleView(@NonNull View view) {
  
    ViewHolder holder = getChildViewHolderInt(view);
    if (holder.isTmpDetached()) {
        // 移除与 RecyclerView 暂时分离状态的视图。
        removeDetachedView(view, false);
    }
    if (holder.isScrap()) {
        // 如果在某种缓存中，应该从当前所处缓存中移除，因为它将要放入 mRecyclerPool 中。
        holder.unScrap();
    } else if (holder.wasReturnedFromScrap()) {
        holder.clearReturnedFromScrapFlag();
    }
    recycleViewHolderInternal(holder);
}

void recycleViewHolderInternal(ViewHolder holder) {
    // 省略部分代码
    boolean cached = false;
    boolean recycled = false;

    if (forceRecycle || holder.isRecyclable()) {
        // 满足条件，则先存到 mCachedViews 中。
        if (mViewCacheMax > 0
                && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                | ViewHolder.FLAG_REMOVED
                | ViewHolder.FLAG_UPDATE
                | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
            int cachedViewSize = mCachedViews.size();
            if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                // 若 mCachedViews 缓存满后，移除 mCachedViews 中最先被添加的 item，并将移除的 Item 放入回收池 mRecyclerPool 中。
                recycleCachedViewAt(0);
                cachedViewSize--;
            }

            int targetCacheIndex = cachedViewSize;
            if (ALLOW_THREAD_GAP_WORK
                    && cachedViewSize > 0
                    && !mPrefetchRegistry.lastPrefetchIncludedPosition(holder.mPosition)) {
                // when adding the view, skip past most recently prefetched views
                int cacheIndex = cachedViewSize - 1;
                while (cacheIndex >= 0) {
                    int cachedPos = mCachedViews.get(cacheIndex).mPosition;
                    if (!mPrefetchRegistry.lastPrefetchIncludedPosition(cachedPos)) {
                        break;
                    }
                    cacheIndex--;
                }
                targetCacheIndex = cacheIndex + 1;
            }
            // 将 RecyclerView 最新被移除的 Item 添加到数组最后。
            mCachedViews.add(targetCacheIndex, holder);
            cached = true;
        }
        if (!cached) {
            // 没有被缓存到 mCachedViews，则回收到 mRecyclerPool。
            addViewHolderToRecycledViewPool(holder, true);
            recycled = true;
        }
    }
    // 省略部分代码
 
}

```

#### 2.1.3.3 getViewForPosition()

```java
public View getViewForPosition(int position) {
    return getViewForPosition(position, false);
}

View getViewForPosition(int position, boolean dryRun) {
    return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
}

@Nullable
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
        boolean dryRun, long deadlineNs) {
    if (position < 0 || position >= mState.getItemCount()) {
        throw new IndexOutOfBoundsException("Invalid item position " + position
                + "(" + position + "). Item count:" + mState.getItemCount()
                + exceptionLabel());
    }
    boolean fromScrapOrHiddenOrCache = false;
    ViewHolder holder = null;
    // 1. 如果处于预布局状态，则尝试从缓存 mChangedScrap 中依次对比 ViewHolder.getLayoutPosition()、 StableId 复用 ViewHolder,
    // mChangedScrap 也只在预布局使用。
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
        fromScrapOrHiddenOrCache = holder != null;
    }
    if (holder == null) {
        // 2. 依次尝试从 mAttachedScrap、mCachedView 中通过对比 ViewHolder.getLayoutPosition() 复用 ViewHolder。
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
        // 忽略部分代码
    }
    if (holder == null) {
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        if (offsetPosition < 0 || offsetPosition >= mAdapter.getItemCount()) {
            throw new IndexOutOfBoundsException("Inconsistency detected. Invalid item "
                    + "position " + position + "(offset:" + offsetPosition + ")."
                    + "state:" + mState.getItemCount() + exceptionLabel());
        }

        final int type = mAdapter.getItemViewType(offsetPosition);
        // 3. 依次尝试从 mAttachedScrap、mCachedView 中通过对比 StableId 复用 ViewHolder。
        if (mAdapter.hasStableIds()) {
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                    type, dryRun);
            if (holder != null) {
                // update position
                holder.mPosition = offsetPosition;
                fromScrapOrHiddenOrCache = true;
            }
        }
        if (holder == null && mViewCacheExtension != null) {
            // 4. 尝试从开发者自定义缓存 mViewCacheExtension 中复用 ViewHolder。
            final View view = mViewCacheExtension
                    .getViewForPositionAndType(this, position, type);
            if (view != null) {
                holder = getChildViewHolder(view);
                if (holder == null) {
                    throw new IllegalArgumentException("getViewForPositionAndType returned"
                            + " a view which does not have a ViewHolder"
                            + exceptionLabel());
                } else if (holder.shouldIgnore()) {
                    throw new IllegalArgumentException("getViewForPositionAndType returned"
                            + " a view that is ignored. You must call stopIgnoring before"
                            + " returning this view." + exceptionLabel());
                }
            }
        }
        if (holder == null) { // fallback to pool
            if (DEBUG) {
                Log.d(TAG, "tryGetViewHolderForPositionByDeadline("
                        + position + ") fetching from shared pool");
            }
            // 5. 最后尝试从缓存池 mRecyclerPool 中复用 ViewHolder，这也是我们最常见的缓存方式。
            holder = getRecycledViewPool().getRecycledView(type);
            if (holder != null) {
                holder.resetInternal();
                if (FORCE_INVALIDATE_DISPLAY_LIST) {
                    invalidateDisplayListInt(holder);
                }
            }
        }
        if (holder == null) {
            long start = getNanoTime();
            if (deadlineNs != FOREVER_NS
                    && !mRecyclerPool.willCreateInTime(type, start, deadlineNs)) {
                // abort - we have a deadline we can't meet
                return null;
            }
            // 6. 如果没有 ViewHolder 可以从缓存中复用，则调用 Adapter.createViewHolder() 创建一个 ViewHolder。
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
            // 忽略部分代码
        }
    }

    // 忽略部分代码

    boolean bound = false;
    if (mState.isPreLayout() && holder.isBound()) {
        // do not update unless we absolutely have to.
        holder.mPreLayoutPosition = position;
    } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
        if (DEBUG && holder.isRemoved()) {
            throw new IllegalStateException("Removed holder should be bound and it should"
                    + " come here only in pre-layout. Holder: " + holder
                    + exceptionLabel());
        }
        // 若 ViewHolder 没有被绑定或需要更新或视图被标记无效，则尝试重新绑定 viewHolder，
        // 对于开发者来说，则是回调 Adapter.bindViewHolder()。
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
    }

    // 对 ViewHolder 中的 itemView 的 LayoutParams 进行修正。
    final ViewGroup.LayoutParams lp = holder.itemView.getLayoutParams();
    final LayoutParams rvLayoutParams;
    if (lp == null) {
        rvLayoutParams = (LayoutParams) generateDefaultLayoutParams();
        holder.itemView.setLayoutParams(rvLayoutParams);
    } else if (!checkLayoutParams(lp)) {
        rvLayoutParams = (LayoutParams) generateLayoutParams(lp);
        holder.itemView.setLayoutParams(rvLayoutParams);
    } else {
        rvLayoutParams = (LayoutParams) lp;
    }
    rvLayoutParams.mViewHolder = holder;
    rvLayoutParams.mPendingInvalidate = fromScrapOrHiddenOrCache && bound;
    return holder;
}
```

#### 2.1.3.4 缓存池小结

RecyclerView 一共有 5 种视图缓存池：

**（1）mAttachedScrap 和 mChangedScrap**

mAttachedScrap 和 mChangedScrap 皆不参与滑动过程中的回收复用，主要在布局时（onLayoutChildren()）进行回收复用，它们不同在于 mChangedScrap 只在预布局状态使用。

通过对比 mAttachedScrap 中的 StableId 复用的 ViewHolder 会回调 onBindViewHolder()，而通过对比 getLayoutPosition() 复用的 ViewHolder 不会回调 onBindViewHolder()。

**（2）mCachedView**

- mCachedViews：保存最新被移除的 ViewHolder（调用 recycleViewHolderInternal(ViewHolder holder))，它的作用是在需要新的 ViewHolder 时，精确匹配是不是 **刚移除** 的那个。如果是，就直接返回给 RecyclerView 进行展示；如果不是，那么即使 mCachedViews 中有 ViewHolder，也不会返回给 RecyclerView 使用。对开发者来说最直接的区别就是从 mCachedView 拿到的 ViewHolder 并不会调用 onBindViewHolder()。

mCachedViews 的默认大小为 2，当 mCachedViews 满了以后，会利用先进先出原则，将出的 ViewHoloder 存放在 mRecyclerPool 中。

**（3）mViewCacheExtension**

RecyclerView 允许我们自己扩展回收池，我们可以通过调用 setViewCacheExtension() 去自定义缓存。一般情况下使用系统自带的缓存即可，它的缓存优先级在比 mCachedView 低，比 mRecyclerPool 高。

**（4）mRecyclerPool**

在 mCachedViews 中复用的 ViewHolder 都是精确匹配的，真正废弃的是存放在 mRecyclerPool 中的 ViewHolder。从 mRecyclerPool 拿到的 ViewHolder 会回调 onBindViewHolder() 重新绑定数据。

## 2.2 自定义 LayoutManager 流程

自定义 LayoutManager 和自定义 ViewGroup 是流程是相似的，只不过多了回收复用以及滑动的处理。一般情况下流程如下：

1. 进行布局之前，调用 detachAndScrapAttachedViews() 将当前屏幕上所有的 ViewHolder 与屏幕分离（因为不一定是首次布局）；
2. 遍历 getItemCount()，调用 recycler.getViewForPosition(position) 拿到 view 后通过 addView() 添加到 RecyclerView 中。此处可以进行性能优化，通过计算只添加首屏会出现的 Item，然后在滑动的过程中进行复用。而不是把所有 Item 都变为一个子 View，否则数据量大时可能会直接 OOM；
3. 调用 measureChild() 或 measureChildWithMargins() 对子 View 进行测量。也可以自己手写限制的测量规格 MeasureSpec，再调用 child.measure(widthSpec, heightSpec)；
4. 调用 layoutDecorated() 或 layoutDecoratedWithMargins() 对子 View 进行布局；
5. RecyclerView 滑动过程的回收复用。

## 2.3 回收复用的实现思路

以竖直线性布局为例，简述回收复用的主要思路：

**（1）在 onLayoutChildren 初始布局时**
 
1. 使用 **detachAndScrapAttachedViews(recycler)** 将所有的可见 ViewHolder 剥离。
2. 只初始化撑满一屏的 Item 数即可，不要多创建。

**（2）在 scrollVerticallyBy 滑动时**

都得控制一个变量作为累计偏移量。

方式一：

1. 先判断在滚动 dy 距离后，哪些 ViewHolder 需要回收，需要回收的 ViewHolder 就调用 removeAndRecycleView(child, recycler) 先将它回收。
2. 调用 recycler.getViewForPosition(position) 获取 view 来填充滚动出来的空白区域。
3. 调用 offsetChildrenVertical() 对所有子 View 进行偏移。

方式二：

1. 先判断在滚动 dy 距离后，哪些 ViewHolder 需要回收，需要回收的 ViewHolder 就调用 removeAndRecycleView(child, recycler) 先将它回收。
2. 调用 detachAndScrapAttachedViews(recycler)，暂时分离所有子 View。
2. 重新遍历所有 Item（根据需求可优化遍历的数量），添加撑满一屏的 Item 数即可，不要多创建。

## 2.4 技巧

### 2.4.1 getChildDrawingOrder()

重写 getChildDrawingOrder() 可改变子 View 的绘制顺序。

### 2.4.2 滑动时回收

在滑动过程中，可以把 mAttachedScrap 中的缓存全部放进 mRecyclerPool 中，mAttachedScrap 中可重用的 ViewHolder 已经在 onLayoutChildren() 中复用。

```java
private void recycleChildren(RecyclerView.Recycler recycler) {
    List<RecyclerView.ViewHolder> scrapList = recycler.getScrapList();
    for (int i = 0; i < scrapList.size(); i++) {
        RecyclerView.ViewHolder holder = scrapList.get(i);
        removeAndRecycleView(holder.itemView, recycler);
    }
}

```

// 未完待续

# 三、RecyclerView 性能优化

## 3.1 RecyclerView.setHasFixdSize()

若 Adapter 的数据变化不会导致 RecyclerView 的大小变化，则将该方法设置为 true。它可以在 RecyclerView 内容发生变化时不需要调用 requestLayout()，而直接对子 View 进行 layout。

## 3.2 RecyclerView.setRecycledViewPool()

多个 RecyclerView 在 viewType 一样（布局文件也一致）的情况下可以共用同一个 RecycledViewPool，例如订单的不同状态使用了多个 RecyclerView。

## 3.3 DiffUtil

适用于整个页面需要刷新，但是部分数据可能相同的情况（是否相同的标准由开发者决定）。它可以比较出不同的数据源进行局部刷新（局部刷新可以是某个 ViewHolder 的全量刷新或是单单某个 View 的方法调用），达到最小刷新的效果。具体的使用方式可看
https://github.com/CymChad/BaseRecyclerViewAdapterHelper/blob/master/library/src/main/java/com/chad/library/adapter/base/BaseQuickAdapter.java。