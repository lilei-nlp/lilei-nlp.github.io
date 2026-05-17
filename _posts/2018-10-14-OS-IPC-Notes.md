---
title: "IPC 问题"
date: 2018-10-14
categories:
  - blog
tags: []
---

操作系统线程之间的通信是一个很重要的问题，在用 C 语言在 Linux 实现经典的“生产者和消费者”以及“读者写者”问题时产生了一些疑惑，记录之。

## <a href="#Problem" class="headerlink" title="Problem"></a>Problem

进程间通信的需要解决的核心问题就是**线程之间的同步**，比如说，有一个程序，两个线程同时对一个共享变量 `cnt` 分别进行 `n` 次加法操作，如果其初始值为 0 的话，那么正确的运行结果应该是 `cnt = 2n`，但如果我们在一台单 CPU 的机器上运行的话，结果往往不是 `2n`。为什么呢？因为机器可能不会按照我们思考的方式去执行这样的指令，一种很容易想到的产生错误的方式就是：一个线程修改了值（对 `cnt` 进行加 1 ）之后还没写回去，这个时候另一个线程就读到了 `cnt` 的旧值并且在旧值之上进行了更新操作，二者都写回去的时候，`cnt` 的值只加一而不是加二。一个显而易见的解决方式就是将各自的操作都变成**原子化**，也即一个线程带来的任何改变都能被另一个线程所感知到，不过这样带来的 overhead（额外开销）过大，对性能是个很大的影响。因此伟大的计算机科学家们提出了两个概念用于解决这个问题，分别是 mutex（互斥量）和 semphore（信号量）：

> Strictly speaking, **a mutex is a locking mechanism** used to synchronize access to a resource. Only one task (can be a thread or process based on OS abstraction) can acquire the mutex. It means there will be ownership associated with mutex, and only the owner can release the lock (mutex).
>
> **Semaphore is signaling mechanism** (“I am done, you can carry on” kind of signal). For example, if you are listening songs (assume it as one task) on your mobile and at the same time your friend called you, an interrupt will be triggered upon which an interrupt service routine (ISR) will signal the call processing task to wakeup.

互斥量是一把锁，用于控制对资源的访问，同一时间只有一个线程能够获得这把锁，也只有获得锁的线程才能够释放这把锁；信号量是一个信号，用于线程之间的通讯。特别地，信号量 s 有两种操作：

- P(s)：如果 s 非零，则将 s -1 并返回；如果 s 为 0，则挂起该线程，直到 s 非 0，一般来说，是等待 V 操作的通知。
- V(s)：V 操作将 s + 1。如果有任何线程阻塞在 P 操作等待，则 V 操作会**通知**折线线程中的一个，让他苏醒完成 s - 1 操作。

这里几点需要注意：

1.  P 和 V 操作都是原子化的，也就是说检测信号值、执行操作、写回信号值这三个子操作时不可能被打断的。也只有这样，P、V 操作才有意义。
2.  V 操作唤醒等待线程时是随机的，我们无法预测哪个线程会苏醒，这里没有**先来先走**的事情存在

概念就先介绍到这里，可能还有一些不清楚的地方，就结合接下来的两个问题的代码实现，再进一步的讲解。  
<span id="more"></span>

## <a href="#Producer-amp-Consumer" class="headerlink" title="Producer &amp; Consumer"></a>Producer & Consumer

问题描述：生产者和消费者线程共享一个具有 n 个槽（slot）的有限缓冲区，生产者线程反复生产新的物品，并把它们放到缓冲区中；消费者从缓冲区中取出这些物品并且使用它们。要模拟这个问题，我们需要保证对共享变量，即缓冲区的访问是互斥的，并且，仅仅互斥还不够，还需要调度这个访问，以为存在以下两种情况：

1.  缓冲区已满：这种情况，生产者需要等待消费者消费掉缓冲区中的物品后，才有空槽来防止生产的物品，所以需要等待
2.  缓冲区已空：消费者需要等待生产者生产出物品才能够消费

现实生活这样的例子比比皆是，比如我们看视频，生产者编码视频帧，消费者解码视频并且显示出来，其中就有一个缓冲。下面就是一个代码的实现：

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
71
72
73</code></pre></td>
<td class="code"><pre><code>#include&lt;stdio.h&gt;
#include&lt;pthread.h&gt;
#include&lt;semaphore.h&gt;

#define MAX 8 // buffer size 缓冲区大小

sem_t full; // 信号量 full slots
sem_t empty; // 信号量 empty slots
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER; // 互斥量 mutex

//sem_t mutex;
int front = 0; // buffer array front
int rear = 0; // buffer array rear

void* produce(void* arg) { // 生产者函数
 int i ;
  for(i = 0 ; i &lt; MAX * 4 ; i ++) { // 生产 MAX * 4 个物品
//     printf(&quot;Producer is preparing data...\n&quot;);
     sem_wait(&amp;empty);
        //sem_wait(&amp;mutex);
       pthread_mutex_lock(&amp;mutex); // lock 
     front = (front + 1 )% MAX ;
//       printf(&quot;now top is %d \n&quot;, top);
       //sem_post(&amp;mutex);
      pthread_mutex_unlock(&amp;mutex); // unlock
      sem_post(&amp;full); 
 }
    return (void*) 1;
}


void* consume(void* arg) {
   int i ;
  for(i = 0 ; i &lt; MAX * 4; i ++ ){
//     printf(&quot;consumer is preparing ...\n&quot;);
     sem_wait(&amp;full) ; // check if there is slot to consume
       pthread_mutex_lock(&amp;mutex); // lock
      //sem_wait(&amp;mutex);
      rear  = (rear + 1 ) % MAX ;
//       printf(&quot;now bottom is %d \n&quot;, bottom);
     pthread_mutex_unlock(&amp;mutex);
     //sem_post(&amp;mutex);
      sem_post(&amp;empty);
 }
    return (void*) 2;
}

int main() {
    pthread_t t1;;
   pthread_t t2;
    pthread_t t3;
    pthread_t t4;

  int ret1, ret2, ret3, ret4;

    sem_init(&amp;full, 0, 0);
    sem_init(&amp;empty, 0, MAX);
    //sem_init(&amp;mutex, 0, 1);
 // create thread
 pthread_create(&amp;t1, NULL, produce, NULL);
   pthread_create(&amp;t2, NULL, consume, NULL);
   pthread_create(&amp;t3, NULL, produce, NULL);
   pthread_create(&amp;t4, NULL, consume, NULL);

 pthread_join(t1, (void**) &amp;ret1);
    pthread_join(t2, (void**) &amp;ret2);
    pthread_join(t3, (void**) &amp;ret3);
    pthread_join(t4, (void**) &amp;ret4);

  printf(&quot;final front: %d\n&quot;,front);
  printf(&quot;final rear: %d\n&quot;, rear);
   return 0;
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

首先我们需要使用的两个头文件：`semphore.h` 和 `pthread.h` ，其中包含了我们的函数和需要的信号量类型，这里主要介绍一下几个重要的函数：

1.  `sem_init()`：其函数原型为 `extern int sem_init __P ((sem_t *__sem, int __pshared, unsigned int __value));` 第一个参数 sem 为指向信号量结构的一个指针；第二个参数 pshared 不为０时此信号量在进程间共享，否则只能为当前进程的所有线程共享，我们的代码中都是在线程中共享，因此均为 0；value 给出了信号量的初始值。
2.  `pthread_create()` 和 `pthread_join()`：分别是创建线程的函数和等待线程结束。前者的参数分别为设定线程对象，线程属性，线程函数，最后一个参数是运行函数的参数；`join` 函数则就是线程对象，以及接受返回结果的指针。

接下来讲解核心的函数 `consume()` ：

我们先在一个循环之中进行生产操作，在消费时我们先进行对信号量 `full` 的 P 操作，检测是否有物品可供消费，如果没有，则挂起线程等待；如果有，则接下来我们要对缓冲区进行操作，所以需要拿到进入缓冲区的钥匙，也就是 **mutex**，通过 `pthread_mutex_lock(&mutex)`来获取这一把锁，同样的，如果这把锁现在在别人手里，我们也会挂起等待。拿到这把锁之后，我们就可以对缓冲去进行操作了，这里我们模拟消费的操作时将缓冲区的尾部 `rear` 加 1。而后释放缓冲区的锁，并且对 `empty` 进行 V 操作，也就是通知生产者我消耗了一个物品。

另一个生产者的函数 `produce`：就不再详细赘述了，和消费者相反，这里先要检查是否有空槽，因此对 `empty` 进行 P 操作，而后操作缓冲区，对 `front` 进行加 1 操作，结束后释放缓冲区的 mutex，以及对 `full` 进行 V 操作。

运行这个程序的时候在编译时需要加上 `-pthread` 参数，可以通过执行下列命令来运行：

> gcc -pthread consumer\_and\_producer.c -o c\_p
>
> ./c\_p

我们可以看到的结果是最终的 `front` 和 `rear` 均为 0。

这里有一点需要注意的时，同步信号量的检查（empty 和 full）要放在互斥量 mutex 的外面，否则，可能会导致死锁。比如生产者获得了缓冲区的锁，之后却无法发现没有空槽，从而挂起，而另一边消费者又无法获得缓冲区的锁导致挂起，从而你等我我等你造成死锁。

我在学习这个代码的时候有个疑问，就是我们能不能用一个 binary semphore（二值信号量）来替代 mutex？也就是把上面的 mutex 也换成一个 `sem_t`，并且将其初始值设置为 1（代码中黄色注释部分）。Stack Overflow 上有一个很不错的<a href="https://stackoverflow.com/questions/62814/difference-between-binary-semaphore-and-mutex" target="_blank" rel="noopener">回答</a>，而我的实验结果是**不建议**，我修改之后运行的结果在我的多核电脑上并不是均为 0（猜测是因为多核的原因，有待思考），而在单核的云服务器上运行结果则正确。所以，正如之前我们解释这两个概念的差别的时候，尽管 binary semphore 和 mutex 很类似，但还是要小心期间细微的差别。

## <a href="#Reader-amp-Writer" class="headerlink" title="Reader &amp; Writer"></a>Reader & Writer

有读者和写者两组并发进程，共享一个文件，当两个或以上的读进程同时访问共享数据时不会产生副作用，但若某个写进程和其他进程（读进程或写进程）同时访问共享数据时则可能导致数据不一致的错误。因此要求：

1.  允许多个读者可以同时对文件执行读操作；
2.  同一时刻，只允许一个写者往文件中写信息；
3.  任一写者在完成写操作之前不允许其他读者或写者工作；
4.  写者执行写操作前，应让已有的读者和写者全部退出。

这里有两种实现的方式，一种是写者优先，一种是读者优先，先来看读者优先实现的代码：

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
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85</code></pre></td>
<td class="code"><pre><code>#include&lt;stdlib.h&gt;
#include&lt;stdio.h&gt;
#include&lt;pthread.h&gt;
#include&lt;semaphore.h&gt;
#include&lt;time.h&gt;

struct data{
  int read_counter;
    int write_counter;
   int value;
} share_data;


int READ_THREAD_NUM = 5;
int WRITE_THREAD_NUM = 3;
int TOTAL_TRY = 10000;

int stop = 0;

int reader_num = 0;
sem_t lock_reader;
sem_t lock_writer;
sem_t w_or_r;

void* read(void* ptr) {
    while(!stop)  {
     sem_wait(&amp;lock_reader);
       reader_num += 1;
      share_data.read_counter ++ ;
      if(reader_num == 1) 
         sem_wait(&amp;w_or_r); // first reader, check if there is any writer
     sem_post(&amp;lock_reader);
       
      int tmp = share_data.value; // access the data
        
        sem_wait(&amp;lock_reader);
       reader_num --;
        if(reader_num == 0 ) 
            sem_post(&amp;w_or_r); // no more reader
     sem_post(&amp;lock_reader) ;
      // use the data read
//     printf(&quot;Reader read : %d \n&quot;, tmp);
    }
    return NULL;
}

void* write(void* ptr) {
    int idx = *(int * ) ptr;
    while(!stop) {
      sem_wait(&amp;lock_writer);
       sem_wait(&amp;w_or_r);
        share_data.write_counter ++;
      share_data.value = idx;
//        printf(&quot;writer write : %d \n&quot;, idx);
      if(share_data.write_counter &gt;= TOTAL_TRY) 
                stop = 1;
     sem_post(&amp;w_or_r);
        sem_post(&amp;lock_writer);
   }
    return NULL;
}

int main() {
 share_data.read_counter = 0;
  share_data.write_counter = 0;

   sem_init(&amp;lock_reader, 0, 1);
 sem_init(&amp;lock_writer, 0 , 1);
    sem_init(&amp;w_or_r, 0 , 1 );

  pthread_t readers[READ_THREAD_NUM];
  pthread_t writers[WRITE_THREAD_NUM];

   int i;
   for(i = 0 ; i &lt; READ_THREAD_NUM; i ++ ) 
      pthread_create(&amp;readers[i], NULL, read, NULL);
  
  int thread_args[] = {1, 2, 3};
 for(i = 0 ; i &lt; WRITE_THREAD_NUM ; i ++ )
     pthread_create(&amp;writers[i], NULL, write, (thread_args + i));
 
  printf(&quot;Final result, reader count: %d, writer count : %d \n&quot;, share_data.read_counter,
                 share_data.write_counter);
    return 0;
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这里我们用到了三个 binary 信号量 `lock_reader` 、`lock_writer` 以及 `w_or_r` ，分别用于读者数量、写者数量以及读写模式的控制。所谓读者优先，我们可以从下面两种序列来分析：

1.  `(r1, w1, r2)`： 如果 `r1` 先获得了读的权限，随后来的 `w1` 写者就会阻塞在获取 `w_or_r` 这一信号量上，注意到**`r1` 在对数据进行读取之前已经对 `lock_reader` 进行了 V 操作**，这时候 `r2` 就不会阻塞而能够读取数据，只有当读者全部完成读取退出之后，才会对 `w_or_r` 进行 V 操作，`w1` 才有机会获取写的权限；
2.  而如果是`(w1, r1, w2)` 这样的序列，`w1` 写后，`r1` 和 `w2` 同时到达，他们就会处于一个公平竞争的地位，谁先拿到 `w_or_r` 谁就运行。

这边我列出了五次运行的结果：

> Final result, reader count: 16, writer count : 2
>
> Final result, reader count: 40, writer count : 5
>
> Final result, reader count: 21, writer count : 4
>
> Final result, reader count: 31, writer count : 0
>
> Final result, reader count: 28, writer count : 5

可以看到，读者的数量是远大于写者的数量，这也就是**读优先**体现。

另一种写者优先的实现如下：

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
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
109
110
111
112
113
114</code></pre></td>
<td class="code"><pre><code>#include&lt;stdio.h&gt;
#include&lt;stdlib.h&gt;
#include&lt;semaphore.h&gt;
#include&lt;pthread.h&gt;

struct data{
    int r_c;
 int w_c;
 int value;
   int v1;
} share_data;

int READ_THREAD_NUM = 5;
int WRITE_THREAD_NUM = 3;
int TOTAL_TRY = 1000;

int stop = 0;
int reader_num = 0;
int writer_num = 0;

sem_t rlock, wlock;
sem_t w_or_r;
sem_t try_read;

void* read(void* ptr) {

    while(!stop) {
        // extra query
     sem_wait(&amp;try_read);
      sem_wait(&amp;rlock);

       reader_num += 1;
      share_data.r_c += 1;
      if(reader_num == 1 ) {
          sem_wait(&amp;w_or_r);
        }

      sem_post(&amp;rlock);
     sem_post(&amp;try_read);
      // do read
       int tmp = share_data.value;
      if( tmp != share_data.v1) {
         printf(&quot;error happen: %d v.s %d\n &quot;, tmp, share_data.v1);
           stop = 1;
     }
        sem_wait(&amp;rlock);
     reader_num --;
        if(reader_num == 0)
          sem_post(&amp;w_or_r);
        sem_post(&amp;rlock);
 }
    return NULL;

}

void* write(void* ptr) {
  int idx = * (int*) ptr;
 while(!stop) {
      sem_wait(&amp;wlock);
     writer_num += 1;
      if(writer_num == 1) 
         sem_wait(&amp;try_read);

        sem_post(&amp;wlock);

       sem_wait(&amp;w_or_r); // set write
      share_data.w_c ++;
        share_data.value = idx;
       share_data.v1 = idx;

        if(share_data.w_c &gt;= TOTAL_TRY) {
            stop = 1;
     }
        sem_post(&amp;w_or_r);
        sem_wait(&amp;wlock);
     writer_num -= 1;
      if(writer_num == 0) 
         sem_post(&amp;try_read); // no more writer
       sem_post(&amp;wlock);
//     sleep(0.001);
  }
    return NULL;
}

int main() {
 share_data.r_c = 0;
   share_data.w_c = 0;

 sem_init(&amp;lock, 0, 1);
    sem_init(&amp;wlock, 0, 1);
   sem_init(&amp;w_or_r, 0 , 1);
 sem_init(&amp;try_read, 0, 1);

  pthread_t readers[READ_THREAD_NUM];
  pthread_t writers[WRITE_THREAD_NUM];

   int i;
   for(i = 0 ; i &lt; READ_THREAD_NUM ; i ++ ) 
     pthread_create(&amp;readers[i], NULL, read, NULL);

    int thread_args[WRITE_THREAD_NUM];
   for(i = 0; i &lt; READ_THREAD_NUM ; i++ ) {
     thread_args[i] = i + 1;
       pthread_create(&amp;writers[i], NULL, write, (thread_args + i ));
    }

  for(i = 0 ; i &lt; READ_THREAD_NUM ; i++) 
       pthread_join(readers[i], 0);
  
  printf(&quot;Final Result: Reader Count: %d  Writer Count: %d \n&quot;, share_data.r_c, share_data.w_c);
    
    return 0;

}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

和读优先的区别在哪呢？就在多了一个信号量 `try_read`，放在最外层，也就是第一次获取 `rlock` 之前，我们再来思考之前的两个例子：

1.  `(r1, w1, r2)`，`r1` 在读的时候，`w1` 和 `r2` 来了，`w1` 和 `r1` 都阻塞在 `try_read` 上，处于一个公平的竞争状态。
2.  `(w1, r1, w2)`：`w1` 在写的时候，`r1` 和 `w2` 来了，`r1` 阻塞在获取 `try_read` 上，`w2` 则阻塞在 `wlock` 上，而我们注意到 **wlock 的 V 操作在 try\_read 之前**，所以 `w2` 在 `w1` 释放 `wlock` 后先于 `r1` 被唤醒，并且进入写，这个时候写者数量不为 0，`try_read` 还无法进行 V 操作，所以 `r1` 就必须等待写者完成写入才能进入。

同样，看看结果：

> Final Result: Reader Count: 5 Writer Count: 100003
>
> Final Result: Reader Count: 59630 Writer Count: 100001
>
> Final Result: Reader Count: 4 Writer Count: 100001
>
> Final Result: Reader Count: 104908 Writer Count: 100003
>
> Final Result: Reader Count: 87291 Writer Count: 100000

**在大部分情况下，读者的数量少于写者**，注意，这不是一定的，因为我们不知道线程运行之后调度的情况，并且不同的系统和硬件配置都可能有不一样的结果，**出现读者比写者多也是可能的**。就这两个实现而言，在我的多核电脑（MacOS）和单核服务器（Ubuntu）上，结果都是不一样的。

## <a href="#Conclusion" class="headerlink" title="Conclusion"></a>Conclusion

总结一下，IPC 的实现借助了信号量和互斥量完成资源的共享访问。换一个视角，原先我说的那种 overhead 很大的实现方式（即每个操作都做成原子化），其实差别就在于**原子化的粒度**，粒度越大越能够保证线程间不会相互干扰，但随之而来的就是性能的下降；而信号量和互斥量用一个比较小的粒度，对一些关键变量进行保护，从而在不带来大量 overhead 的情况下实现了线程之间的共享变量。

OS 真的是博大精深，是需要好好学的，个人觉得虽然不至于要到能搓一个内核的水平，但对其实现的原理必须还是有个清楚的认知，不然愧对科班出身这个身份。

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color3">OS</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2018/10/14/OS-IPC-Notes/)