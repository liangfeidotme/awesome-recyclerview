# IRecyclerView

> [源码地址](https://github.com/aspsine/irecyclerview)

_这篇文章只分析了如何实现 FOOTER & HEADER，不包含下拉刷新的功能，以后用到了再补。_

## WrapperAdapter

`WrapperAdapter` 继承自 `RecyclerView.Adapter<RecyclerView.ViewHolder>`，内部定义了一个 `RecyclerView.Adapter` 类型的成员变量 `mAdapter`：

```java
private final RecyclerView.Adapter mAdapter;
```

`mAdapter` 通过构造函数由外部传递进来：

```java
public WrapperAdapter(RecyclerView.Adapter adapter, RefreshHeaderLayout refreshHeaderContainer,
                      LinearLayout headerContainer, LinearLayout footerContainer,
                      FrameLayout loadMoreFooterContainer) {
  mAdapter = adapter;
  mRefreshHeaderContainer = refreshHeaderContainer;
  mHeaderContainer = headerContainer;
  mFooterContainer = footerContainer;
  mLoadMoreFooterContainer = loadMoreFooterContainer;
  mAdapter.registerAdapterDataObserver(mObserver);
}
```

其中 `mRefreshHeaderContainer`、`mHeaderContainer`、`mFooterContainer`、`mLoadMoreFooterContainer` 会在创建 **ViewHolder** 的时候被用到。

以上分析可见 `WrapperAdapter` 确实是 `mAdapter` 的一个 **wrapper**，`mAdapter` 由外部传递进来，存储了“真正”的数据。

`WrapperAdapter` 在 `mAdapter` 的基础上增加了 4 个 类型的 item：

```java
protected static final int REFRESH_HEADER = Integer.MIN_VALUE;
protected static final int HEADER = Integer.MIN_VALUE + 1;
protected static final int FOOTER = Integer.MAX_VALUE - 1;
protected static final int LOAD_MORE_FOOTER = Integer.MAX_VALUE;
```

因此 `WrapperAdapter` 的 **itemCount** 比 `mAdapter` 多了 4 个：

```java
@Override
public int getItemCount() {
  return mAdapter.getItemCount() + 4;
}
```

相应的 `position` 所对应的 **itemType** 也发生了变化：

```java
@Override
public int getItemType(int position) {
  if (position == 0) return REFRESH_HEADER;
  if (position == 1) return HEADER; 
  if (1 < position && position < mAdapter.getItemCount() + 2) return mAdapter.getItemViewType(position - 2); 
  if (position == mAdapter.getItemCount() + 2) return FOOTER; 
  if (position == mAdapter.getItemCount() + 3) return LOAD_MORE_FOOTER;
}
```

那么 **ItemType** 对应的 **ViewHolder** 也会发生变化：

```java
@Override
public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
  if (viewType == REFRESH_HEADER) return new RefreshHeaderContainerViewHolder(mRefreshHeaderContainer);
  if (viewType == HEADER) return new HeaderContainerViewHolder(mHeaderContainer);
  if (viewType == FOOTER) return new FooterContainerViewHolder(mFooterContainer);
  if (viewType == LOAD_MORE_FOOTER) return new LoadMoreFooterContainerViewHolder(mLoadMoreFooterContainer);
  return mAdapter.onCreateViewHolder(parent, viewType);
}
```

构造 **ViewHolder** 时用到的 View 就是经由 `WrapperAdapter` 构造函数传递进来的四个 **View**。
其中 **ViewHolder** 的实现非常简单，都是继承自 `RecyclerView.ViewHolder`，例如 `RefreshHeaderContainerViewHolder`：

```java
static class RefreshHeaderContainerViewHolder extends RecyclerView.ViewHolder {
  public RefreshHeaderContainerViewHolder(View itemView) {
    super(itemView);
  }   
}  
```

由以上分析可得出 `WrapperAdapter` 对应着如下布局结构：

```

--------------------------
|     REFRESH_HEADER     |  position = 0
--------------------------
|         HEADER         |  position = 1
--------------------------
|         ......         |
--------------------------
|         ......         |
--------------------------
|         ......         |
--------------------------
|         FOOTER         |  position = mAdapter.getItemCount() + 2
--------------------------
|    LOAD_MORE_FOOTER    |  position = mAdapter.getItemCount() + 3
--------------------------
```

由上图可以看出，新增的 4 种 **item** 都需要与 `RecyclerView` 同宽：

```java
private boolean isFullSpanType(int type) {
  return type == REFRESH_HEADER || type == HEADER || type == FOOTER || type == LOAD_MORE_FOOTER;
}
```

因为 `RecyclerView` 有三种 **Layout Manager**，所以要分成三种情况来对待，其中 `LinearLayoutManager` 所有的 **item** 都能满足同宽的条件，所以只考虑 `GridLayoutManager` 和 `StaggeredGridLayoutManager` 两种即可。

当 **Layout Manager** 是 `GridLayoutManager` 时，需要修改 **header view** 与 **footer view** 的 __spanSize__ ，由于 `GridLayoutManager` 的 __spanCount__ 事先已经定义好了，因此可以在 `onAttachedToRecyclerView` 方法中一次性设定 - 当 **itemViewType** 满足 `isFullSpanType` 时，就让它占满一行的 **spans**：

```java
@Override
public void onAttachedToRecyclerView(final RecyclerView recyclerView) {
  RecyclerView.LayoutManager layoutManager = recyclerView.getLayoutManager();
  if (layoutManager instanceof GridLayoutManager) {
    final GridLayoutManager gridLayoutManager = (GridLayoutManager) layoutManager;
    gridLayoutManager.setSpanSizeLookup(new GridLayoutManager.SpanSizeLookup() {
      @Override
      public int getSpanSize(int position) {
        WrapperAdapter wrapperAdapter = (WrapperAdapter) recyclerView.getAdapter();
        if (isFullSpanType(wrapperAdapter.getItemViewType(position))) {
          return gridLayoutManager.getSpanCount();
        }
        return 1;
      }
    });
  }
}
```

但是 `StaggeredGridLayoutManager` 并没有 `setSpanSizeLookup` 方法，可以通过 [`setFullSpan`][setFullSpan] 方法达到想要的效果。

根据 [StackOverFlow 上的问题](setFullSpan-SOF) 可知，通常会在 `onBindViewHolder` 方法中通过 `setFullSpan` 来设置 **item** 的宽度，但是 `WrapperAdapter` 中只有 `mAdapter` 带进来的 **holder** 才需要复用：

```java
@Override
public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {
  if (1 < position && position < mAdapter.getItemCount() + 2) {
    mAdapter.onBindViewHolder(holder, position - 2);
  }
}
```

因此，只能在 `onViewAttachedToWindow` 方法中设置满足 `isFullSpanType` 的 `itemView` 的宽度。

```java
@Override
public void onViewAttachedToWindow(RecyclerView.ViewHolder holder) {
  super.onViewAttachedToWindow(holder);
  int position = holder.getLayoutPosition();
  int type = getItemViewType(position);
  if (isFullSpanType(type)) {
    ViewGroup.LayoutParams layoutParams = holder.itemView.getLayoutParams();
    if (layoutParams instanceof StaggeredGridLayoutManager.LayoutParams) {
      StaggeredGridLayoutManager.LayoutParams lp = (StaggeredGridLayoutManager.LayoutParams) layoutParams;
      lp.setFullSpan(true);
    }
  }
}
```

`WrapperAdapter` 构造函数的最后一行为 `mAdapter` 注册了一个 **AdapterDataObserver**：
```java
mAdapter.registerAdapterDataObserver(mObserver)
```
这个 `mObserver` 会把 `mAdapter` 中发生的**数据变化**转发给 `WrapperAdapter`，这样 `WrapperAdapter` 就接管了 `mAdapter` 的数据变化。

```java
private RecyclerView.AdapterDataObserver mObserver = new RecyclerView.AdapterDataObserver() {
    @Override
    public void onChanged() {
        WrapperAdapter.this.notifyDataSetChanged();
    }

    @Override
    public void onItemRangeChanged(int positionStart, int itemCount) {
        WrapperAdapter.this.notifyItemRangeChanged(positionStart + 2, itemCount);
    }

    @Override
    public void onItemRangeChanged(int positionStart, int itemCount, Object payload) {
        WrapperAdapter.this.notifyItemRangeChanged(positionStart + 2, itemCount, payload);
    }

    @Override
    public void onItemRangeInserted(int positionStart, int itemCount) {
        WrapperAdapter.this.notifyItemRangeInserted(positionStart + 2, itemCount);
    }

    @Override
    public void onItemRangeRemoved(int positionStart, int itemCount) {
        WrapperAdapter.this.notifyItemRangeRemoved(positionStart + 2, itemCount);
    }

    @Override
    public void onItemRangeMoved(int fromPosition, int toPosition, int itemCount) {
        WrapperAdapter.this.notifyDataSetChanged();
    }
};
```

## RefreshHeaderLayout

继承自 `ViewGroup`，重新实现了 `onMeasure` & `onLayout`，只处理第一个子 View（`View view = getChildAt(0)`)。
在 `onMeasure` 中把子 View 的 `measureSpec` 修改为 `MeasureSpec.UNSPECIFIED`，**交给上层 View 去测量大小**。

```java
int childHeightSpec = MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED);
View view = getChildAt(0);
measureChildWithMargins(view, widthMeasureSpec, 0, childHeightSpec, 0)
```

## IViewHolder

继承自 `RecyclerView.ViewHolder`，因为增加了 `HeaderView` 和 `RefreshHeaderLayout`， 所以 `position`、`layoutPosition`、`adateprPosition`、`oldPosition` 都有 2 个 offset。

```java
public final int getIPosition() { return getPosition() - 2; }
public final int getILayoutPosition() { return getLayoutPosition() - 2; }
public final int getIAdapterPosition() { return getAdapterPosition() - 2; }
public final int getIOldPosition() { return getOldPosition() - 2; }
```

## Listeners
### OnLoadMoreScrollListener

继承自 `RecyclerView.OnScrollListener`，在 `onScrollStateChanged(RecyclerView recyclerView, int newState)` 方法中通过如下条件判断是否能够触发 `onLoadMore`：

0. `visibleItemCount > 0`
0. `newState == RecyclerView.SCROLL_STATE_IDLE`
0. 最底部 item 的 `position == totalItemCount - 1`

### OnLoadMoreListener

上拉加载回调
```java
public interface OnLoadMoreListener {
    void onLoadMore(View loadMoreView);
}
```

### OnRefreshListener

下拉刷新回调
```java
public interface OnRefreshListener {
    void onRefresh();
}
```

### SimpleAnimatorListener

`Animator.AnimatorListener` 的空实现，其实可以用 [`AnimatorListenerAdapter`](http://developer.android.com/reference/android/animation/AnimatorListenerAdapter.html)

[setFullSpan]:http://developer.android.com/reference/android/support/v7/widget/StaggeredGridLayoutManager.LayoutParams.html#setFullSpan(boolean)
[setFullSpan-SOF]:http://stackoverflow.com/questions/33696096/setting-span-size-of-single-row-in-staggeredgridlayoutmanager/33707897#33707897
