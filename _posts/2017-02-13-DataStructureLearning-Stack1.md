---
title: "数据结构学习笔记-栈的实现"
date: 2017-02-13
categories:
  - blog
tags: []
---

栈和向量、列表一样，是**线性序列**的一种  
但它的核心特点在于：**只能访问和操作序列的一端（栈顶）** 并且是 **LIFO (Last In First Out)**  
可以用一叠盘子来进行比喻，对于一叠盘子，我们对它进行操作的时候，要么是拿走顶上的一块盘子，要么就是在顶上放一块盘子。  
所以对应的，栈有最为基本的两个操作，push(入栈)和pop(出栈)

以下就用C语言来实现一个数据类型为int的简单的栈结构

<span id="more"></span>

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
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65</code></pre></td>
<td class="code"><pre><code>#include&lt;stdio.h&gt;
#include&lt;stdlib.h&gt;

typedef struct Node{ // 栈节点 其实实现的是链表的一个节点
    int data ; //数据域
    struct Node* next; // 下一个节点的指针
}Node;

typedef struct Stack{ //栈结构 
    int size ; // 栈的大小
    Node* top ; 
    Node* bottom ; 
}Stack;

void init(Stack* ps){ //初始化栈函数
    ps-&gt;size = 0 ; // 这里 ps是一个栈指针 结构体指针-&gt;成员 是 结构体-&gt;成员 的一种简便写法 
    Node* node = malloc(sizeof(Node));
    ps-&gt;bottom = ps-&gt;top = node ; //初始化一个空指针作为栈底
    ps-&gt;top-&gt;next = NULL;
}

void push(Stack* ps,int data){ // 入栈函数
    ps-&gt;size ++ ; //  栈长度加 1
    Node* newNode = malloc(sizeof(Node)); //为新节点申请内存空间
    newNode-&gt;data = data ; // 为新节点赋值
    newNode-&gt;next = ps-&gt;top ; // 注意接下来两步的顺序  先将new-&gt;next设为ps-&gt;top 再将newNode作为栈顶
    ps-&gt;top = newNode;
}

void pop(Stack* ps){
    if(ps-&gt;size &lt; 1){
        printf(&quot;Empty Stack!&quot;);//空栈无法出栈
        return ;
    }
    ps-&gt;size -- ; // 栈长度减一
    Node* temp = ps-&gt;top ; //temp指针用来进行内存释放操作 避免内存泄露
    ps-&gt;top = ps-&gt;top-&gt;next ; // 将栈顶的下一个节点作为栈顶
    free(temp); //释放栈顶内存空间
    temp = NULL ; // 安全操作
}

void tranverse(Stack* ps){ //遍历栈结构 并输出数据
    int i ;
    if(ps-&gt;size &lt; 1){
        printf(&quot;Empty Stack!&quot;);//空栈无法遍历
        return ;
    }
  while( ps-&gt;size &gt; 0){
     printf(&quot;%d\n&quot;,ps-&gt;top-&gt;data);
     pop(ps);
  }
}


int main(){
    Stack stack ;
    init(&amp;stack);
    push(&amp;stack,1);
    push(&amp;stack,5);
    push(&amp;stack,2);
    printf(&quot;size :%d\n&quot;,ps-&gt;size);
    pop(&amp;stack);
    printf(&quot;size :%d\n&quot;,ps-&gt;size);
    tranverse(&amp;stack);
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

输出结果

> size :3  
> size :2  
> 5  
> 1

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color5">数据结构</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color2">栈</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2017/02/13/DataStructureLearning-Stack1/)