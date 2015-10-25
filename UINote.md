#UINote

UINote is to record some information about android UI. Of cource, contains implement UI by self and some question.

由于当前IOS和Android两个系统的交互设计相互渗透，现在很多时候我们都需要自己去自定义一些比较炫酷或者复杂的UI。其实我们自定义UI是可以通过这三种方法实现：
  1.继承View，直接在Canvas上面绘制。
  2.继承ViewGroup，根据ViewGroup的绘制流程实现自己需要的效果。
  3.利用已有的UI控件(RelativeLayout,LinearLayout,FrameLayout....)。

##自定义UI（利用View）
  前面简单的介绍了自定义UI控件实现的方法，这部分主要说一下如何利用View实现自定UI控件。上面说了，利用View实现主要是继承View在OnDraw()方法中绘制。
  
  
