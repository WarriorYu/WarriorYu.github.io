---
title: 深度剖析Android 10大开源框架
categories:
  - Android源码分析
tags:
  - Android第三方库源码分析
date: 2020-08-05 10:09:26
---

**第一章：介绍**

流程，原理，机制，核心类源码剖析

1. 网络框架：Okhttp / retrofit
2. 依赖注入：butterknife / dagger2
3.  异步处理：rxjava / eventbus
4. 图片框架:：glide / picasso
5. 性能优化：leakcanary / blockcanary

**第二章：okhttp**

1. OkHttpClient 和 Request 都可以通过Builder模式创建。OkHttpClient可以通过buil模式设置请求的

2. client.dispatcher().enqueue(new AsyncCall(responseCallback));

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp1.png" alt="okhttp1" style="zoom:30%;" />

3.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp2.png" alt="okhttp1" style="zoom:30%;" />

4. 问题1：okhttp 如何实现同步异步请求？

​    答：发送同步  / 异步请求都会在dispatcher 中管理其状态。

5. 问题2：到底什么是dispatcher?

​    答：dispatcher 的作用为维护请求的状态，并维护一个线程池，用于执行请求。

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp3.png" alt="okhttp1" style="zoom:30%;" />

6.问题3：异步请求为什么需要两个队列？

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp4.png" alt="okhttp1" style="zoom:30%;" />

7. Okhttp 的任务调度

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp5.png" alt="okhttp1" style="zoom:30%;" />

​       答：会在一个请求结束后，调用 Dispatcher 里的finish方法时，finish方法里又调用了promoteCalls()，这里会把readyAsycCalls里等待的请求添加到runningAsyncCalls，并且通过线程池executorService().execute(call)执行这个请求。

8. 

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp6.png" alt="okhttp1" style="zoom:30%;" />

9. Okhttp 拦截器

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp7.png" alt="okhttp1" style="zoom:30%;" />

10. 

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp8.png" alt="okhttp1" style="zoom:30%;" />

11. 

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp9.png" alt="okhttp1" style="zoom:30%;" />

12. getResponseWithInterceptorChain()

这个方法所做的就是构成了一个拦截器的链，然后通过依次执行每一个不同功能的拦截器，来获取我们服务器的响应返回。

13.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp10.png" alt="okhttp1" style="zoom:30%;" />

14.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp11.png" alt="okhttp1" style="zoom:30%;" />

15.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp12.png" alt="okhttp1" style="zoom:30%;" />

16.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp13.png" alt="okhttp1" style="zoom:30%;" />

17. 

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp14.png" alt="okhttp1" style="zoom:30%;" />

18.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp15.png" alt="okhttp1" style="zoom:30%;" />

okhttp会将客户端和服务端的连接抽象成一个Connection接口类，它的实现类就是RealConnection。

19. ConnectionPool

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp16.png" alt="okhttp1" style="zoom:30%;" />

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp17.png" alt="okhttp1" style="zoom:30%;" />

20.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp18.png" alt="okhttp1" style="zoom:30%;" />

21. CallServerInterception 

这里面几个重点类：

- RealInterception:okhttp拦截器中的一个链，所有的网络请求正式通过这个链，链接到一起完成相应的功能。正是proceed方法能让依次进行下一个链的操作。
- Httpcodec：okhttp中将所有的流对象都封装成了Httpcodec这个类，这是一个接口，CallServerInterceptor这个拦截器正是通过Httpcodec里的Socket流来完成操作的 。简单讲就是Httpcodec将request编码，将response解码。HttpCodec是对 HTTP 协议操作的抽象，有两个实现：Http1Codec和Http2Codec，顾名思义，它们分别对应 HTTP/1.1 和 HTTP/2 版本的实现。在这个方法的内部实现连接池的复用处理

- StreamAllocation：它是用来建立okhttp请求所需要的其他的网络设施的组件，它的作用是用来分配Stream。它相当于一个管理类，维护了服务器连接、并发流。
- RealConnection:okhttp会将客户端和服务端的连接抽象成一个Connection连接，而这个抽象连接的具体实现就是RealConnection。
- Request: 代表网络请求。

22.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp19.png" alt="okhttp1" style="zoom:30%;" />

拦截器链里的拦截器

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp20.png" alt="okhttp1" style="zoom:30%;" />

-  RetryAndFollowUpInterceptor ：主要是重试和重定向我们的请求。
-  CacheInterceptor：处理缓存。负责读取缓存直接返回、更新缓存。
-  BridgeInterceptor：负责okhttp请求和响应对象与实际的http协议中请求和响应的转换。转换过程中可以处理一些cookie相关的内容。比如请求时，对必要的Header进行一些添加，接收响应时，移除必要的Header。
- ConnectInterceptor： 负责和服务器建立连接。
-  CallServerInterceptor：负责完成最终的网络请求。负责发送请求和响应两大工作。负责向服务器发送请求数据、从服务器读取响应数据。

23.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp21.png" alt="okhttp1" style="zoom:30%;" />

24.

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/okhttp22.png" alt="okhttp1" style="zoom:30%;" />

**第三章：Retrofit**

1.  

- App应用程序通过Retrofit请求网络，实际上是使用Retrofit接口层封装请求参数，之后由OkHttp完成后续的请求操作。
- 在服务端返回数据之后，OkHttp 将原始的结果交给Retrofit,Retrofit根据用户的需求对结果进行解析。

2.  

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/retrofit1.png" alt="retrofit1" style="zoom:30%;" />

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/retrofit2.png" alt="retrofit1" style="zoom:30%;" />

Retrofit 将每一个Http请求抽象成了java 接口，然后用注解配置请求的参数。内部实现原理就是用动态代理将接口的注解翻译成一个一个的请求，再有我们的线程池来执行请求。

3. **动态代理**：代理类在程序运行时创建的代理方式。不用频繁的修改每一个代理类的函数。

4.  

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/retrofit3.png" alt="retrofit1" style="zoom:30%;" />

5.  

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/retrofit4.png" alt="retrofit1" style="zoom:30%;" />

6. 涉及到的设计模式：

-  工厂设计模式：将类实例化的操作与使用对象的操作相分开，这样客户端使用的时候不需要知道具体参数，就可以通过Factory提供给我们的静态方法，来创建我们需要的产品。
- 抽象工厂模式：CallAdapter
- 简单工厂模式：Platform 获取平台
- 外观模式：屏蔽了Retrofit内部细节，使用者只关注Retrofit，通过Retrofit设置参数就行了。
- 策略模式：工厂模式强调的是生产不同的对象。策略模式是这些不同对象的策略方法具体实现。 
- 适配器模式：将OkHttpCall适配成不同平台（Android、Java8、ios、Rxjava）的Call。
- 观察者模式：Call是被观察者，OkHttpCall实现了Call，是实际的被观察者。Callback是被观察者。  

7.  

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/retrofit5.png" alt="retrofit1" style="zoom:30%;" />

8.  

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/retrofit6.png" alt="retrofit1" style="zoom:30%;" />

9. 

<img src="/Users/yuxibing/Work_codes/MyBlog/WarriorYu.github.io/source/images/retrofit7.png" alt="retrofit1" style="zoom:30%;" />



> BAT大牛带你深度剖析Android 10大开源框架

