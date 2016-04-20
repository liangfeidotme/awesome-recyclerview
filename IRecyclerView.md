## WrapperAdapter

`WrapperAdapter` 继承自 `RecyclerView.Adapter<RecyclerView.ViewHolder>`，内部定义了一个 `RecyclerView.Adapter` 类型的成员变量 `mAdapter`：

```java
private final RecyclerView.Adapter mAdapter;
```

`mAdapter` 通过构造函数由外部传递进来。

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

其中 `mRefreshHeaderContainer`、`mHeaderContainer`、`mFooterContainer`、`mLoadMoreFooterContainer` 会在创建 **ViewHolder** 的时候会被用到。

以上分析可见 `WrapperAdapter` 确实是 `mAdapter` 的一个 **wrapper**，`mAdapter` 中存储了“真正”的数据，并且由外部传递进来。

`WrapperAdapter` 在 `mAdapter` 的基础上增加了 4 个 类型的 item:

```java
protected static final int REFRESH_HEADER = Integer.MIN_VALUE;
protected static final int HEADER = Integer.MIN_VALUE;
protected static final int REFRESH_HEADER = Integer.MIN_VALUE;
protected static final int REFRESH_HEADER = Integer.MIN_VALUE;
protected static final int REFRESH_HEADER = Integer.MIN_VALUE;
```

因此 `WrapperAdapter` 的 **itemCount** 比 `mAdapter` 多了 4 个。

```java
@Override
public int getItemCount() {
  return mAdapter.getItemCount() + 4;
}
```

相应的 `position` 所对应的 **itemType** 也发生了变化。

```java
@Override
public int getItemType(int position) {
  if (position == 0) return REFRESH_HEADER
  if (position == 1) return HEADER; 
  if (1 < position && position < mAdapter.getItemCount() + 2) return mAdapter.getItemViewType(position - 2); 
  if (position == mAdapter.getItemCount() + 2) return FOOTER; 
  if (position == mAdapter.getItemCount() + 3) return LOAD_MORE_FOOTER;
}
```

那么 **ViewType** 对应的 `ViewHolder` 也会发生变化：

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
********************
* REFRESH_HEADER   * position = 0
********************
*     HEADER       * position = 1
********************
*     .....        *
********************
*     .....        *
********************
*     ......       * 
********************
*     FOOTER       * position = mAdapter.getItemCount() + 2
********************
* LOAD_MORE_FOOTER * position = mAdapter.getItemCount() + 3
********************
```

由上图可以看出，新增的 4 种 **item** 都需要与 `RecyclerView` 同宽。

---

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


