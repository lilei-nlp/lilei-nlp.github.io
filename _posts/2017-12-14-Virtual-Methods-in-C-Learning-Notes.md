---
title: "Virtual Methods in C++ Learning Notes"
date: 2017-12-14
categories:
  - blog
tags: []
---

早就听闻 C++ 中虚函数的大名，今天 OOP 课上老师讲了，感觉还有一些不清晰的地方，上课老师几个比较坑的例子也有一些细节需要注意。回来翻看了一下《C++ Primer Plus》，做笔记以供日后复习。

## <a href="#前置知识" class="headerlink" title="前置知识"></a>前置知识

在学习虚函数之前需要一些 C 语言和基本的 C++ 知识，这里不展开，简单带过：

1.  函数调用的方式：在 C/C++ 中，函数调用，实质上是**通过地址作为入口**找到对应的代码块，进行执行。
2.  继承：子类继承父类之后，会获得父类所有的成员，并且，在构造函数（constructor)的调用链中，以**父类在前，子类在后**的顺序依次调用；析构（deconstructor）函数恰好相反，**子类在先，父类在后**。

## <a href="#静态-动态联编" class="headerlink" title="静态/动态联编"></a>静态/动态联编

> 编译器负责将源代码中的函数调用解释为执行特定的函数代码块，这一过程被称为函数名联编（binding）。在编译过程（Compile time）进行联编称为静态联编（static binding），而虚函数的存在使得编译器有时需要在运行时（Run time）选择正确的虚方法的代码，称为动态联编。
>
> ——《C++ Primer Plus》

似乎还是有点抽象，需要用代码来解释，在此之前，再讨论一个小问题：指针和引用类型的兼容性。

<span id="more"></span>

### <a href="#指针和引用类型的兼容性" class="headerlink" title="指针和引用类型的兼容性"></a>指针和引用类型的兼容性

<figure class="highlight c++">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3</code></pre></td>
<td class="code"><pre><code>double x = 2.4;
int * pi = &amp; x; // 非法赋值，指针类型不匹配
long &amp; rl = x; // 非法赋值，引用类型不匹配</code></pre></td>
</tr>
</tbody>
</table>
</figure>

上面这段代码的错误显而易见，类型不匹配，但是在继承之中：**指向基类的引用或指针可以引用派生类对象**，而不必进行显式的类型转换，比如：

<figure class="highlight c++">
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
5</code></pre></td>
<td class="code"><pre><code>class BaseClass {}
class DerivedClass : public BaseClass {}
DerivedClass dc;
BaseClass * pb = &amp;dc; // 正确
BaseClass &amp; rb = dc; // 正确</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这种将派生类的指针或引用转换为基类的指针或指针的过程，称为向上强制转换（upcasting）。其实也非常好理解，继承就是一个 `is-a` 的关系，既然 `DerivedClass` 的实例 `dc` 是一个 `BaseClass`，那么其指针和引用自然也能够认为是 `BaseClass`的指针和引用。

相反地，如果想要把基类的指针或引用转换为派生类指针或引用，称为（downcasting），如果不用显式类型转换，是不允许的，举个最简单的例子：

<figure class="highlight c++">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3</code></pre></td>
<td class="code"><pre><code>BaseClass bc;
// DerivedClass &amp;dc =  bc; 错误
DerivedClass &amp;dc = (DerivedClass) bc;// 显式类型转换</code></pre></td>
</tr>
</tbody>
</table>
</figure>

同样很好理解，`is-a` 关系不可逆，如果我的派生类里有父类没有的成员，那基类怎么给你变出一个不存在的成员来？所以自然不行。

接下来，就进入到虚函数的部分。

## <a href="#虚函数" class="headerlink" title="虚函数"></a>虚函数

先来看一个例子：

<figure class="highlight c++">
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
<td class="code"><pre><code>class TradePerson {
public:
    void say() {cout &lt;&lt; &quot;Just Hi\n&quot;;}
};

class Tinker : public TradePerson {
public:
    void say() {cout &lt;&lt; &quot;Hi Tinker\n&quot;;}
};</code></pre></td>
</tr>
</tbody>
</table>
</figure>

我们定义了两个类，`Tinker` 继承自 `TradePerson`，接下来：

<figure class="highlight c++">
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
<td class="code"><pre><code>Tinker tinker;
TradePerson * tp;
tp = &amp; tinker;
tp-&gt;say();</code></pre></td>
</tr>
</tbody>
</table>
</figure>

结果是 `Just Hi`，`tp->say()` 根据指针类型 `TradePerson` 调用了 `TradePerson::say()`，没有毛病，在编译是就能够确定这个调用的地址，是静态联编。

而如果我们在 `TradePerson` 的 `say()` 方法前加上 `virtual` 关键字，声明为虚函数（注：声明为虚函数的方法在基类及所有派生类，包括派生类的派生类中都是虚的）：

<figure class="highlight c++">
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
<td class="code"><pre><code>class TradePerson {
public:
    virtual void say() {cout &lt;&lt; &quot;Just Hi\n&quot;;}
};</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这时的输出结果就会变成 `Hi Tinker`，这是为什么呢？这个时候 `tp->say()` 就会根据对象类型 `Tinker` 调用 `Tinker::say()`。而在实际应用中，这个对象类型只有在运行的时候才会确定，也就是我们所说的动态联编。

似乎是很神奇，那么虚函数是怎么工作的呢？

## <a href="#工作原理" class="headerlink" title="工作原理"></a>工作原理

通常编译器处理虚函数的方法是，为每个对象添加一个隐藏成员，这个隐藏成员中保存了一个指向函数地址数组的指针，这个数组称为虚函数表，看下面这个例子：

![Virtual Function Table](/img/vtable.png)

基类是 `Scientist` 类，其中隐藏的成员 `vptr` 指向一个函数地址数组；同样，派生类`Physicity` 也有一个 `vptr` 指向另一个函数地址数组。值得注意的是，对于所有声明为虚函数的函数，如果其没有在派生类中被重新定义，**则派生类中该函数的指向和基类相同**，也就是说，我们会去调用父类的这个函数；而如果被重新定义了，则相应的更新其地址。

上面的例子已经非常详细了，另外还有一点值得注意，就是不管有多少的虚函数，我们都只需要在对象中添加**一个地址成员 vptr**，区别仅仅在于 **vptr** 所指向的地址表的大小而已。

### <a href="#例子" class="headerlink" title="例子"></a>例子

对于使用基类引用或指针作为参数的函数调用，将进行向上转换，这一点很重要，请看下面的例子：

<figure class="highlight c++">
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
23</code></pre></td>
<td class="code"><pre><code>class Brass{
  public:
  virtual void ViewAcct();
}
class BrassPlus : public Brass{
  public:
  void ViewAcct();
}

void fr(Brass &amp; rb) ; // uses rb.ViewAcct()
void fp(Brass * pb) ; // uses pb-&gt;ViewAcct()
void fv(Brass b) ; // uses b.ViewAcct()

int main() {
  Brass b;
  BrassPluss bp;
  fr(b);// 调用 Brass::ViewAcct()
  fr(bp);//  调用 BrassPlus::ViewAcct()
  fp(b);// 调用 Brass::ViewAcct()
  fp(bp); //  调用 BrassPlus::ViewAcct()
  fv(b);  // 调用 Brass::ViewAcct()
  fv(bp); // 调用 Brass::ViewAcct()
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

前面四个输出结果没有问题，前面的几个 `bp` 都通过 upcasting 调用了 `BrassPlus::ViewAcct()`，最后两个为什么都是 `Brass::ViewAcct()`？因为**值传递只把 `BrassPlus` 的 `Brass` 部分给了 `fv()`，只能调用 `Brass::ViewAcct()`**。

## <a href="#注意事项" class="headerlink" title="注意事项"></a>注意事项

1.  构造函数没有虚函数。因为没有意义，我们在构造子类的时候必然会调用父类的构造函数。

2.  一般我们都会把**基类的析构函数声明为虚函数**，Why？看下面的例子：

    <figure class="highlight c++">
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
    22</code></pre></td>
    <td class="code"><pre><code>class A {
        char* p;
    public:
        A() {cout&lt;&lt; &quot;A constructor&quot; &lt;&lt; endl;  p = new char[5];}
        ~A() {cout &lt;&lt; &quot;A deconstructor&quot; &lt;&lt; endl; delete[] p;}
    };

    class Z : public A {
        char * q;
    public:
        Z() {cout&lt;&lt; &quot;Z constructor!&quot; &lt;&lt; endl;  q = new char[50];}
        ~Z() {cout &lt;&lt; &quot;Z deconstructor&quot; &lt;&lt; endl; delete[] q;}
    };

    void f() {
        A * ptr = new Z();
        delete ptr;
    }
    int main() {
      f();
      return 0;
    }</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

    结果是什么呢？

    > A constructor  
    > Z constructor!  
    > A deconstructor

    没有调用 `Z` 的析构函数，也就意味着，有申请的内存没有被释放。原因是 `delete` 会只调用 `~A()`，解决的办法很简单，就是把基类 `A` 声明为虚函数。**通常我们会给基类提供虚析构函数，即使它并不需要析构函数**，其原因就在于这样能够调用子类对应的析构函数，来进行资源的释放。

3.  友元不能是虚函数，因为友元不是类成员，而只有成员才能是虚函数。

4.  如果派生类没有重新定义虚函数，就会使用该函数的基类版本。如果派生类在派生链之中，则使用最新的虚函数版本（即最晚定义的版本，太爷爷声明虚函数，爷爷没重新定义，爸爸重新定义了，儿子没有定义，那么儿子会调用爸爸的版本）。

5.  重新定义将会隐藏方法：

    <figure class="highlight c++">
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
    <td class="code"><pre><code>class A{
      public:
      virtual void print(int a) const;
    }
    class Z : public A{
      public:
      virtual void print() const;
    }</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

    这可能会报错，如果不报错，这段代码意味着:

    <figure class="highlight c++">
    <table>
    <colgroup>
    <col style="width: 50%" />
    <col style="width: 50%" />
    </colgroup>
    <tbody>
    <tr>
    <td class="gutter"><pre><code>1
    2
    3</code></pre></td>
    <td class="code"><pre><code>Z z;
    z.print(); // 正确
    z.print(5); // 错误，父类的版本被隐藏</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

    **重新定义继承的方法不是重载**，而会将所有同名的基类方法隐藏。

    这告诉我们两点：

    如果重新定义继承的方法，确保与原来的原型完全相同。

    如果基类声明被重载了，则需要在派生类中重新定义所有的基类版本。如果之定义一个版本，则其余版本会被隐藏，无法使用

    ## <a href="#感受" class="headerlink" title="感受"></a>感受

    静态联编和动态联编和 JVM 的静态分配合动态分配很类似，不过不同的就是 Java 中的继承和重载比 C++ 简单了很多，更加解放了程序员，而不必操心这些有的没的（逃
