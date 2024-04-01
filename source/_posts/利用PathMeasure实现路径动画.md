---
title: 利用PathMeasure实现路径动画
tags: [Android]
photos:
  - /image/Android/PathMeasure/path_measure.gif
date: 2019-01-04 21:47:21
keywords: Android,PathMeasure,动画,路径动画
---
我们可以利用路径动画实现很多好玩的东西，比如上面图中的类似支付宝支付完成的动画。
主要用到了`PathMeasure`,`ValueAnumator`这两个类
<!--more-->

#### PathMeasure
类似于一个计算器，可以计算一些和路径相关的东西。
两种初始化方式：
``` java
PathMeasure pathMeasure = new PathMeasure();
pathMeasure.setPath(path,false);
```
或者
``` java 
PathMeasure pathMeasure = new PathMeasure(path,false);
```

##### getLength()
用来获取路径长度，并且获取到的是当前曲线的长度，而不是整个Path的长度
``` java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    canvas.translate(150, 150);
    path.addRect(-50,-50,50,50,Path.Direction.CW);
    path.addRect(-100,-100,100,100,Path.Direction.CW);
    path.addRect(-120,-120,120,120,Path.Direction.CW);

    canvas.drawPath(path, paint);
    pathMeasure.setPath(path,false);
    do{
    	Log.e("huangyuan",pathMeasure.getLength()+"");
    }while (pathMeasure.nextContour());

}
```
> 2019-01-04 21:58:28.408 5231-5231/huangyuanlove.com.customwidget E/huangyuan: 400.0
2019-01-04 21:58:28.408 5231-5231/huangyuanlove.com.customwidget E/huangyuan: 800.0
2019-01-04 21:58:28.408 5231-5231/huangyuanlove.com.customwidget E/huangyuan: 960.0

pathMeasure.nextContour()得到的曲线的顺序与添加到Path中的顺序相同

##### getSegment()
函数定义
> public boolean getSegment(float startD, float stopD, Path dst, boolean startWithMoveTo)

懒得翻译，自己看吧
``` java
/**
    * Given a start and stop distance, return in dst the intervening
    * segment(s). If the segment is zero-length, return false, else return
    * true. startD and stopD are pinned to legal values (0..getLength()).
    * If startD >= stopD then return false (and leave dst untouched).
    * Begin the segment with a moveTo if startWithMoveTo is true.
*/
```
我们可以这么用
``` java
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        canvas.translate(150, 150);
        path.addRect(-50,-50,50,50,Path.Direction.CW);
        Path dst = new Path();
        pathMeasure.setPath(path,false);
        pathMeasure.getSegment(0,150,dst,true);
        canvas.drawPath(dst,paint);
    }

```
得到：
![getSegment](/image/Android/PathMeasure/dst.png  "getSegment")
如果dst路径不为空

``` java
Path dst = new Path();
dst.lineTo(10,100);
```
得到
![dst路径不空](/image/Android/PathMeasure/dst_not_null.png  "dst路径不空")
如果dst路径不空，`startWithMoveTo`为false，得到
![starWithMoveToFalse，](/image/Android/PathMeasure/starWithMoveToFalse.png  "starWithMoveToFalse")

##### 路径加载动画

``` java
 public TestPathMeasure(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {
        //禁用硬件加速
        setLayerType(LAYER_TYPE_SOFTWARE,null);
        paint = new Paint(Paint.ANTI_ALIAS_FLAG);
        paint.setColor(Color.BLACK);
        paint.setStrokeWidth(4);
        paint.setStyle(Paint.Style.STROKE);
        dstPath = new Path();
        circlePath = new Path();
        circlePath.addCircle(100,100,50,Path.Direction.CW);
        pathMeasure = new PathMeasure(circlePath,true);


        ValueAnimator valueAnimator = ValueAnimator.ofFloat(0,1);
        valueAnimator.setRepeatCount(ValueAnimator.INFINITE);


        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                currentAnimValue = (float) animation.getAnimatedValue();
                invalidate();
            }
        });
        valueAnimator.setDuration(2000);
        valueAnimator.start();

    }


    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);


        float length = pathMeasure.getLength();

        float stop = pathMeasure.getLength() * currentAnimValue;

        float start = (float) (stop- ((0.5- Math.abs(currentAnimValue-0.5))*length));

        dstPath.reset();
        pathMeasure.getSegment(start,stop,dstPath,true);
        canvas.drawPath(dstPath,paint);
    }

```

我们可以在路径动画上加个箭头，

``` java
public class TestPathMeasure extends View {

    private Paint paint;
    private Path path;
    private Path dstPath;
    private Path circlePath;
    private PathMeasure pathMeasure;
    private float currentAnimValue;

    private Bitmap icChevronRight;

    private float[] pos = new float[2];
    private float[] tan = new float[2];


    public TestPathMeasure(Context context) {
        super(context);
        init(context);
    }

    public TestPathMeasure(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        init(context);
    }

    public TestPathMeasure(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init(context);
    }

    private void init(Context context) {
        //禁用硬件加速
        setLayerType(LAYER_TYPE_SOFTWARE, null);
        paint = new Paint(Paint.ANTI_ALIAS_FLAG);
        paint.setColor(Color.BLACK);
        paint.setStrokeWidth(4);
        paint.setStyle(Paint.Style.STROKE);
        dstPath = new Path();
        circlePath = new Path();
        circlePath.addCircle(300, 300, 150, Path.Direction.CW);
        pathMeasure = new PathMeasure(circlePath, true);
        BitmapFactory.Options options = new BitmapFactory.Options();

        icChevronRight = BitmapFactory.decodeResource(context.getResources(), R.drawable.right);


        ValueAnimator valueAnimator = ValueAnimator.ofFloat(0, 1);
        valueAnimator.setRepeatCount(ValueAnimator.INFINITE);


        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                currentAnimValue = (float) animation.getAnimatedValue();
                invalidate();
            }
        });
        valueAnimator.setDuration(2000);
        valueAnimator.start();

    }


    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);


        float length = pathMeasure.getLength();

        float stop = pathMeasure.getLength() * currentAnimValue;

        float start = (float) (stop - ((0.5 - Math.abs(currentAnimValue - 0.5)) * length));

        dstPath.reset();
        pathMeasure.getSegment(start, stop, dstPath, true);
        canvas.drawPath(dstPath, paint);


        pathMeasure.getPosTan(stop, pos, tan);
        float degrees = (float) (Math.atan2(tan[1], tan[0]) * 180 / Math.PI);

        Matrix matrix = new Matrix();
//        matrix.postRotate(degrees,icChevronRight.getWidth()/2,icChevronRight.getHeight()/2);
//        matrix.postTranslate(pos[0] -icChevronRight.getWidth()/2,pos[1]-icChevronRight.getHeight()/2);

        pathMeasure.getMatrix(stop, matrix, PathMeasure.POSITION_MATRIX_FLAG | PathMeasure.TANGENT_MATRIX_FLAG);
        matrix.preTranslate(-icChevronRight.getWidth() / 2, -icChevronRight.getHeight() / 2);
        canvas.drawBitmap(icChevronRight, matrix, paint);
    }

}
```

这里主要用到了Matrix和getPosTan(),用来得到路径上某一长度的位置以及该位置的正切值。函数原型如下：

``` java

    /**
     * Pins distance to 0 <= distance <= getLength(), and then computes the
     * corresponding position and tangent. Returns false if there is no path,
     * or a zero-length path was specified, in which case position and tangent
     * are unchanged.
     *
     * @param distance The distance along the current contour to sample
     * @param pos If not null, returns the sampled position (x==[0], y==[1])
     * @param tan If not null, returns the sampled tangent (x==[0], y==[1])
     * @return false if there was no path associated with this measure object
    */
    public boolean getPosTan(float distance, float pos[], float tan[]) 
```

参数解释：

* float distance: 距离Path起点的长度，取值范围0<=distance<=getLength
* float[] pos:该点的坐标值，pos[0]表示x坐标，pos[y]表示y坐标
* float[] tan:该点的正切值。

在上面的代码中需要注意的是

* pos、tan数组在使用时必须先使用new关键词分配存储空间，而PathMeasure.getPosTan函数只会向数组中的元素赋值。
* 通过Math.atan2(tan[1],tan[0])得到的是弧度值，而不是角度
* 先利用matrix.postRotate将图片旋转指定角度，然后用matix.postTranslate将图片移动到当前路径最前端(注释掉的那两句)

pathMeasure有个getMatrix函数，是对我们自己实现的那种方式的封装，我们只需要将图片移动一下就好了

#### 山寨支付宝支付成功动画

就是先画一个圆，然后圆形里面画个对勾。。。。。很简陋的一种实现方式

``` java
public class AliPaySuccess extends View {

    private Paint paint;
    private Path dstPath;
    private Path circlePath;

    private int centerX = 500;
    private int centerY = 500;
    private int radius = 250;
    private PathMeasure pathMeasure;
    private float currentAnimValue;
    private boolean switchLine;

    public AliPaySuccess(Context context) {
        super(context);
        init();
    }

    public AliPaySuccess(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public AliPaySuccess(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init(){
        setLayerType(LAYER_TYPE_SOFTWARE, null);
        paint = new Paint(Paint.ANTI_ALIAS_FLAG);
        paint.setColor(Color.BLACK);
        paint.setStrokeWidth(4);
        paint.setStyle(Paint.Style.STROKE);
        dstPath = new Path();
        circlePath = new Path();
        circlePath.addCircle(centerX,centerY,radius,Path.Direction.CW);
        circlePath.moveTo(centerX-radius/2,centerY);
        circlePath.lineTo(centerX,centerY+radius/2);
        circlePath.lineTo(centerX+radius/2,centerY-radius/3);
        pathMeasure = new PathMeasure(circlePath,false);
        ValueAnimator animator = ValueAnimator.ofFloat(0,2);
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                currentAnimValue = (float) animation.getAnimatedValue();
                invalidate();
            }
        });
        animator.setDuration(2000);
        animator.setInterpolator(new AccelerateInterpolator());
        animator.start();
    }


    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if(currentAnimValue<1){
            float stop = pathMeasure.getLength()*currentAnimValue;
            pathMeasure.getSegment(0,stop,dstPath,true);
        }else if (currentAnimValue >1 && !switchLine){
            pathMeasure.getSegment(0,pathMeasure.getLength(),dstPath,true);
            switchLine = true;
            pathMeasure.nextContour();
        }else   {
            float stop = pathMeasure.getLength()*(currentAnimValue-1);
            pathMeasure.getSegment(0,stop,dstPath,true);
        }
        canvas.drawPath(dstPath,paint);
    }
}

```

1. 首先初始化各种参数：画笔、路径和动画

    初始化路径时，先添加了外面的圆形，然后是里面的对勾。设置动画从0～2，0-1时画圆，1-2时画对勾。

2. 重写onDraw，判断当前的动画值，在0-1时画圆，当动画值第一次大于1时，切换到对号那条线上。

   这里的`currentAnimValue`不一定会有等于1的时候，至少我执行了十几遍也没出现一次。所以取的是第一次大于时切换。

3. 这里面的数值和颜色之类的属性都是直接写死的，应该通过xml文件读取。



----

以上

