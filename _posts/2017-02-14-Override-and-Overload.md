---
title: "深入理解Java虚拟机-重写和重载的实现"
date: 2017-02-14
categories:
  - blog
tags: []
---

在《深入理解Java虚拟机》看到8.3 方法调用这一节的时候，突然就想到上次看到的三道清华Java测试题，里面就有一题考察的是关于Java 方法的Overload。  
涉及到的一个核心问题就是： Java是怎么知道要调用哪个方法的?  
首先，方法调用不等同于方法执行。方法调用唯一的任务就是确定调用方法的版本。  
而Class文件中不包含方法调用的入口地址，而只是一个符号引用。这给Java带来了动态扩展的能力，但也使得方法调用的过程变的更为复杂。

<span id="more"></span>

Java中的方法调用分为两大类：

1、解析调用(Resolution)： 在类加载的解析阶段，会把其中的一部分符号引用转化为直接引用。前提是：方法在程序运行之前，就有一个可确定的调用版本，且该版本在运行期不可变。即“编译期可知，运行期不变”，符合这个要求的主要包括静态方法和私有方法两大类，前者与类型直接关联，后者外部无法调用，因此无法通过继承重写。

2、分派调用(Dispatch)：又分为 “静态分派” “动态分派” “多分派” “单分派”。在运行期间才能确定调用方法的版本。

## <a href="#解析调用：" class="headerlink" title="解析调用："></a>解析调用：

Java虚拟机中提供了5条方法调用的字节码指令

> 

1.  invokestatic: 调用静态方法

2.  invokespecial: 调用实例构造器方法、私有方法和父类方法

3.  invokevirtual:调用所有的虚方法

4.  invokeinterface:调用接口方法，会在运行时确定一个实现此接口的对象

5.  invokedynamic: 先在运行时动态解析出调用点限定符所引用的方法，然后再执行

    上面前4条调用指令、分派逻辑是固化在Java虚拟机内部的，而invokedynamic则由用户指定的引导方法决定。  
    只要能被 invokestatic 和 invokespecial 调用的方法，都可以在解析阶段确定唯一的调用版本。符合这个条件的有 **静态方法、私有方法、实例构造器、父类方法** 四种，他们在类加载阶段就会把**方法的符号引用解析为直接引用（内存地址入口）**。这类方法也称为非虚方法。  
    以下是《深入理解Java虚拟机》的示例代码

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
    <td class="code"><pre><code>public class StaticResolution {
        public static  void sayHello(){
            System.out.println(&quot;hello world&quot;);
        }
        public static void main(String[] args) {
            StaticResolution.sayHello();
        }
    }</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

在命令行通过javac 编译后得到.class文件 再利用javap来查看字节码发现

> $ javap -verbose StaticResolution  
> public static void main(java.lang.String\[\]);  
> descriptor: (\[Ljava/lang/String;)V  
> flags: ACC\_PUBLIC, ACC\_STATIC  
> Code:  
> stack=0, locals=1, args\_size=1  
> 0: invokestatic \#5 // Method sayHello:()V  
> 3: return  
> LineNumberTable:  
> line 6: 0  
> line 7: 3  
> }

确实是通过invokestatic来调用了方法。  
Java中非虚方法还有一种，final方法,虽然是用invokevirtual来调用的，但是因为它无法被覆盖，没有其他版本，多态的选择也一定是唯一的。**在Java语言规范中规定了final方法是一种非虚方法**

## <a href="#分派调用：" class="headerlink" title="分派调用："></a>分派调用：

分派调用揭示了OOP多态性的一些最基本的体现。“重载”和“重写”，就是其中之一。

### <a href="#1-静态分派" class="headerlink" title="1.静态分派"></a>1.静态分派

先来看一段代码  

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
24
25</code></pre></td>
<td class="code"><pre><code>public class StaticDispatch {
    static abstract class Human{}
    static class Man extends  Human{}
    static class Woman extends  Human{}
    
    public void sayHello(Human guy){
        System.out.println(&quot;Hello human&quot;);
    }

    public void sayHello(Man guy){
        System.out.println(&quot;Hello man&quot;);
    }

    public void sayHello(Woman guy){
        System.out.println(&quot;Hello woman&quot;);
    }

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        StaticDispatch sr = new StaticDispatch();
        sr.sayHello(man);
        sr.sayHello(woman);
    }
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

输出是

> Hello human  
> Hello human

为了弄清楚这是为什么，需要先明白一个定义

> Human man = new Man();

这里 “Human”是 man变量的 静态类型 (Static Type) 或者叫 外观类型(Apparent Type)  
而后面的 “Man” 则是 man 变量的 实际类型(Actual Type)  
静态类型都实际类型在程序中都可以发生变化，区别在于**静态类型的变化仅仅是在使用时发生，而其本身的静态类型并不发生改变**  
看一个例子  

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
6</code></pre></td>
<td class="code"><pre><code>//实际类型改变
Human man = new Man();
man = new Woman();
//静态类型改变
sr.sayHello((Man) man);
sr.sayHello((Woman) man);</code></pre></td>
</tr>
</tbody>
</table>
</figure>

上述代码中的实际类型改变之后,Man的实际类型就由”Man”变成了”Woman”  
而通过强制转换的静态类型转换，只是通知调用者，以”Man”或者”Woman”的方式来看待man变量，**其静态类型“Human”并没有发生改变**

方法调用确定版本的要素有两个:  
1.方法的调用者 这里都是sr  
2.方法的参数的数量和数据类型  
静态类型在编译期可知，而动态类型只有实际运行时能够获知。  
**虚拟机（准确说是编译器）是通过参数静态类型作为重载的判定依据**  
因此在编译阶段，Javac编译器会根据参数的静态类型决定使用哪个重载版本。  
两个变量实际类型不同，然并卵，静态类型相同，就决定了他们会使用同一个重载函数。

所以，所有依赖静态类型来定位方法执行版本的分派动作就是静态分派。典型应用就是方法重载(Overload)  
**静态分派发生在编译阶段** 但有些时候，这个版本并不唯一，只能确定一个“更加合适的版本”，就是上次清华Java题里面null的情况。  
这种情况主要发生在 字面量作为变量的情况下，因为字面量不需要定义，所以也没有显式的静态类型。  
来看一个极其恶心人的代码。  

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
24
25
26
27
28</code></pre></td>
<td class="code"><pre><code>public class Overload {
    public static void sayHello(Object obj){
        System.out.println(&quot;Hello object&quot;);
    }
    public static void sayHello(int arg){
        System.out.println(&quot;Hello int&quot;);
    }
    public static void sayHello(long arg){
        System.out.println(&quot;Hello long&quot;);
    }
    public static void sayHello(Character arg){
        System.out.println(&quot;Hello character&quot;);
    }
    public static void sayHello(char arg){
        System.out.println(&quot;Hello char&quot;);
    }
    public static void sayHello(char ...arg){
        System.out.println(&quot;Hello char ...&quot;);
    }

    public static void sayHello(Serializable arg){
        System.out.println(&quot;Hello Serializable &quot;);
    }

    public static void main(String[] args) {
        sayHello(&#39;a&#39;);
    }
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

运行之后输出

> Hello char

没毛病。如果我们把sayHello(char arg)给注释掉 输出就变成了

> Hello int

也能理解， ‘a’ 发生一次自动类型转换，变成一个整形数97，再把 sayHello(int arg)注了

> Hello long

这里又发生了一次自动类型转换,’a’-&gt;int 97 -&gt; long 97 匹配了long的重载  
虽然这里没有 double float类型的重载 但事实上如果有 就还会发生  
按照 char -&gt; int -&gt; long -&gt;float -&gt; double 的顺序发生自动类型转换 但不会匹配byte 和short类型的重载  
，因为类型转换不安全。再把long也注释掉。

> Hello character

嘿呀，这里就发生了一次自动装箱(AutoBoxing) 原始类型char ‘a’被包装它对应的封装类型 java.lang.Character 所以匹配了Character的重载，继续注掉Character，输出

> Hello Serializable

有点莫名其妙，为什么会输出这个呢？java.lang.Serializable 是 java.lang.Character实现的一个接口  
当’a’被自动装箱成Character还是找不到装箱类匹配的函数,他就找到了装箱类实现的接口类型。所以又发生了一次自动转型，Character转型为Serializable 安全地转型为它的接口或者父类 但是肯定不能变成Integer。  
Character还实现了Comparable这个接口，如果有一个Comparable&lt; Character &gt;的重载方法，那么这两个接口的优先级是相同的，会提示Ambiguous method call 模糊的方法调用而拒绝编译  
除非在调用时显示的指明字面量的类型  
比如:  

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1</code></pre></td>
<td class="code"><pre><code>sayHello(Comparable&lt;Character&gt; &#39;a&#39;);</code></pre></td>
</tr>
</tbody>
</table>
</figure>

继续我们的注释，输出是

> Hello object

Amazing!为什么不选貌似更为接近的变长参数char … 呢，因为变长参数的优先级是重载方法中最低的。  
‘a’装箱成Character以后开始向父类转型，辈分越高，优先级越低，即使是null，也是这样一个顺序。  
再注释掉，自然只剩下一个结果

> Hello char …

注意不要混淆 **解析和静态分配**：  
两种确定方式，是在不同层次上去筛选的过程。  
静态方法会在类加载期进行解析，但是静态方法也可能有多个重载版本，这就是静态分派了。  
eg.  

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
<td class="code"><pre><code>public class ResolutionAndDispatch{
    static void sayHello(int arg){
        System.out.println(&quot;Hello int&quot;);
    }
    static void sayHello(char arg){
        System.out.println(&quot;Hello char&quot;);
    }
    public static void main(String[] args){
        ResolutionAndDispatch.sayHello(&#39;a’);
    }
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

在类加载阶段，Class中的指向sayHello的符号引用全部被换成直接引用，  
而在编译时,’a’ 实际上是 一个char 静态类型 就作为编译器静态分派的判断依据。

### <a href="#2、动态分派" class="headerlink" title="2、动态分派"></a>2、动态分派

动态分派和多态性另一个重要体现 重写 (Override) 有密切的关系。  
看代码：  

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
24
25
26
27</code></pre></td>
<td class="code"><pre><code>public class DynamicDispatch {
    static abstract class Human{
        protected abstract void sayHello();
    }
    static class Man extends Human{
        @Override
        protected void sayHello() {
            System.out.println(&quot;man say hello&quot;);
        }
    }

    static class Woman extends Human{
        @Override
        protected void sayHello() {
            System.out.println(&quot;woman say hello&quot;);
        }
    }

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        man.sayHello();
        woman.sayHello();
        man = new Woman();
        man.sayHello();
    }
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

输出:

> man say hello  
> woman say hello  
> woman say hello

非常顺眼的输出，感觉没有什么奇怪，但是为什么这里静态类型都是Human 没有去调用父类的方法呢??  
用javap 输出字节码看一下  

<figure class="highlight plain">
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
<td class="code"><pre><code>public static void main(java.lang.String[]);
  descriptor: ([Ljava/lang/String;)V
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    stack=2, locals=3, args_size=1
       0: new           #2                  // class DynamicDispatch$Man
       3: dup
       4: invokespecial #3                  // Method DynamicDispatch$Man.&quot;&lt;in                                  it&gt;&quot;:()V
       7: astore_1
       8: new           #4                  // class DynamicDispatch$Woman
      11: dup
      12: invokespecial #5                  // Method DynamicDispatch$Woman.&quot;&lt;                                  init&gt;&quot;:()V
      15: astore_2
      16: aload_1
      17: invokevirtual #6                  // Method DynamicDispatch$Human.sa                                  yHello:()V
      20: aload_2
      21: invokevirtual #6                  // Method DynamicDispatch$Human.sa                                  yHello:()V
      24: new           #4                  // class DynamicDispatch$Woman
      27: dup
      28: invokespecial #5                  // Method DynamicDispatch$Woman.&quot;&lt;                                  init&gt;&quot;:()V
      31: astore_1
      32: aload_1
      33: invokevirtual #6                  // Method DynamicDispatch$Human.sa                                  yHello:()V
      36: return</code></pre></td>
</tr>
</tbody>
</table>
</figure>

0~15 在做准备动作 我们看到调用了两次 invokespecial 是调用了实例构造器 构造了man 和woman两个实例，并且把他们的引用放在1、2个局部变量表Slot中  
接下来的16~21，16和20两句aload\_1和aload\_2 把创建的对象的引用压到栈顶，这两个对象是将要执行的方法sayHello()的执行者，称作接受者(Receiver) 17和21两句的方法调用指令 和参数 都是一样的，但是最终执行的目标方法不同，原因就是invokevirtual指令的多态查找:

> 

1、找到操作数栈顶的第一个元素所指向的对象的实际类型，记作C (这里就是Man或者是Woman)  
2、如果在类型C中找到和常量中描述符合简单名称都相符合的方法，则进行访问权限校验，如果通过就返回这个方法的直接引用，查找结束；不通过，返回 IllegalAccessError异常  
3、否则（没找到符合的方法），按照继承关系从下往上对C的各个父类进行第2步的搜索和验证过程  
4、如果始终没有找到合适的方法，则跑出java.lang.AbstractMethodError异常

**invokevirtual指令 第一步就是确定接受者的实际类型**  
所以两次调用把  
**常量池相同的类方法符号引用解析到了不同的直接引用上，这就是Java方法重写的本质**  
这种在运行期间根据实际类型确定方法执行版本的分派过程就是 动态分派

### <a href="#3-单分派与多分派" class="headerlink" title="3.单分派与多分派"></a>3.单分派与多分派

方法的接受者与方法的参数统称为方法的 **宗量**  
单分派和多分派，顾名思义，根据一个宗量来对目标方法选择就是单分派，多于一个，则是多分派。  
看代码  

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
24
25
26
27
28
29</code></pre></td>
<td class="code"><pre><code>public class Dispatch {
    static class QQ{}
    static class _360{}
    
    public static class Father{
        public void hardChoice(QQ arg){
            System.out.println(&quot;father choose QQ&quot;);
        }
        public void hardChoice(_360 arg){
            System.out.println(&quot;father choose 360&quot;);
        }
    }
    
    public static class Son extends Father{
        public void hardChoice(QQ arg){
            System.out.println(&quot;son choose QQ&quot;);
        }
        public void hardChoice(_360 arg){
            System.out.println(&quot;son choose 360&quot;);
        }
    }
    
    public static void main(String[] args) {
        Father father = new Father();
        Father son = new Son();
        father.hardChoice(new _360());
        son.hardChoice(new QQ());
    }
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

输出

> father choose 360  
> son choose QQ

编译期间编译器的选择，也就是静态分派，  
依据有两条：1.方法的接受者是father还是son 2.参数的静态类型是360还是QQ  
这次选择最终产生

> 24: invokevirtual \#8 // Method Dispatch$Father.hardChoice:(LDispatch$\_360;)V  
> 35: invokevirtual \#11 // Method Dispatch$Father.hardChoice:(LDispatch$QQ;)V

这两条分别指向Father.hardChoice(QQ)和Father.hardChoice(360)的invokevirtual指令  
根据两个宗量进行选择，所以Java的静态分派是多宗量分派。

运行期间，JVM的选择，也就是动态分派  
编译期间已经确定了son.sayHello(new QQ()) 这句 目标方法的签名必须是 hardChoice(QQ) 所以虚拟机不管是“腾讯QQ”还是“奇瑞QQ” 参数的静态类型和实际类型不会对目标方法的选择产生影响  
唯一能够影响JVM选择的，只由接受者的实际类型是Son还是Father  
仅仅根据接受者这一宗量，所以，动态分派是 单宗量分派

## <a href="#虚拟机动态分派的实现" class="headerlink" title="虚拟机动态分派的实现"></a>虚拟机动态分派的实现

动态分派是很频繁的过程，虚拟机对此用了一种“稳定优化”的手段来加快这一过程。  
为类在方法区中建立一个虚方法表，用虚方法表索引来代替元数据查找以提高性能。  
虚方法表中存放着各个方法的实际入口地址，如果子类没有重写父类方法，那么子类的虚方法表中该函数的入口地址和父类相同方法的地址是一致的。  
为了程序实现的方便，具有相同签名的方法，父类和子类虚方法表中都应当具有相同的索引，这样类型改变，只要换一张虚方法表就可以了，迅速通过相同的索引找到对应方法的入口地址。  
方法表一般在类加载的连接阶段进行初始化。

实际上跟着样例敲一遍代码走一遍javap 看看字节码收获还是很大的。  
总结一下：  
Java是一门静态多宗量，动态单宗量的语言。  
静态分派，编译期间选择，多和重载有关。（类内部的方法多态性）  
动态分派，运行期才执行，与重写有关。（类继承的多态性）

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color5">Java</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">JVM</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2017/02/14/Override-and-Overload/)