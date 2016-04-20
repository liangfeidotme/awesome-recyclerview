## WrapperAdapter

`WrapperAdapter` 继承自 `RecyclerView.Adapter<RecyclerView.ViewHolder>`，内部定义了一个 `RecyclerView.Adapter` 类型的成员变量 `mAdapter`：

```java
private final RecyclerView.Adapter mAdapter;
```

因此 `WrapperAdapter` 是 `mAdapter` 的 **wrapper**，`mAdapter` 中存储了“真正”的数据。
`WrapperAdapter` 在 `mAdapter` 的基础上增加了 4 个 类型的 item:

```java
protected static final int REFRESH_HEADER = Integer.MIN_VALUE;
protected static final int HEADER = Integer.MIN_VALUE;
protected static final int REFRESH_HEADER = Integer.MIN_VALUE;
protected static final int REFRESH_HEADER = Integer.MIN_VALUE;
protected static final int REFRESH_HEADER = Integer.MIN_VALUE;
```

因此 **itemCount** 比 `mAdapter` 增加了 4 个。

```java
@Override
public int getItemCount() {
  return mAdapter.getItemCount() + 4;
}
```

重新调整了 `position` 对应的 **itemType**：

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

其中 `mRefreshHeaderContainer`、`mHeaderContainer`、`mFooterContainer`、`mLoadMoreFooterContainer` 是通过构造函数传递进来的 View：

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

构造函数的最后一行为 `mAdapter` 注册了一个 **AdapterDataObserver**：
```java
mAdapter.registerAdapterDataObserver(mObserver)
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


