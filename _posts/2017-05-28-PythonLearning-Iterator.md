---
title: "PythonLearning-Iterator"
date: 2017-05-28
categories:
  - blog
tags: []
---

​迭代器和生成器应该算是Python的黑魔法，一开始学习的时候还是感觉相当的迷糊的，这篇文章就记录一下学习生成器和迭代器的过程。

## <a href="#Iterator-迭代器" class="headerlink" title="Iterator 迭代器"></a>Iterator 迭代器

首先问一个问题：迭代器是什么？

> - 迭代协议：实现`__next__()` 方法的对象会前进到下一个结果，而在到达一系列结果的末尾时，会引发 StopIteration 异常
> - 任何遵循上述迭代协议的对象，都被认为是迭代器
> - 任何可迭代对象都能通过 for 循环或其它迭代工具遍历。实际上，所有迭代工具内部都是在每次迭代中调用迭代器的 **next**() 方法，并通过捕捉 StopIteration 异常来确定何时离开

但是在这个定义之前，我们其实已经在无数场合用过它了:

<figure class="highlight python">
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
<td class="code"><pre><code>my_list = [1,2,3]
for i in my_list:
    print(i)</code></pre></td>
</tr>
</tbody>
</table>
</figure>

<span id="more"></span>

这里的for语句中的my\_list，实际上是通过调用了Python的build-in function`iter()`获取到的一个迭代器对象。

随后通过for循环，对它不断地执行迭代协议。

所以我们可以通过实现`__iter__()`和`next()/__next__()`方法来手搓一个简单的迭代器：

<figure class="highlight python">
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
19</code></pre></td>
<td class="code"><pre><code>class MyListIter:
    def __init__(self, list_data):
        # 其迭代的内容来自于list_data 用一个list来维护
        self.list_data = list_data
        self.index = 0
   
    def __iter__(self):
        return self

    def __next__(self):
        # 迭代的计算发生在__next__函数中
        if self.index &lt; len(self.list_data):
            # 这里是把列表中的值取出作为返回值
            val = self.list_data[self.index]
            # 下标增加 指向后一个元素
            self.index += 1
            return val
        else:
            raise StopIteration()</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这里我们定义了一个简单的MyListIter对象，**其`__iter__()`函数返回它自身**，那么后续的迭代协议就是在**调用它自身的`__next__()`方法**。

<figure class="highlight python">
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
<td class="code"><pre><code>my_list = MyListIter([1,2,3])
it = iter(my_list) # 获取一个iterator对象 其实现了__next__()方法 这里就是它自己
for i in it # 写成 for i in my_list 也完全可以
   print(i) # 1 2 3
print(next(it)) # StopIteration 停止迭代 index = 3迭代器已经用完了</code></pre></td>
</tr>
</tbody>
</table>
</figure>

需要注意的一点是，因为这里我们的`__next__()`方法的实现其实是通过其自身下标指针这一成员，不断自增，来实现取得下一个元素的。

所以当我们在for循环中使用过it进行过一次完整的从头到尾的迭代之后，index的值已经变成了3，再次调用next()方法就会出现`StopIteration`这一Exception。

**很多迭代器的实现都是通过其内部成员的变化来实现迭代，所以迭代器基本就只能用一次，用完就无法再进行迭代了!**

## <a href="#Generator-和-yield" class="headerlink" title="Generator 和 yield"></a>Generator 和 yield

问题：求无数多个素数，怎么实现？

因为不可能存的下无数个素数，自然会想到迭代协议，不断的next计算出下一个素数并且返回，就好了。

有个形象的比喻：

> 有个人很喜欢吃M&M豆，一天他想吃很多很多的M&M豆，怎么办?
>
> 非生成器的实现，就是把所有的M&M豆交给他，可能堆积如山，甚至出现放不下的情况。
>
> 而生成器就给了他一台可以生产M&M豆的机器，上面有一个按钮，按一下(调用next()方法)，出一颗M&M豆。
>
> 这样就可以获得无穷多的M&M豆了！

​可是，如果按照上面的MyListIter的实现，我内部需要持有一个成员来保存当前计算的数，可能还要重写两个方法，太麻烦了不是吗？

​这样实现很大的一个原因是因为**传统的函数只有一次返回的机会，无法多次返回**,而迭代就意味着我们需要保存当前的上下文，从而根据上一次的结果计算得到下一次，因而这里就用类的成员来作为这个上下文的记录者。

​有没有简单的一点的方法呢？

​Yes! 那就是`yield`关键字

### <a href="#yield" class="headerlink" title="yield"></a>yield

yield关键字,能够让函数变成一个生成器，那么什么是生成器呢？

> - Generator functions create generator iterators.
>
>   生成器函数产生一个生成器迭代器
>
> - Generator iterator are almost always referred to as “generators”.
>
>   生成器迭代器一般就叫做生成器
>
> - Just remember that a generator is a special type of iterator.
>
>   生活器就是一种特别的迭代器
>
> - To be considered an iterator, generators must define a few methods, one of which is **next**().
>
>   作为一种迭代器，生成器必须实现几个方法，包括next()

说白了，就是生成器和迭代器类似，是可以不断地对他调用next()函数，

也可以应用在`for i in generator:`这样的语句之中的一种对象。

yield关键字的作用和return类似，它返回一个值。

但是！！！**这个值的当前状态会被保存，而不同于普通函数结束之后内存会被回收掉**

这就是生成器的作用了。

其最核心的特点就是**惰性计算**，优点就是**节省空间**。

来看一个例子:

<figure class="highlight python">
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
<td class="code"><pre><code>def fibonacci():
    prev, curr = 0, 1
    while True:
        yield curr
        prev, curr = curr, prev + curr
f = fibonacci() # 得到一个生成器
next(f) # 1
next(f) # 1
next(f) # 2
next(f) # 4
# ...</code></pre></td>
</tr>
</tbody>
</table>
</figure>

在这里我们定义了一个`fibonacci()`函数，来求斐波那契数列，理论上，它可以求无限多个斐波那契。

为什么呢？因为我们每计算出一个curr（当前的斐波那契数) 就通过yield关键字将它返回。

我们通过`f = fibonacci()` 获取了一个生成器

而当我们调用next(f)时，程序会在初始化prev和curr后会进入循环体，并且在yield关键字处返回curr的值并保存当前上下文，再下一次调用时会根据上下文从yield关键字之后继续执行。

所以我们看到在不断执行next(f)会获得一个又一个的斐波那契数。

那么回到我们一开始的那个问题，求无限个素数：

<figure class="highlight python">
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
20</code></pre></td>
<td class="code"><pre><code>def is_prime(number):
    &#39;&#39;&#39;判断素数函数&#39;&#39;&#39;
    if number &gt; 1:
        if number == 2:
            return True
        if number % 2 == 0:
            return False
        for current in range(3, int(mat.sqrt(number) + 1), 2):
            if number % current == 0:
                return False
        return True
    return False

def get_prime():
    cnt = 2
    while True:
        if(is_prime(cnt)):
            yield cnt # return cnt 并且保存其当前的值
        cnt += 1</code></pre></td>
</tr>
</tbody>
</table>
</figure>

​总结一下的话，**generator就是函数版的iterator，其通过yield关键字来实现迭代状态的保存，更加简洁**。

## <a href="#Java中的Iterator-和-Iterable-接口" class="headerlink" title="Java中的Iterator 和 Iterable 接口"></a>Java中的Iterator 和 Iterable 接口

其实我看到迭代器的第一反应就Java里的Iterator和Iterable这两个接口。

接口定义:

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
<td class="code"><pre><code>public interface Iterable&lt;T&gt; {
  Iterator&lt;T&gt; iterator();
}

public interface Iterator&lt;E&gt; {
  boolean hasNext();
  E next();
  void remove();
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

两个接口有什么区别呢？

> An `Iterable` is a simple representation of a series of elements that can be iterated over. It does not have any iteration state such as a “current element”. Instead, it has one method that produces an `Iterator`.
>
> An `Iterator` is the object with iteration state. It lets you check if it has more elements using `hasNext()` and move to the next element (if any) using `next()`.
>
> Typically, an `Iterable` should be able to produce any number of valid `Iterator`s.
>
> 摘自StackOverFlow

实际上Java的两个接口的设计和Python的实现非常类似，

实现了Iterable接口返回一个迭代器，而不在乎迭代器的内部实现（也就是iteration state的保存），甚至可能根据需求返回不同iterator。

而在Iterator内部则需要实现`hasNext()`和`next()`这两个方法，用的很多的场合就是 `for-each loop`里面。

看个简单的例子：

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
29
30</code></pre></td>
<td class="code"><pre><code>class MyIterator implements Iterator&lt;Integer&gt; {

    private int[] myList;
    private int index;

    public MyIterator(int[] myList) {
        this.myList = myList;
        this.index = 0;
    }

    @Override
    public boolean hasNext() {
        return index &lt; myList.length;
    }

    @Override
    public Integer next() {
        return myList[index++];
    }
}

public static void main(String[] args) {
        int[] list = {1,2,3};
        MyIterator myIterator = new MyIterator(list);
        while(myIterator.hasNext()) {
            System.out.println(myIterator.next());
              // 1 2 3
        }

}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

和Python非常的类似，只不过Java没有yield这种黑科技，不再赘述了。（Java代码真的长…)

## <a href="#Summary" class="headerlink" title="Summary"></a>Summary

1.  **迭代器就是符合迭代协议的对象**
2.  **生成器是通过yield关键字来产生迭代器的函数**
3.  **Java中类似的实现的Iterator和Iterable接口**
4.  **多多纵向对比语言的实现，可以帮助更好的理解**
