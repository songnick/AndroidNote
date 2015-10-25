#UINote
  由于当前IOS和Android两个系统的交互设计相互渗透，现在很多时候我们都需要自己去自定义一些比较炫酷或者复杂的UI。其实我们自定义UI是可以通过这三种方法实现：
1.继承View，直接在Canvas上面绘制。
2.继承ViewGroup，根据ViewGroup的绘制流程实现自己需要的效果。
3.利用已有的UI控件(RelativeLayout,LinearLayout,FrameLayout....)。

##自定义UI（利用View）
  前面简单的介绍了自定义UI控件实现的方法，这部分主要说一下如何利用View实现自定UI控件。上面说了，利用View实现主要是继承View在OnDraw(Canvas canvas)方法中绘制。利用canvas主要可以绘制矩形(包括圆角矩形)、圆、点、线、扇形等，同时还可以根据类[Path](http://developer.android.com/reference/android/graphics/Path.html)设置需要的路径，利用Canvas将Path绘制出来。具体的使用通过实践学习吧。今天利用View实现如下的效果（这里实现第二个效果）：
  
  ![alt tag](/Resource/my_implement.gif)
  
###分解效果图
  动画效果的某个静态的界面如下所示，可以看出是一个圆环和实心圆组合而成的。那么现在第一步要做的就是把这个静态页面绘制出来。
  1.绘制圆环
  
  
