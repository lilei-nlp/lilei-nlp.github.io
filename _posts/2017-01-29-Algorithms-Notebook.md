---
title: "Algorithms Notebook"
date: 2017-01-29
categories:
  - blog
tags: []
---

突然萌生了一个想法 就是收集一些经典、优雅的算法  
没事多翻翻看看 能够沉淀下来一些就好了

1、最大子序列的和  

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
13</code></pre></td>
<td class="code"><pre><code>public static int maxSubsequenceSum(int[] a ,int N){
    int thisSum,MaxSum;
    thisSum = MaxSum = 0 ;
    for (int i = 0; i &lt; N; i++) {
        thisSum += a[i];
        if(thisSum &gt; MaxSum){
            MaxSum = thisSum;
        }else if(thisSum &lt; 0){
            thisSum = 0 ;
        }
    }
    return MaxSum;
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

在数列不是全部为负数的前提下  
算法很巧妙的用两个变量来储存 当前子列和 以及 最大子列和  
并且及时地交换二者的值或者是 在当前子列和 小于0时 重置为 0  
实现 **一遍遍历就能得到最大子列和的目的** 即 复杂度为O(n)

<span id="more"></span>

2、二分查找  

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
<td class="code"><pre><code>public static int binarySearch(int key,int[] a){
    int lo = 0 ;
    int hi = a.length -1 ;
    while( lo &lt;= hi){
         int mid = (lo + hi) / 2 ;
         if ( key &lt; a[mid]) hi = mid - 1 ;
         else if( a[mid] &lt; key) lo = mid + 1 ;
         else return mid ;
    }

    return -1 ;
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

是一个很经典的算法 前提是数列有序 复杂度为 O(lg n)

3、进制转换  

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
4
5
6
7
8</code></pre></td>
<td class="code"><pre><code>char digits[] = {&#39;0&#39;,&#39;1&#39;,&#39;2&#39;,&#39;3&#39;,&#39;4&#39;,&#39;5&#39;,&#39;6&#39;,&#39;7&#39;,&#39;8&#39;,&#39;9&#39;,&#39;A&#39;,
               &#39;B&#39;,&#39;C&#39;,&#39;D&#39;,&#39;E&#39;,&#39;F&#39;}; //全局变量
    void convert(int x,int y){
       if( x != 0) {
           convert(x /y , y);
            printf(&quot;%c&quot;,digits[x % y]);
     }
    }</code></pre></td>
</tr>
</tbody>
</table>
</figure>

一个精巧的进制转换代码，将十进制整数 x 转换为 y 进制 然后输出。  
进制转换无非就是 除 然后 取余 ，但是有一个输出和处理过程是颠倒的问题。  
精妙之处在于利用递归解决了输出需要逆置的问题，也可以用栈结构来解决。

待续…

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color3">算法</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2017/01/29/Algorithms-Notebook/)