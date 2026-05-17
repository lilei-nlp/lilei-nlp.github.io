---
title: "BitOperators"
date: 2017-01-31
categories:
  - blog
tags: []
---

昨天看的几个很有意思的问题 都是用 二进制方法解决的  
今天学习的PKU的算法课里面有很有意思的熄灯问题 也是用二进制方法来简便的解决思路  
二进制方法 最为重要的一点是  
**二进制的思想 和 二进制位的操作**

这篇主要讲二进制的位操作，以后再记录一下关于思想的一些思考。

按位运算符主要有：  
& 按位与 （AND）  
| 按位或 （OR）  
^ 按位异或 （XOR）  
左移&lt;&lt;  
右移&gt;&gt;  
~ 按位取反

& 的使用很简单 1 & 0 = 0, 1 & 1 = 1 , 0 & 0 = 0 ; 只有两边都为 1 时为 1  
常用于 将二进制位 置0  
| 同样 1 | 0 = 1, 1 | 1 = 1 ，0 | 0 = 0 ；只有两边都为 0 时为 0  
常用于 将二进制位 置1  
^当两个操作数对应位不同时 为 1 相同时 为 0 1 ^ 0 = 1, 1 ^ 1 = 0 , 0 ^ 0 = 0;  
移位操作 &lt;&lt; &gt;&gt;分别用于将运算的操作数进行左移 与右移  
00111 &lt;&lt; 2 得到的结果就是 11100 等价于对表达式 乘 4  
左移补0，右移不同机器有不同的移位方式 有用符号位填补（算数移位） 也有用0填补 （逻辑移位)

<span id="more"></span>

看一个TCPL上的例子  

<figure class="highlight c">
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
<td class="code"><pre><code>unsigned getBits(unsigned x ,int p, int n ){
    return (x &gt;&gt; (p + 1 - n ) &amp; ~ (~ 0 &lt;&lt; n ) ) ;
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

二进制位 习惯上 **从右往左数**  
x &gt;&gt; ( p +１－ｎ） 将期望得到的字段移到最右端  
要保留这n个位 制作出 00000…111 进行 &  
所以 利用 ~0 全是1的屏蔽码 左移n位 得到 1111..000  
然后取反 得到 ~(~0 &lt;&lt; n )  
再与 x 的右端进行 And 操作  
即 (x &gt;&gt; (p + 1 - n ) & ~ (~ 0 &lt;&lt; n ) )

再来看一道练习题

> 编写一个函数 setBits(x,p,n,y) 该函数返回对x执行下列操作后的结果值:  
> 将x中从第p位开始的n个二进制位设置成y中最右边的n位的值 x的其余各位保持不变。

我们就是要把  
x: xxxx…nnn…xxx 和  
y: yyyy…yyy…nnn 进行一个对应位的交换  
怎么做呢  
首先是我们要把 x 的 n个位 置零 得到  
xxxx…000…xxx 然后和 经过 置 1 操作的 y’  
0000…nnn…000 进行 OR 操作 就可以完成赋值了

怎么得到 xxxx…000…xxx 呢  
也就是要把原来的  
xxxx…nnn…xxx 和  
1111…000…111 进行一个 AND 操作  
得到 1111…000…111的方法 就很简单  
利用 **~0** 这个全部都是1的屏蔽码 左移n位 ~0 &lt;&lt; n得到 1111…111…000(n个)  
然后 取反 即 ~(~0 &lt;&lt; n ) 得到 0000…000…111(n个)  
再将其移动到 p位 处 即 (~(~0 &lt;&lt; n) &lt;&lt; (p -n + 1） 得到 0000…111…000  
取反再移动到对应位上 利用自动补0这一特点 就变得更为轻松了 先移动的话就会得到 1111…000…000 这种尴尬地屏蔽码了  
然后再取一次反 就得到我们最终需要的屏蔽码了 即 ~((~(~0 &lt;&lt; n) &lt;&lt; (p -n + 1）)  
x & ~((~(~0 &lt;&lt; n) &lt;&lt; (p -n + 1）) 就是我们要的 p以后n位置0 以后的 x

怎么得到 1111…nnn…111呢  
其实就是保留y右端的n位  
利用 ~0 &lt;&lt; n 得到 1111…111…000  
取反 ~(~0 &lt;&lt; n ) 得到 0000…000…111  
再和y进行 AND 即 y & ~(~0 &lt;&lt; n )得到 0000…000…nnn  
左移 (y & ~(~0 &lt;&lt; n )) &lt;&lt; (p -n + 1 ) 得到 0000…nnn…000

所以仿照着例子 我们可以写出这样的函数  

<figure class="highlight c">
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
<td class="code"><pre><code>unsigned setBits(unsigned x ,int p, int n ,int y ){
    return x &amp;  ~((~(~0 &lt;&lt; n) &lt;&lt; (p -n + 1）) |
        (y &amp; ~(~0 &lt;&lt; n )) &lt;&lt; (p -n + 1 ) ;
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

大功告成

总结一下：主要用到的技巧就是  
OR 和 0 来进行保留原有位  
AND 和 0 来 进行置 0  
利用取反 和 移位 来构造恰当的 屏蔽码 实现位操作
