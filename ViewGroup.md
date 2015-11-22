 前面我们介绍了利用View和Android已有的控件RLF...(RelativeLayout、LinearLayout、FrameLayout...)实践自定义UI，感兴趣的小伙伴请移步：

[实践自定UI—View](http://www.jianshu.com/p/11210b14f743)

[实践自定义UI—RLF...(RelativeLayout LinearLayout FrameLayout....)](http://www.jianshu.com/p/ff8dcefce371)

接下来我们将利用ViewGroup实践自定义UI，首先还是看看效果图：


![效果图](http://upload-images.jianshu.io/upload_images/1098335-db1cfb6fad7fcdef.gif?imageMogr2/auto-orient/strip)

 这个效果是来源于Keep_Growing群里面的一个小伙伴，好像是在项目中需要，问有没有开源的，后来我发现好像还真的没有（如果你知道，请告诉我），所有就想着自己利用ViewGroup实现这个效果。这里利用ViewGroup自定义UI控件，我们主要是注意一下下面两点：

1.定义规则、属性：定义一下布局规则，类似于LinearLayout中的orientation、RelativeLayout中的alignParentLeft等。这些规则主要是告诉我们这些子View如何放置他们的位置，以及如何设置大小等属性。

2.处理交互事件：主要是触摸事件的处理。

##分解效果图

 我们从上面的效果图可以很清晰的发现，ViewGroup的子View在滑动的时候，是可以放大和缩小的。那么我们的主要任务之一就是解决这个放大和缩小的效果。我们看一下进入界面的效果如下图：


![静态图](http://upload-images.jianshu.io/upload_images/1098335-4cbba0da820f1f61.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从这个静态的页面可以看到，就是两个View，其中第二个View我们可以认为只是按照一定的比例缩小了。根据上面的分析，我们可以这么想象，在ViewGroup中我们添加的一定数量的子View，并且第一个View保持原始大小，剩下的View按一定比例缩小。他们的布局如下图所示：



![示意图](http://upload-images.jianshu.io/upload_images/1098335-013ca24c133c7710.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在滑动的过程中，假如从右向左滑动，那么当前的View会逐渐缩小，下一个View会逐渐放大；假如从左向右滑动，当前的View会逐渐缩小，上一个View会逐渐放大(可以参考效果图理解)。

##实现分解效果图

 根据上面的分解我们来一步一步实现。

1.测量大小和布局
 为了布局和设置大小的需要，这里我们定义两个属性：marginLeftRight和gutterSize，其中marginLeftRight是确定子View与left和right的间距，gutterSize是确定原始大小View与缩小View之间的距离。知道这两个属性后我们首先要确定每个View的大小，我们知道这个过程是在onMeasure()方法中完成的(其实onMeasure()方法就是确定当前ViewGroup和子View大小的地方，我们自定义View和ViewGroup都是一样的)，这里还是直接看代码吧：

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(0, widthMeasureSpec),
                getDefaultSize(0, heightMeasureSpec));
        int measuredWidth = getMeasuredWidth();
        int measuredHeight = getMeasuredHeight();

        int childCount = getChildCount();
        int width = measuredWidth - (int) (mMarginLeftRight * 2);
        int height = measuredHeight;
        int childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width, MeasureSpec.EXACTLY);
        int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height, MeasureSpec.EXACTLY);
        for (int i = 0; i < childCount; i++) {
            getChildAt(i).measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
        mSwitchSize = width;
        confirmScaleRatio(width, mGutterSize);
    }

这里首先设置的当前ViewGroup的大小，然后确定每个子View的大小。子View的高度是和ViewGroup的高度相同的，子View的宽度是需要减去刚才设置与两边的间距，并调用child.measure()方法确定子View的大小。

 当前ViewGroup的大小和每个子View的大小确定了，接下来的工作就是确定他们在当前ViewGroup中的位置，这个工作当然由onLayout()方法来确定啦，还是直接看代码吧：

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int childCount = getChildCount();
        int originLeft = (int) mMarginLeftRight;

        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            int left = originLeft + child.getMeasuredWidth() * i;
            int right = originLeft + child.getMeasuredWidth() * (i + 1);
            int bottom = child.getMeasuredHeight();
            child.layout(left, 0, right, bottom);
            if (i != 0) {
                child.setScaleX(SCALE_RATIO);
                child.setScaleY(SCALE_RATIO);
            }
        }

    }


其实这个位置确定的过程可以参考上面的示意图，首先按照原始的大小将每个子View通过调用child.layout()方法告诉他们在当前ViewGroup中的位置，他们在绘制自己的时候就会在给定的区域内绘制。当这些子View都确定位置时，他们是一个挨着一个的，并没有缩小的效果图，我们调用child.setScaleX()和child.setScaleY()两个方法设置缩放的大小，那么当child在绘制的时候就会缩小。这里我们怎么知道缩小多少呢，还是看看代码：

    private void confirmScaleRatio(int width, float gutterSize) {
        SCALE_RATIO = (width - gutterSize * 2) / width;
    }

这里是根据gutterSize的大小占用整个子View宽度大小的比例，就是缩小的比例，如果不是很理解这个计算方法，可以参考下图理解一下：


![计算示意图](http://upload-images.jianshu.io/upload_images/1098335-fd42149e106d24cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.滑动效果

 上面我们简单的将测量大小和布局的过程介绍了一下，接下来的工作就是左右滑动的效果实现了，以及处理好滑动过程中的放大和缩小的效果。为了会实现这个效果我们这里简单的介绍一下需要使用到的类和方法。

(1)Scroller
 滑动的过程我们用到了Scroller这个类，它的主要作用是配合computeScroll()，让子View滑动到固定的位置。我们先看看Scoller中我们需要使用的方法：

    startScroll(int startX, int startY, int dx, int dy, int duration)

这个方法主要的功能是模拟在duration的时间内，在X轴方向上从startX的位置(这里我们只关心X方向，Y方向类似)移动了dx的距离。在这个移动的过程中通过getCurrX() 获取当前移动到的位置(其实这里大家可以自己查一下这个累的具体用法)。

(2)VelocityTracker
 这个类的主要作用就是检测手势滑动的速度。我们滑动View的时候会有一定的速率，当达到一定的速率时我们切换子View。

(3)scrollBy(int x, int y)方法、scrollTo(int x, int y)方法和computeScroll()方法
 scrollBy()方法是在X轴上移动距离为x和Y轴上移动距离为y；scrollTo()方法是移动到(x, y)的位置；computeScroll()方法在我们需要View进行重绘时，就会触发该方法。当我们需要在规定时间内将View从某个位置滑动到某个固定位置时，可以通过Scroller类模拟这个过程，并通过scrollTo方法配合使用，就可以达到View移动的效果。

 接下来我们将利用上面介绍的方法实现滑动的效果。实现滑动的效果，肯定是对Touch事件的处理，还是直接看代码：

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        LogUtils.LogD(TAG, " onInterceptTouchEvent hit touch event");
        final int actionIndex = MotionEventCompat.getActionIndex(event);
        mActivePointerId = MotionEventCompat.getPointerId(event, 0);

        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(event);
        switch (event.getAction() & MotionEvent.ACTION_MASK) {
            case MotionEvent.ACTION_DOWN:
                mDownX = event.getRawX();
                if (mScroller != null && !mScroller.isFinished()) {
                    mScroller.abortAnimation();
                }
                break;
            case MotionEvent.ACTION_MOVE:

                //calculate moving distance
                float distance = -(event.getRawX() - mDownX);
                mDownX = event.getRawX();
                LogUtils.LogD(TAG, " current distance == " + distance);
                performDrag((int)distance);
                break;
            case MotionEvent.ACTION_UP:
                releaseViewForTouchUp();
                cancel();
                break;
        }
        return true;
    }

    private void performDrag(int distance) {
        if (mOnPagerChangeListener != null){
            mOnPagerChangeListener.onPageScrollStateChanged(SCROLL_STATE_DRAGGING);
        }
        LogUtils.LogD(TAG, " perform drag distance == " + distance);
        scrollBy(distance, 0);
        if (distance < 0) {
            dragScaleShrinkView(mCurrentPosition, LEFT_TO_RIGHT);
        } else {
            LogUtils.LogD(TAG, " current direction is right to left and current child position =  " + mCurrentPosition);
            dragScaleShrinkView(mCurrentPosition, RIGHT_TO_LEFT);
        }
    }

这里处理的是在滑动的时候child的变化，当然最主要的就是放大缩小的变化，由于代码比较多这里就简单的分析一下计算过程。我们知道放大过程就是从SCALE_RATIO变化到1.0，缩小的过程就是从1.0变化到SCALE_RATIO，所以移动过程中计算方法如下：

    shrinkRatio = SCALE_RATIO + (1.0f - SCALE_RATIO) * ratio;
    scaleRatio = 1.0f - (1.0f - SCALE_RATIO) * ratio;

这里的ratio是指移动距离占整个切换一个页面需要移动距离的比例。这个值的变化范围为：0-1。定义切换一个页面需要移动的距离为mSwitchSize，当前处于原始大小child的位置为position，当我们向左滑动的时候(向右滑动的过程大家可以试着算一下)，计算的过程为：

    int moveSize = getScrollX() - position * mSwitchSize;
    float ratio = (float) moveSize / mSwitchSize;

这个计算的过程估计会有点难理解，大家还是自己想象一下滑动的过程，这样便于理解。以上是滑动过程中的变化，用户一直处于滑动的状态。当用户松手之后，那么我们需要根据滑动的速率和当前移动的距离是否超过mSwitchSize的一半，判断是否切换页面。

      private void releaseViewForTouchUp() {

        final VelocityTracker velocityTracker = mVelocityTracker;
        velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
        int initialVelocity = (int) VelocityTrackerCompat.getXVelocity(
                velocityTracker, mActivePointerId);
        float xVel = mVelocityTracker.getXVelocity();
        if (xVel > SNAP_VELOCITY && mCurrentPosition > 0) {
            smoothScrollToItemView(mCurrentPosition - 1, true);
        } else if (xVel < -SNAP_VELOCITY && mCurrentPosition < getChildCount() - 1) {
            smoothScrollToItemView(mCurrentPosition + 1, true);
        } else {
            smoothScrollToDes();
        }
        setScrollState(SCROLL_STATE_SETTLING);
    }

    private void smoothScrollToDes() {
        int scrollX = getScrollX();
        //confirm the position to scroll
        int position = (scrollX + mSwitchSize / 2) / mSwitchSize;
        LogUtils.LogD(TAG, " smooth scroll to des position == before =" + mCurrentPosition
                + " scroll X = " + scrollX + " switch size == " + mSwitchSize + " position == " + position);
        smoothScrollToItemView(position, mCurrentPosition == position);
    }

    private void smoothScrollToItemView(int position, boolean pageSelected) {
        mCurrentPosition = position;
        if (mCurrentPosition > getChildCount() - 1) {
            mCurrentPosition = getChildCount() - 1;
        }
        if (mOnPagerChangeListener != null && pageSelected){
            mOnPagerChangeListener.onPageSelected(position);
        }
        int dx = position * (getMeasuredWidth() - (int) mMarginLeftRight * 2) - getScrollX();
        mScroller.startScroll(getScrollX(), 0, dx, 0, Math.min(Math.abs(dx) * 2, MAX_SETTLE_DURATION));
        invalidate();
    }

当调用startScroll方法后会调用invalidate()方法，这个过程就会触发computeScroll()方法，我们看看在该方法中我们怎么处理滑动的效果吧，直接看代码：

    @Override
    public void computeScroll() {
        if (!mScroller.isFinished() && mScroller.computeScrollOffset()) {
            dragScaleShrinkView(mCurrentPosition, mCurrentDir);
            scrollTo(mScroller.getCurrX(), 0);
        }

    }


上面我们说了，startScroll方法只是模拟移动的过程，通过模拟的过程我们可以在duration的时间内获取移动到的位置(getCurrX()方法获取)，正真的移动效果还是通过scrollTo()方法实现的，由于我们需要不停的获取和移动，所以就需要在模拟的时间内不停的调用scrollTo方法，达到滑动的效果，这个这个过程中同时实现放大缩小的过程。
&emsp;好了，上面我基本上把需要实现了滑屏以及滑动过程中放大缩小的效果了，这个过程其实涉及的东西还是蛮多的，也比较繁琐，不过不是非常的难。只要仔细的理解每一个过程，还是比较容易理解的。主要还是多多练习！