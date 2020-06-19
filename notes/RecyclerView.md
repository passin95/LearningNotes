
<!-- TOC -->

- [一、自定义 ItemDecoration](#%E4%B8%80%E8%87%AA%E5%AE%9A%E4%B9%89-itemdecoration)
- [二、自定义 LayoutManager](#%E4%BA%8C%E8%87%AA%E5%AE%9A%E4%B9%89-layoutmanager)
  - [2.1 测量](#21-%E6%B5%8B%E9%87%8F)
  - [2.2 布局](#22-%E5%B8%83%E5%B1%80)
  - [2.3 回收复用](#23-%E5%9B%9E%E6%94%B6%E5%A4%8D%E7%94%A8)
    - [2.3.1 缓存池](#231-%E7%BC%93%E5%AD%98%E6%B1%A0)
    - [2.3.2 detachAndScrapAttachedViews()](#232-detachandscrapattachedviews)
      - [2.3.2.1 scrapView()](#2321-scrapview)
    - [2.3.3 removeAndRecycleView()](#233-removeandrecycleview)
      - [2.3.3.1 recycleView()](#2331-recycleview)
    - [2.3.4 getViewForPosition()](#234-getviewforposition)
  - [2.4 自定义 LayoutManager 流程](#24-%E8%87%AA%E5%AE%9A%E4%B9%89-layoutmanager-%E6%B5%81%E7%A8%8B)
  - [2.5 回收复用的实现思路](#25-%E5%9B%9E%E6%94%B6%E5%A4%8D%E7%94%A8%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%80%9D%E8%B7%AF)
  - [2.6 技巧](#26-%E6%8A%80%E5%B7%A7)
    - [2.6.1 getChildDrawingOrder()](#261-getchilddrawingorder)
    - [2.6.2 滑动时回收](#262-%E6%BB%91%E5%8A%A8%E6%97%B6%E5%9B%9E%E6%94%B6)
- [三、RecyclerView 源码分析](#%E4%B8%89recyclerview-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)
  - [3.1 onMeasure()](#31-onmeasure)
    - [3.1.1 mLayout == null](#311-mlayout--null)
    - [3.1.2 LayoutManager 开启自动测量](#312-layoutmanager-%E5%BC%80%E5%90%AF%E8%87%AA%E5%8A%A8%E6%B5%8B%E9%87%8F)
      - [3.1.2.1 dispatchLayoutStep1()](#3121-dispatchlayoutstep1)
      - [3.1.2.2 dispatchLayoutStep2()](#3122-dispatchlayoutstep2)
    - [3.1.3 LayoutManager 不开启自动测量](#313-layoutmanager-%E4%B8%8D%E5%BC%80%E5%90%AF%E8%87%AA%E5%8A%A8%E6%B5%8B%E9%87%8F)
  - [3.2 onLayout()](#32-onlayout)
    - [3.2.1 dispatchLayoutStep3()](#321-dispatchlayoutstep3)
  - [3.3 draw() 和 onDraw()](#33-draw-%E5%92%8C-ondraw)
- [四、RecyclerView 性能优化](#%E5%9B%9Brecyclerview-%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96)
  - [4.1 RecyclerView.setHasFixdSize()](#41-recyclerviewsethasfixdsize)
  - [4.2 RecyclerView.setRecycledViewPool()](#42-recyclerviewsetrecycledviewpool)
  - [4.3 DiffUtil](#43-diffutil)

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
     * @param state：用在 LayoutManager、 Adapter 等组件之间共享 RecyclerView 状态的。
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

在测量和布局上，LayoutManager 和 ViewGroup 使用的方法基本上都是一一对应的，不同在于 LayoutManager 需要考虑 ItemDecoration 的偏移。

## 2.1 测量

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

在 ViewGroup 中测量完毕后就可以通过 View.getMeasureWidth()、View.getMeasureHeight() 拿到测量的宽高，但在 LayoutManager 中，应该通过 LayoutManager.getDecoratedMeasuredWidth(View child) 拿到测量结果，它会将 ItemDecoration 的偏移加入一起计算，从而适配 ItemDecoration.getItemOffsets()。

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

## 2.2 布局

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

## 2.3 回收复用

### 2.3.1 缓存池

RecyclerView 一共有 5 种视图缓存池，按缓存优先级从先到后先进行一个小结：

**（1）mAttachedScrap 和 mChangedScrap**

mAttachedScrap 和 mChangedScrap 不参与滑动过程中的回收复用，主要在布局时（onLayoutChildren()）进行回收复用，缓存的是从 RecyclerView 分离出来的 View，但是又即将添加上去的 ViewHolder。在多次 layout 的情况下，ViewHolder 的位置和数据都是不变的，一般情况下可以直接复用从而不需要重新调用 onBindViewHolder() 从而提高性能。

mAttachedScrap 和 mChangedScrap 的主要区别如下：

1. 被标记 update 的 ViewHolder 以及必须设置了 mItemAnimator，且 ItemAnimator 不可以重用的 ViewHolder 缓存进 mChangedScrap，其余情况缓存进 mAttachedScrap。
2. mChangedScrap 仅在预布局下使用（mRunPredictiveAnimations = true）。

**（2）mCachedView**

保存最新被移除的 ViewHolder（调用 recycleViewHolderInternal(ViewHolder holder))，它的作用是在需要新的 ViewHolder 时，精确匹配是不是 **刚移除** 的那个（通过判断 ViewHolder 的 position）。如果是，就直接返回给 RecyclerView 进行展示；如果不是，那么即使 mCachedViews 中有 ViewHolder，也不会返回给 RecyclerView 使用。除非 ViewHolder 有数据上的更新，否则在正常滑动可以直接复用从而不需要重新调用 onBindViewHolder() 从而提高性能。

mCachedViews 的默认大小为 2，可通过 RecyclerView.setItemViewCacheSize() 修改该值大小。当 mCachedViews 满了以后，会利用先进先出原则，将出的 ViewHoloder 存放在 mRecyclerPool 中。

**（3）mViewCacheExtension**

RecyclerView 允许我们自己扩展回收池，我们可以通过调用 setViewCacheExtension() 去自定义缓存。一般情况下使用系统自带的缓存即可，它的缓存优先级在比 mCachedView 低，比 mRecyclerPool 高。

**（4）mRecyclerPool**

真正废弃的是存放在 mRecyclerPool 中的 ViewHolder。从 mRecyclerPool 拿到的 ViewHolder 会回调 onBindViewHolder() 重新绑定数据。根据 ViewType 来缓存 ViewHolder，每个 ViewType 的数组大小为 5，可以动态的改变。

### 2.3.2 detachAndScrapAttachedViews()

detachAndScrapXXX() 用于暂时分离子视图，并将其添加到 Recycler 缓存堆中。一般只用于 onLayoutChildren() 中。

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
    // 忽略的 ViewHolder 不处理。
    if (viewHolder.shouldIgnore()) {
        if (DEBUG) {
            Log.d(TAG, "ignoring view " + viewHolder);
        }
        return;
    }
    
    if (viewHolder.isInvalid() && !viewHolder.isRemoved()
            && !mRecyclerView.mAdapter.hasStableIds()) {
        // viewHolder.isInvalid() 为 true，一般有 3 种可能：
        // 1.调用 Adapter.notifyDataSetChanged()；
        // 2.调用 RecyclerView.invalidateItemDecorations()；
        // 3.调用 RecyclerView.setAdapter() 或者 RecyclerView.swapAdapter()。

        // 因此一般情况下是调用了 Adapter.notifyDataSetChanged() 会走到该代码块。（同时需要满足没有设置 Adapter.setHasStableIds(true) 且该 ViewHolder 没有被调用 notifyItemRangeRemoved()）
        // 将 View 从 RecyclerView 中移除，本质上是调 ViewGroup.removeViewXXX()，
        removeViewAt(index);
        // 将 viewHolder 缓存到 mCachedView 或 mRecyclerPool 中，该方法在下文进行分析。
        recycler.recycleViewHolderInternal(viewHolder);
    } else {
        // 一般都是局部的视图变化（调用了 Adapter.notifyItemXXX()）或设置了 RecyclerView.setHasFixedSize(true)，会走到这里。
        // 将 View 从 RecyclerView 上分离，分离的本质是仅仅将 View 和 RecyclerView 互相之间的引用置空，最终调用的是 ViewGroup.detachViewFromParent()。
        detachViewAt(index);
        // 将 ViewHolder 缓存到 mAttachedScrap 或 mChangedScrap 中，以便稍后直接重用。
        recycler.scrapView(view);
        mRecyclerView.mViewInfoStore.onViewDetached(viewHolder);
    }
}
```

#### 2.3.2.1 scrapView()

该方法可以直接看出 mAttachedScrap 和 mChangedScrap 的区别。

```java
void scrapView(View view) {
    final ViewHolder holder = getChildViewHolderInt(view);
    if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
            || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
        if (holder.isInvalid() && !holder.isRemoved() && !mAdapter.hasStableIds()) {
            throw new IllegalArgumentException("Called scrap view with an invalid view."
                    + " Invalid views cannot be reused from scrap, they should rebound from"
                    + " recycler pool." + exceptionLabel());
        }
        // 满足以下任意一个条件就存到 mAttachedScrap 中：
        // 1.被同时标记 remove 和 invalid。
        // 2.不需要进行更新的 ViewHolder（例如 notifyItemChanged 会标记 ViewHolder 为 update）。
        // 3.没有设置 mItemAnimator，或者 mItemAnimator 可以重用被标记为 updated 的 ViewHolder（默认的 DefaultItemAnimator 可以重用）。
        
        // 因此从上面的限制条件可以得出，一般请下，只有满足以下 2 个条件才会放到 mChangedScrap，其它情况都放到 mAttachedScrap 中，也就是说绝大部分时候都是放到 mAttachedScrap 中。
        // 1. 被标记 update 的 ViewHolder。
        // 2. 必须设置了 mItemAnimator，且 ItemAnimator 不可以重用 ViewHolder。
        holder.setScrapContainer(this, false);
        mAttachedScrap.add(holder);
    } else {
        if (mChangedScrap == null) {
            mChangedScrap = new ArrayList<ViewHolder>();
        }
        holder.setScrapContainer(this, true);
        mChangedScrap.add(holder);
    }
}

boolean canReuseUpdatedViewHolder(ViewHolder viewHolder) {
    return mItemAnimator == null || mItemAnimator.canReuseUpdatedViewHolder(viewHolder,
            viewHolder.getUnmodifiedPayloads());
}
```

### 2.3.3 removeAndRecycleView()

从 RecyclerView 移除子视图并其缓存到 mCachedView 或 mRecyclerPool 中，一般用于滑动过程中的视图回收。

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

#### 2.3.3.1 recycleView()

```java
public void recycleView(@NonNull View view) {
  
    ViewHolder holder = getChildViewHolderInt(view);
    if (holder.isTmpDetached()) {
        // 完成对分离视图的移除。
        removeDetachedView(view, false);
    }
    if (holder.isScrap()) {
        // 如果 holder 已在 mAttachedScrap 或 mChangedScrap 缓存，应该从当前所处缓存中移除，因为将要它放入 mRecyclerPool 中。
        holder.unScrap();
    } else if (holder.wasReturnedFromScrap()) {
        // 清除无用 flag。
        holder.clearReturnedFromScrapFlag();
    }
    recycleViewHolderInternal(holder);
}

void recycleViewHolderInternal(ViewHolder holder) {
    // 省略部分代码
    boolean cached = false;
    boolean recycled = false;

    if (forceRecycle || holder.isRecyclable()) {
        // 先尝试缓存到 mCachedViews 中。
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
            // 将 holder 缓存到 mCachedViews 中。
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

### 2.3.4 getViewForPosition()

给定一个数据源在容器的位置从而得到一个视图。

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
    // 1. 如果处于预布局状态，则尝试从缓存 mChangedScrap 中依次对比 ViewHolder.getLayoutPosition()、StableId 复用 ViewHolder。
    // 为什么会这样呢？因为处于预布局状态必定会展示预测性动画（mRunPredictiveAnimations=true），也仅在开启动画时，才可能有 ViewHolder 缓存到 mChangedScrap 中。
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
        fromScrapOrHiddenOrCache = holder != null;
    }
    if (holder == null) {
        // 2. 依次尝试从 mAttachedScrap、mCachedView 中通过对比 ViewHolder.getLayoutPosition() 复用 ViewHolder。
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
        if (holder != null) {
            // 如果获取到的 ViewHolder 是无效的。
            if (!validateViewHolderForOffsetPosition(holder)) {
                if (!dryRun) {
                    // 做一些清理操作，然后重新放入 mCacheViews 或 RecyclerViewPool 中。
                    holder.addFlags(ViewHolder.FLAG_INVALID);
                    if (holder.isScrap()) {
                        removeDetachedView(holder.itemView, false);
                        holder.unScrap();
                    } else if (holder.wasReturnedFromScrap()) {
                        holder.clearReturnedFromScrapFlag();
                    }
                    recycleViewHolderInternal(holder);
                }
                holder = null;
            } else {
                fromScrapOrHiddenOrCache = true;
            }
        }
    }
    if (holder == null) {
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        if (offsetPosition < 0 || offsetPosition >= mAdapter.getItemCount()) {
            throw new IndexOutOfBoundsException("Inconsistency detected. Invalid item "
                    + "position " + position + "(offset:" + offsetPosition + ")."
                    + "state:" + mState.getItemCount() + exceptionLabel());
        }

        // 获取第 position 个数据所对应的 ItemViewType，不同的 ItemViewType 会对应不同的视图样式，因此之间是不能复用的。
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
            // 5. 最后尝试从 mRecyclerPool 中获取 ViewHolder，这也是我们最常见的缓存方式。
            // mRecyclerPool 中每个 mItemViewType 的最大缓存容量默认是 5 个，实现开发过程中应当根据需求动态调整优化性能。
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
        // 7. 若 ViewHolder 没有被绑定或需要更新或视图被标记无效，则尝试重新绑定 viewHolder，对于开发者来说，则是回调 Adapter.bindViewHolder()。
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
 
## 2.4 自定义 LayoutManager 流程

自定义 LayoutManager 和自定义 ViewGroup 是流程是相似的，只是多了回收复用以及滑动的处理。一般情况下流程如下：

1. 在 onLayoutChildren() 方法中，进行布局之前，调用 detachAndScrapAttachedViews() 将当前屏幕上所有的 ViewHolder 与屏幕分离（因为不一定是首次布局）；
2. 遍历 getItemCount()，调用 recycler.getViewForPosition(position) 拿到子 View ，再通过 addView() 添加到 RecyclerView 中。此处需要进行性能优化，通过计算只添加首屏会出现的 Item，然后在滑动的过程中进行复用，而不是把所有 Item 都变为一个子 View，否则数据量大时可能会直接 OOM 或卡顿；
3. 调用 measureChild() 或 measureChildWithMargins() 对子 View 进行测量。也可以自己手写限制的测量规格 MeasureSpec，再调用 child.measure(widthSpec, heightSpec)；
4. 调用 layoutDecorated() 或 layoutDecoratedWithMargins() 对子 View 进行布局；
5. RecyclerView 滑动过程的回收复用。

## 2.5 回收复用的实现思路

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

## 2.6 技巧

### 2.6.1 getChildDrawingOrder()

重写 getChildDrawingOrder() 可改变子 View 的绘制顺序。

### 2.6.2 滑动时回收

在滑动过程中，可以把 mAttachedScrap 中的缓存转进 mRecyclerPool 中，mAttachedScrap 中可重用的 ViewHolder 已经在 onLayoutChildren() 中复用。

```java
private void recycleChildren(RecyclerView.Recycler recycler) {
    List<RecyclerView.ViewHolder> scrapList = recycler.getScrapList();
    for (int i = 0; i < scrapList.size(); i++) {
        RecyclerView.ViewHolder holder = scrapList.get(i);
        removeAndRecycleView(holder.itemView, recycler);
    }
}
```

# 三、RecyclerView 源码分析

本文基于 androidx.recyclerview:recyclerview:1.0.0。

Recyclerview 的本质依然是一个 View，因此它的源码分析依旧从 View 的三大流程 onMeasure()、onLayout()、onDraw() 开始。

## 3.1 onMeasure()

dispatchLayoutStep1()、dispatchLayoutStep2()、dispatchLayoutStep3() 会在下文经常出现，因此先对这 3 个方法和所对应的的状态进行一个小结，从而对整个测量布局过程有一个大体的认知：

```java
public static class State {
    static final int STEP_START = 1;
    static final int STEP_LAYOUT = 1 << 1;
    static final int STEP_ANIMATIONS = 1 << 2;

    int mLayoutStep = STEP_START;
}
```

- dispatchLayoutStep1：mLayoutStep 处于 STEP_START 状态，在方法执行要结束时状态置为 STEP_LAYOUT。本方法的作用主要有四点：
  1. 处理 Adapter 更新;
  2. 决定应运行哪个动画;
  3. 保存布局前的视图信息。
  4. 如果有必要，进行预布局并保存布局后的视图信息。
- dispatchLayoutStep2：一般情况下处于 State.STEP_LAYOUT 状态，也可能由于二次测量的原因处于 State.STEP_ANIMATIONS 状态，在方法执行要结束时置为 State.STEP_ANIMATIONS。该方法主要作用是进行子视图的测量和布局。
- dispatchLayoutStep3：处于 STEP_ANIMATIONS 状态。在方法执行要结束时状态重新置为 STEP_START， 这个方法的作用执行在 dispatchLayoutStep1 方法里面保存的动画信息。本方法不是本文的介绍重点，后面在介绍 ItemAnimator 时，会重点分析这个方法。

接下来开始正文，下文中变量 mLayout 的类型都是 RecyclerView.LayoutManager。

RecyclerView 的 onMeasure() 可以分为三种情况：

```java
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout == null) {
        // 第一种情况， mLayout==null。对开发者所设置的 padding 值和父视图的限制进行一个默认测量。
        return;
    }
    if (mLayout.isAutoMeasureEnabled()) {
        // 第二种情况，LayoutManager 启动自动测量，这也是绝大部分 layoutManager 的处理方式，可以分为 2 种情况：
        // 1.父视图的宽高限制都是 MeasureSpec.EXACTLY，则直接确定 RecyclerView 的宽高。dispatchLayoutStep1() 和 dispatchLayoutStep2() 推迟到 onLayout() 进行。
        // 2.根据父视图的限制以及子视图想要的宽高去决定 RecyclerView 的测量宽高，子视图的测量和布局直接发生在 onMeasure() 阶段。
    } else {
        // 第三种情况。LayoutManager 不启动自动测量，这种情况极少见，可以分为 2 种情况：
        // 1. 调用方确定 RecyclerView 的大小不跟随子视图的个数、大小变化而变化（mHasFixedSize = true），则直接使用 mLayout 的默认测量方式。
        // 2. 仅参考数据源的个数和父视图的限制去确定测量算法从而确定 RecyclerView 的测量宽高，子视图的测量和布局发生在 onLayout() 阶段。
    }
}
```

### 3.1.1 mLayout == null

直接看一下源码：

```java
if (mLayout == null) {
    defaultOnMeasure(widthSpec, heightSpec);
    return;
}

void defaultOnMeasure(int widthSpec, int heightSpec) {

    final int width = LayoutManager.chooseSize(widthSpec,
            getPaddingLeft() + getPaddingRight(),
            ViewCompat.getMinimumWidth(this));
    final int height = LayoutManager.chooseSize(heightSpec,
            getPaddingTop() + getPaddingBottom(),
            ViewCompat.getMinimumHeight(this));

    setMeasuredDimension(width, height);
}

// 该方法在 LayoutManager 中。
public static int chooseSize(int spec, int desired, int min) {
    // desired 在 defaultOnMeasure() 就是水平或者垂直方向的 padding 值之和。
    // min 则是所设置的最小宽度或高度。
    final int mode = View.MeasureSpec.getMode(spec);
    final int size = View.MeasureSpec.getSize(spec);
    switch (mode) {
        case View.MeasureSpec.EXACTLY:
            return size;
        case View.MeasureSpec.AT_MOST:
        // 限制上线时，取最小。
            return Math.min(size, Math.max(desired, min));
        case View.MeasureSpec.UNSPECIFIED:
        // 不限制时，取最大。
        default:
            return Math.max(desired, min);
    }
}
```

### 3.1.2 LayoutManager 开启自动测量

此时的测量阶段（onMeasure）可以主要分为两步，分别调用 dispatchLayoutStep1() 和 dispatchLayoutStep2()，还有一个 dispatchLayoutStep3() 在布局阶段（onLayout）中调用。

接着开始分析源码，可以将其分成以下关键的节点：

```java
if (mLayout.isAutoMeasureEnabled()) {
    final int widthMode = MeasureSpec.getMode(widthSpec);
    final int heightMode = MeasureSpec.getMode(heightSpec);
    // 1、默认实现是 mRecyclerView.defaultOnMeasure(widthSpec, heightSpec)，除非有特殊需求，否则一般情况下在开启自动测量的情况下，不需要重写。
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
    final boolean measureSpecModeIsExactly =
            widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
    // 2、若父视图宽高的限制为确定的大小（MeasureSpec.EXACTLY），则不用再测量子视图去确定 RecyclerView 的测量宽高，dispatchLayoutStep1() 和 dispatchLayoutStep2() 推迟到 onLayout() 进行。
    if (measureSpecModeIsExactly || mAdapter == null) {
        return;
    }
    if (mState.mLayoutStep == State.STEP_START) {
    // 2、执行 dispatchLayoutStep1()。
        dispatchLayoutStep1();
    }
    // 3、将父视图的测量限制传给 LayoutManager，之后会在 dispatchLayoutStep2() 中使用该限制对子视图进行测量和布局。
    mLayout.setMeasureSpecs(widthSpec, heightSpec);
    mState.mIsMeasuring = true;
    // 4、dispatchLayoutStep2()，主要是对子视图进行测量和布局。
    dispatchLayoutStep2();

    // 5、这时可以拿到所有子视图所占的区域和父视图的限制，然后确定最终 RecyclerView 所需的宽高。
    mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

    // 6、mLayout 是否需要进行二次测量。例如父视图的限制为不限制（MeasureSpec.UNSPECIFIED），又存在开发者对子视图的要求为铺满父视图时，则需要二次测量。
    if (mLayout.shouldMeasureTwice()) {
        // 将第一次的测量结果作为测量的限制。
        mLayout.setMeasureSpecs(
                MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
        mState.mIsMeasuring = true;
        // 重新测量和布局。
        dispatchLayoutStep2();
        // 拿到所有子视图所占的区域和父视图的限制，然后确定最终 RecyclerView 所需的宽高。
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
    }
}
```

#### 3.1.2.1 dispatchLayoutStep1()

```java
private void dispatchLayoutStep1() {
    // dispatchLayoutStep1 执行时必须处于 State.STEP_START。
    mState.assertLayoutStep(State.STEP_START);
    // 如果处于滑动的过程中发生了重测，则存下剩余滑动的距离。
    fillRemainingScrollValues(mState);
    mState.mIsMeasuring = false;
    startInterceptRequestLayout();
    // 清空原视图动画信息。
    mViewInfoStore.clear();
    onEnterLayoutOrScroll();
    // 1、处理适配器的更新，并设置动画标识 mRunSimpleAnimations 和 mRunPredictiveAnimations，看下文前先看该方法的解析。
    processAdapterUpdatesAndSetAnimationFlags();
    saveFocusInfo();
    mState.mTrackOldChangeHolders = mState.mRunSimpleAnimations && mItemsChanged;
    mItemsAddedOrRemoved = mItemsChanged = false;
    // 可以发现是否是预布局阶段与 mRunPredictiveAnimations 有直接关系。
    mState.mInPreLayout = mState.mRunPredictiveAnimations;
    mState.mItemCount = mAdapter.getItemCount();
    findMinMaxChildLayoutPositions(mMinMaxLayoutPositions);

    if (mState.mRunSimpleAnimations) {
        int count = mChildHelper.getChildCount();
        for (int i = 0; i < count; ++i) {
            final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
            if (holder.shouldIgnore() || (holder.isInvalid() && !mAdapter.hasStableIds())) {
                continue;
            }
            
            // 2、遍历所有子视图，拿到没有被移除（有效）的子视图所对应的 ViewHolder。
            final ItemHolderInfo animationInfo = mItemAnimator
                    .recordPreLayoutInformation(mState, holder,
                            ItemAnimator.buildAdapterChangeFlagsForAnimations(holder),
                            holder.getUnmodifiedPayloads());
            // 3、记录布局前的视图信息存入 mViewInfoStore 中（动画肯定是有布局前的样式移动到布局后的样式）。
            mViewInfoStore.addToPreLayout(holder, animationInfo);
            if (mState.mTrackOldChangeHolders && holder.isUpdated() && !holder.isRemoved()
                    && !holder.shouldIgnore() && !holder.isInvalid()) {
                long key = getChangedHolderKey(holder);
                // This is NOT the only place where a ViewHolder is added to old change holders
                // list. There is another case where:
                //    * A VH is currently hidden but not deleted
                //    * The hidden item is changed in the adapter
                //    * Layout manager decides to layout the item in the pre-Layout pass (step1)
                // When this case is detected, RV will un-hide that view and add to the old
                // change holders list.
                mViewInfoStore.addToOldChangeHolders(key, holder);
            }
        }
    }
    if (mState.mRunPredictiveAnimations) {
        // 保存 ViewHolder 进行预布局前的位置。
        saveOldPositions();
        final boolean didStructureChange = mState.mStructureChanged;
        mState.mStructureChanged = false;
        // 4、使用 RecyclerView 旧的宽高进行预布局。
        mLayout.onLayoutChildren(mRecycler, mState);
        mState.mStructureChanged = didStructureChange;

        for (int i = 0; i < mChildHelper.getChildCount(); ++i) {
            final View child = mChildHelper.getChildAt(i);
            final ViewHolder viewHolder = getChildViewHolderInt(child);
            if (viewHolder.shouldIgnore()) {
                continue;
            }
            if (!mViewInfoStore.isInPreLayout(viewHolder)) {
                int flags = ItemAnimator.buildAdapterChangeFlagsForAnimations(viewHolder);
                boolean wasHidden = viewHolder
                        .hasAnyOfTheFlags(ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
                if (!wasHidden) {
                    flags |= ItemAnimator.FLAG_APPEARED_IN_PRE_LAYOUT;
                }
                // 记录预布局的后的视图信息。
                final ItemHolderInfo animationInfo = mItemAnimator.recordPreLayoutInformation(
                        mState, viewHolder, flags, viewHolder.getUnmodifiedPayloads());
                if (wasHidden) {
                    recordAnimationInfoIfBouncedHiddenView(viewHolder, animationInfo);
                } else {
                    mViewInfoStore.addToAppearedInPreLayoutHolders(viewHolder, animationInfo);
                }
            }
        }
        // we don't process disappearing list because they may re-appear in post layout pass.
        clearOldPositions();
    } else {
        clearOldPositions();
    }
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
    mState.mLayoutStep = State.STEP_LAYOUT;
}
```

```java
private void processAdapterUpdatesAndSetAnimationFlags() {
    // 在触发重测期间，数据源又发生了改变。
    if (mDataSetHasChangedAfterLayout) {
        mAdapterHelper.reset();
        // 若通知整个数据源都发生了变化，则回调给 mLayout 相应的方法。
        if (mDispatchItemsChangedEvent) {
            mLayout.onItemsChanged(this);
        }
    }

    if (predictiveItemAnimationsEnabled()) {
    // 简单的说就是若可能处于预布局状态（用于展示动画），则将数据的更新回调推迟执行。
        mAdapterHelper.preProcess();
    } else {
        mAdapterHelper.consumeUpdatesInOnePass();
    }
    boolean animationTypeSupported = mItemsAddedOrRemoved || mItemsChanged;
    // mRunSimpleAnimations 为 true 需要注意的有 2 点：
    // 1.必须调用过一次 onLayout()，也就是说首次加载出来的视图是没有动画的（mFirstLayoutComplete 为 true）。
    // 2.mItemAnimator != null。
    mState.mRunSimpleAnimations = 
            && mItemAnimator != null
            && (mDataSetHasChangedAfterLayout
            || animationTypeSupported
            || mLayout.mRequestedSimpleAnimations)
            && (!mDataSetHasChangedAfterLayout
            || mAdapter.hasStableIds());
    // 简单动画（mRunSimpleAnimations）是高级动画（mRunPredictiveAnimations）的子集。
    mState.mRunPredictiveAnimations = mState.mRunSimpleAnimations
            && animationTypeSupported
            && !mDataSetHasChangedAfterLayout
            && predictiveItemAnimationsEnabled();
}
```

#### 3.1.2.2 dispatchLayoutStep2()

dispatchLayoutStep2() 进行子视图的实际测量和布局。

```java
private void dispatchLayoutStep2() {
    startInterceptRequestLayout();
    onEnterLayoutOrScroll();
    // 一般情况下处于 State.STEP_LAYOUT 状态，也可能由于二次测量的原因处于 State.STEP_ANIMATIONS 状态。
    mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
    mAdapterHelper.consumeUpdatesInOnePass();
    mState.mItemCount = mAdapter.getItemCount();
    mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;

    mState.mInPreLayout = false;
    // 核心代码，具体的 LayoutManager 会真正对子视图进行测量和布局。
    mLayout.onLayoutChildren(mRecycler, mState);

    mState.mStructureChanged = false;
    mPendingSavedState = null;

    // 调用 mLayout.onLayoutChildren 时可能会关闭 mItemAnimator，重新检查。
    mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator != null;
    mState.mLayoutStep = State.STEP_ANIMATIONS;
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
}
```

### 3.1.3 LayoutManager 不开启自动测量

```java
// 不开启自动测量且 mHasFixedSize 为 true 时（使用者确定 RecyclerView 的大小不会随着子视图的变化而变化），直接使用 mLayout 的默认测量方式。
if (mHasFixedSize) {
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
    return;
}
// 如果在测量前，适配器的数据源有变化。
if (mAdapterUpdateDuringMeasure) {
    startInterceptRequestLayout();
    onEnterLayoutOrScroll();
    // 处理适配器更新并设置动画标志。
    processAdapterUpdatesAndSetAnimationFlags();
    onExitLayoutOrScroll();

    if (mState.mRunPredictiveAnimations) {
        mState.mInPreLayout = true;
    } else {
        // consume remaining updates to provide a consistent state with the layout pass.
        mAdapterHelper.consumeUpdatesInOnePass();
        mState.mInPreLayout = false;
    }
    mAdapterUpdateDuringMeasure = false;
    stopInterceptRequestLayout(false);
} else if (mState.mRunPredictiveAnimations) {
    // mState.mRunPredictiveAnimations 仅在 processAdapterUpdatesAndSetAnimationFlags() 中可能设为 true。
    // 直接使用上一次的测量结果。
    setMeasuredDimension(getMeasuredWidth(), getMeasuredHeight());
    return;
}

if (mAdapter != null) {
    mState.mItemCount = mAdapter.getItemCount();
} else {
    mState.mItemCount = 0;
}
startInterceptRequestLayout();
// 参考数据源的个数和父视图的限制去确定测量算法从而确定 RecyclerView 的测量宽高。
mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
stopInterceptRequestLayout(false);
mState.mInPreLayout = false; // clear
```

## 3.2 onLayout()

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
    dispatchLayout();
    TraceCompat.endSection();
    // mFirstLayoutComplete 仅在此处置为 true。
    mFirstLayoutComplete = true;
}
```

主要代码为 dispatchLayout()，目的也很简单，保证 RecyclerView 的测量布局必须经历 3 个步骤 dispatchLayoutStep1、dispatchLayoutStep2、dispatchLayoutStep3。

```java
void dispatchLayout() {
    // Adapter 或者 LayoutManager 任意一个为空，都不会对子视图进行测量布局。
    if (mAdapter == null) {
        Log.e(TAG, "No adapter attached; skipping layout");
        // leave the state in START
        return;
    }
    if (mLayout == null) {
        Log.e(TAG, "No layout manager attached; skipping layout");
        // leave the state in START
        return;
    }
    mState.mIsMeasuring = false;
    if (mState.mLayoutStep == State.STEP_START) {
        // 执行到这里可能出现的情形：
        // 1：LayoutManager 不启动自动测量。
        // 2：LayoutManager 启动自动测量但父视图给 RecyclerView 的宽高限制是确定的（MeasureSpec.EXACTLY）。
        dispatchLayoutStep1();
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
            || mLayout.getHeight() != getHeight()) {
        // 该种情况出现于有数据更新或 RecyclerView 想要的大小和最后父视图给的大小不一致，即测量大小和最终的视图大小不一致。
        mLayout.setExactMeasureSpecsFrom(this);
        // 因此需要重新测量和布局子视图。
        dispatchLayoutStep2();
    } else {
        // 确保 layoutManager 最终拿到 RecyclerView 的大小是准确的。
        mLayout.setExactMeasureSpecsFrom(this);
    }
    dispatchLayoutStep3();
}
```

### 3.2.1 dispatchLayoutStep3()

```java
private void dispatchLayoutStep3() {
    mState.assertLayoutStep(State.STEP_ANIMATIONS);
    startInterceptRequestLayout();
    onEnterLayoutOrScroll();
    mState.mLayoutStep = State.STEP_START;
    if (mState.mRunSimpleAnimations) {
        // Step 3: 获到布局后 ItemView 的位置信息，保存在 ViewInfoStore 里面。
        for (int i = mChildHelper.getChildCount() - 1; i >= 0; i--) {
            ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
            if (holder.shouldIgnore()) {
                continue;
            }
            long key = getChangedHolderKey(holder);
            final ItemHolderInfo animationInfo = mItemAnimator
                    .recordPostLayoutInformation(mState, holder);
            ViewHolder oldChangeViewHolder = mViewInfoStore.getFromOldChangeHolders(key);
            if (oldChangeViewHolder != null && !oldChangeViewHolder.shouldIgnore()) {
                // 说明某个 viewHolder 在 2 次布局中都使用
                final boolean oldDisappearing = mViewInfoStore.isDisappearing(
                        oldChangeViewHolder);
                final boolean newDisappearing = mViewInfoStore.isDisappearing(holder);
                if (oldDisappearing && oldChangeViewHolder == holder) {
                    // run disappear animation instead of change
                    mViewInfoStore.addToPostLayout(holder, animationInfo);
                } else {
                    final ItemHolderInfo preInfo = mViewInfoStore.popFromPreLayout(
                            oldChangeViewHolder);
                    // we add and remove so that any post info is merged.
                    mViewInfoStore.addToPostLayout(holder, animationInfo);
                    ItemHolderInfo postInfo = mViewInfoStore.popFromPostLayout(holder);
                    if (preInfo == null) {
                        handleMissingPreInfoForChangeError(key, holder, oldChangeViewHolder);
                    } else {
                        animateChange(oldChangeViewHolder, holder, preInfo, postInfo,
                                oldDisappearing, newDisappearing);
                    }
                }
            } else {
                mViewInfoStore.addToPostLayout(holder, animationInfo);
            }
        }

        // 处理视图信息列表并触发动画。
        mViewInfoStore.process(mViewInfoProcessCallback);
    }

    mLayout.removeAndRecycleScrapInt(mRecycler);
    mState.mPreviousLayoutItemCount = mState.mItemCount;
    mDataSetHasChangedAfterLayout = false;
    mDispatchItemsChangedEvent = false;
    mState.mRunSimpleAnimations = false;

    mState.mRunPredictiveAnimations = false;
    mLayout.mRequestedSimpleAnimations = false;
    if (mRecycler.mChangedScrap != null) {
        mRecycler.mChangedScrap.clear();
    }
    if (mLayout.mPrefetchMaxObservedInInitialPrefetch) {
        // Initial prefetch has expanded cache, so reset until next prefetch.
        // This prevents initial prefetches from expanding the cache permanently.
        mLayout.mPrefetchMaxCountObserved = 0;
        mLayout.mPrefetchMaxObservedInInitialPrefetch = false;
        mRecycler.updateViewCacheSize();
    }

    mLayout.onLayoutCompleted(mState);
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
    mViewInfoStore.clear();
    if (didChildRangeChange(mMinMaxLayoutPositions[0], mMinMaxLayoutPositions[1])) {
        dispatchOnScrolled(0, 0);
    }
    recoverFocusFromState();
    resetFocusInfo();
}
```

## 3.3 draw() 和 onDraw()

RecylcerView 的 draw() 方法主要作用为：加入了 Decorations（装饰）的支持，它的作用从本质上讲，就是在绘制 Item 的前后允许我们插入自己的绘制需求。因此我们从 draw() 开始分析源码：

```java
public void draw(Canvas c) {
    super.draw(c);

    final int count = mItemDecorations.size();
    for (int i = 0; i < count; i++) {
        mItemDecorations.get(i).onDrawOver(c, this, mState);
    }
    // 忽略部分非核心代码。
}

public void onDraw(Canvas c) {
    super.onDraw(c);

    final int count = mItemDecorations.size();
    for (int i = 0; i < count; i++) {
        mItemDecorations.get(i).onDraw(c, this, mState);
    }
}
```

super.draw(c) 执行的 View.draw(c) ，从它的 [绘制内容和流程](./View%20的工作流程.md#%E4%BA%94%E7%BB%98%E5%88%B6%E6%B5%81%E7%A8%8Bdraw) 我们可以得出装饰的执行流程为：

1. 在 super.draw(c) 中先执行 onDraw()，也就是先执行所有装饰的 onDraw() 方法。
2. 在 super.draw(c) 中遍历子视图的 draw()（绘制子视图）。
3. 最后在 draw() 中执行所有装饰的 onDrawOver() 方法。

# 四、RecyclerView 性能优化

## 4.1 RecyclerView.setHasFixdSize()

若 Adapter 的数据变化不会导致 RecyclerView 的大小变化，则将该方法设置为 true。它可以在 RecyclerView 内容发生变化时不需要调用 requestLayout()，直接对子 View 进行 测量和布局。

## 4.2 RecyclerView.setRecycledViewPool()

多个 RecyclerView 在 viewType 一样（布局文件也一致）的情况下可以共用同一个 RecycledViewPool，例如订单的不同状态使用了多个 RecyclerView。

## 4.3 DiffUtil

适用于整个页面需要刷新，但是部分数据可能相同的情况（是否相同的标准由开发者决定）。它可以比较出不同的数据源进行局部刷新（局部刷新可以是某个 ViewHolder 的全量刷新或是单单某个 View 的方法调用），达到最小刷新的效果。具体的使用方式可看
https://github.com/CymChad/BaseRecyclerViewAdapterHelper/blob/master/library/src/main/java/com/chad/library/adapter/base/BaseQuickAdapter.java。