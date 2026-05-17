---
title: "数据结构学习笔记-栈的应用"
date: 2017-02-13
categories:
  - blog
tags: []
---

栈的特点 LIFO 后进先出 使得它在处理一些相关问题的时候非常强大

> 1、逆序输出(conversion)  
> 特点：1.输出与处理过程相反 2.递归深度、输出长度不易预知  
> 典型问题: 进制转换  
> 2、递归嵌套  
> 特点:1.具有自相似的问题可递归描述 2.分支和位置不确定  
> 典型问题: 括号匹配  
> 3、延迟缓冲  
> 特点:线性扫描算法中要预读足够长才能够处理前缀  
> 典型问题: 中缀表达式  
> 4、RPN (逆波兰表达式) 基于栈结构的特定计算模式  
> —— 清华大学 邓俊辉《数据结构》

<span id="more"></span>

这里就举一道 括号匹配 和 布尔表达式求值 的题目作为例子，尝试利用栈结构来解决一些问题。

> 括号匹配  
> 对于给定的字符串 只包含( { \[ \] } ) 括号可以嵌套使用  
> 合法如 “((){}\[{}\])” 则输出Yes 非法如 “((){}\[)” 则输出 No

对题目进行初步分析后，我们想，怎么才能缩小问题的规模呢?  
对于给定的一串括号 我们发现  
**遇到右括号时如果存在对应的左括号 则可以消去这对括号 原问题的合法性和其消去一对括号的合法性是相同的**  
比如说 如果”((){})”合法 那么 “({})”也必然是合法 那么”()”合法 就回归到一个最基本的情形了  
我们通过消去匹配的括号，来缩小问题规模  
也就是 **减而治之(Decrease and Conquer)** 的策略  
利用栈结构，我们对输入的字符串进行扫描  
如果我们遇到一个左括号，则令其入栈，遇到右括号，则令栈顶弹出。  
如果不匹配，则终止扫描，输出No;如果匹配，弹出栈顶的左括号，就达到了消去括号的目的。  
如果最后整个栈为空栈，则证明所有的括号都成功消去，即匹配。  
以下是C语言实现的代码

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
65
66
67
68
69
70
71</code></pre></td>
<td class="code"><pre><code>#include&lt;stdio.h&gt;
#include&lt;stdlib.h&gt;
#include&lt;string.h&gt;

//括号匹配 通过栈来实现
//将所有的左括号 压入栈 然如果碰到右括号 则令左括号弹出 
...
//为节省篇幅 省略定义栈结构相关代码

char pop(Stack* ps){ //出栈 并且返回出栈的节点的数据 这里的数据域是char类型
   if(ps-&gt;size == 0){
       printf(&quot;Empty stack!&quot;);
     return ;
 }else{
     ps-&gt;size -- ;
      char ret = ps-&gt;top-&gt;data;
      Node* temp = ps-&gt;top ;
     ps-&gt;top = ps-&gt;top-&gt;next;
     free(temp);
     temp = NULL ;
        return ret ;
 }
}

int isEmpty(Stack* ps){ // 判断栈是否为空栈
 return ps-&gt;size &gt; 0 ? 0 : 1;
}

int checkPro(Stack* stack,char * s ){ //检测括号匹配
  int i ;
  char c ;
 for( i = 0 ; i &lt;strlen(s); i++){
       if(s[i] == &#39;(&#39;){
            push(stack,&#39;s&#39;); // 小括号 对应的数据为 &#39;s&#39; 入栈
      }else if (s[i] == &#39;[&#39;){
           push(stack,&#39;m&#39;);// 中括号 对应的数据为 &#39;m&#39; 入栈
       }else if(s[i] == &#39;{&#39;){
           push(stack,&#39;l&#39;);// 大括号
     }else if(s[i] == &#39;)&#39; &amp;&amp; !isEmpty(stack)){ //碰到右小括号 且 栈不为空
          c = pop(stack); // 栈顶出栈
            if( c != &#39;s&#39;){ // 弹出的栈 不匹配
             return 0;
            }
        }else if(s[i] == &#39;]&#39; &amp;&amp; !isEmpty(stack)){//同小括号
            c = pop(stack);
         if( c != &#39;m&#39;){
              return 0;
            }
        }else if(s[i] == &#39;}&#39; &amp;&amp; !isEmpty(stack)){//同小括号
           c = pop(stack);
         if( c != &#39;l&#39;){
              return 0;
            }
        }
    }
    return isEmpty(stack); // 如果扫描进行结束没有退出 栈为空则匹配 非空则说明有多余的左括号
}


int main(){
    Stack stack ;
   init(&amp;stack);
   char s[100];
 scanf(&quot;%s&quot;,&amp;s);
 if(checkPro(&amp;stack,s)){
       printf(&quot;yes&quot;);
  }else{
     printf(&quot;No&quot;);
   } 
   return 0;
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

Boolean Expressions  
\[POJ Boolean Expressions\]\[1\]  
\[1\]: <a href="http://poj.org/problem?id=2106" target="_blank" rel="noopener">http://poj.org/problem?id=2106</a>  
利用栈结构进行表达式求值，主要实现的思路是：  
1、操作数与值分离，用独立的两个栈来存放操作数和值  
2、判断操作数的优先级，一般来说，如果当前操作数优先级低于栈顶操作数的优先级，则说明栈顶的操作数可以执行了，就弹出操作数栈顶和相应的值栈，进行运算，并且将运算结果重新压入值栈中  
这一题布尔表达式求值 实际上操作数只有 () ! | & 五个  
(优先级最高 !次之 &再次 然后是| )可以定义为最低  
然后因为这题值栈事实上就是V/F 操作数栈也不过5个操作数 所以可以用两个数组来进行模拟简单的栈  
又由于!的优先级是很高的 所以我们可以把 !V !F 看做是一个值 来进行压入值栈的操作  
计算只要负责 | 和 & 就可以了

以下是借鉴自某CSDN博客的代码

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
65
66
67
68
69
70
71</code></pre></td>
<td class="code"><pre><code>#include&lt;cstdio&gt;  
#define MAX 150

bool val[MAX]; // 值栈 
int op[MAX];  //操作数栈 
int vp, pp; // 栈顶下标

/* 定义优先级  
    (:  0  
    | : 1 
   &amp; : 2
    ! : 3
    ) : 4
*/   

void insert(bool b){  //入栈操作 
    while( pp &amp;&amp;  (op[pp-1] == 3) ){ // 处理!操作 如果操作数栈优先级为3 则说明为!操作 
     b = ! b; // 取反
       --pp; // 栈顶下标-1 相当于 弹出栈顶操作数
  }
    val[vp++] = b; // 将值压入栈中 并且vp++ 栈顶下标+1 
}

void calc(){ // 计算操作
  bool b = val[--vp]; // 取出一个值 
   bool a = val[--vp];
  int opr = op[--pp]; //取出一个操作数  这里的pp 和 vp 原来都是比实际栈顶下标大 1的
   insert(opr == 1 ? a | b : a &amp; b ); // 压入结果  如果操作数为1即为 | 不为1 则为 &amp;
}

int main(){
   int cnt = 0; // 计算第几个表达式的值
  char c ;
 while( ~(c = getchar()) ){ // 这是个小技巧 EOF = -1 按位取反 等价于 (c = getchar()) != EOF
      vp = pp = 0 ; // 初始化下标为 0 pp也代表着操作数的数量
       
      do{
         if( c == &#39;(&#39;){ //左括号 压入栈中 
             op[pp++] = 0 ;
            }else if( c == &#39;)&#39;){
              while(pp &amp;&amp; op[pp-1]){ //处理左括号以前所有的操作 处理完之后相当于消去了()
                    calc();
               }
                --pp; // pp-- 相当于弹出 ) 
               insert(val[--vp]); // 如果表达式为 !() 形式 还要计算一次! 
         }else if( c == &#39;!&#39;){
              op[pp++] = 3; //操作数 ! 入栈  
           }else if( c == &#39;|&#39;){ 
             while( pp &amp;&amp; op[pp-1] &gt;= 1){  // 计算|之前 优先级比|高的操作
                    calc();
               }
                op[pp++] = 1; // 压入 | 操作
         }else if( c == &#39;&amp;&#39;){
              while( pp &amp;&amp; op[pp-1] &gt;= 2){ // 计算&amp;之前 优先级比&amp;高的操作
                 calc();
               }
                op[pp++] = 2 ;// 压入 &amp; 操作
         }else if( c == &#39;V&#39; || c == &#39;F&#39;){
              insert(c == &#39;V&#39; ? true : false); //压入 true or false
            }
            // 空格被忽略
     }while((c = getchar())!= &#39;\n&#39; &amp;&amp;  ~c); // ~c 等价于 c != EOF
      
      while(pp){ // 如果还存在操作数 继续计算
            calc(); 
      }
        //输出结果
       printf(&quot;Expression %d: %c\n&quot;,++cnt,(val[0] ? &#39;V&#39; : &#39;F&#39;));
   }
    return 0;
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这个代码写的我个人觉得非常的优雅，  
首先是通过将运算符分优先级后编号，然后就利用两个数组来实现值栈和操作数栈，简洁而有效。  
然后是两个pp vp 操作数下标 和 值的下标 既作为栈顶 又作为操作数的数量 实际写的过程中 ++ – 其实是很容易搞错的  
并且将 !的计算放在了 calc()的函数中 这样就可以同等对待剩余的 | 和 &  
还有特殊的 !()模式的处理 以及 剩余操作数的 处理  
非常值得学习的一段代码，努力把它消化了

总结一下：  
栈结构很强大，LIFO的特点使得它在表达式求值、相关匹配的算法题中起到很大作用。  
但也不一定要拘泥于栈结构的形式，数组也可以作为栈。而且一般向量(数组)作为栈的  
会以**向量尾部**作为栈顶，因为这样出栈的操作时间复杂度就是O(1) 而如果在头部， 则是**O(n) 和向量长度成正比**，值得注意。

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color5">数据结构</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color2">栈</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2017/02/13/DataStructureLearning-Stack2/)