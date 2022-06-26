---
title: 自定义View--贝塞尔曲线、Shader
tags: [Android]
mathjax: true
math: true
date: 2019-01-20 21:56:52
keywords: [自定义View,贝塞尔曲线,波浪动画,望远镜效果]
---

贝塞尔曲线可以将Path中的moveTo、LineTo等连接的生硬路径变得平滑，也能实现很多好看的效果。

1. 一阶贝塞曲线

   $B(t)=P_0(1-t)+tP_1,t\in[0,1]$

   ![一阶贝塞曲线](/image/Android/customview/Bézier_curve_1.gif)

2. 二阶贝塞尔曲线

   $B(t)=P_0(1-t)^2+2t(1-t)P_1+  t^2P_2,t\in[0,1]$

   ![二阶贝塞曲线](/image/Android/customview/Bézier_curve_2.gif)

3. 三阶贝塞尔曲线

   $B(t)=P_0(1-t)^3+3P_1t(1-t)^2+  3P_2t^2(1-t)+P_3t^3,t\in[0,1]$

   ![三阶贝塞曲线](/image/Android/customview/Bézier_curve_3.gif)

<!--more-->

#### 贝塞尔曲线的应用

##### 手势追踪，改变moveTo、LineTo生硬路径现象

传统手势划线，重写onTouchEvent方法，在ACTION_MOVE事件中划线就好

``` java
@Override
public boolean onTouchEvent(MotionEvent event) {
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            path.moveTo(event.getX(),event.getY());
            break;
        case MotionEvent.ACTION_MOVE:
            path.lineTo(event.getX(),event.getY());
            postInvalidate();
            break;
        case MotionEvent.ACTION_UP:
            break;
    }
    return true;
}
```

这样的话，画出来的线条在转弯处比较生硬，我们可以使用贝塞尔曲线来优化一下

``` java
@Override
public boolean onTouchEvent(MotionEvent event) {
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            path.moveTo(event.getX(), event.getY());
            preX = event.getX();
            preY = event.getY();
            return true;
        case MotionEvent.ACTION_MOVE:
            float endX = (preX + event.getX()) / 2;
            float endY = (preY + event.getY()) / 2;
            path.quadTo(preX,preY,endX,endY);
            preX = event.getX();
            preY = event.getY();
            postInvalidate();
            break;
        default:
            break;
    }
    return super.onTouchEvent(event);
}
```

##### 波浪效果

![波浪效果](/image/Android/customview/wave_anim.gif)

``` java
//贝塞尔曲线实现波纹效果
public class WaveAnimationView extends View {
    private Paint paint;
    private Path path;
    private int itemWaveLength = 1080;
    private int dx;

    public WaveAnimationView(Context context) {
        super(context);
        init();
    }

    public WaveAnimationView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public WaveAnimationView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init(){
        path = new Path();
        paint = new Paint();
        paint.setColor(Color.GREEN);
        paint.setStyle(Paint.Style.FILL);
        startAnim();
    }


    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        path.reset();
        int originY = 300;
        int halfWaveLen = itemWaveLength/2;
        path.moveTo(-itemWaveLength+dx,originY);
        for(int i = -itemWaveLength;i<getWidth()+itemWaveLength;i+=itemWaveLength){
            path.rQuadTo(halfWaveLen/2,-100,halfWaveLen,0);
            path.rQuadTo(halfWaveLen/2,100,halfWaveLen,0);
        }
        path.lineTo(getWidth(),getHeight());
        path.lineTo(0,getHeight());
        path.close();
        canvas.drawPath(path,paint);

    }

    private void startAnim(){
        ValueAnimator animator = ValueAnimator.ofInt(0,itemWaveLength);
        animator.setDuration(2000);
        animator.setRepeatCount(ValueAnimator.INFINITE);
        animator.setInterpolator(new LinearInterpolator());
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                dx = (int) animation.getAnimatedValue();
                postInvalidate();
            }
        });
        animator.start();
    }
}
```

先放代码，再一点点解释。

首先init里面初始化需要用到的变量。然后在onDraw方法中，整个横向上画满波浪

``` java
path.reset();
int originY = 300;
int halfWaveLen = itemWaveLength/2;
path.moveTo(-itemWaveLength+dx,originY);
for(int i = -itemWaveLength;i<getWidth()+itemWaveLength;i+=itemWaveLength){
    path.rQuadTo(halfWaveLen/2,-100,halfWaveLen,0);
    path.rQuadTo(halfWaveLen/2,100,halfWaveLen,0);
}
```

然后将波浪底部填满

``` java
path.lineTo(getWidth(),getHeight());
path.lineTo(0,getHeight());
path.close();
canvas.drawPath(path,paint);
```

之后将画面移动起来就好了

``` java
private void startAnim(){
    ValueAnimator animator = ValueAnimator.ofInt(0,itemWaveLength);
    animator.setDuration(2000);
    animator.setRepeatCount(ValueAnimator.INFINITE);
    animator.setInterpolator(new LinearInterpolator());
    animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animation) {
            dx = (int) animation.getAnimatedValue();
            postInvalidate();
        }
    });
    animator.start();
}
```

#### 望远镜效果(点击哪里则哪里出现图像)

![望远镜效果](/image/Android/customview/shader.gif)

先上代码

``` java

public class ShadowLayerView extends View {

    private Paint paint;
    private Bitmap bitmap;
    private int preX;
    private int preY;

    public ShadowLayerView(Context context) {
        super(context);
        init();
    }

    public ShadowLayerView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public ShadowLayerView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {
        paint = new Paint();
        bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.test);
        setLayerType(LAYER_TYPE_SOFTWARE, null);
        paint.setColor(Color.BLACK);
        paint.setTextSize(125);
        paint.setShader(new BitmapShader(bitmap, Shader.TileMode.REPEAT, Shader.TileMode.MIRROR));
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        paint.clearShadowLayer();
        if (preY != -1 && preX != -1) {
            canvas.drawCircle(preX, preY, 150, paint);
        }
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                preX = (int) event.getX();
                preY = (int) event.getY();
                break;
            case MotionEvent.ACTION_MOVE:
                preX = (int) event.getX();
                preY = (int) event.getY();
                break;
            case MotionEvent.ACTION_UP:
                preX = -1;
                preY=-1;
                break;
        }
        postInvalidate();
        return true;
    }
}
```

init中初始化需要的变量及图片资源，paint.setShader是为了让图片铺满整个屏幕(可以省略)。

在onTouchEvent中保存了当前点击以及移动时的坐标，在结束点击的时候将坐标点重置为-1。

在onDraw方法中，当坐标不是(-1,-1)时，则以点击的坐标为圆心，画一个圆圈，把圆圈部分显示出来。

当然，刮奖效果也可以用这种方式实现，只需要保存一下移动的坐标(结合上面的手势追踪)，把路径画出来就好了。

----

以上