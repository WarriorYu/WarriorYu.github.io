---
title: 实现在Recycleview中一键返回指定Position
categories:
  - Android
tags:
  - View
date: 2019-11-19 14:25:40
---

**问题：**

怎么实现Recyclerview 中某个Item的子View(如图片中的TextView3)，无论当前列表滑动到什么位置，让这个子View可以平滑滚动到屏幕顶部？

<img src="/images/img_recyclerview_position.jpg" style="zoom:21%;" />

**答：**

如果想平滑滚动到RecyclerView的某个Item,可以通过RecyclerView.smoothScrollToPosition(position);我们一般是当列表往上滑动一段距离后，显示一个「返回顶部」的按钮，点击返回到顶部，这个方法很完美。但是有一个特殊的场景：当点击按钮后，不是滑动到顶部，而是滑动到还没有显示出来的某个Item（比如要求滑动到position=10，当前在position=1）,这时候滑动出来的position=10的Item的底部贴在在屏幕的底部，而不是Item的顶部贴在屏幕的顶部。如果RecyclerView是垂直的，想解决这个问题，需要自定义一个LinearLayoutManager，通过RecyclerView.setLayoutManager(new LinearLayoutManagerWithSmoothScroller(context));替换掉原来的LinearLayoutManager。看代码：

```java
public class LinearLayoutManagerWithSmoothScroller extends LinearLayoutManager {

    public LinearLayoutManagerWithSmoothScroller(Context context) {
        super(context, RecyclerView.VERTICAL, false);
    }

    public LinearLayoutManagerWithSmoothScroller(Context context, int orientation, boolean 	reverseLayout) {
        super(context, orientation, reverseLayout);
    }

    @Override
    public void smoothScrollToPosition(RecyclerView recyclerView, RecyclerView.State state, int position) {
        RecyclerView.SmoothScroller smoothScroller = new TopSnappedSmoothScroller(recyclerView.getContext());
        smoothScroller.setTargetPosition(position);
        startSmoothScroll(smoothScroller);
    }

    private class TopSnappedSmoothScroller extends LinearSmoothScroller {
        public TopSnappedSmoothScroller(Context context) {
            super(context);

        }

        @Override
        public PointF computeScrollVectorForPosition(int targetPosition) {
            return LinearLayoutManagerWithSmoothScroller.this
                    .computeScrollVectorForPosition(targetPosition);
        }

        @Override
        protected int getVerticalSnapPreference() {
            return SNAP_TO_START;
        }
    }
}
```

再来看题目的要求，如果想把Textview3置顶的话，可以计算Textview3距离屏幕顶部的距离，通过RecyclerView.smoothScrollBy(x, y)平滑滚到到指定的距离。这个距离的计算可以通过下面我贴出来的「代码一」获取Textview3在屏幕中的坐标，但是在小米手机上有时候获取的坐标偶尔有正确的值，偶尔是[0,0]，我也没有找到是什么原因导致的这个问题，如果你有好的想法，欢迎来告诉我。但是我们可以通过下面的「代码二」获取Textview3距离屏幕顶部的精确距离。

「代码一」

```java
@Override
 public void onWindowFocusChanged(boolean hasFocus) {
     super.onWindowFocusChanged(hasFocus);
         int[] location = new int[2];
        //获取在当前窗口内的绝对坐标
         TextView3.getLocationInWindow(location); 
         int y = location[1];
 }
```

「代码二」

```java
TextView3.post(new Runnable() {
    @Override
    public void run() {
       int  top = TextView3.getTop()
    }
});
```

因为没有RecyclerView.smoothScrollTo(x,y)这个方法，只获取到坐标还实现不了功能，我们可以通过一个迂回的方法，通过下面的「代码三」监听RecyclerView的滑动距离，通过「代码三」中计算的总的滑动距离和「代码二」中获取的top可以计算出来一个差值，通过这个差值，就可以使用RecyclerView.smoothScrollBy(0, top - scrollHeight)平滑滚到到指定的距离了。

「代码三」

```java
  recyView.addOnScrollListener(new RecyclerView.OnScrollListener() {
            @Override
            public void onScrolled(@NonNull RecyclerView recyclerView, int dx, int dy) {
                super.onScrolled(recyclerView, dx, dy);
                int scrollHeight += dy;             
            }
        });
```

注意：如果通过「代码三」这种方式监听滑动距离，千万不能使用View.scrollTo()这种方法让View跳到某个位置，这会导致「代码三」中的监听滑动距离失败，可以理解为scrollTo()这种方法是一下跳过一段距离，所以无法获取到滑动的距离。

如果你有更好的实现方式，或者对于在复杂列表中返回到指定的Position有什么分享的，欢迎来讨论。