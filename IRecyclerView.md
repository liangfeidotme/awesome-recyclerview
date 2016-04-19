## RefreshHeaderLayout
---

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
