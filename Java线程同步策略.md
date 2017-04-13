---
title: Java线程同步策略
date: 2016-06-10 20:48:25
tags: [Java,JVM,并发编程]
categories: 当我们在谈论多线程的时候，我们在谈论什么
---
![多线程并发安全](multithread.png "multithread")
# 线程安全
## 概述
引用《Java Concurrency In Practice》对线程安全的定义
>当多个线程访问一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，或者在调用方进行任何其他的协调操作，调用这个对象的行为都可以获得正确的结果，那这个对象是线程安全的。

这意味着如若要实现线程安全，代码本身必须要封装所有必要的正确性保障手段（比如锁的实现），以确保程序无论在多线程环境下如何调用该方法，将始终保持返回正确的结果。
## Java的线程安全
我们在讨论Java的线程安全，实际上讨论的是“相对线程安全”。需要保证的是单独对象操作是线程安全的，调用过程中不需要额外的保障措施，但是涉及到某些业务场景需要特定顺序连续调用，就可能需要调用者考虑使用额外的同步手段保证同步。引用《深入理解Java虚拟机》一书中的例子很能说明问题：
```java
private static Vector<Integer> vector = new Vector<Integer>();

public static void main(String[] args){
    while(true){
        for(int i = 0; i<10; i++){
            vector.add(i);
        }
    }

    //一个线程删数据
    Thread removeThread = new Thread(new Runnable(){
        @Override
        public void run(){
            for(int i=0 ; i<vector.size() ;i++){
                vector.remove(i);   
            }
        }
    });

    //一个线程读数据
    Thread printThread = new Thread(new Runnable() {
        @Override
        public void run(){
            for(int i=0 ; i<vector.size() ;i++){
                System.out.println(vector.get(i));
            }
        }
    });

    removeThread.start();
    printThread.start();

    //防止过多线程，内存溢出
    while(Thread.activeCount() > 20);
}
```
运行报错：
``java.lang.ArrayIndexOutOfBoundsException``
虽然说Vector已经是Java中所谓的“线程安全”了，但是在这种特殊的情况下，无法保证正确的输出结果。这里就需要做一些额外的同步手段，如下：
```java
    //一个线程删数据
    Thread removeThread = new Thread(new Runnable(){
        @Override
        public void run(){
            synchronized (vector){
                for(int i=0 ; i<vector.size() ;i++){
                    vector.remove(i);   
                }
            }
        }
    });

    //一个线程读数据
    Thread printThread = new Thread(new Runnable() {
        @Override
        public void run(){
            synchronized (vector){
                for(int i=0 ; i<vector.size() ;i++){
                    System.out.println(vector.get(i));  
                }
            }
        }
    });
```
# Java中的互斥同步
## Synchronized
说到线程安全，离不开讨论锁的实现方式。说到锁，Java开发者们第一想到的肯定是Synchronized关键字，我们就先从这个关键字切入，来阐述Java中锁的实现。
#### 同步原理
>JVM规范规定JVM基于进入和退出Monitor对象来实现方法同步和代码块同步，但两者的实现细节不一样。代码块同步是使用monitorenter和monitorexit指令实现，而方法同步是使用另外一种方式实现的，细节在JVM规范里并没有详细说明，但是方法的同步同样可以使用这两个指令来实现。monitorenter指令是在编译后插入到同步代码块的开始位置，而monitorexit是插入到方法结束处和异常处， JVM要保证每个monitorenter必须有对应的monitorexit与之配对。任何对象都有一个 monitor 与之关联，当且一个monitor 被持有后，它将处于锁定状态。线程执行到 monitorenter 指令时，将会尝试获取对象所对应的 monitor 的所有权，即尝试获得对象的锁。

Sychronized采取的同步策略是**互斥同步**，怎么理解互斥同步呢？通常情况下，临界区（Critical Section）、互斥量（Mutex）和信号量（Semaphore）都是主要的互斥实现形式。在每次获取资源之前，都需要检查是否有线程占用该资源。这里有两个关键点需要注意：

 - Sychronized是可重入的；
 - 已经进入的线程尚未执行完，将会阻塞后面其他线程；

#### 锁的本质是对象实例
对于非静态方法来说，Synchronized 有两种呈现形式，Synchronized方法体和Synchronized语句块。两种呈现形式本质上的锁都是对象实例。
来看看代码实现
```Java
package org.leon.concurent;

public class SynchronizeDemo {

    static SynchronizeDemo synchronizeDemo = new SynchronizeDemo();

    public static void main(String[] args) {
        Thread t1 = new Thread(synchronizeDemo::doSth1);
        Thread t2 = new Thread(synchronizeDemo::doSth1);
        t1.start();
        t2.start();
    }

    public void doSth1() {
        /**
         * 锁对象实例 synchronizeDemo
         */
        synchronized (synchronizeDemo){
            try {
                System.out.println("正在执行方法");
                Thread.sleep(10000);
                System.out.println("正在退出方法");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public void doSth2() {
        /**
         * 锁对象实例 this 等同于 synchronizeDemo
         */
        synchronized (this){
            try {
                System.out.println("正在执行方法");
                Thread.sleep(10000);
                System.out.println("正在退出方法");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public synchronized void doSth3() {
        /**
         *  表面呈现是锁方法体，实际上是synchronized (this) ，等价于上面
         */
        try {
            System.out.println("正在执行方法");
            Thread.sleep(10000);
            System.out.println("正在退出方法");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

那么对于静态方法来说锁定的又是什么呢？

```Java
package org.leon.concurent;

public class SynchronizeDemo {

    public static void main(String[] args) {
        Thread t1 = new Thread(SynchronizeDemo::doSth1);
        Thread t2 = new Thread(SynchronizeDemo::doSth1);
        t1.start();
        t2.start();
    }
    /**
     * 锁定Synchronized 的Class对象
     */
    public static void doSth1() {
        synchronized (SynchronizeDemo.class){
            try {
                System.out.println("正在执行方法");
                Thread.sleep(10000);
                System.out.println("正在退出方法");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 锁定当前类的Class对象，所以和上边等价
     */
    public synchronized static void doSth2() {
        try {
            System.out.println("正在执行方法");
            Thread.sleep(10000);
            System.out.println("正在退出方法");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

我们可以看出，从本质上而非呈现形式上看，synchronized同步也分两种。

- 锁类的对象实例，针对于某个具体实例普通方法/语句块的互斥；
- 锁类的Class对象，针对于Class类静态方法/语句块的互斥；


#### 进程切换导致的系统开销
Java的线程是直接映射到操作系统线程之上的，线程的挂起、阻塞、唤醒等都需要操作系统的参与，因此在线程切换的过程中是有一定的系统开销的。在多线程环境下调用Synchronized方法，有可能需要多次线程状态切换，因此可以说Synchronized是在Java语言中一个**重量级**操作。
虽然如此，JDK1.6版本后还是对Synchronized关键字做了相关优化，加入**锁自旋**特性减少系统线程切换导致的开销，几乎与ReentrantLock的性能不相上下，因此建议在能满足业务需求的前提下，优先使用Sychronized。

## ReentrantLock (可重入锁)
与Synchronized的实现原理类似，采用的都是**互斥同步**策略，用法和实现效果上来说也很相似，也具备可重入的特性。
不同点是，Sychronized是原生语法层面上同步控制，其颗粒度更大；相比而言，ReentranLock是从API层面来控制锁（``lock()``与``unlock()``方法），开发者的自主权更强，可控制粒度更细，优化空间更高。
#### 高级特性
这里可以直接引用《深入理解Java虚拟机》的描述
> - **等待可中断**是指当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情，可中断特性对处理执行时间非常长的同步块很有帮助。
- **公平锁**是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序依次获得锁；而非公平锁则不保证这一点，在锁被释放时，任何一个等待锁的线程都有机会获得锁。Sychronized的锁是非公平的，``ReentrantLock``默认情况下也是非公平的，但可以通过带boolean值的构造函数要求使用公平锁；
- **锁绑定多个条件**是指一个ReentrantLock对象可以同时绑定多个Condition对象，而在Synchronized中，锁对象的``wait()``和``notify()``或``notifyAll()``方法可以实现一个隐含的条件，如果要多于一个条件关联的时候，就不得不额外添加一个锁，而``ReentrantLock``无需这样做，只需要多次调用``newCondition()``方法即可。

#### 公平锁正确的打开方式
```java
package org.leon.concurent.lock;

import org.junit.Test;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 公平锁和非公平锁区别
 *
 * Created by LeonWong on 16/4/28.
 */
public class FairAndUnfairTest {

    private static Lock fairLock = new ReentrantLockRev(true);
    private static Lock unfairLock = new ReentrantLockRev(false);

    @Test
    public void fair() {
        testLock(fairLock);
    }

    @Test
    public void unfair() {
        testLock(unfairLock);
    }

    /**
     * 启动十个Job
     *
     * @param lock
     */
    private void testLock(Lock lock) {
        for (int i = 0; i < 10; i++) {
            new Thread(new Job(lock), "Thread-" + i).start();
        }
    }

    private static class Job extends Thread {
        private Lock lock;

        public Job(Lock lock) {
            this.lock = lock;
        }

        @Override
        public void run() {
            lock.lock();
            try {
                // 连续多次打印当前Tread和队列中的Thread
                for (int i = 0; i < 6; i++) {
                    System.out.println("Lock by ['" + Thread.currentThread().getName() + "']");
                }
            } finally {
                lock.unlock();
            }
        }
    }

    private static class ReentrantLockRev extends ReentrantLock {
        public ReentrantLockRev(boolean fair) {
            super(fair);
        }

        // 颠倒列表顺序
        public Collection<Thread> getQueuedThreads() {
            List<Thread> threads = new ArrayList<Thread>(super.getQueuedThreads());
            Collections.reverse(threads);
            return threads;
        }
    }

}
```
可以发现，使用非公平锁的时候，打印出来的线程名称顺序是乱的；而使用公平锁后，线程名称显示顺序则是按照先进先出的原则打印出来。

#### 绑定条件 -- Condition用法
在jdk1.5以前，线程的等待与唤醒功能，只能通过执行Object自带的``notify()``和``wait()``方法实现。来看个简单的栗子
```java
package org.leon.concurent.volatileUsage;

import org.junit.Test;
import org.leon.concurent.SleepUtils;

import java.text.SimpleDateFormat;
import java.util.Date;

public class WaitAndNotifyDemo {

    // 内存可见
    private static volatile boolean flag = true;

    private static final Object lock = new Object();

    @Test
    public void doLauncher() throws Exception {
        Thread waitThread = new Thread(new Wait(), "WaitThread");
        waitThread.start();
        SleepUtils.sleepForSecond(1);
        Thread notifyThread = new Thread(new Notify(), "NotifyThread");
        notifyThread.start();
        // 防止主线程关闭后导致子线程关闭
        SleepUtils.sleepForSecond(1000000);
    }

    private static class Wait implements Runnable {
        @Override
        public void run() {
            // 加锁,拥有lock的Monitor
            synchronized (lock) {
                // 当条件不满足时,继续wait,同时释放了lock的锁
                while (flag) {
                    try {
                        System.out.println(Thread.currentThread() + " flag is true. wait @ "
                                + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(Thread.currentThread() + " Thread has been woke!!!!. wait @ "
                            + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                }
                // 条件满足,完成工作
                System.out.println(Thread.currentThread() + " flag is false. done!!! @ "
                        + new SimpleDateFormat("HH:mm:ss").format(new Date()));
            }
        }
    }

    private static class Notify implements Runnable {
        @Override
        public void run() {
            // 加锁,拥有lock的Monitor
            synchronized (lock) {
                // 获取lock的锁,然后进行通知,通知时不会释放lock的锁
                // 直到当前线程释放了lock后,waitThread才能从wait方法中返回
                System.out.println(Thread.currentThread() + " hold lock. notify @ "
                        + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                System.out.println(Thread.currentThread() + " do notifyAll,but I wanna sleep 4 5 secs. notify @ "
                        + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                lock.notifyAll();
                flag = false;
                SleepUtils.sleepForSecond(5);
            }
            // 再次加锁
            synchronized (lock) {
                System.out.println(Thread.currentThread() + " hold lock another 5 secs and. notify @ "
                        + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                SleepUtils.sleepForSecond(5);
            }
        }
    }

}

```
这里要注意的是，执行``wait()``方法后，线程进入等待状态，并且释放了锁，等待其他线程执行``notify()``方法将其唤醒。
而Condition的用法与其极其类似，先是从``Lock``对象中执行``newCondition()``实例化Condition对象，且允许实例化多个。通过调用``await()``和``notify()``方法完成等待唤醒一系列操作，来看个有界阻塞队列的实现。
```java
package org.leon.concurent.condition;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 有界阻塞队列<br/>
 * 当队列为空,队列将会阻塞删除并获取操作的线程,直到队列中有新元素;<br/>
 * 当队列已满,队列将会阻塞添加操作的线程，直到队列有空位置；
 * <p>
 * Created by LeonWong on 16/4/29.
 */
public class BoundedQueue<T> {
    private Object[] items;

    // 添加的下标,删除的下标和数组当前数量
    private int addIndex, removeIndex, count;
    private Lock lock = new ReentrantLock();
    private Condition notEmpty = lock.newCondition();
    private Condition notFull = lock.newCondition();

    public BoundedQueue() {
        items = new Object[5];
    }

    public BoundedQueue(int size) {
        items = new Object[size];
    }

    /**
     * 添加一个元素,数组满则添加线程进入等待状态
     *
     * @param t
     * @throws InterruptedException
     */
    public void add(T t) throws InterruptedException {
        lock.lock();
        try {
            while (items.length == count) {
                System.out.println("添加队列--陷入等待");
                notFull.await();
            }
            items[addIndex] = t;
            addIndex = ++addIndex == items.length ? 0 : addIndex;
            count++;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    /**
     * 删除并获取一个元素,数组空则进入等待
     *
     * @return
     * @throws InterruptedException
     */
    public T remove() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                System.out.println("删除队列--陷入等待");
                notEmpty.await();
            }
            Object tmp = items[removeIndex];
            items[removeIndex] = null;// 这一步可以有可无
            removeIndex = ++removeIndex == items.length ? 0 : removeIndex;
            count--;
            notFull.signal();
            return (T) tmp;
        } finally {
            lock.unlock();
        }
    }

    public Object[] getItems() {
        return items;
    }

    public void setItems(Object[] items) {
        this.items = items;
    }

    public int getAddIndex() {
        return addIndex;
    }

    public void setAddIndex(int addIndex) {
        this.addIndex = addIndex;
    }

    public int getRemoveIndex() {
        return removeIndex;
    }

    public void setRemoveIndex(int removeIndex) {
        this.removeIndex = removeIndex;
    }

    public int getCount() {
        return count;
    }

    public void setCount(int count) {
        this.count = count;
    }

}
```
测试用例
```java
package org.leon.concurent.condition;

import org.junit.Test;
import org.leon.concurent.SleepUtils;

public class BoundedQueueTest {

    /**
     * 队列初始化size为10
     */
    private BoundedQueue<String> boundedQueue = new BoundedQueue<>(10);

    @Test
    public void testBoundedQueue() throws InterruptedException {
        // 删除队列,起初队列为空,务必陷入等待
        new Thread(new DoRemove()).start();

        SleepUtils.sleepForSecond(2);

        // 添加11条数据入队,队列“有可能”会满,并陷入等待
        for (int i = 0; i < 11; i++) {
            new Thread(new DoAdd()).start();
        }

        System.out.println("添加队列完毕");
        SleepUtils.sleepForSecond(4);

        // 再删一次
        boundedQueue.remove();

        SleepUtils.sleepForSecond(2);

        System.out.println("操作完毕,其中addIndex为" + boundedQueue.getAddIndex() + ",removeIndex为"
                + boundedQueue.getRemoveIndex() + ",count为" + boundedQueue.getCount());
    }

    private class DoRemove implements Runnable {
        @Override
        public void run() {
            try {
                boundedQueue.remove();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    private class DoAdd implements Runnable {
        @Override
        public void run() {
            try {
                boundedQueue.add("Something");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
#### AbstractQueuedSynchronizer（简称AQS）
ReentrantLock实现是基于AQS这个抽象方法的，然而篇幅有限，将在下一篇文章里从代码的角度着重探讨。
简单来说，AQS是通过管理状态的方式来实现相对线程安全的。Java中信号量（Semaphore）、读写锁（ReadWriteLock）、计数器（CountDownLatch）以及FutureTask等都是基于AQS实现的，可见这个抽象类的地位多么不一般。

#### 与其说ReentrantLock性能更好不如说Synchronized优化空间更大
上面介绍过，Synchronized在JDK1.6以后性能有所增强，因此在能满足业务复杂度需求的情况下，采用Synchronized也未尝不可。然而**互斥同步**终究属于悲观的并发策略，在对性能要求极高的业务场景下使用以上**互斥同步**策略并不合适。接下来进而介绍如何实现乐观的同步策略。
## Java中的非阻塞同步策略
#### CAS指令与原子性
原子操作的业务表现形式是“不可被中断或不可被分割操作”。所谓CAS（Compare And Swap）比较并交换就是一种原子操作。简单来说执行CAS需要两个参数，一个新值，一个旧值，当比较内存的值与旧值相符时，则替换为新值，否则不执行替换操作。CPU如何实现，这里不多说，Java若要实现CAS则需要CPU指令集配合，JDK1.5加入了这个特性，并在随后的版本对其进行丰富。
```java
package org.leon.concurent.atomic;

import org.junit.Test;

import java.util.concurrent.atomic.AtomicInteger;

/**
 * Created by LeonWong on 16/6/10.
 */
public class AtomicItegerUpdaterTest {
    private static AtomicInteger couter = new AtomicInteger(0);

    @Test
    public void doUpdate() {
        couter.compareAndSet(0, 1);
        System.out.println("结果为" + couter.get());// 结果为1
        couter.compareAndSet(0, 3);
        System.out.println("结果为" + couter.get());// 结果为1
    }
}
```
除了Integer以外，还支持包括CAS更新实例、更新实例的属性等功能。
阅读源码不难发现，Java是通过一个``sun.misc.Unsafe``的类，完成CAS指令操作的，然而我们从AQS的源码中也发现了``sun.misc.Unsafe``类的踪影。
```java
    /**
     * Atomically sets synchronization state to the given updated
     * value if the current state value equals the expected value.
     * This operation has memory semantics of a {@code volatile} read
     * and write.
     *
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that the actual
     *         value was not equal to the expected value.
     */
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```
其实不难理解，AQS负责管理状态（也可以理解为互斥资源）—— 这里狭义来说可以是锁是否被线程占用的标记，当然，状态的判定规则以及互斥资源数目由AQS的继承者们负责实现，而状态的更新只能是通过CAS指令完成，以确保线程安全。

## volatile 关键字
``volatile``在多线程环境下保证了共享变量内存可见性。意思就是线程A修改了``volatile``修饰的共享变量，线程B能够感知修改。如果``volatile``合理使用的话，将会比Synchronized的执行成本更低。
从底层的角度来说，为了提高处理速度，CPU不直接和内存进行通信，而是先将数据读入到CPU缓存后在进行操作，但不知何时将会更新到内存。声明变量加入``volatile``关键字后，每次修改该变量，JVM就会通知处理器将CPU缓存内的值强制更新到内存中，这就是所谓的“可见性”。再来看代码。

```java
package org.leon.concurent.volatileUsage;

/**
 * VolatileDemo 与 SynchronizedDemo 实现效果等价
 *
 * Created by LeonWong on 16/6/10.
 */
public class VolatileDemo {
    volatile long vl = 0L;

    public void set(long l) {
        vl = l;
    }

    public void getAndIncrement() {
        vl++;
    }

    public long get() {
        return vl;
    }
}

class SynchronizedDemo {
    long vl = 0L;

    public synchronized void set(long l) {
        vl = l;
    }

    public void getAndIncrement() {
        long temp = get();
        temp += 1L;
        set(temp);
    }

    public synchronized long get() {
        return vl;
    }
}
```
``VolatileDemo`` 与 ``SynchronizedDemo``实现效果等价，但是性能来说前者更优，原因前面也说了，``Synchronized``在多线程环境下会引起上下文切换及调度，在并发量大的前提下，有不小的性能开销。因此合理使用``volatile``有助于我们代码性能的优化。

## 无同步策略
这就比较容易理解了，同步只是线程安全的一个手段，无同步并不意味着线程不安全。大致两种方法的代码可以保证没有使用同步方案的前提下的线程安全。

 - **可重入代码。**例如纯计算的函数之类的，方法运行间不需要获取外部资源就可以进行计算。
 - **线程本地存储资源**。很好理解，线程本地维护自己的资源，根本不存在与其他线程资源冲突的可能。


# 总结
本篇简单介绍了Java并发环境下有关线程安全的语法糖，也做了一些粗略的利弊分析，想必在未来开发的过程中，我们会对线程安全的实现方式有一个基本的考量。如果可以，尽可能用计算机的底层去思考我们编码会造成怎样的影响，我们的代码是否适用于业务本身，诸如此类。关于并发编程，未来还会继续更新。
