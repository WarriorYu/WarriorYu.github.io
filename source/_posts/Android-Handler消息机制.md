---
title: Android - Handler消息机制
categories:
  - Android源码分析
tags:
  - 源码分析
date: 2020-02-29 19:58:16
---

#  Android - Handler 消息机制 



## 一. 概念

### 1. Handler

**个人理解**：

- 消息辅助类，Handler 是主线程和子线程沟通的桥梁，可以通过以 send / post 为前缀的一些方法（例如:sendMessage、postDelayed）发送消息（Message）到消息队列（MessageQueue）；也可以处理消息循环器（Looper）分发过来的消息（Message），轻松的将一个任务切换到 Handler 所在的线程中执行。Android不允许在子线程中更新UI，在子线程执行耗时操作后，通过Handler将更新UI的操作切换到主线程执行。

**Google官方文档的介绍（我的蹩脚翻译）：**

- 通过 Handler 可以发送和处理 Message / Runnable 对象，Message 和 Runnable 跟线程的[MessageQueue](https://developer.android.google.cn/reference/android/os/MessageQueue) 相关。每个 Handler 实例都关联一个线程（Thread）和该线程的消息队列（MessageQueue）。创建Handler的同时会绑定到 [Looper](https://developer.android.google.cn/reference/android/os/Looper)  。Looper 可以在 MessageQueue 里 添加 /取出 Message / Runnable ，并且 Looper 是运行在创建Handler的线程中。
- Handler 有两个主要的用途：
  1. 使 Message / Runnable 在未来的某个时间点得到处理。
  2. 处理不同线程的队列消息  
- Handler可以通过`post(Runnable)`, `postAtTime(java.lang.Runnable, long)`, `postDelayed(Runnable, Object, long)`, `sendEmptyMessage(int)`, `sendMessage(Message)`, `sendMessageAtTime(Message, long)`, and `sendMessageDelayed(Message, long)`发送消息。以 post 版本的方法是将 Runnable 对象封装成Message添加到消息队列，当Handler接收到消息时，会回调Runnable的run()方法;sendMessage 版本的方法将携带一些信息的Message对象添加到消息队列，当接收消息时，会回调handleMessage(Message)方法，这个方法在你创建Handler的时候由自己来重写。

### 2. Message

**个人理解：**

- 存储通信数据信息，是Handler发送、处理的消息对象。用于在主线程和子线程中传递。在子线程中将各种需要更新到UI的数据信息封装到Message中，然后通过Handle在主线程中解析出Message的信息，更新到UI。

**Google官方文档的介绍（我的翻译）：**

- Message 可用于发送给 Handler ，它包含三种int类型（what、arg1、arg2）和一个对象（obj）字段，不需要你自己再添加配置字段。

  ☆  虽然 Message 的构造方法是 public 的,但是最高效的方式是通过调用 Message.obtain() 或者		    Handler#obtainMessage 方法获得 Message 对象, 这种方式是通过对象池来获取 Message 。

### 3. MessageQueue

**个人理解：**

- 消息队列, 消息队列的主要功能是向消息池发送消息(`MessageQueue.enqueueMessage()`)和取走消息池的消息(`MessageQueue.next()`)；尽管MessageQueue叫消息队列，但是它的内部实现并不是用的队列，实际上它是通过一个单链表的数据结构（即Message）来维护消息列表，单链表在插入和删除上比较有优势。   

**Google官方文档的介绍（我的翻译）：**

- 一个简单的类，持有 Looper 分发的消息列表。Message 不是直接添加到 MessageQueue 中，而是通过跟Looper相关的Handler对象。
- 可以通过 Looper.myQueue() 获取当前线程的MessageQueue。

### 4. Looper :

**个人理解：**

- 消息循环器，通过 Looper.loop() 负责在消息队列（MessageQueue）循环取出消息（Message），并且分发给对应的处理者（Handler）。是消息队列（MessageQueue）与处理者（Handler）的通信媒介。每个线程只能拥有一个 Looper  ；一个 Looper 可以绑定多个线程的 Handler；所以多个线程通过Handler 发送 Message 时，是往同1个 Looper 所持有的 MessageQueue 中发送消息，提供了线程间通信的可能。

**Google 官方文档的介绍（我的翻译）：**

- Looper这个类用于在线程中进行消息循环。默认创建的子线程没有 Looper，如果创建的话，在线程中调用 prepare() ，然后调用 loop() 开启消息循环。

- 大部分与消息循环器（Looper）的交互是通过 Handler 类。

- 下面是一个 Looper/Thread 的典型例子，通过 prepare() 和 loop() ，创建 Handler 和Looper交互。

```java
class LooperThread extends Thread {
      public Handler mHandler;

      public void run() {
          Looper.prepare();

          mHandler = new Handler() {
              public void handleMessage(Message msg) {
                  // process incoming messages here
              }
          };

          Looper.loop();
      }
  }
```

## 二.  源码解读：

### 从一个普通的例子说起：

```java
//1.在主线程以匿名内部类的方式创建Handler对象
private Handler mHandler = new Handler() {
    @Override
    public void handleMessage(@NonNull Message msg) {
        //更新UI
    }
};

new Thread(new Runnable() {
    @Override
    public void run() {
        //2.创建消息对象
        Message msg = Message.obtain();
        msg.what = 1;
        //3.在子线程中通过Handler发送消息到消息队列中
        mHandler.sendMessage(msg);
    }
}).start();
```

### 分析1：

```java
/**
 * Handler的无参构造方法
 *
 */
public Handler() {
  	//待分析：1
    this(null, false);
}

//分析：1
public Handler(@Nullable Callback callback, boolean async) {
    //1.1 匿名类、内部类、本地类都必须声明为static,
    //否则就会出现我们常看到的黄色代码块的警告，
  	//提示可能会出现内存泄漏。
  	//async默认是false，即创建Handler时，决定了以后创建的普通消息就是同步消息
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || 							klass.isLocalClass())&&
             (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +klass.getCanonicalName());
        }
    }
   
    //1.2 在ThreadLocal中获取当前线程的Looper对象，如果没有Looper则无法创建Handler
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    //1.3 消息队列，来自Looper对象
    mQueue = mLooper.mQueue;
    mCallback = callback;//回调方法
    mAsynchronous = async;//设置消息是否为异步处理方式
}	
```

###  **问题：上述1.2 的Looper对象是怎么来的呢？**

####  **答：分两种情况：**

1. **如果在主线程中创建Handler。**在 Android 应用进程启动时，会默认创建 1 个主线程 ActivityThread（Android的主线程就是 ActivityThread ，也叫UI线程），主线程的入口方法为 main() ，在 main() 方法中系统会通过 Looper.prepareMainLooper() 来创建主线程的 Looper 以及 MessageQueue ，并通过 Looper.loop() 来开启主线程的消息循环。这个过程就是上面代码所示。

2. **如果在子线程中创建Handler。**则必须为当前线程创建 Looper，所以需要手动调用Looper.prepare() 创建Looper，然后调用 loop() 开启消息循环。

```java
public final class ActivityThread extends ClientTransactionHandler {
   public static void main(String[] args) {
       //...省略部分无关代码...

       //通过Looper.prepareMainLooper()来创建主线程的Looper以及MessageQueue
       //待分析：2
       Looper.prepareMainLooper();

       //...省略部分无关代码...

       ActivityThread thread = new ActivityThread();
       thread.attach(false, startSeq);

       //...省略部分无关代码...

       //待分析：3
       //通过Looper.loop()来开启主线程的消息循环
        Looper.loop();
   }
}
```

###   分析2、分析3：

```java
/**
* Looper这个类用于在线程中进行消息循环。
*/
public final class Looper {

   //创建Looper的同时，创建了MessageQueue，并且绑定了当前线程
   private Looper(boolean quitAllowed) {
       mQueue = new MessageQueue(quitAllowed);
       mThread = Thread.currentThread();
}

  /**分析：2
    * 初始化当前线程的Looper，并且标记为应用程序的主线程Looper。
    * 应用程序的主线程Looper是Android环境自动创建的，所以
    * 不需要你自己去调用这些方法。
    */
   public static void prepareMainLooper() {
       prepare(false);
       synchronized (Looper.class) {
           if (sMainLooper != null) {
              throw new IllegalStateException("The main Looper has already been prepared.");
           }
         //主线程的Looper
        sMainLooper = myLooper();
       }
   }

  /** 默认手动创建的子线程没有Looper，如果创建的话，在子线程中调用prepare()，
    * 然后调用 loop() 开启消息循环。
    */
   private static void prepare(boolean quitAllowed) {
     if (sThreadLocal.get() != null) {
         //在这里也可以看出每个线程只能有 1 个Looper
         throw new RuntimeException("Only one Looper may be created per thread");
     }
     //在这里创建了Looper，在上面Looper的构造方法里可以看出，创建Looper的同时，
     //创建了MessageQueue，并且绑定了当前线程
     sThreadLocal.set(new Looper(quitAllowed));
  }

  /**分析：3
    * 在当前线程执行消息循环。在消息循环结束后要调用 quit() 方法
    */
   public static void loop() {
       //获取ThreadLocal存储的Looper对象
       final Looper me = myLooper();

       if (me == null) {
           //在调用loop()之前必须通过Looper.prepare()获取Looper对象
           throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
       }

       //获取Looper对象中的消息队列
       final MessageQueue queue = me.mQueue;

       //进入loop的消息循环
       for (;;) {
           //取出消息队列里的消息，这里会导致阻塞
           //待分析：4
           Message msg = queue.next();

           if (msg == null) {
               // No message indicates that the message queue is quitting.
               return;
           }

           //通过Message的target（target是Handler对象）进行消息的派发,
           //即回调Hander.dispatchMessage(msg)
           //待分析：5
           msg.target.dispatchMessage(msg);

           //回收Message，放入消息池。
           msg.recycleUnchecked();
       }
   }
}
```

### 分析4：

```java
public final class MessageQueue {  
  /**分析：4
    * 在消息队列中取出下一条消息
    */
  Message next() {

   final long ptr = mPtr;
   //当消息循环已经退出，则直接返回
   if (ptr == 0) {
       return null;
   }

   //在第一次循环迭代时为-1
   int pendingIdleHandlerCount = -1;
   int nextPollTimeoutMillis = 0;
   for (;;) {
       //...省略部分无关代码

       //这是native的方法。nativePollOnce是阻塞操作，
       //其中nextPollTimeoutMillis代表下一个消息到来前，
       //还需要等待的时长；当nextPollTimeoutMillis = -1时，表示消息队列中无消息，会一直等待下去。
       //当nextPollTimeoutMillis时长已经消耗完，或者消息队列被唤醒，都会返回。
       nativePollOnce(ptr, nextPollTimeoutMillis);

       synchronized (this) {
           // Try to retrieve the next message.  Return if found.
           final long now = SystemClock.uptimeMillis();
           android.os.Message prevMsg = null;
           android.os.Message msg = mMessages;
            //当消息的Handler为空时，则查询异步消息
           if (msg != null && msg.target == null) {
               //当查询到异步消息，则立刻退出循环
               do {
                   prevMsg = msg;
                   msg = msg.next;
               } while (msg != null && !msg.isAsynchronous());
           }
           if (msg != null) {
               if (now < msg.when) {

                   //当异步消息的触发时间大于当前时间，则设置下一次轮循的超时时长。
                   //在下次循环时nativePollOnce(ptr, nextPollTimeoutMillis)执行等待这个时长。
                   //源码的注释是这样说的：下一个消息还没有就绪。设置一个唤醒该消息的超时时间。
                   nextPollTimeoutMillis = (int) Math.min(msg.when - now, 			 	 		     																							Integer.MAX_VALUE);
               } else {
                   // 获取一条消息，并返回给Looper.loop()的for循环里
                   mBlocked = false;
                   if (prevMsg != null) {
                       prevMsg.next = msg.next;
                   } else {
                       mMessages = msg.next;
                   }
                   msg.next = null;
                   if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                   //设置消息的使用状态
                   msg.markInUse(); // void markInUse() { flags |= FLAG_IN_USE; }
                   return msg; // 返回MessageQueue中的下一条即将执行的消息
               }
           } else {
               // 没有消息则返回-1
               nextPollTimeoutMillis = -1;
        }

        //所有挂起的消息已经处理完毕，消息正在退出，返回null
           if (mQuitting) {
               dispose();
               return null;
           }

           //当消息队列为空，或者消息队列里即将处理的消息是是第一条时。
           if (pendingIdleHandlerCount < 0 && (mMessages == null || now < 																	mMessages.when)) {
               pendingIdleHandlerCount = mIdleHandlers.size();
           }

           if (pendingIdleHandlerCount <= 0) {
               //没有dle handlers需要运行。则循环并等待
               mBlocked = true;
               continue;
           }

           if (mPendingIdleHandlers == null) {
               mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 									4)];
           }
           mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
       }

       //只在第一次迭代循环时会进入下面的代码来运行idle handlers 。
       for (int i = 0; i < pendingIdleHandlerCount; i++) {
           final IdleHandler idler = mPendingIdleHandlers[i];
           mPendingIdleHandlers[i] = null; // release the reference to the handler

           boolean keep = false;
           try {
               keep = idler.queueIdle();//idle时执行的方法
           } catch (Throwable t) {
               Log.wtf(TAG, "IdleHandler threw exception", t);
           }

           if (!keep) {
               synchronized (this) {
                   mIdleHandlers.remove(idler);
               }
           }
       }

       //重置idle handler个数为0，以保证不会再次重复运行
       pendingIdleHandlerCount = 0;

       //当调用一个空闲handler时，一个新message能够被分发，
       //因此无需等待可以直接查询pending message.
       nextPollTimeoutMillis = 0;
   }
}
}
```

###  分析5 ：

```java
public class Handler {
    /**分析：5
    * 根据传入的消息方式，来决定处理消息的回调方法
    */
   public void dispatchMessage(@NonNull Message msg) {

       // 优先级1： 若msg.callback属性不为空，则代表使用了post（Runnable r）发送消息
       // 则执行handleCallback(msg)，即回调Runnable对象里复写的run（）
       if (msg.callback != null) {
           handleCallback(msg);
       } else {
           //优先级2：如果mCallback！=null，则执行
           if (mCallback != null) {
               if (mCallback.handleMessage(msg)) {
                   return;
               }
           }
           //优先级3：执行创建Handler时重写的handleMessage(msg)方法
           handleMessage(msg);
       }
   }

  private static void handleCallback(Message message) {
       message.callback.run();
   }
}
```

###    **至此，消息循环、分发的过程分析完毕，总结一下：**

   1. 通过 Looper.loop() 里的 for 循环来不断轮询消息；

   2. 通过 for 循环里的 MessageQueue.next() 方法获取下一条要处理的消息；

   3. MessageQueue 的 next() 方法去获取消息队列里的消息；  

      - 成功获取到一条消息，则通过Message的target属性（即Handler）调用dispatchMessage(msg)方法处理消息，在该方法里又有三种不同的消息处理回调方式（代码里可以看出是有3种优先级顺序的），回调方式因创建Handler和发送消息方式的不同而不同。

      - 如果消息的触发时间大于当前时间，则在next()方法里阻塞，阻塞时间根据创建消息时设定的延迟时间决定；这里涉及到native的方法，有利于节省CPU的资源。

        


### **下一步，我们继续分析消息的发送、入队过程：**

##### **在子线程中通过Handler发送消息到消息队列中 :**

-  send/post 发送消息的各种方法，最终都是调用 MessageQueue.enqueueMessage()方法


#### **send方法：**

```java
public class Handler {
  //将消息发送到消息队列
  public final boolean sendMessage(@NonNull Message msg) {
      return sendMessageDelayed(msg, 0);
  }

  public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
      if (delayMillis < 0) {
          delayMillis = 0;
      }
      return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
  }

  public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    	//获取对应的消息队列对象
      MessageQueue queue = mQueue;
      if (queue == null) {
          RuntimeException e = new RuntimeException(
                  this + " sendMessageAtTime() called with no mQueue");
          Log.w("Looper", e.getMessage(), e);
          return false;
      }
      return enqueueMessage(queue, msg, uptimeMillis);
  }

  public final boolean sendEmptyMessage(int what){
      return sendEmptyMessageDelayed(what, 0);
  }

  public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
      Message msg = Message.obtain();
      msg.what = what;
      return sendMessageDelayed(msg, delayMillis);
  }

  public final boolean sendMessageAtFrontOfQueue(@NonNull Message msg) {
      //获取对应的消息队列对象
      MessageQueue queue = mQueue;
      if (queue == null) {
          RuntimeException e = new RuntimeException(
              this + " sendMessageAtTime() called with no mQueue");
          Log.w("Looper", e.getMessage(), e);
          return false;
      }
      return enqueueMessage(queue, msg, 0);
  }

  private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
          long uptimeMillis) {
      //将当前的Handler对象赋值给msg.target；
    	//这里可以和Looper.loop()的for循环里的msg.target.dispatchMessage(msg)联系起来理解；
    	//target就是在这里被赋值的。
      msg.target = this;
      msg.workSourceUid = ThreadLocalWorkSource.getUid();
      if (mAsynchronous) {
          msg.setAsynchronous(true);
      }
    	//调用消息队列的enqueueMessage()，将消息加入消息队列
      return queue.enqueueMessage(msg, uptimeMillis);
  }
}
```

#### **post方法：**

```java
public class Handler {
    public final boolean post(@NonNull Runnable r) {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }

    public final boolean postAtFrontOfQueue(@NonNull Runnable r) {
       return sendMessageAtFrontOfQueue(getPostMessage(r));
    }

    public final boolean postAtTime(@NonNull Runnable r, long uptimeMillis) {
       return sendMessageAtTime(getPostMessage(r), uptimeMillis);
    }

    public final boolean postDelayed(@NonNull Runnable r, long delayMillis) {
       return sendMessageDelayed(getPostMessage(r), delayMillis);
    }

    //可以看出，post方式是通过将Runnable赋值给Message的callback属性，封装成Message，
    //归根到底和send方式一样，都是发送的Message消息。
    private static Message getPostMessage(Runnable r, Object token) {
       Message m = Message.obtain();
       m.obj = token;
       m.callback = r;
       return m;
    }
}
```

###  开始分析 MessageQueue.enqueueMessage() 方法

```java
public final class MessageQueue {
	boolean enqueueMessage(android.os.Message msg, long when) {
    //Message必须有一个target（即Handler）
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            //如果是正在退出时，回收msg，加入到消息池。recycle()方法我们在下面会分析。
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        android.os.Message p = mMessages;
        boolean needWake;
        //p == null 即消息队列里没有消息，或者msg的触发时间是队列中最早的，则将当前消息作为队头插入；
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            //如果此时消息队列的循环处于阻塞状态，则唤醒
            needWake = mBlocked;
        } else {
            //将消息按时间顺序插入到MessageQueue。一般地，不需要唤醒事件队列，						
          	//除非消息队头存在barrier，并且同时Message是队列中最早执行的异步消息。
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            android.os.Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    //1.第一次循环，msg的触发时间比队头的消息还要早，则跳出for循环，直接加到队头。
                  	//2.往后再次循环，如果遍历到队尾（即 p 节点==null），
                  	//或者当前遍历到的 p 节点的触发时间大于msg的触发时间，则跳出for循环
                  	//将此msg添加到p节点之前(即msg.next = p)。
                    break;
                }
              	//如果新加入的消息是异步消息，并且通过上面的if判断可知:如果新消息按照时间顺序排列的话，不
              	//是队头的元素(即比队头的元素触发时间还要晚），则不需要唤醒(即needWake = false)。
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
          	//链表的操作，将msg添加到链表相应的位置。
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        // 消息没有退出，我们认为此时mPtr != 0
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}  
```

### 至此，消息入队的过程分析完毕，总结一下：

- Message入队是按照Message触发时间的先后顺序排列的，队头的消息是将要最早触发的消息。当有消息需要加入消息队列时，会从队头开始遍历，直到找到消息应该插入的合适位置，以保证所有消息的时间顺序。


### 下面，我们继续分析创建消息（Message）对象的最佳方式：

```java
//创建消息对象
Message msg = Message.obtain();

/**
 * 在消息池里返回一个Message对象。避免了创建新对象。
 */
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;//取出一个消息对象
            sPool = m.next;
            m.next = null;//将取出的消息在链表里断开连接
            m.flags = 0; // 清除 in-use flag
            sPoolSize--; //消息池的可用大小减一
            return m;
        }
    }
  	//当消息池为空时，直接新建消息对象
    return new Message();
}
```

### 以上获取Message实例的过程总结为：

- 如果消息池不是空的，从消息池（链表结构）的头部取走一个Message，并将链表的头部更新为取走消息的next（即下一条Message）。否则新建Message对象。

### 下面分析recycle（即Message的回收）：

- 其中在Looper.loop()方法取出一条消息交给Handler处理后，有一个回收Message到消息池的操作，调用的是recycleUnchecked()：


```java
public final class Message implements Parcelable {
  public void recycle() {
     //判断消息是否正在使用
      if (isInUse()) {
          if (gCheckRecycle) {
              throw new IllegalStateException("This message cannot be recycled because it "
                      + "is still in use.");
          }
          return;
      }
      recycleUnchecked();
  }

  /**
   * 该方法在处理排队的消息时，在MessageQueue 和 Looper内部调用
   */
  void recycleUnchecked() {
     	//将消息标示为IN_USE，并清空消息所有的参数。
      flags = FLAG_IN_USE;
      what = 0;
      arg1 = 0;
      arg2 = 0;
      obj = null;
      replyTo = null;
      sendingUid = UID_NONE;
      workSourceUid = UID_NONE;
      when = 0;
      target = null;
      callback = null;
      data = null;
		
      synchronized (sPoolSync) {
        	//消息池的大小是MAX_POOL_SIZE = 50，当消息池没有满时，将Message对象加入消息池
          if (sPoolSize < MAX_POOL_SIZE) {
              next = sPool;
              sPool = this;//将消息添加到消息池的头部
            	//消息池的可用大小进行加1操作
              sPoolSize++;
          }
      }
  }
}
```

以上总结为：为了提高效率，减少对象频繁的创建和回收过程，提供了一个消息缓存队列。消息池的默认大小是50，当需要回收消息时（比如：分发消息给Handler处理后），将消息的属性参数清空，然后将回收的消息添加到消息池的头部。

#### 至此，Android的消息机制分析完毕，Android的消息机制源码远没有这么简单，以上仅从Java层进行了分析，对于Native层的分析没有涉及到。希望大家一起加油，踏踏实实分析源码的脉络，遇到同样的问题，做到胸有成竹。也欢迎在留言区一起讨论，一起成长！



### 如果想及时收到我的文章更新，可关注以下公众号。 



![我的公众号二维码](/images/我的公众号二维码.png)



> 参考资料：
>
> 1.https://developer.android.google.cn/reference/android/os/Handler?hl=en
>
> 2.http://gityuan.com/2015/12/26/handler-message-framework/
>
> 3 .https://www.jianshu.com/p/b4d745c7ff7a
>
> 4.Android开发艺术探索第十章