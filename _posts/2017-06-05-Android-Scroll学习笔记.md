---
title: "Android Scroll学习笔记"
date: 2017-06-05
categories:
  - blog
tags: []
---

​友球要写一个跟纸片一样的能够翻转过来跟纸片一样的自定义View，所以我就去研究了一下View的Animation和滑动、拖拽。这篇就记录一下学习《Android群英传》里的滑动部分。

## <a href="#Key-Points" class="headerlink" title="Key Points"></a>Key Points

滑动的实现：**不断地改变View的坐标**

Android坐标系: **左上角为原点，水平向右为X轴正方向，垂直向下为Y轴正方向**

视图坐标系: 描述子视图在父视图中的位置关系，子视图的坐标以**父视图左上角为原点**

### <a href="#触控事件—MotionEvent" class="headerlink" title="触控事件—MotionEvent"></a>触控事件—MotionEvent

MotionEvent中封装的常用事件常量:

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7</code></pre></td>
<td class="code"><pre><code>public static fianl int ACTION_DOWN = 0; // 单点触摸 按下
public static fianl int ACTION_UP = 1; // 单点触摸 离开
public static fianl int ACTION_MOVE = 2; // 触摸点 移动
public static fianl int ACTION_CANCEL = 3; // 触摸动作 取消
public static fianl int ACTION_OUTSIDE = 4; // 触摸动作 超出边界
public static fianl int ACTION_POINTER_DOWN = 5; // 多点触摸 按下
public static fianl int ACTION_POINTER_UP = 6; // 多点触摸 离开</code></pre></td>
</tr>
</tbody>
</table>
</figure>

实现触摸实现的代码套路:在`onTouchEvent(MotionEvent event)`中通过`event.getAction()`获取触控事件类型，并且用switch-case语句来筛选我们所需要的动作。

<span id="more"></span>

#### <a href="#触摸事件的代码套路" class="headerlink" title="触摸事件的代码套路"></a>触摸事件的代码套路

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18</code></pre></td>
<td class="code"><pre><code>@Override 
public boolean onTouchEvent(MotionEvent event) {
  // 获取触摸事件坐标
   int x = (int) event.getX();
   int y = (int) event.getY();
  switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
               // 处理按下
                break;
            case MotionEvent.ACTION_MOVE:
                // 处理移动
                break;
         case MotionEvent.UP:
                // 处理离开
                break;
        }
        return true;
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

Android提供的获取一系列坐标的方法示意图:

![PositionMethods](/img/position_methods.png)

以上方法分成两类:

- View提供的获取坐标方法
  1.  getTop() : 获取View自身顶边到其父布局顶边的距离
  2.  getLeft(): 获取View自身左边到其父布局左边距离
  3.  getBottom(): 获取View自身的底边到父布局顶边的距离
  4.  getRight(): 获取View自身的右边到父布局左边的距离
- MotionEvent提供的方法
  1.  getX(): 获取点击事件距离**控件**左边的距离，即视图坐标
  2.  getY(): 获取点击事件距离**控件**顶边的距离，即视图坐标
  3.  getRawX(): 获取点击事件距离**整个屏幕**左边的距离，即绝对坐标
  4.  getX(): 获取点击事件距离**整个屏幕**顶边的距离，即绝对坐标

只要牢记**坐标获取都是以左/顶边为参考的**即可，配合图片理解就好了。

## <a href="#实现滑动的若干种方法" class="headerlink" title="实现滑动的若干种方法"></a>实现滑动的若干种方法

​接下来就用这些系统提供的API来实现一个动态的修改一个View的坐标，也就是实现滑动效果。不管哪种方式，实现的思路是一样的：

​**触摸View时，系统记录下此时坐标；手指一动时，记录下移动后的触摸点坐标，获取到偏移量，并通过偏移量来修改View坐标；不断重复，就实现了滑动**

​首先，自定义一个DragView继承自View，重写Constructor方法，并且有两个私有成员准备用来记录上一次的坐标。

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8</code></pre></td>
<td class="code"><pre><code>public class DragView extends View {
    private int lastX = 0;
    private int lastY = 0;
  
    public DragView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

​写一个简单的布局，把我们的DragView设成100\*100，并且让他填上背景色方便观察。

<figure class="highlight xml">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12</code></pre></td>
<td class="code"><pre><code>&lt;?xml version=&quot;1.0&quot; encoding=&quot;utf-8&quot;?&gt;
&lt;LinearLayout
    xmlns:android=&quot;http://schemas.android.com/apk/res/android&quot;
    xmlns:tools=&quot;http://schemas.android.com/tools&quot;
    android:layout_width=&quot;match_parent&quot;
    android:layout_height=&quot;match_parent&quot;
    tools:context=&quot;top.tobiaslee.viewtest.ScrollActivity&quot;&gt;
    &lt;top.tobiaslee.viewtest.DragView
        android:layout_width=&quot;100dp&quot;
        android:layout_height=&quot;100dp&quot;
        android:background=&quot;@color/colorAccent&quot;/&gt;
&lt;/LinearLayout&gt;</code></pre></td>
</tr>
</tbody>
</table>
</figure>

​接下来，照搬套路。我们重写`onTouchEvent(MotionEvent event)`这个方法

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18</code></pre></td>
<td class="code"><pre><code>@Override 
public boolean onTouchEvent(MotionEvent event) {
  // 获取触摸事件坐标
   int x = (int) event.getX();
   int y = (int) event.getY();
  switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
               // 处理按下
                break;
            case MotionEvent.ACTION_MOVE:
                // 处理移动
                break;
            case MotionEvent.UP:
                // 处理离开
                break;
        }
        return true;
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

### <a href="#法1-layout方法" class="headerlink" title="法1. layout方法"></a>法1. layout方法

View进行绘制的时候，会调用`onLayout()`方法来设置显示的位置，我们可以在ACTION\_MOV事件中计算偏移量，然后在layout中增加偏移量就可以了。

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15</code></pre></td>
<td class="code"><pre><code>case MotionEvent.ACTION_DOWN:
   // 记录触摸点坐标
   lastX = x;
    lastY = y; 
   break;
case MotionEvent.ACTION_MOVE:
   // 计算偏移量
 int offsetX = x - lastX;
 int offsetY = y - lastY;
 // 调用layout() 在四个方向上增加偏移量
    layout(getLeft() + offsetX, 
           getTop() + offsetY, 
           getRight() + offsetX, 
           getBottom() + offsetY);
  break;</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这里是通过视图坐标来计算偏移量的，我们同样可以通过绝对坐标来计算:

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24</code></pre></td>
<td class="code"><pre><code>@Override 
public boolean onTouchEvent(MotionEvent event) {
  // 获取触摸事件坐标
   int rawX = (int) event.getRawX();
   int rawY = (int) event.getRawY();
  switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                 lastX = rawX;
                 lastY = rawY;
                break;
            case MotionEvent.ACTION_MOVE:
                int offsetX = rawX - lastX;
              int offsetY = rawY - lastY;
                  layout(getLeft() + offsetX, 
                       getTop() + offsetY, 
                      getRight() + offsetX, 
                    getBottom() + offsetY);
                  // 重设初始坐标
                lastX = rawX;
                 lastY = rawY;
                break;
        }
        return true;
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

​需要注意的是，因为通过绝对坐标计算偏移量，我们要及时更新上一个触摸点的坐标，所以在ACTION\_MOVE方法中也要记得更新`lastX`和`lastY`，否则就会出现偏移量不准确的情况。

### <a href="#法2-offsetLeftAndRight-与offsetTopAndBottom" class="headerlink" title="法2. offsetLeftAndRight()与offsetTopAndBottom()"></a>法2. offsetLeftAndRight()与offsetTopAndBottom()

这两个方法就是系统提供的一个对左右、上下移动的API封装，计算出偏移量后，直接调用即可。

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4</code></pre></td>
<td class="code"><pre><code>// 左右移动
offsetLeftAndRight(offsetX);
// 上下移动
offsetTopAndBottom(offsetY);</code></pre></td>
</tr>
</tbody>
</table>
</figure>

### <a href="#法3-LayoutParams" class="headerlink" title="法3. LayoutParams"></a>法3. LayoutParams

LayoutParams保存了View的布局参数，我们可以通过改变LayoutParams的参数来动态改变View的位置参数，从而实现View的移动。我们可以通过`getLayoutParams()`获取到View的LayoutParams，然后通过`setLayoutParams()`来改变其参数。

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11</code></pre></td>
<td class="code"><pre><code>// 通过layoutParams来实现移动
// 通过父布局
LinearLayout.LayoutParams layoutParams = (LinearLayout.LayoutParams) 
                        getLayoutParams();
// 通过MarginLayout来实现
ViewGroup.MarginLayoutParams layoutParams =(ViewGroup.MarginLayoutParams)
                          getLayoutParams();

layoutParams.leftMargin = getLeft() + offsetX;
layoutParams.topMargin = getTop() + offsetY;
setLayoutParams(layoutParams);</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这里有两种方法获取LayoutParams。

一种根据父布局获取对应的LayoutParams，这里我们用的是LinearLayout，所以对应的也就要使用这种类型的布局参数，如果是父布局RelativeLayout，那就是`RelativeLayout.LayoutParams layoutParams = (RelativeLayout.LayoutParams) getLayoutParams();`。但要注意，**如果没有父布局，就无法获取LayoutParams**

还有一种是通过`ViewGroup.MarginLayoutParams`来实现，因为我们实现View的移动，通常是改变这个View的Margin属性。这种实现就不用考虑父布局，更加方便。

### <a href="#法4-scrollTo和scrollBy" class="headerlink" title="法4. scrollTo和scrollBy"></a>法4. scrollTo和scrollBy

在一个View中，系统提供了`scrollTo()`和`scrollBy()`来改变一个View位置。

从字面上就可以看出**`scrollBy()`移动一个增量(dx, dy)，`scrollTo()`移动到指定的坐标位置(x, y)**

但是要注意一点，这两个方法**移动的是View的content，也就是让View的内容移动**，比如一个TextView，content就是它的文本；ImageView，就是它的drawable对象。

所以如果我们要移动这个DrawView，就要移动它的父View

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1</code></pre></td>
<td class="code"><pre><code>((View)getParent()).scrollBy(offsetX, offsetY)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

但是这样的实现，我们会发现拖动时候，View的移动是反向的!

Why? 因为我们移动的是ViewGroup，而计算的偏移量是通过子View来计算的。

假设我们希望View移动(20,10),那么上面语句执行结果就是让父布局向右下方移动了(20, 10)，对于View来说就相当于移动了(-20, -10)，他们的移动是相反的。

所以这个和在显微镜下移动载玻片相类似，需要向相反的方向移动。

也就是:

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1</code></pre></td>
<td class="code"><pre><code>((View)getParent()).scrollBy(-offsetX, -offsetY)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

### <a href="#法5-Scroller" class="headerlink" title="法5. Scroller"></a>法5. Scroller

我们之前通过scrollBy()方法，通过不断微分移动的距离，切割成N个小片段，然后瞬间移动一个小片段，实现了滑动，但未免有些不够流畅；而Scroller是用来实现平滑移动效果的一个类。

Scroller的使用，我们需要在添加一个私有成员mScroller,并且在初始化的时候创建一个scroller对象

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4</code></pre></td>
<td class="code"><pre><code>public DragView(Context context, @Nullable AttributeSet attrs) {
       super(context, attrs);
       mScroller = new Scroller(context);
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

然后重写`onComputeScroll()`方法，实现模拟滑动，这是Scroller类的核心

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12</code></pre></td>
<td class="code"><pre><code>    
@Override
public void computeScroll() {
 super.computeScroll();
   // 判断Scroller是否执行完毕
  if(mScroller.computeScrollOffset()) {
       ((View)getParent()).scrollTo(mScroller.getCurrX(),
                                    mScroller.getCurrY());
        // 通过重绘 不断调用computeScroll
        invalidate();
 }
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

Scroller类提供了`computeScrollOffset()`来判断滑动是否完成，也提供了`getCurrX()`和`getCurrY()`来获取当前的滑动坐标。

**需要注意的是，computeScroll()方法是不会自动的调用的，但我们只能在这里获取currX和currY，所以只能通过invalidate()-&gt;draw()-&gt;computeScroll()来间接调用，从而不断获得坐标**

接下来就是要启动Scroller了

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2</code></pre></td>
<td class="code"><pre><code>public void startScroll(int startX, int startY, int dx, int dy, int duration）
public void startScroll(int startX, int startY, int dx, int dy)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

Scroller启动有两个方法，区别就在于一个指定了时间长短，一个没有。（startX, startY）起始坐标，(dx, dy)就是坐标偏移量。

然后我们在ACTION\_UP中启动，让View滑动回原来的位置:

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10</code></pre></td>
<td class="code"><pre><code>case MotionEvent.ACTION_UP:
 View viewGroup = (View)getParent();
   // 滑动回原位置
    mScroller.startScroll(viewGroup.getScrollX(),
                          viewGroup.getScrollY(),
                          -viewGroup.getScrollX(),
                          -viewGroup.getScrollY());
 // 通知重绘
  invalidate();
 break;</code></pre></td>
</tr>
</tbody>
</table>
</figure>

### <a href="#法6-7-属性动画和ViewDragHelper" class="headerlink" title="法6.7 属性动画和ViewDragHelper"></a>法6.7 属性动画和ViewDragHelper

限于篇幅，就不放在这里讲了。

## <a href="#效果" class="headerlink" title="效果"></a>效果

最后我们看看实现的效果:

![scroll](/img/scroll_view.gif)

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color3">Android</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color3">自定义View</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2017/06/05/Android-Scroll学习笔记/)