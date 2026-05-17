---
title: "多线程学习笔记"
date: 2016-11-06
categories:
  - blog
tags: []
---

看着马士兵的视频学习了一些简单的多线程知识，  
就着《Java核心技术1》 把相关知识梳理一下。

- 1 线程和进程的区别:  
  机器上的每一个exe文件，打开之后就会形成一个“进程”，进程是一个运行中的程序，是系统分配资源的单位，而“线程”，则是进程中多条指令集执行的路径，线程之间共享数据。与进程相比，线程更为轻量级。如果把进程比作一条马路，那么线程就是各个车道。

- 2 实现多线程的方法：

  - 1 定义Class t 实现Runnable接口 并且重写run方法 在主线程中New Thread 并且将t传入Thread（）

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
    <td class="code"><pre><code>class R implements Runnable{
        public void run(){}
    }
    public void static void main(String[]args){
        R r = new R();
        Thread t = new Thread(r);
        t.start();
    }</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

  - 2 定义Class MyThread 继承Thread类 并且重写run方法

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
    7</code></pre></td>
    <td class="code"><pre><code>class MyThread extends Thread{
        public void run(){}
    }
     public void static void main(String[]args){
        MyThread mt = new MyThread();
        mt.start();
    }</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

    需要注意的是 **直接调用对象的run方法是方法调用 而不会启动新线程**

    <span id="more"></span>

- 3 线程的一些属性和状态：
  - 1 优先级：  
    一个线程的子类继承其父类的优先级，优先级的范围从1~10,线程调度器在有机会选择新线程时，会优先选择优先级别较高的线程。MIN\_PRIORITY = 1，MAX\_PRIORITY = 10，可以用setPriority方法来改变线程的优先级。
  - 2 状态：
    - 1 New
    - 2 Runnable **Runnable不代表线程正在运行!** 只能说线程具有获得运行时间的能力
    - 3 Blocked 被阻塞 当线程试图获得一个内部对象锁 而该锁被其他线程所持有时 该线程被阻塞
    - 4 Wating 等待 Object.wait或Thread.join 使线程“挂起” 等待通知 即Object.notify
    - 5 Timed Wating Waiting状态的计时版 当时间超出设定或接到通知 终止该态度
    - 6 Terminated 被终止 两种情况 因为run方法运行结束而自然死亡 和因未捕获的异常而异常死亡

- 4 线程同步：  
  线程同步是为了解决，因多线程而导致代码片段执行不完整，造成的异常状况。  
  实现线程同步的方法即给代码块加锁，保证其能够完整的一次执行完，而不会因为CPU的切换造成线程中断从而导致数据出错。  
  加锁的方法有两种：

  - 1 Java SE 5.0 引入了ReentrantLock类 可以用来保护代码块

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
    <td class="code"><pre><code>myLock.lock();//a ReentrantLock object

    try{
        //critical section
    }catch{
    }
    finally{
        myLock.unlock()l; //make sure the lock is unlocked even if an exception is thown
    }</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

  - 2 利用synchronized关键字来保护方法的执行

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
    20</code></pre></td>
    <td class="code"><pre><code>int b = 100;
    public synchronized void m1(){
        b = 1000;
        try{
            Thread.sleep(5000);
        }catch(InterruptedException e ){
            e.printStackTrace();
        }
        System.out.println(&quot;m1 b = &quot;  + b );
    }    
    public  synchronized void m2(){
        try{
            Thread.sleep(3000);
        }catch(InterruptedException e ){
            e.printStackTrace();
        }

        b = 2000;
        System.out.println(&quot;m2 b = &quot; + b );
    }</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

    在方法前加**sychronized**关键字来声明，那么对象的锁将保护该方法，如果该对象的其他方法试图获得该锁，那么他将进入阻塞状态，需等待拥有锁的方法执行完毕才能获得锁。  
    Tips:**1 所有阻塞式方法 sleep,wait,await都有可能抛出InterruptedException 需要处理**

        **2 sychronized关键字使用起来较为方便 应尽可能的多使用它**

- 5 条件对象和死锁

  - 1 条件对象  
    实际在设计多线程同步的程序的时候，如设计一个银行相互转账的系统，需要用同步方法来保证存取款时不被打断，金额才能正确。  
    不过在取款之前，需要设计一个变量，检测账户是否有足够的余额供取出才能继续执行取出操作。

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
    5</code></pre></td>
    <td class="code"><pre><code>public sychronized void getMoney(acount[],amount){
        while(accounts[from] &lt; amount){
            this.wait(); //当取出金额大于被取出账户余额的时候 调用wait方法 挂起线程
        }
    }</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

    当无法执行操作时，调用Object类的wait方法，使该线程**放弃锁**进入等待集，当锁可用时，该线程不能马上解除，相反，它仍会处于阻塞状态，需要等待另一个线程的通知”notify”

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
    3</code></pre></td>
    <td class="code"><pre><code>public sychronized void saveMoney(acount[];amount){
        this.notify();// this.notifyAll();
    }</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

    当有用户向账户中存钱时，需要调用notify方法，通知被阻塞的取钱的线程，检查是否有足够金额可以取出。  
    注意区别wait和sleep方法：**wait方法会释放对象锁 而sleep仍持有锁 wait方法是属于Object类的 而sleep是属于Thread类的**

  - 2 死锁  
    考虑下面的程序

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
    30
    31
    32
    33
    34
    35</code></pre></td>
    <td class="code"><pre><code>class TestDeadLock extends Runnable{
         static Object o1 = new Object();
         static Object o2 = new Object();
         public int flag = 1;
         public void run(){
         
             if( flag == 1){
                 sychronized(o1){
                  
                  }
                 sychronized(o2){
                     System.out.println(&quot;1&quot;);
                  }
             }
             if(flag == 0){
                 sychronized(o2){
                     
                 }
                 sychronized(o1){
                     System.out.println(&quot;0&quot;);
                 }
             }
         }
     }
     
     public static void main(String[]args){
         TestDeadLock td1 = new TestDeadLock();
         TestDeadLock td2 = new TestDeadLock();
         td1.flag = 1;
         td2.falg = 0;
         Thread t1 = new Thread(td1);
         Thread t2 = new Thread(td2);
         t1.start();
         t2.start();
     }</code></pre></td>
    </tr>
    </tbody>
    </table>
    </figure>

    当td1的run方法执行 锁住o1对象后试图锁住o2 而t2线程锁住了o2 t1无法得到o2的锁  
    而t2也无法得到o1的锁 于是….就发生了线程死锁 两个线程都被阻塞了 这样就是死锁 (DeadLock)  
    令人悲伤是，Java并没有任何东西可以完全避免或者打破死锁，必须仔细设计程序，确保不会出现死锁现象。

差不多就学到了这里，线程的学习就暂时告一段落，日后应用再多多钻研。  
加油~
