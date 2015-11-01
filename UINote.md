#UINote
&emsp;由于当前IOS和Android两个系统的交互设计相互渗透，现在很多时候我们都需要自己去自定义一些比较炫酷或者复杂的UI。其实我们自定义UI是可以通过这三种方法实现：

1.继承View，直接在Canvas上面绘制。

2.继承ViewGroup，根据ViewGroup的绘制流程实现自己需要的效果。

3.利用已有的UI控件(RelativeLayout,LinearLayout,FrameLayout....)。

接下来就以上的方法，一个一个实践来练习吧！

##自定义UI（利用View）
&emsp;利用View实现自定义主要是继承View并利用OnDraw(Canvas canvas)方法绘制自己需要的UI。利用canvas主要可以绘制矩形(包括圆角矩形)、圆、点、线、扇形等，同时还可以根据类[Path](http://developer.android.com/reference/android/graphics/Path.html)设置需要的路径，利用Canvas将Path绘制出来。具体的使用通过实践学习吧。今天利用View实现如下的效果(gif 效果不是很好0_0)：
  
  ![alt tag](/Resource/jianshu.gif)
  
###分解效果图
&emsp;动画效果的某个瞬间的静态UI如下所示：
  
  
通过这个静态的UI，可以看出它的组成很简单：一个大圆，一个实心的小圆————这个小圆的中心点是在大圆的圆环上。这样就把这个静态的UI分解为：一个大圆和一个小圆。
  
###实现分解效果图
&emsp;这里将实现上面分解的结果啦，应该很简单的！

1.绘制圆环

&emsp;在onDraw(Canvas canvas)这个方法中利用canvas绘制，直接上代码：

    private void drawBigCircle(Canvas canvas){
        canvas.drawCircle(rectF.centerX(), rectF.centerY(), getBigCircleRadius(), mCirclePaint);
    }
    
这里主要是利用了canvas的方法(当然，绘制circle的方法还有其他几个方法可以调用，根据自己的需求):

    drawCircle(float cx, float cy, float radius, @NonNull Paint paint)
    
简单介绍一下参数，这里cx,cy表示的是圆心位置；radius表示圆半径的大小，paint主要是画笔,可以设置颜色等。这个应该比较简单吧+_+

2.绘制实心小圆——小球

&emsp;还是直接看代码，然后分析^_^

    double sweepAngle = Math.PI/180 * 45;
    //caculate the circle's center(x, y)
    float y = (float)Math.sin(sweepAngle)*(getBigCircleRadius());
    float x = (float)Math.cos(sweepAngle)*(getBigCircleRadius());
    int restoreCount = canvas.save();
    //change aix center position
    canvas.translate(rectF.centerX(), rectF.centerY());
    canvas.drawCircle(x, y, mAccBallRadius, mAccBallPaint);
    canvas.restoreToCount(restoreCount);

这里的第一步是计算角度，第二是计算实心小圆的中心位置，这两步根据上面分解章节的数学计算应该很好理解的。

接下来就是绘制这个“小球”啦。首先移动画布(改变坐标系的中心位置):在计算小球的中心位置(cx,cy)的时候，是根据大圆的半径计算的，所以这里的(cx,cy)是到大圆中心点的距离。因此，Canvas的起始位置应该移动到大圆的中心位置。

这里首先要保存一下画布，在绘制结束了还要恢复到原来的状态，为什么？因为在绘制之前我们需要对画布进行移动，当绘制结束的时候如果不恢复到原来的状态，Canvas将一直保持在目前的状态，而后面的有可能是需要在没有移动的画布上绘制其他图案，这时候就会存在问题。

3.让小球动起来

&emsp;上面的两步我们已经把静态的状态绘制完成了，但是还需要让这个小球动起来(顺着大圆环跑Y(^_^)Y)。其实在绘制第二步绘制小球的时候我们是固定了一个45度角，如果我们让这个角度变化(0-360)，那么不就OK了。那么怎么才能做到呢？那就想啊..........，这里想到了利用ValueAimator来实现，关于这个类的使用这里就不做介绍。大家可以另外寻找相应的文章自行补脑！

    AccTypeEvaluator accCore = new AccTypeEvaluator();
    ValueAnimator animator = ValueAnimator.ofObject(accCore, 0.0f, 360.0f);
    animator.setDuration(mDuration);
    animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animation) {
            Float value = (Float) animation.getAnimatedValue();
            sweepAngle = value;
            invalidate();
        }
    });
    animator.setRepeatCount(ValueAnimator.INFINITE);
    animator.start();
  
这里的AccTypeEvaluator定义如下：

    public class AccTypeEvaluator implements TypeEvaluator<Float>{

    @Override
    public Float evaluate(float fraction, Float startValue, Float endValue) {
        Log.d("", " current fraction == " + fraction);
        return fraction * (endValue - startValue);
      }
    }
    
根据上面的代码，我们就可以获得0-360变化了，并且是随着时间的变化。当我们获取到角度时通过调用invalidate()方法，触发View重新绘制界面。这样就可以达到我们需要的动画效果啦。

## 总结
&emsp;&emsp;上面一步一步介绍了自定义UI的过程，现在应该对这个过程有所了解了——其实主要还是一个模型的建立过程。这里简单总结一下：

1.一点一点的分解UI，把复杂的界面分解成一个一个静态、简单的界面，并实现它们。

2.当静态界面绘制完了，需要动画的——观察需要改变哪些参数，并且不断的触发View绘制，不就动起来了^_^。

3.一些比较复杂的动画或者效果就需要一些物理、数学方面的知识来辅助了——塞贝尔曲线等

4.多多的练习，只有练习你才能对自定义有更深的理解。

这里放一个简单的自定义UI，让大家自己实现吧！

