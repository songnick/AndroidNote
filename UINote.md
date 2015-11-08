#UINote
&emsp;目前IOS和Android两个系统的交互设计都在相互的借鉴，所以有的时候我们需要在Android系统中实现IOS的一些UI效果，那我们就必须自己实现啦（没有现成的控件）。现在也看到很多公司也把能够自定义UI放到招聘要求中，可见我们还是要学习如何去自定义。当然我们自定义UI的过程中，可以让我们对Android系统的一些机制理解的更加透彻。那么自定义UI真的很难吗，我感觉没有那么难，只要多练习，自然就可以了。按照我的理解，其实我们自定义UI是可以通过这三种方法实现：

1.继承View，直接在Canvas上面绘制。

2.继承ViewGroup，根据ViewGroup的绘制流程实现自己需要的效果。

3.利用已有的UI控件(RelativeLayout,LinearLayout,FrameLayout....)，继承改造或者组合使用等。

接下来就以上的方法，一个一个实践来练习吧！

##自定义UI（利用View）
&emsp;利用View实现自定义主要是继承View并利用OnDraw(Canvas canvas)方法绘制自己需要的UI，利用canvas主要可以绘制矩形(包括圆角矩形)、圆、点、线、扇形等，同时还可以根据类[Path](http://developer.android.com/reference/android/graphics/Path.html)设置需要的路径，利用Canvas将Path绘制出来。具体的使用通过实践学习吧。这里我们还要关心View中的OnMeasure（）方法，这个方法主要是设定我们当前这个View的大小。关于View中的这些方法的分析可以看看[这篇文章](http://yifeiyuan.me/2015/10/12/%E8%87%AA%E5%AE%9A%E4%B9%89View%E7%9A%84onMeasure%E3%80%81onLayout/)，分析的比较详细。好了，还是实战吧（在实战中遇到问题不断的解决，自然对很多东西就了解了，总是看理论的东西还是理解的不透彻哦），今天利用View实现如下的效果(gif 效果不是很好0_0)：
  
![alt tag](/Resource/jianshu.gif)
  
###分解效果图
&emsp;任何复杂的事情都要学会分解成多个简单的事情去处理。动画效果的某个瞬间的静态UI如下所示：
  
![alt tag](/Resource/static.png)

通过这个静态的UI，可以看出它的组成很简单：一个大圆，一个实心的小圆————这个小圆的中心点是在大圆的圆环上。这样就把这个静态的UI分解为：一个大圆和一个小圆。

接下来想想怎么让实心小圆动起来呢？前面说了，小圆的中心在大圆的圆环上，如果这个中心不停的沿着大圆的圆环上走，不就动起来的。怎么沿着走呢，看看下面的图！

![alt tag](/Resource/location.jpeg)

上图我们绘制了小球（实心小圆）的中心位置，这里我们选取的顺时针方向为正方向，并且0度角为X轴正方向。这里选取了45度角的一个静态分析，假设定义一个变量为angle，当这个angle不断的从0到360变化时，小球就是沿着大圆跑了。根据上面的图和前面的分析，应该比较清晰的知道原理了吧^_^！
  
###实现分解效果图
&emsp;这里将实现上面分解的结果啦，应该很简单的！

1.绘制圆环

&emsp;在onDraw(Canvas canvas)这个方法中利用canvas绘制，直接上代码：

    private void drawBigCircle(Canvas canvas){
        canvas.drawCircle(rectF.centerX(), rectF.centerY(), getBigCircleRadius(), mCirclePaint);
    }
    
（这里的rect变量主要是利用Rect累框定绘制圆的范围）这里主要是利用了canvas的方法(当然，绘制circle的方法还有其他几个方法可以调用，根据自己的需求):

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

这里的第一步是计算角度，第二是计算实心小圆的中心位置，这两步根据上面分解章节的数学计算应该很好理解的。第三步是绘制实心小球，这里实现实心是通过Paint的方法setStyle(Paint.Style.FILL)设置充满整个圆;


接下来就是绘制这个“小球”啦。首先移动画布(改变坐标系的中心位置):在计算小球的中心位置(cx,cy)的时候，是根据大圆的半径计算的，所以这里的(cx,cy)是到大圆中心点的距离。因此，Canvas的起始位置应该移动到大圆的中心位置。

这里首先要保存一下画布，在绘制结束了还要恢复到原来的状态，为什么？因为在绘制之前我们需要对画布进行移动，当绘制结束的时候如果不恢复到原来的状态，Canvas将一直保持在目前的状态，而后面的有可能是需要在没有移动的画布上绘制其他图案，这时候的中心位置改变了，就会存在问题。

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

### 总结
&emsp;&emsp;上面一步一步介绍了自定义UI的过程，现在应该对这个过程有所了解了——其实主要还是一个模型的建立过程。这里简单总结一下：

1.一点一点的分解UI，把复杂的界面分解成一个一个静态、简单的界面，并实现它们。

2.当静态界面绘制完了，需要动画的——观察需要改变哪些参数，并且不断的触发View绘制，不就动起来了^_^。

3.一些比较复杂的动画或者效果就需要一些物理、数学方面的知识来辅助了——塞贝尔曲线等，其实这些就需要花费你一些时间去研究了（但是，还是那句话，从简单到复杂）。

4.多多的练习，只有练习你才能对自定义有更深的理解。

这里放一个简单的自定义UI，让大家自己实现吧(这里是静态的，大家也可以让它动起来0_0)！

![alt tag](/Resource/practice.png)

PS:后面我们将介绍如何利用ViewGroup和RLF...(RelativeLayout LinearLayout FrameLayout....)自定义。


##  自定义UI（利用Android现有的RelativeLayout、LinearLayout、FrameLayout等）

&emsp;&emsp;我们利用现有的UI控件，主要是利用它们的一些属性，并且根据这些属性的改变可以，达到我们预期的效果。还是看看今天我们实现的效果吧，No picture，it's so hard。效果图如下所示，就是我们常见的Tab和SeekBar，看看今天怎么用现有的UI控件实现它；老样子我们还是来一步一步分析吧。


![alt tag](/Resource/slidprogressbar.gif)

### 分解效果图

&emsp;我们看到Horizontal和Vertical两个Tab是不可以滑动的，只有通过点击来触发左右切换，在Tab的下面会有一个带颜色的bar标志当前选中的位置。想这样的UI目前现有的Android控件应该没有可以直接使用的(据我的了解)。一般我们在实现这样的UI时，我们会用TextView和一个带颜色的View组合实现。这样好像也可以，但是还是有一定的代码量的。那么我们怎么用最少的代码实现这样的需求呢！首先我们看到左右切换的选中，类似于RadioButton这样的控件——在多个选项中只能选取其中一个，但是RadioButton这样的控件底部好像也没有这样的带颜色的bar啊，难道还是要利用一个View和它组合使用吗，那这样还是太low了。我们想啊，既然底部没有这样一个bar，那我们可以让RadioButton在底部画一个啊。怎么画？那当然是继承它，在[onDraw()](http://developer.android.com/training/custom-views/custom-drawing.html)方法里面画啦。具体怎么画，我相信画一个矩形应该很简单吧^_^。

&emsp;上面分析完了Tab的部分，这下我们来看看这个Seekbar吧，我们先看看下面分解的4张图片。

![alt tag](/Resource/first_bar.png)

![alt tag](/Resource/second_bar.png)

![alt tag](/Resource/thumb.png)

![alt tag](/Resource/seekbar.png)

从上面的四张图片可以看出，这个Seekbar是通过前面三张一个一个叠加过后达到第四张图片的效果。当我们拖动小thumb(小圆球)滑动的时候，不断的改变第二层上圆角矩形的宽度就可以达到想要的效果了。那么具体怎么实现，当然还是上代码啦。

###  实现分解效果图

1.实现Tab的切换效果

&emsp;上面我们分析了，需要实现Tab底部的效果主要是继承RadioButton，并在onDraw()方法中绘制一个矩形就可以了，还是直接上代码吧。

        @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        int height = getMeasuredHeight();
        //the indicator's height
        int indicatorHeight = getResources().getDimensionPixelSize(R.dimen.radio_button_indicator_height);
        if (isChecked()){
            mPaint.setColor(getCurrentTextColor());
        }else {
            mPaint.setColor(Color.TRANSPARENT);
        }
        canvas.drawRect(0, height - indicatorHeight, getMeasuredWidth(), height, mPaint);
    }

这里我们设置了选中色块的高度，再调用drawRect()方法绘制矩形（选中色块）。这里简单介绍一个这个方法

    drawRect(float left, float top, float right, float bottom, @NonNull Paint paint)

先看看下面的图解吧。

![alt tag](/Resource/rect.jpeg)  

结合上面的图解，我们这里把top的值设为height - indicatorHeight，这个height为整个RadioButton的高度，由图应该可以很清楚的知道top的坐标计算方法了。我们在RadioButton选中的时候通过Paint的值设置选中颜色，当处于未选中状态时设置为透明色，并且绘制同一块区域，达到切换的效果。

2.实现Seekbar

&emsp;在分析Seekbar的时候，我们把它分解成了几个层次的叠加，那接下来我们的任务就是实现这些层次的叠加。我们这部分是通过现有的UI控件实现这个效果的，那么我们用哪一个UI控件可以实现层次的叠加呢，那当然只有FrameLayou和RelativeLayout啦！那到底用FrameLayout还是RelativeLayout呢？这里我们使用RelativeLayout，对于FrameLayout的使用可以自行实践。

(1)实现第一层圆角——progressbar

![alt tag](/Resource/first_bar.png)

&emsp;那我们这一层用什么实现呢，这里我们可以使用Andorid UI控件中的LinearLayout、FrameLayou等都可以，但是我们这里使用ViewGroup的子类——LinearLayout(当然也可以选取其他的)，这样我们可以对这一层进行扩展，在里面添加TextView、ImageView等。那么怎么实现圆角呢，当时是使用drawble进行配置的，但是对于使用drawable进行配置是有问题的——不能代码控制圆角的大小，这个问题导致可扩展性太差。那怎么解决呢，当然得想办法啊，最后找到了使用[GradientDrawable](http://developer.android.com/reference/android/graphics/drawable/GradientDrawable.html)，这里我想说的就是，我们在自定义的过程中总会出现问题，然后不停的找到具体的解决方法，一步一步实现。其实其他的编程问题都是这样的。好了，我们这里还是直接看看具体代码吧，如下：

        mFirstBar = new LinearLayout(context);
        GradientDrawable drawable = new GradientDrawable();
        drawable.setColor(Color.parseColor("#FFBB33"));
        drawable.setCornerRadius((float) mProgressHeight / 2);
        mFirstBar.setBackgroundDrawable(drawable);
//        mFirstBar.setBackgroundResource(R.drawable.firtbar_bkg);

        mFirstBarLp = new RelativeLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, mProgressHeight);
        mFirstBarLp.addRule(CENTER_IN_PARENT);
        mFirstBar.setClickable(false);
        addView(mFirstBar, mFirstBarLp);

好了，第一层的实现应该很简单吧。

(2)实现第二层圆角——secondProgressbar

![alt tag](/Resource/second_bar.png)

&emsp;其实secondProgressbar的实现基本上同progressbar的实现类似，还是直接上代码吧

        mSecondBar = new LinearLayout(context);
        GradientDrawable secondDrawable = new GradientDrawable();
        secondDrawable.setColor(Color.parseColor("#99CC00"));
        secondDrawable.setCornerRadius((float)mProgressHeight/2);
        mSecondBar.setBackgroundDrawable(secondDrawable);
//        mSecondBar.setBackgroundResource(R.drawable.secondbar_bkg);
        mSecondBarLp = new RelativeLayout.LayoutParams(mThumbRadius, mProgressHeight);
//        mSecondBarLp.leftMargin = 0;
        mSecondBarLp.addRule(CENTER_VERTICAL);
//        mSecondBarLp.addRule(LEFT_OF, THUMB_ID);
        mSecondBarLp.addRule(ALIGN_PARENT_LEFT);
        addView(mSecondBar, 1, mSecondBarLp);
        mSecondBar.setClickable(false);

这里我们看到了，在等一LayoutParams的时候我们设置的width的大小为圆角半径的大小，为什么要这么做？这里还是先解释一下吧，这样做的目的主要是将secondProgressbar的最右端总是在thumb的中心位置，这里记住这一点后面我们在介绍。

(3)实现游标——thumb

&emsp;其实thumb很简单啦，就是一个圆形的View。这里直接使用TextView，不要问我为什么——我任性$_$。直接看代码吧。

        mThumb = new TextView(context);
        GradientDrawable thumb = new GradientDrawable();
        thumb.setColor(Color.parseColor("#33b5e5"));
        thumb.setCornerRadius((float)mThumbRadius);
        mThumb.setBackgroundDrawable(thumb);
        mThumbLp = new RelativeLayout.LayoutParams(mThumbRadius*2, mThumbRadius*2);
        mThumbLp.addRule(CENTER_VERTICAL);
        mThumbLp.addRule(ALIGN_PARENT_LEFT);
        mThumbLp.leftMargin = 0;
        addView(mThumb, mThumbLp);
        mThumb.setId(THUMB_ID);
        mThumb.setGravity(Gravity.CENTER);
        if (mThumbTextSize != 0)
        mThumb.setTextSize(mThumbTextSize);

这里在我们使用TextView，我们还可以在上面显示一些信息，不是很好！

(4)实现滑动——Seekbar

&emsp;好了，上面我们分解的图都实现了，现在应该要实现滑动了吧！那么怎么实现按住Thumb就会滑动，并且secondProgressbar和滑动同时显示当前的进度。当然是使用监听OnTouchEvent事件啦，根据滑动的distance不断更新thumb和secondProgressbar的参数让他们动起来。那这里就直接实现RelativeLayout的OnTouchEvent方法，还是先看代码吧，如下：

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        int action = MotionEventCompat.getActionMasked(event);
        boolean isDraged = false;
        Rect rect = new Rect();
        mThumb.getHitRect(rect);
        switch (action){
            case MotionEvent.ACTION_DOWN:
                float x = event.getX();
                float y = event.getY();
                boolean contain = rect.contains((int)x, (int)y);
                if (contain){
                    mLastMotionX = event.getX();
                    isDraged = true;
                }
                break;
            case MotionEvent.ACTION_MOVE:
                dragThumb(event.getX());
                break;

            case MotionEvent.ACTION_UP:

                break;
        }
        return isDraged;
    }

    private void dragThumb(float x){
        float distance = (x - mLastMotionX);
        mLastMotionX = x;
        mThumbLp.leftMargin = (int) (mThumbLp.leftMargin + distance);
        mSecondBarLp.width = (int) (mSecondBarLp.width + distance);
        LogUtils.LogD(TAG, " horizontal current distance == " + distance);
        //confirm this thumb is show, no anywhere is hide
        if (mThumbLp.leftMargin <= 0) {
            mThumbLp.leftMargin = 0;
            mSecondBarLp.width = mThumbRadius;
        } else if (mThumbLp.leftMargin >= getMeasuredWidth() - mThumbRadius * 2) {
            mThumbLp.leftMargin = getMeasuredWidth() - mThumbRadius * 2;
            mSecondBarLp.width = getMeasuredWidth() - mThumbRadius;
        }
        updateViewLayout(mThumb, mThumbLp);
        updateViewLayout(mSecondBar, mSecondBarLp);
    }

这里我们实现的是整个RelativeLayout的OnTouchEvent方法，所以它的touch事件是针对整个RelativeLayout的。所以这里我们要做一下过滤，点击范围在不在thumb上面，只有点击和拖动都在thumb上面这次的touch对thumb才有效。当确定拖动有效的时候，在开始初始化的时候，设置了thumb相对于父控件为ALIGN_PARENT_LEFT，所以通过改变mThumbLp.leftMargin就可以改变thumb于左边的距离啦。对于secondProgressbar，只要改变mSecondBarLp.width的大小就可以改变它的宽度，最后调用updateViewLayout()方法更新UI。这里我们要注意两点：1.防止thumb滑动到最右端时超出边界。2.防止thumb滑动回来到最左端时超出边界。好了，到这里通过简单的介绍将这个Seekbar的基本功能完成。

### 总结

&emsp;上面我们利用Android控件实现了一个简单的Seekbar，现在简单的总结一下：

1.和自定义View的时候一样，我们还是把这个Seekbar进行了分解，然后一步一步实现。

2.在自定义的过程中我们会遇到很多问题，我在这里遇到了这些问题：圆角怎么可以用代码控制、在开始的时候我没有利用RelativeLayou的onTouchEvent方法实现滑动，而是直接将onTouchEvent事件直接set在thumb上面，结果滑动的时候出现了问题（有兴趣的可以自己试试看看是什么问题）....。这里面遇到了很多问题，但都一个一个击破，所以我们在自定UI的时候不要心急，一点一点将没有问题解决，最后就会实现。

3.这里只是通过这个例子分析怎样去利用Android UI 自定义我们自己需要的UI。这里只是引导，更多的还是靠实践、实践、实践...。重要的事说三遍^_^

好了，国际惯例，可以自己练习一下垂直方向的Seekbar，如下图：

![alt tag](/Resource/vertical.png)