---
title: "动态规划学习笔记1"
date: 2018-02-02
categories:
  - blog
tags: []
---

菜鸡还是直面自己的弱点才能成长啊，算法还是一块巨大的短板，需要好好补一补。从自己觉得很玄的动态规划（Dynamic Programming，DP） 开始，从几个经典的题型入手，主要参考刘汝佳的《算法竞赛入门经典》。

## <a href="#Longest-Increasing-Subsequence" class="headerlink" title="Longest Increasing Subsequence"></a>Longest Increasing Subsequence

POJ 2533 这道最长上升子序列问题是非常经典的 DP 问题，问题描述摘录如下:

> A numeric sequence of *ai* is ordered if *a<sub>1</sub>* &lt; *a<sub>2</sub>* &lt; … &lt; *a<sub>N</sub>*. Let the subsequence of the given numeric sequence (*a1<sub>1</sub>,* a<sub>2</sub>, …, *a<sub>N</sub>*) be any sequence (*a<sub>i1</sub>*, *a<sub>i2</sub>*, …, *a<sub>iK</sub>*), where 1 &lt;= *i1* &lt; *i2* &lt; … &lt; *iK* &lt;= *N*. For example, sequence (1, 7, 3, 5, 9, 4, 8) has ordered subsequences, e. g., (1, 7), (3, 4, 8) and many others. All longest ordered subsequences are of length 4, e. g., (1, 3, 5, 8).
>
> Your program, when given the numeric sequence, must find the length of its longest ordered subsequence.

题意很简单，怎么做呢？简单粗暴的穷举必然是一种方法，找出所有的子串，验证其单调性，复杂度为O(2^n)。

另外一种普遍的解法就是动态规划了，设 `d[i]` 是 `以 i 位置结尾的数能够成的上升子序列的长度最大值` ，那么必然有：

**$d\[i\] = max(d\[i\], d\[j\]+1) \\ if \\ a\[j\] &lt; a\[i\] \\ and \\ i &lt; j$**

（PS：刘汝佳的书上的公式给的是 `d[i] = max(0, d[j]) + 1`，有问题。

有了这个式子，我们就不难写出下面这段核心代码：

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
<td class="code"><pre><code>for(int i = 0 ; i &lt; n ; i++) d[i] = 1; // 初始化
for (int i = 0; i &lt; n; ++i)
 for (int j = 0;  j &lt; i; ++j) 
        if(arr[j] &lt; arr[i]) 
          d[i] = max(d[j] + 1, d[i]); // 状态转移方程</code></pre></td>
</tr>
</tbody>
</table>
</figure>

上面的那个式子叫做状态转移方程，而 `d[i]` 叫做位置 `i` 的状态。很显然，因为**自身构成一个 LIS，长度为 1**，所以我们需要把所有状态都初始化为 1。最后的解雇就是 `max(d[i])`。复杂度是O(n^2)，还存在着 O(nlogn) 的优化方法，就不展开了。

从这个例子我们能够发现，动态规划最重要的就是**设计合适的状态和状态转移方程**，从而把问题变小，也是一种分而治之的思想。

## <a href="#0-1-背包问题" class="headerlink" title="0-1 背包问题"></a>0-1 背包问题

我一开始听到 0-1 背包就觉得是个厉害的不行的东西肯定很难，其实也还是一个动态规划问题，问题描述如下：

> 0-1 背包
>
> 有一个窃贼在偷窃一家商店时发现有n件物品，第i件物品价值为vi元，重量为wi，假设vi和wi都为整数。他希望带走的东西越值钱越好，但他的背包中之多只能装下C磅的东西，W为一整数。他应该带走哪几样东西？

这里的核心就是设计状态转移方程，因为每一件物品只有两种状态，取或者不取，所以利用这一点，对于第 i 件物品，考虑当前剩余容量 j，那么用 **d(i, j) 表示取第 i 个，i+1，…，n 件物品能取得的最大价值**，则有

> d(i, j) = max( d(i+1, j), d(i+1, j-W<sub>i</sub>) + V<sub>i</sub>)

d(i+1, j) 的含义即为不取第 i 个物品，考虑下一个；d(i+1, j-W<sub>i</sub>) + V<sub>i</sub> 则是取了第 i 个，相应的重量减少为 `j - W<sub>i</sub>`继续考虑第 i+1 个，因此我们可以写出下面的这段核心代码

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
6</code></pre></td>
<td class="code"><pre><code>for(int i = n ; i &gt;= 1; i-- )  // 从后向前考虑
 for(j = C ; j &gt;= 0 ; j --){
      d[i][j] = (i == n ? 0 : d[i+1][j]);// 边界情况 
          if( j &gt;= W[i]) // 容量允许
               d[i][j] = max(d[i][j], d[i+1][j-W[i]] + V[i]);
    }</code></pre></td>
</tr>
</tbody>
</table>
</figure>

其中有一点需要注意：N 从后向前考虑，也就是最简单的情况，保证 i+1 能够继续递推，最终的答案即为 d(1, C) ；如果要从前向后的话，即定义 **f(i, j) 为把前 i 个物品装入容量为 j 的最大价值**，类似我们可以得到

> f(i, j) = max( f(i-1, j), f(i-1, j - W<sub>i</sub>) + V<sub>i</sub>)

这里的边界情况也对应的修改为 `i=0` 时，价值为 0。这种从前向后的的代码如下：

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
7</code></pre></td>
<td class="code"><pre><code>for(int i = 1; i &lt;= n; i ++)
    for(int j = 0 ; j &lt;= W; j++) {
        cin &gt;&gt; W &gt;&gt; V; // 直接计算 不用保存
        f[i][j] = (i == 0 ? 0 : f[i-1][j]);
        if(j &gt;= V)
            f[i][j] = max(f[i][j], f[i-1][j-W]+V);
    }</code></pre></td>
</tr>
</tbody>
</table>
</figure>

其带来的一个显著的好处就是不用对数据进行保存。

然而使用二维数组，其大小 N\*M 是一种巨大的浪费，甚至会超出一些题目的限制，因此我们可以使用滚动数组来节约空间的开销，即吧 f 数组变成一维的

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
7</code></pre></td>
<td class="code"><pre><code>memset(f, 0, sizeof(f)); // tips: memset 对int数组只设置为 0/-1 
for(int i = 1; i&lt;= n ; i ++) {
    cin &gt;&gt; W &gt;&gt; V;
   for(int j = C; j&gt;= 0; j --) 
        if(j &gt;= V)
            f[j] = max(f[j], f[j-W] + V);
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这边我们先把 j 的顺序改成了逆序，在前面两种方法中，j 枚举的顺序是没关系的。这样，f 数组的计算顺序就是从上到下，从右往左。在计算 `f(i, j)` 之前，`f[j]` 里保存着`f(i-1, j)` ，而 `f[j- W]` 里保存着的是 `f(i-1, j - W)`，因为 `f(i, j)` 的计算所依赖的 `f(i-1, j)` 和 `f(i-1, j - W)` 都已经在之前计算出来，所以我们可以利用先前的值更新并且覆盖，达到节约空间的效果。

所以，可以得出一个经验，**使用滚动数组在一些旧值使用不多的情况下，可以大幅减少内存**。当然，滚动数组也会带来打印结果不便的困难，再一次证明了“天下没有免费的午餐”这一定律。

## <a href="#Summary" class="headerlink" title="Summary"></a>Summary

DP 的学习才刚刚开始，但这篇文章已经拖了很久，所以决定在年三十先告一段落吧。再小小的总结一下：

1.  DP 思想，是分治的一种，核心是设计好的状态转移方程
2.  何时用滚动数组，优点缺点
3.  memset 的使用注意点，对 int 数组只能设置为 0 或 -1（保证高字节和低字节相同）
4.  边界情况的考虑
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">ACM</a>
