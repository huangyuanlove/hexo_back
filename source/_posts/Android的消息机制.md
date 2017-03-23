---
title: Android的消息机制
date: 2017-03-23 14:00:09
tags: [Android]
---
Android消息机制主要是指Handler的运行机制，Handler的运行需要底层的MessageQueue和Looper的支撑。MessageQueue的中文翻译是消息队列，他的内部存储了一组消息，以队列的形式对外提供插入和删除的工作。虽然叫消息队列，但是它的内部存储结构并不是真正的队列，而是采用单链表的数据机构来存储消息队列，但是它的内部存储结构并不是真正的队列，而是采用单链表的数据结构来存储消息列表。Looper可以理解为消息循环。由于MessageQueue只是一个消息的存储单元，它不能去处理消息，而Looper就填补了这个功能，Looper会以无限昏眩的形式去查找是否有新消息，如果有的话就处理消息，否则就等待。Looper中海油一个特殊的概念，那就是ThreadLocal，ThreadLocal并不是现成，它的作用是可以在每个现成中存储数据。Handler创建的时候会采用当前线程的Looper来构造消息循环系统，ThreadLocal可以在不同的线程中互不干扰的存储并提供数据，Handler可以通过ThreadLocal轻松获取每个现成的Looper。需要注意的是，现成默认是没有Looper的，如果需要使用Handler就必须为现成创建Looper，我们经常提到的主线程，也叫UI线程，它就是ActivityThread，ActivityThread被创建时就会初始化Looper，这也是在主线程中默认可以使用Handler的原因。
<!--more-->

### ThreadLocal的工作原理
ThreadLocal是一个县城内部的数据存储类，通过它可以在制定的线程中存储数据，数据存储以后，只能在制定的线程中才可以获取到存储的数据，对于其他线程来说则无法获取到，我们在使用的时候：
``` java
 final ThreadLocal<Boolean> mBooleanThreadLocal = new ThreadLocal<>();
        new Thread("Thread#1"){
            @Override
            public void run() {
                mBooleanThreadLocal.set(false);
                Log.d("huangyuan","Thread#1" + mBooleanThreadLocal.get());
            }
        }.start();

        new Thread("Thread#2"){

            @Override
            public void run() {
                mBooleanThreadLocal.set(true);
                Log.d("huangyuan","Thread#2" + mBooleanThreadLocal.get());
            }
        }.start();
```
这样，我们虽然在不同的线程中访问的是同一个ThreadLocal对象，但是他们的值确是不一样的。这是因为不同的线程访问同一个ThreadLocal的get方法，ThreadLocal内部会从各自的线程中取出一个数组，然后再从数组中根据当前ThreadLocal的索引去查找出对应的value值。
ThreadLocal是一个泛型类，定义为`public class ThreadLocal<T>`,首先看ThreadLocal的set方法，如下：
``` java
 public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```
上面的方法中，首先会通过getMap方法获取ThreadLocalMap（存储线程的ThreadLocal数据），ThreadLocalMap是ThreadLocal类的静态内部类，其内部包含了一个静态内部类Entry，声明如下`static class Entry extends WeakReference<ThreadLocal>`,ThreadLocalMap类中有一个Entry类型的数组`private Entry[] table;`,下面是set方法的具体实现：
``` java
  private void set(ThreadLocal key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```
ThreadLocal的get方法如下：
``` java
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    } 
```
从ThreadLocal的set和get方法可以看出，他们所操作的对象都是当前线程的ThreadLocalMap对象的table数组，因此在不同线程中访问同一个ThreadLocal的set和get方法，他们对ThreadLocal所做的读写操作仅限于各自线程的内部，这就是为什么ThreadLocal可以在多个线程中互不干扰的存储和修改数据。
### 消息队列的工作原理
消息队列在Android中指的是MessageQueue，MessageQueue主要包含两个操作，插入和读取。读取操作本身会伴随着删除操作，插入和读取对应的方法分别为enqueueMessage和next，其中enqueueMessage的作用是往消息队列中插入一跳消息，而next的作用是从消息队列中取出一条消息并将其从消息队列中移除，尽管MessageQueue叫消息队列，但是它的内部实现并不是用的队列，实际上它是通过一个单链表的数据结构来维护消息列表，单链表在插入和删除上比较有优势。下面是enqueueMessage的实现：
``` java
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
从其实现来安，主要操作其实就是单链表的插入操作。
next的实现如下：
``` java
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
可以返现next方法是一个无线循环的方法，如果消息队列为空，那么next方法会一直阻塞在这里，当有消息到来时，next方法会返回这条消息并将其从单链表中移除
### Looper的工作原理
Looper在Android的消息机制中扮演者消息循环的角色，具体来说就是它会不停地从MessageQueue中查看是否有新消息，如果有新消息就会立刻处理，否则就一直阻塞在那里。首先看一下它的构造方法，在构造中它会创建一个MessageQueue也就是消息队列，然后将当前线程的对象保存起来，如下：
``` java
  private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```
Handler的工作需要Looper，没有Looper的线程就会报错，我们可以通过Looper.prepare()即可为当前线程创建一个Looper，接着通过Looper.loop()来开启消息循环。Looper除了prepare方法外，还提供了prepareMainLoop方法，这个方法主要是给主线程也就是ActivityThread创建Looper使用的，其本质也是通过prepare方法来实现的。由于主线程的Looper比较特殊，所有Looper提供了一个getMainLooper方法，通过它可以在任何地方获取到主线程的Looper。Looper也是可以退出的，Looper提供了quit个quitSafely来退出一个Looper，二者的区别是：quit会直接突出Looper，而quitSafely只是设定一个退出标记，然后把消息队列中的已有消息处理完毕后才安全的退出。Looper退出后，通过Handler发送的消息会失败，这个时候Handler的send方法会返回false。在子线程中，如果手动为其创建了Looper，那么在所有的事情完成以后应该调用quit方法来总之消息循环，否则这个子线程就会一直处于等待的状态，而如果退出Looper以后，这个县城就会立刻终止，因此甲乙不需要的时候终止Looper。
 Looper最重要的一个方法是loop方法，只有调用了loop后，消息循环系统才会真正的起作用，实现如下：
 ``` java
  /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            if (traceTag != 0) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
 ```
 loop方法是一个死循环，唯一跳出循环的方式是MessageQueue的next方法返回了null，当Looper的quit方法被调用时，Looper就会通知消息队列退出，当消息队列被标记为退出状态时，它的next方法就会返回null，也就是说，Looper必须退出，否则loop方法就会无限循环下去，loop方法会调用MessageQueue的next方法来获取新消息，而next方法是一个阻塞操作，当没有消息时，next方法会一直阻塞在那里，这也导致loop方法一直阻塞在那里，如果MessageQueue的next方法返回了新消息，Looper就会处理这条消息，  msg.target.dispatchMessage(msg)，这里的  msg.target是发送这条消息的handler对象，这样Handler发送的消息最终又交个它的  dispatchMessage(msg);方法来处理了，但是这里不同的是，Handler的dispatchMessahe方法是在创建Handler时所使用的Looper中执行的，这样就成功的将代码逻辑切换到指定的线程中去执行了。