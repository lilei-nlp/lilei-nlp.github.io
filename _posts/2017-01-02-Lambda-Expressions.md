---
title: "Lambda学习笔记"
date: 2017-01-02
categories:
  - blog
tags: []
---

昨天学习了一下Java 8 中非常隆重推出的Lambda表达式  
不过说实话好像非常的鸡肋???  
个人感觉仅仅是表达方式上的更为简洁一点 似乎和我想象的函数式编程不那么一样?  
不过也不是不能接受，毕竟作为一门以健壮性称道的语言，  
**如果什么东西好就往里加，那Java就不是Java啦。**

Oracle 文档中，中对Lambda表达式的描述时这样的是这样的

> Lambda expressions are a new and important feature included in Java SE 8. They provide a clear and concise way to represent one method interface using an expression.

<span id="more"></span>

我个人认为的重点就是三个词 “clear” “concise” “method interface”  
来看看使用Lambda的三个例子，来看看Lambda到底怎么用以及优雅在哪里。

第一个例子:  

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
<td class="code"><pre><code>//lambda examples
Thread t1 = new Thread( () -&gt;{
    System.out.println(&quot;Hello world!,I&#39;m Lambda!&quot;);
});

//non lambda
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println(&quot;Hello world,I&#39;m not lambda&quot;);     
    }
});</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这个例子在日常Codeing中使用频率还是比较高的，需要new 一个线程来多线程工作的时候,在没有Lambda以前的实现方式就是上面的t2的实现。  
在引进Lambda表达式以后呢，就变成了t1。  
非常直观的一点，就是Lambda确实更为简洁和清晰，将原来的6行代码缩减到3行，更加优雅了。  
Lambda在这里究竟是怎么样一个作用呢？  
当我们新建一个线程的时候，我们都会需要传入一个实现了Runnable接口的匿名内部类，然后通过重写这个匿名内部类的run()方法来执行我们想要执行的代码。  
那么对于这种只有一个抽象方法的接口（比如Runnable）需要实现这种接口的对象时，我们就可以提供一个Lambda表达式，这种接口称为函数式接口。  
所以在Lambda表达式中 我们利用()-&gt; {…} 替代了匿名内部类，产生的效果还是一样的。

第二个例子：

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
<td class="code"><pre><code>JButton jButton = new JButton(&quot;Test Button&quot;);
//non-lambda expression
jButton.addActionListener(new ActionListener() {
    @Override
    public void actionPerformed(ActionEvent e) {
        System.out.println(&quot;Button Clicked&quot;);
    }
});
//lambda expression
jButton.addActionListener(e -&gt;
        System.out.println(&quot;Clicked detected by lambda&quot;));</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这个例子描述的是给JButton添加事件监听，事实上也就是实现了ActionListener中actionPerformed的一段代码。这里把6行代码缩减到2行，看起来更舒服，更为一目了然。  
所以，**Lambda表达式把很多臃肿的匿名内部类的创建过程给省略了，而直接通过实现函数式接口来完成代码块传递的这样一个过程**

第三个例子：  

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
9</code></pre></td>
<td class="code"><pre><code>public static void repeatMsg(String text,int delay){
    ActionListener listener = e -&gt; {
        System.out.println(text);
        Toolkit.getDefaultToolkit().beep();
    };
    new Timer(delay,listener).start();
}

    repeatMsg(&quot;Hello&quot;,1000);  //print &quot;Hello&quot; every second</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这里有一个问题，如果text这个参数变量已经不存在了，那怎么lambda表达式不就无法知道text是什么了吗？  
巩固一下理解，Lambda表达式有三个部分:

1.  一个代码块；
2.  参数；
3.  自由变量的值，这是指非参数而且不在代码中定义的变量。  
    在这个例子里面，lambda表达式有一个自由变量text。  
    **表示lambda表达式的数据结构必须储存自由变量的值**  
    这里就是”Hello”这个字符串，我们说它被lambda给捕获了（captured）  
    关于代码块和自由变量值有一个术语叫做“闭包”（closure），所以，Java也是有闭包的。  
    **但是，Java所捕获的外部变量的值，必须是明确定义的，而且，其引用值不会改变**  
    这一点主要是考虑到了并发时线程安全的问题，也就是说如果该自由变量可能被外部函数所改变的话，那么就无法保证数据的准确性，也就不合法了。

什么时候用Lambda表达式呢，Java 核心技术卷 I 中给出了以下几个原因：

1.  在一个单独的线程中运行代码（Thread的例子）；

2.  多次运行代码（下面的repeat)；

3.  在算法的适当位置运行代码（比如排序中的比较操作）；

4.  发生某种情况时执行代码（按钮点击监听，数据到达等等）；

5.  只在必要时才运行代码。

    在使用Lambda的时候，只要定义自己需要的函数接口，然后实现它就可以了。  
    下面这个repeat的例子，就是重复操作的一个例子。

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
9</code></pre></td>
<td class="code"><pre><code>public interface IntConsumer {
    void accept(int value);
}
public static void repeat(int n ,IntConsumer action){
    for (int i = 0 ; i &lt; n ; i ++) 
        action.accept(i);
}

repeat(10,value -&gt; System.out.println(&quot;Countdown:&quot; + ( 9  - value)));</code></pre></td>
</tr>
</tbody>
</table>
</figure>

比较复杂的是在repeat这段代码中需要一个int类型的参数，而IntConsumer是处理int值的标准接口  
通过value -&gt; {…} 将这个int传给了accept(int value)，实现相应的代码。

总结一下：  
Lambda表达式主要用在函数式接口（只有一个抽象方法的接口）上来取代臃肿的内部类创建过程。  
注意传入的参数要和函数的参数匹配。  
Lambda有闭包，但是只能捕获最终变量（effective final) 即改变量初始化后不会再被更新，比如字符串常量。
