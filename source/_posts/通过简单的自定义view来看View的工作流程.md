---
title: 通过简单的自定义view来看View的工作流程
date: 2017-03-18 14:54:01
tags: [Android]
keywords: 简单自定义View
---

通过简单的自定义View(画个圆)，来了解一下View的工作流程以及自定义View应该注意的地方。
<!--more-->
### 一、自定义View的分类
#### 1.1 继承View重写onDraw方法
这种方法主要用于实现一些不规则的效果，比如动态或者静态显示一些不规则的图形，采用这种方式需要自己支持`wrap_content`,并且`padding`需要自己处理.
#### 1.2 继承ViewGroup派生特殊的Layout
这种方式主要用于实现自定义布局，如流式布局。采用这种方式需要合适的处理ViewGroup的测量、布局这两个过程，并同时处理子元素的测量和布局过程。
#### 1.3 继承特定的View(如TextView)
这种方法一般用于扩展某种已有的`View`的功能，比如`TextView`，这种方法比较容易实现，不需要自己支持`wrap_content`和`padding`。
#### 1.4 继承特定的ViewGroup
采用这种方式不需要自己处理`ViewGroup`的测量和布局这两个过程。
### 二、值得注意的地方
#### 2.1 让View支持wrap_content
这是因为直接继承View或者ViewGroup的控件，如果不在`onMeasure`中对`wrap_content`做特殊处理，那么当外界在适用`wrap_content`时就无法达到预期的效果。
#### 2.2 如果有必要，让View支持padding
这是因为如果直接继承View，如果不在draw方法中处理padding，那么padding属性是无法起作用的。另外，直接继承自ViewGroup的控件需要在`onMeasure`和`onLayout`中考虑padding和子元素的margin对其造成的影响，不然将导致padding和子元素的margin失效。
#### 2.3 尽量不要在View中使用Handler，没必要
因为View内部本身就提供了post系列的方法， 完全可以替代Handler的作用，当然除非你很明确要使用Handler来发送消息
#### 2.4 及时停止动画和线程
如果需要停止线程或者动画，可以在`onDetachedFromWindow`方法中处理，当包含此View的Activity退出或者当前View被remove时，View的`onDetachedFromWindow`方法会被调用，和此方法对应的是`onAttachedToWindow`，当包含此View的Activity启动时，View的`onAttachedToWindow`方法会被调用。
#### 2.5 View带有滑动嵌套情形时，需要处理好滑动冲突
如果有滑动冲突的话，那么要合适的处理滑动冲突，否则将会严重影响View的效果。
### 三、自定义View
#### 3.1 继承View重写onDraw方法
首先来看一下代码
``` java
public class TestCustomCircleView extends View {

    private int color = Color.RED;
    private Paint paint;
    public TestCustomCircleView(Context context) {
        super(context);
        init();
    }

    public TestCustomCircleView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public TestCustomCircleView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }
    private void init(){
        paint = new Paint();
        paint.setColor(color);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        int width = getWidth();
        int height = getHeight();
        int radius = Math.min(width,height)/2;
        canvas.drawCircle(width/2,height/2,radius,paint);
    }
}
```
上面的代码只是一种初级的实现，当我们进行使用的时候，只支持margin属性，并不支持padding属性。对`onDraw`方法进行修改，只要在绘制的时候考虑一下padding即可。
```java
 @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        int paddingLeft = getPaddingLeft();
        int paddingRight = getPaddingRight();
        int paddingTop = getPaddingTop();
        int paddingBottom = getPaddingBottom();
        int width = getWidth() - paddingLeft - paddingRight;
        int height = getHeight() - paddingBottom - paddingTop;
        int radius = Math.min(width, height) / 2;
        canvas.drawCircle(paddingLeft + width / 2, paddingTop + height / 2, radius, paint);
    }
```
但是现在还不能支持`warp_content`属性，现在使用`wrap_content`和使用`match_parent`没有任何区别：对于直接继承自View的控件，如果不对`wrap_content`做特殊处理，那么使用`wrap_content`就相当于使用`match_parent`.这里就需要我们重写`onMeasure`方法，当宽高属性为`wrap_content`时，取一个默认值。
```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int desiredWidth = 100;
        int desiredHeight = 100;
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);

        int width;
        int height;

        //宽度
        if (widthMode == MeasureSpec.AT_MOST) {
            width = Math.min(desiredWidth, widthSize);
        } else if (widthMode == MeasureSpec.EXACTLY) {
            width = widthSize;
        } else {
            width = desiredWidth;
        }
        //高度
        if (heightMode == MeasureSpec.AT_MOST) {
            height = Math.min(desiredHeight, heightSize);
        } else if (heightMode == MeasureSpec.EXACTLY) {
            height = heightSize;
        } else {
            height = desiredHeight;
        }
        setMeasuredDimension(width, height);
    }
```
这样，当我们使用wrap_content时，就是使用默认的100px的值。
### 四、使用自定义属性

#### 4.1 创建自定义的配置文件
在values目录下面创建自定义属性的XML，比如attrs.xml，也可以选择类似于attrs_circle_view.xml等这种以attrs_开头的文件名，当然这个文件名并没有什么限制，可以随便取名字。我们选择创建attrs.xml文件。
``` xml
<resources>
    <declare-styleable name="TestCustomCircleView">
        <attr name="circle_color" format="color"/>
    </declare-styleable>
</resources>
```
#### 4.2 在构造方法中解析自定义的属性值并做相应的处理
``` java
    public TestCustomCircleView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        TypedArray typedArray = context.obtainStyledAttributes(attrs, R.styleable.TestCustomCircleView);
        color = typedArray.getColor(R.styleable.TestCustomCircleView_circle_color,Color.RED);
        typedArray.recycle();
        init();
    }
```
首先加载自定义属性集合，接着解析属性集合中`TestCustomCircleView_circle_color`，如果没有指定属性值，则使用`Color.RED`作为默认值。
#### 4.3 在布局文件中使用自定义属性
``` xml
    <com.example.huangyuan.custom.TestCustomCircleView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center_horizontal"
        app:circle_color="@color/grey"
        />
``` 
