---
title: Android消息机制
date: 2017-1-7
tags: Android
---

-----

# 概论
Android的消息机制由Handler，MessageQueue与Looper支撑，Handler是消息机制的上层接口，在开发过程中主要与Handler打交道。
MessageQueue为消息队列，它在内部存储了一组消息，以队列的形式对外提供插入与删除的工作（内部存储采用单链表的数据结构）。
Looper的中文翻译为循环，它以无限循环的方式去查找MessageQueue中是否有消息，如果有就处理消息，否则一直等待。
Handler在创建的时候会采用当前线程的Looper构建消息循环系统，而Looper使用ThreadLocal（一个线程内部的数据存储类，可以在不同线程中互不干扰地存储并提供数据）使得每个线程都有一个Looper。
不过线程默认没有Looper（ui线程除外），如果需要使用Handler就必须为线程创建Looper。UI线程就是ActivityThread，它会在创建的时候初始化Looper。
<!-- more --> 
## UI不能在子线程中访问的原因
Handler并不是专门用于更新UI的，但很多时候它都被开发者用来更新UI，那为什么必须在主线程中才能操作UI呢？主要是因为Android的UI控件不是线程安全的，如果在多线程并发访问会导致UI控件出现不可预料到结果。那为什么不能加锁呢？首先是加锁会让UI访问得逻辑变得复杂，然后也会降低UI访问的效率，因为锁机制会阻塞一些线程的执行。所以最简单高效的方法就是采用单线程模型来处理UI操作。
# Message与MessageQueue工作原理
对于Android消息机制我们从最熟悉的Message开始，Message用于存储传递的数据
```java
/**
* arg1 and arg2 are lower-cost alternatives to using﻿
* {@link #setData(Bundle) setData()} if *you only need to store a﻿ few integer *values.﻿
*arg1和arg2是低成本数据的选择方案
*如果只要存储一些integer只用选择它*们，不必使用setData();
    */﻿
   public int arg1;﻿﻿
   /**﻿
    * arg1 and arg2 are lower-cost alternatives to using﻿
    * {@link #setData(Bundle) setData()} if you only need to store a﻿
    * few integer values.﻿
    */﻿
   public int arg2;
   
   /**
   User-defined message code so that the recipient can identify what this message is about. Each Handler has its own name-space for message codes, so you do not need to worry about yours conflicting with other handlers.
   
用户自定义的消息代码，这样接受者可以确定这个消息的信息。每个handler各自包含自己的消息代码，所以不用担心自定义的消息跟其他handlers有冲突。
   */
   public int what
   
   //...
   public Object obj
```
创建Message不必使用它的构造方法，使用obtain();可以从Message池中取出一个Message,避免多次创建新的Message对象。
```java
* Return a new Message instance from the global pool. Allows us to﻿
    * avoid allocating new objects in many cases.﻿
    */﻿
   public static Message obtain() {﻿
       synchronized (sPoolSync) {﻿
           if (sPool != null) {﻿
               Message m = sPool;﻿
               sPool = m.next;﻿
               m.next = null;﻿
               m.flags = 0; // clear in-use flag﻿
               sPoolSize--;﻿
               return m;﻿
           }﻿
       }﻿
       return new Message();﻿
   }﻿﻿
   /**﻿
    * Same as {@link #obtain()}, but copies the values of an existing﻿
    * message (including its target) into the new one.﻿
    * @param orig Original message to copy.﻿
    * @return A Message object from the global pool.﻿
    */﻿
   public static Message obtain(Message orig) {﻿
       Message m = obtain();﻿
       m.what = orig.what;﻿
       m.arg1 = orig.arg1;﻿
       m.arg2 = orig.arg2;﻿
       m.obj = orig.obj;﻿
       m.replyTo = orig.replyTo;﻿
       m.sendingUid = orig.sendingUid;﻿
       if (orig.data != null) {﻿
           m.data = new Bundle(orig.data);﻿
       }﻿
       m.target = orig.target;﻿
       m.callback = orig.callback;﻿﻿
       return m;﻿
   }﻿﻿
   /**﻿
    * Same as {@link #obtain()}, but sets the value for the target member on the Message returned.﻿
    * @param h  Handler to assign to the returned Message object's target member.﻿
    * @return A Message object from the global pool.﻿
    */﻿
   public static Message obtain(Handler h) {﻿
       Message m = obtain();﻿
       m.target = h;﻿﻿
       return m;﻿
   }
```
接下来看下常用的sendMessage()
```java
public final boolean sendMessage(Message msg)﻿{﻿
return sendMessageDelayed(msg, 0);﻿
   }
```
```java
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {﻿
       Message msg = Message.obtain();﻿
       msg.what = what;﻿
       return sendMessageDelayed(msg, delayMillis);﻿
   }
```
都调用到sendMessageDelayed
```java
public final boolean sendMessageDelayed(Message msg, long delayMillis)﻿
  {﻿
      if (delayMillis < 0) {﻿
          delayMillis = 0;﻿
      }﻿
      return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);﻿
  }
```
```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {﻿
       MessageQueue queue = mQueue;﻿
       if (queue == null) {﻿
           RuntimeException e = new RuntimeException(﻿
                   this + " sendMessageAtTime() called with no mQueue");﻿
           Log.w("Looper", e.getMessage(), e);﻿
           return false;﻿
       }﻿
       return enqueueMessage(queue, msg, uptimeMillis);﻿}
```
```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {﻿
       msg.target = this;﻿
       if (mAsynchronous) {﻿
           msg.setAsynchronous(true);﻿
       }﻿
       return queue.enqueueMessage(msg, uptimeMillis);﻿
   }
```
这里就给target赋值为this,就是上面说的target就是Handler了，所以最终会调用queue的enqueueMessage的方法，这就是MessageQueue的插入了。
```java
boolean enqueueMessage(Message msg, long when) {
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
                msg.recycle();
                return false;
            }
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }
            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
enqueueMessage的实现中可以看出，它的主要操作就是单链表的插入操作。

在MessageQueue中还有一个重要方法：读取 next方法。
```java
Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }
                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }
                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }
            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler
                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }
            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;
            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```
可以看出next方法就是一个无限循环的方法，消息队列没有消息，next方法就一直阻塞在那里，当有消息的时候，next方法就会返回这条消息并将其从单链表中移除。

# Looper工作原理
Looper在Android消息机制中扮演着消息循环的角色，具体来说就是以无限循环的方式去查找MessageQueue中是否有消息，如果有就处理消息，否则一直阻塞在那里。
我们知道，Handler工作需要Looper，没有Looper就会报错，那么，怎么创建Looper呢？其实就是通过Looper.prepare()就可以为当前线程创建一个Looper，接着通过Looper.loop()就可以开启消息循环了。
首先我们看一下它的构造方法
```java
   private Looper(boolean quitAllowed){
      mQueue = new MessageQueue(quitAllowed);
      mThread = Thread.currentThread();
   }
```
在构造方法中它会创建一个MessageQueue即消息队列，然后将当前线程的对象保存起来。

```java
/** Initialize the current thread as a looper.﻿
   * This gives you a chance to create handlers that then reference﻿
   * this looper, before actually starting the loop. Be sure to call﻿
   * {@link #loop()} after calling this method, and end it by calling﻿
   * {@link #quit()}.﻿
   */﻿
 public static void prepare() {﻿
     prepare(true);﻿
 }﻿﻿
 private static void prepare(boolean quitAllowed) {﻿
     if (sThreadLocal.get() != null) {﻿
         throw new RuntimeException("Only one Looper may be created per thread");﻿
     }﻿
     sThreadLocal.set(new Looper(quitAllowed));﻿
 }
```
hreadLocal是一个ThreadLocal对象，可以在一个线程中存储变量。可以看到，将一个Looper的实例放入了ThreadLocal，并且11行判断了sThreadLocal是否为null，否则抛出异常。这也就说明了Looper.prepare()方法不能被调用两次，同时也保证了一个线程中只有一个Looper实例,接下来是loop()方法
```java
/**﻿
  * Run the message queue in this thread. Be sure to call﻿
  * {@link #quit()} to end the loop.﻿
  */﻿
 public static void loop() {﻿
     final Looper me = myLooper();﻿
     if (me == null) {﻿
         throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");﻿
     }﻿
     final MessageQueue queue = me.mQueue;﻿﻿
     // Make sure the identity of this thread is that of the local process,﻿
     // and keep track of what that identity token actually is.﻿
     Binder.clearCallingIdentity();﻿
     final long ident = Binder.clearCallingIdentity();﻿﻿
     for (;;) {﻿
         Message msg = queue.next(); // might block﻿
         if (msg == null) {﻿
             // No message indicates that the message queue is quitting.﻿
             return;﻿
         }﻿﻿
         // This must be in a local variable, in case a UI event sets the logger﻿
         Printer logging = me.mLogging;﻿
         if (logging != null) {﻿
             logging.println(">>>>> Dispatching to " + msg.target + " " +﻿
                     msg.callback + ": " + msg.what);﻿
         }﻿﻿
         msg.target.dispatchMessage(msg);﻿﻿
         if (logging != null) {﻿
             logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);﻿
         }﻿﻿
         // Make sure that during the course of dispatching the﻿
         // identity of the thread wasn't corrupted.﻿
         final long newIdent = Binder.clearCallingIdentity();﻿
         if (ident != newIdent) {﻿
             Log.wtf(TAG, "Thread identity changed from 0x"﻿
                     + Long.toHexString(ident) + " to 0x"﻿
                     + Long.toHexString(newIdent) + " while dispatching to "﻿
                     + msg.target.getClass().getName() + " "﻿
                     + msg.callback + " what=" + msg.what);﻿
         }﻿﻿
         msg.recycleUnchecked();﻿
     }﻿
 }﻿﻿
 /**﻿
  * Return the Looper object associated with the current thread.  Returns﻿
  * null if the calling thread is not associated with a Looper.﻿
  */﻿
 public static Looper myLooper() {﻿
     return sThreadLocal.get();﻿
 }
```
方法中直接获取了sThreadLocal存储的Looper实例，如果me为null则抛出异常，这就是说looper方法必须在prepare方法之后才运行。﻿然后拿到该looper实例中的mQueue，就进入了无限循环。﻿然后一直取出一条消息，直到没有消息则阻塞。子线程Looper用完建议终止（调用quit方法），否则loop方法会无限循环下去，子线程会一直处于等待的状态。

获取消息后使用msg.target.dispatchMessage(msg);把消息交给msg的target的dispatchMessage方法去处理。Msg的target是什么呢？其实就是handler对象。但是这里不同的是，Handle的dispatchMessage方法是在创建Handle时所使用的Looper中执行的，这样就成功的完成了线程切换。
# Handler原理
Handler的工作主要包括消息的发送和接受过程。消息发送可以通过post或者send的一系列方法来实现，post的一系列方法最终是通过send的一系列方法来实现的。处理消息最终会调用Handler的dispatchMessage方法。
```java
/**﻿
 * Handle system messages here.﻿
 */﻿
public void dispatchMessage(Message msg) {﻿
    if (msg.callback != null) {﻿
        handleCallback(msg);﻿
    } else {﻿
        if (mCallback != null) {﻿
            if (mCallback.handleMessage(msg)) {﻿
                return;﻿
            }﻿
        }﻿
        handleMessage(msg);﻿
    }﻿
}
```
```java
/**﻿
 * Subclasses must implement this to receive messages.﻿
 */﻿
public void handleMessage(Message msg) {﻿
}
```
这个就是个空方法，所以需要我们对它进行复写。于是乎，处理Message就交到我们手中了。