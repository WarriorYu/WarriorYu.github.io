---
title: Java 并发核心知识体系
categories:
  - Java基础
tags:
  - 并发
date: 2020-08-05 11:08:13
---



 

 **笔记**

**Java 并发核心知识体系精讲**

**一. 线程八大核心**

**第3章：实现线程的方式：** 

![Introduction](/images/thread1.png)

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread1.png" alt="thread1" style="zoom:33%;" />

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread2.png" alt="thread1" style="zoom:33%;" />

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread3.png" alt="thread1" style="zoom:33%;" />

**第4章：线程启动**

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread4.png" alt="thread1" style="zoom:33%;" />

**直接调用run()是在主线程或者当前子线程执行。而start()方法，先检查线程的状态，通过native方法start0()再启动新线程。**

**第5章：如何正确停止线程**

**1. 原理介绍：使用interrupt来通知，而不是强制。**

**2.**

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread5.png" alt="thread1" style="zoom:33%;" />

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread6.png" alt="thread1" style="zoom:33%;" />

**3.**

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread7.png" alt="thread1" style="zoom:33%;" />

**4.**

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread8.png" alt="thread1" style="zoom:33%;" />

**5.** 

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread9.png" alt="thread1" style="zoom:33%;" />

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread10.png" alt="thread1" style="zoom:33%;" />

```java
/**
 * Create   : 2020/7/31
 * Describe : 注意Thread.interrupted()方法的目标对象是"当前线程"，而不管本方法来自于哪个对象
 */
public class RightWayInterrupted {
    public static void main(String[] args) throws InterruptedException {
        Thread threadOne = new Thread(new Runnable() {
            @Override
            public void run() {
                for (; ; ) {

                }
            }
        });

        threadOne.start();
        threadOne.interrupt();
        // 获取中断标志
        System.out.println("isInterrupted:" + threadOne.isInterrupted());
        // 获取中断标志并重置
        // 因为interrupted()是静态方法，不管是threadOne还是Thread，
        // 执行它的都是主函数main函数，main函数没有经过任何的中断，所以返回false。
        System.out.println("isInterrupted:" + threadOne.interrupted());
        // 获取中断标志并重置 原理同上
        System.out.println("isInterrupted:" + Thread.interrupted());
        // 获取中断标志
        System.out.println("isInterrupted:" + threadOne.isInterrupted());
        threadOne.join();
        System.out.println("Main thread is over");

        /**
         * 打印结果：
         *
         * isInterrupted:true
         * isInterrupted:false
         * isInterrupted:false
         * isInterrupted:true
         */
    }
}
```

**6.线程的 6个状态（生命周期）**

1. Java线程的6种状态及切换(透彻讲解)：https://blog.csdn.net/pange1991/article/details/53860651

2.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread11.png" alt="thread1" style="zoom:20%;" />

3.

1. **初始(NEW)**：新创建了一个线程对象，但还没有调用start()方法。

2. **运行(RUNNABLE)**：Java线程中将就绪（ready）和运行中（running）两种状态笼统的称为“运行”。

线程对象创建后，其他线程(比如main线程）调用了该对象的start()方法。该状态的线程位于可运行线程池中，等待被线程调度选中，获取CPU的使用权，此时处于就绪状态（ready）。就绪状态的线程在获得CPU时间片后变为运行中状态（running）。

3. **阻塞(BLOCKED)**：表示线程阻塞于锁。一定是Synchronized修饰的代码块或者方法才有这个状态。

4. **等待(WAITING)**：进入该状态的线程需要等待其他线程做出一些特定动作（通知或中断）。

5. **超时等待(TIMED_WAITING)**：该状态不同于WAITING，它可以在指定的时间后自行返回。

6. **终止(TERMINATED)**：表示该线程已经执行完毕。

4.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread12.png" alt="thread1" style="zoom:33%;" />

5.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread13.png" alt="thread1" style="zoom:33%;" />

6.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread14.png" alt="thread1" style="zoom:33%;" />

根据上面第4点的图解答即可。

**第7章 核心5：趣解Thread和Object类中线程相关方法**

1.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread15.png" alt="thread1" style="zoom:25%;" />

2.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread16.png" alt="thread1" style="zoom:25%;" />

3.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread17.png" alt="thread1" style="zoom:25%;" />

4.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread18.png" alt="thread1" style="zoom:25%;" />

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread19.png" alt="thread1" style="zoom:25%;" />

5.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread20.png" alt="thread1" style="zoom:25%;" />

6.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread21.png" alt="thread1" style="zoom:25%;" />

7.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread22.png" alt="thread1" style="zoom:25%;" />

8.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread23.png" alt="thread1" style="zoom:25%;" />

9.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread24.png" alt="thread1" style="zoom:25%;" />

**第8章 核心6：一网打尽线程属性**

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread25.png" alt="thread1" style="zoom:25%;" />

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread26.png" alt="thread1" style="zoom:25%;" />

**第10章 核心8：追寻并发的崇高理想-线程安全**

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/thread27.png" alt="thread1" style="zoom:25%;" />



> **本文配套用于慕课网悟空老师的《Java并发核心知识精讲》课，本文所涵盖的知识导图的内容，在课程中有详细讲解。**
>
> **课程地址：**[**https://coding.imooc.com/class/362.html**](https://coding.imooc.com/class/362.html)
>
>  
>
> **以下思维导图会持续更新，所以请访问线上版本：**
>
>  
>
> **线程8大核心基础：**
>
> [**http://naotu.baidu.com/file/07f437ff6bc3fa7939e171b00f133e17?token=6744a1c6ca6860a0**](http://naotu.baidu.com/file/07f437ff6bc3fa7939e171b00f133e17?token=6744a1c6ca6860a0)
>
>  
>
> **Java内存模型——底层原理：**
>
> [**http://naotu.baidu.com/file/60a0bdcaca7c6b92fcc5f796fe6f6bc9?token=bcdbae34bb3b0533**](http://naotu.baidu.com/file/60a0bdcaca7c6b92fcc5f796fe6f6bc9?token=bcdbae34bb3b0533)
>
>  
>
> **死锁——从产生到消除：**
>
> [**http://naotu.baidu.com/file/ec7748c253f4fc9d88ac1cc1e47814f3?token=bb71b5895a747d67**](http://naotu.baidu.com/file/ec7748c253f4fc9d88ac1cc1e47814f3?token=bb71b5895a747d67)
>
>  
>
> **二级大纲（复习用） 成体系的Java并发多线程核心+内存模型+死锁——从用法到原理，面试必考：**
>
> [**http://naotu.baidu.com/file/29942292cd032adfae23c09783676004?token=130df3d389cab703**](http://naotu.baidu.com/file/29942292cd032adfae23c09783676004?token=130df3d389cab703)
>
>  
>
> **并发工具类分类：**
>
> [**http://naotu.baidu.com/file/3902a010470d7c1cf76fe719be124797?token=bec25304cc071bf1**](http://naotu.baidu.com/file/3902a010470d7c1cf76fe719be124797?token=bec25304cc071bf1)