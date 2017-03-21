---
title: View的滑动
date: 2017-03-15 11:01:46
tags: [Android,滑动动画]
---

#### 一 使用ScrollTo/ScrollBy

调用方式 `View.scrollTo(int x, int y)`,`View.scrollBy(int x, int y)`
方法源码：
``` java
/**
     * Set the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the x position to scroll to
     * @param y the y position to scroll to
     */
    public void scrollTo(int x, int y) {
        if (mScrollX != x || mScrollY != y) {
            int oldX = mScrollX;
            int oldY = mScrollY;
            mScrollX = x;
            mScrollY = y;
            invalidateParentCaches();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (!awakenScrollBars()) {
                postInvalidateOnAnimation();
            }
        }
    }

    /**
     * Move the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the amount of pixels to scroll by horizontally
     * @param y the amount of pixels to scroll by vertically
     */
    public void scrollBy(int x, int y) {
        scrollTo(mScrollX + x, mScrollY + y);
    }
```
从源码中可以看出，`scrollBy`实际上也是调用`scrollBy`的方法。需要注意的是在`View`的滑动过程中，`mScrollX`和`mScrollY`的改变规则：
在滑动过程中，`mScrollX`的值总是等于View左边缘和View内容左边缘在水平方向的距离，而mScrollY的值总是等于View上边缘和View内容上边缘在竖直方向的距离。其中`mScrollX`和`mScrollY`的单位是像素，并且当View左边缘在View内容左边缘的右边时，`mScrollX`为正值，反之为负值；当View上边缘在View内容上边缘的下边时，`mScrollY`为正值，反之为负值。换句话说：从左向右滑动，`mScrollX`为负值，反之为正值；如果从上往下滑动，`mScrollY`为负值，反之为正值。
**意思就是说ScrollTo/ScrollBy只能滑动View的内容而不能滑动View本身，比如，只能滑动TextView的文字，而不能滑动TextView控件本身**
#### 二 使用动画
这个没什么好介绍的，想要兼容3.0以下的属性动画，建议使用nineoldandroids来实现
#### 三 改变布局参数
改变布局参数，也就是改变·LayoutParams·
#### 四 使用Scroller进行平滑移动
自定义一个控件，添加成员变量·Scroller·，如下：
```java
public class ScrollerTextView extends TextView {

    private Scroller mScroller;

    public ScrollerTextView(Context context) {
        super(context);
        mScroller = new Scroller(context);
    }

    public ScrollerTextView(Context context, AttributeSet attrs) {
        super(context, attrs);
        mScroller = new Scroller(context);
    }

    public ScrollerTextView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mScroller = new Scroller(context);
    }

    public void smoothScrollBy(int dx,int dy){
        mScroller.startScroll(mScroller.getFinalX(),mScroller.getFinalY(),dx,dy,2000);
        invalidate();
    }

    public void smoothScrollTo(int fx, int fy){
        int dx = fx - mScroller.getFinalX();
        int dy = fy - mScroller.getFinalY();
        smoothScrollBy(dx,dy);
    }

    @Override
    public void computeScroll() {
       if(mScroller.computeScrollOffset()){
           scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
           postInvalidate();
       }
        super.computeScroll();
    }
}
```
其实`Scroller`也是通过`ScrollTO/ScrollBy`实现的，同样只能滑动内容，不能滑动本身。
PS：
在调用`startScroll`时，并没有让`View`进行滑动。而是在调用`invalidate()`进行重绘的时候，会去调用`computeScroll`方法，但是`computeScroll`在`View`只是个空实现，因此需要我们自己去实现。在`computeScroll`中进行平移。也就是说当View重绘后在`draw`方法中调用`computeScroll`,而`computeScroll`又会去向`Scroller`获取当前的`scrollX`和`scrollY`,然后通过`scrollTo`方法实现滑动，接着又调用`postInvalidate()`方法来进行第二次重绘，这一次重绘过程和第一次一样，还是会去调用`computeScroll()`方法，如此反复，直到整个滑动过程结束。

----
以上
