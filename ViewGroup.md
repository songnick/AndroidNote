###前面我们介绍了利用View和Android已有的控件RLF...(RelativeLayout、LinearLayout、FrameLayout...)实践自定义UI，感兴趣的小伙伴请移步：

[实践自定UI—View](http://www.jianshu.com/p/11210b14f743)

[实践自定义UI—RLF...(RelativeLayout LinearLayout FrameLayout....)](http://www.jianshu.com/p/ff8dcefce371)

&emsp;接下来我们将利用ViewGroup实践自定义UI，首先还是看看效果图：



&emsp;这个效果是来源于Keep_Growing群里面的一个小伙伴，好像是在项目中需要，问有没有开源的，后来我发现好像还真的没有（如果你知道，请告诉我），所有就想着自己利用ViewGroup实现这个效果。这里利用ViewGroup自定义UI控件，我们主要是注意一下下面两点：

1.定义规则、属性：定义一下布局规则，类似于LinearLayout中的orientation、RelativeLayout中的alignParentLeft等。这些规则主要是告诉我们这些子View如何放置他们的位置，以及如何设置大小等属性。

2.处理交互事件：主要是触摸事件的处理。

##分解效果图

&emsp;我们从上面的效果图可以很清晰的发现，ViewGroup的子View在滑动的时候，是可以放大和缩小的。那么我们的主要任务之一就是解决这个放大和缩小的效果。我们看一下进入界面的效果如下图：


从这个静态的页面可以看到，就是两个View，其中第二个View我们可以认为只是按照一定的比例缩小了。如果在没有按比例缩小的状态下，他们的布局是如下图所示：


根据上面的分析，我们可以这么想象，在ViewGroup中我们添加的一定数量的子View，并且第一个View保持原始大小，剩下的View按一定比例缩小。示意图如下所示：


在滑动的过程中，假如从右向左滑动，那么当前的View会逐渐缩小，下一个View会逐渐放大；假如从左向右滑动，当前的View会逐渐缩小，上一个View会逐渐放大(可以参考效果图理解)。

##实现分解效果图

&emsp;根据上面的分解我们来一步一步实现。

1.测量大小和布局
&emsp;这里我们定义两个属性：marginLeftRight和gutterSize，其中marginLeftRight是确定子View与left和right的间距，gutterSize是确定原始大小View与缩小View之间的距离。知道这两个属性后我们首先要确定每个View的大小，我们知道这个过程是在onMeasure()方法中完成的(其实onMeasure()方法就是确定当前ViewGroup和子View大小的地方，我们自定义View和ViewGroup都是一样的)，这里还是直接看代码吧：

	@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // For simple implementation, our internal size is always 0.
        // We depend on the container to specify the layout size of
        // our view.  We can't really know what it is since we will be
        // adding and removing different arbitrary views and do not
        // want the layout to change as this happens.
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

&emps;每个子View的大小确定了，接下来的工作就是确定他们在当前ViewGroup中的位置，这个工作当然由onLayout()方法来确定啦，还是直接看代码吧：

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

这里是根据gutterSize的大小占用整个子View宽度大小的比例，就是缩小的比例。

2.滑动效果

&emps;上面我们简单的将测量大小和布局的过程介绍了一下，接下来的工作就是左右滑动的效果实现了，以及处理好滑动过程中的放大和缩小的效果。为了会实现这个效果我们这里简单的介绍一下需要使用到的类和方法。

(1)Scroller

&emps;滑动的过程我们用到了Scroller这个类，它的主要作用是配合computeScroll()，让子View滑动到固定的位置。我们先看看Scoller中我们需要使用的方法：

	startScroll(int startX, int startY, int dx, int dy, int duration)


这个方法主要的功能是模拟在duration的时间内，在X轴方向上从startX的位置(这里我们只关心X方向，Y方向类似)移动了dx的距离。在这个移动的过程中通过getCurrX() 获取当前移动到的位置(其实这里大家可以自己查一下这个累的具体用法)。

(2)VelocityTracker

&emps;这个类的主要作用就是检测手势滑动的速度。我们滑动View的时候会有一定的速率，当达到一定的速率时我们切换子View。

(3)scrollBy(int x, int y)方法、scrollTo(int x, int y)方法和computeScroll()方法

&emps;scrollBy()方法是在X轴上移动距离为x和Y轴上移动距离为y；scrollTo()方法是移动到(x, y)的位置；computeScroll()方法在我们需要View进行重绘时，就会触发该方法。当我们需要在规定时间内将View从某个位置滑动到每个固定位置时，可以通过Scroller类模拟这个过程，并通过scrollTo方法配合使用，就可以达到View移动的效果。

&emps;接下来我们将利用上面我们介绍的方法实现滑动的过程。我们实现滑动的效果，肯定需要处理Touch事件













