---
title: 摸清Java并发安全的套路
date: 2016-06-13 10:02:53
tags: [Java,JVM,并发编程]
categories: 当我们在谈论多线程的时候，我们在谈论什么
---
![并发安全](multi.jpg "并发安全")
# 概述
之前的文章《Java线程同步策略》对Java的并发策略有了一个比较详尽的介绍。其中讲到``ReentrantLock``时提及到了``AbstractQueuedSynchronizer``（下文简称AQS）这个类。
从字面含义去揣度，这就是一个**抽象的队列同步器**。所谓抽象，想必有许多方法（或者说模板）还需要开发者具体实现；所谓队列，意味着其中维护一套遵循FIFO原则的存储结构；所谓同步，说明其中蕴含某种系统级别的同步机制，为线程安全设计而生的。
《Java并发编程艺术》一书是这么描述的：
> 队列同步器``AbstractQueuedSynchronizer``，是用来构建锁或者其他同步组件的基础框架，它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作。并发包作者（Doug Lea）期望它能够成为实现大部分同步需求的基础。

以下的介绍也是从这三个维度（抽象、队列、同步）具体展开来说。

# 抽象
上面说的，AQS规定了一套模板，也规定了一套可重写的方法，这就是我们所说的抽象。
> To use this class as the basis of a synchronizer, redefine the following methods, as applicable, by inspecting and/or modifying the synchronization state using getState(), setState(int) and/or compareAndSetState(int, int)(用这个类作为同步器的核心基础类，需要重新定义如下方法来使用，通过调用``getState()``、``setState(int)``和``compareAndSetState(int,int)``这几个方法，获取并改变同步状态)

这一解释已经很能够说明白这个类的使用方式了，我们来阅读一下API文档，了解一下这一套可重写的方法。后面会举例子具体说明如何使用。

方法  | 解释
------------- | -------------
tryAcquire() | 独占式获取同步状态，需先询问是否允许被获取，允许则利用CAS改状态
tryRelease() | 独占式释放同步状态。
tryAcquireShared() | 共享式获取同步状态。
tryReleaseShared() | 共享式释放同步状态。
isHeldExclusively() | 查询当前同步器是否在独占模式下被占用。

PS: 这套方法如果不实现，则抛出``UnsupportedOperationException``异常

# 队列
AQS另外也提供了一套模板方法，这里就牵扯到我们所说的队列。
这里列举一些重要的模板方法。

方法  | 解释
------------- | -------------
void acquire(int arg) | 独占式获取同步，如果获取成功，则方法返回，否则将会进入同步队列等待，该方法会调用重写的tryAcquire(int arg)
void acquireShared(int arg) | 共享式获取同步状态，会调用重写的tryAcquireShared(int arg)。
void acquireInterruptibly(int arg) | 独占式获取同步，可响应中断，即中断后抛出``InterruptedException``。
void acquireSharedInterruptibly(int arg)| 共享可中断式获取同步状态
boolean tryAcquiredNanos(int arg,long nanos) | 独占式获取同步，并设置了超时
boolean tryAcquiredSharedNanos(int arg,long nanos) | 共享式获取同步，并设置了超时
boolean release(int arg) | 独占式释放同步
boolean releaseShared(int arg) | 共享式释放同步
Conllection< Thread > getQueuedThreads() | 获取等待在同步队列上的线程集合

从API文档里不难看出，获取和释放同步支持两种方式，共享式和独占式。而无论独占还是共享，当该线程无法获取资源的时候，则会将该线程作为FIFO中的一个节点（node），插入到队列的尾部，另一边则不停轮询请求，判断该线程的先驱节点是否为头节点（head）以及是否有空闲资源能被获取，从而进一步获取所谓的锁。
以独占式获取锁资源为例，代码很清晰的表达了这一切：
```java
    /**
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     */
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }


    /**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }

    /**
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
# 同步
那么AQS如何保证线程安全的呢？

其实上边也说过，AQS是通过调用``getState()``、``setState(int)``和``compareAndSetState(int,int)``这几个方法，获取并改变同步状态的。以独占式为例，我们可以将state设置为1，表示资源有且仅有一个，当有一个线程占用锁资源后，state则设置为0，此时，其余线程将因为无法获取该资源而进入队列中等待。

那么在多线程环境下，获取并改变state的值是必须要有线程安全的手段的。在CAS指令流行之前，我们很容易就想到了用``synchronized``关键字给这一操作上锁。然而``synchronized``是重量级操作，在频繁的线程上下文切换的场景下，``synchronized``并不可取。而CAS在此时的作用显得尤为耀眼。

关于CAS，在《Java线程同步策略》一文中已经阐述，这里不做多余介绍。我们来看看``compareAndSetState(int,int)``的源码。

```java

    import sun.misc.Unsafe;



    /**
     * Setup to support compareAndSet. We need to natively implement
     * this here: For the sake of permitting future enhancements, we
     * cannot explicitly subclass AtomicInteger, which would be
     * efficient and useful otherwise. So, as the lesser of evils, we
     * natively implement using hotspot intrinsics API. And while we
     * are at it, we do the same for other CASable fields (which could
     * otherwise be done with atomic field updaters).
     */
    private static final Unsafe unsafe = Unsafe.getUnsafe();

    /**
     * Atomically sets synchronization state to the given updated
     * value if the current state value equals the expected value.
     * This operation has memory semantics of a <tt>volatile</tt> read
     * and write.
     *
     * @param expect the expected value
     * @param update the new value
     * @return true if successful. False return indicates that the actual
     *         value was not equal to the expected value.
     */
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```


# 一个例子
我们从三个维度（抽象、队列、同步）来阐述AQS的实现原理以及基于模板自身的扩展，下面就以一个独占式的``Mutex``为例来进一步加深对AQS的理解。
```java
package org.leon.concurent.lock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

/**
 * 独占锁Mutex是一个自定义的同步组件,在同一时刻只允许一个线程占有锁<br/>
 *
 * Created by LeonWong on 16/4/26.
 */
public class Mutex implements Lock {

    // 静态内部类,自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {
        // 是否处于占用状态
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        /**
         * 当状态为0的时候获取锁<br/>
         * 获取锁的过程是首先将同步状态设置为已同步,然后设置持有锁的线程为自己本身
         */
        public boolean tryAcquire(int acquires) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        // 释放锁,将状态设置为0
        protected boolean tryRelease(int releases) {
            if (getState() == 0)
                throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        // 返回一个Condition,每个condition都包含了一个condition队列
        Condition newCondition() {
            return new ConditionObject();
        }
    }

    private final Sync sync = new Sync();

    /**
     * 阻塞方法 只有拿到了同步锁方法才会返回
     */
    @Override
    public void lock() {
        System.out.println("正尝试获取锁,锁是否被独占?" + sync.isHeldExclusively());
        sync.acquire(1);
        System.out.println("获取锁成功");
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }

    public boolean isLocked() {
        return sync.isHeldExclusively();
    }

    public boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }

}

```

测试用例
```java
package org.leon.concurent.lock;

import org.leon.concurent.SleepUtils;

/**
 * Created by LeonWong on 16/4/26.
 */
public class MutexTest {

    public static final Mutex lock = new Mutex();

    public static void main(String args[]) {
        for (int i = 0; i < 10; i++) {
            new Thread(new Runner(), "Runner" + i).start();
        }
    }

    static class Runner implements Runnable {
        @Override
        public void run() {
            lock.lock();
            System.out.println("简易测试锁程序 开始");
            SleepUtils.sleepForSecond(5);
            System.out.println("简易测试锁程序 结束");
            lock.unlock();
        }
    }
}
```
SleepUtil工具类
```java
package org.leon.concurent;

import java.util.concurrent.TimeUnit;

/**
 * Created by LeonWong on 16/4/21.
 */
public class SleepUtils {
    public static void sleepForSecond(long seconds) {
        try {
            TimeUnit.SECONDS.sleep(seconds);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void sleepForMillsSecond(long millis) {
        try {
            TimeUnit.MILLISECONDS.sleep(millis);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

# 总结
在大多数场景下，谈论Java并发编程的时候，AQS的实现思路可以称之为经典，甚至Java许多并发组件都是基于AQS进行开发的，例如ReentrantLock、CountDownLatch、FutureTask、Semaphore等等。可以说理解了AQS，就是理解了Java并发的核心套路。
