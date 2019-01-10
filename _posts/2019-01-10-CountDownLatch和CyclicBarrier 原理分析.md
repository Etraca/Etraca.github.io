---
layout:     post
title:      CountDownLatch和CyclicBarrier 原理分析
subtitle:   CountDownLatch和CyclicBarrier 原理分析
date:       2019-01-10
author:     WPF
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- CountDownLatch
- CyclicBarrier
- java
---

## 简介
在分析完AbstractQueuedSynchronizer（以下简称 AQS）和ReentrantLock的原理后，本文将分析 java.util.concurrent 包下的两个线程同步组件CountDownLatch和CyclicBarrier。这两个同步组件比较常用，也经常被放在一起对比。通过分析这两个同步组件，可使我们对 Java 线程间协同有更深入的了解。同时通过分析其原理，也可使我们做到知其然，并知其所以然。

这里首先来介绍一下 CountDownLatch 的用途，CountDownLatch 允许一个或一组线程等待其他线程完成后再恢复运行。线程可通过调用await方法进入等待状态，在其他线程调用countDown方法将计数器减为0后，处于等待状态的线程即可恢复运行。CyclicBarrier （可循环使用的屏障）则与此不同，CyclicBarrier 允许一组线程到达屏障后阻塞住，直到最后一个线程进入到达屏障，所有线程才恢复运行。它们之间主要的区别在于唤醒等待线程的时机。CountDownLatch 是在计数器减为0后，唤醒等待线程。CyclicBarrier 是在计数器（等待线程数）增长到指定数量后，再唤醒等待线程。除此之外，两种之间还有一些其他的差异，这个将会在后面进行说明。

## 原理
#### CountDownLatch 的实现原理
CountDownLatch 的同步功能是基于 AQS 实现的，CountDownLatch 使用 AQS 中的 state 成员变量作为计数器。在 state 不为0的情况下，凡是调用 await 方法的线程将会被阻塞，并被放入 AQS 所维护的同步队列中进行等待。大致示意图如下：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15257920913484.jpg)

每个阻塞的线程都会被封装成节点对象，节点之间通过 prev 和 next 指针形成同步队列。初始情况下，队列的头结点是一个虚拟节点。该节点仅是一个占位符，没什么特别的意义。每当有一个线程调用 countDown 方法，就将计数器 state–。当 state 被减至0时，队列中的节点就会按照 FIFO 顺序被唤醒，被阻塞的线程即可恢复运行。

CountDownLatch 本身的原理并不难理解，不过如果大家想深入理解 CountDownLatch 的实现细节，那么需要先去学习一下 AQS 的相关原理。CountDownLatch 是基于 AQS 实现的，所以理解 AQS 是学习 CountDownLatch 的前置条件。我在之前写过一篇关于 AQS 的文章 Java 重入锁 ReentrantLock 原理分析，有兴趣的朋友可以去读一读。

#### CyclicBarrier 的实现原理
与 CountDownLatch 的实现方式不同，CyclicBarrier 并没有直接通过 AQS 实现同步功能，而是在重入锁 ReentrantLock 的基础上实现的。在 CyclicBarrier 中，线程访问 await 方法需先获取锁才能访问。在最后一个线程访问 await 方法前，其他线程进入 await 方法中后，会调用 Condition 的 await 方法进入等待状态。在最后一个线程进入 CyclicBarrier await 方法后，该线程将会调用 Condition 的 signalAll 方法唤醒所有处于等待状态中的线程。同时，最后一个进入 await 的线程还会重置 CyclicBarrier 的状态，使其可以重复使用。

在创建 CyclicBarrier 对象时，需要转入一个值，用于初始化 CyclicBarrier 的成员变量 parties，该成员变量表示屏障拦截的线程数。当到达屏障的线程数小于 parties 时，这些线程都会被阻塞住。当最后一个线程到达屏障后，此前被阻塞的线程才会被唤醒。

## 源码分析
通过前面简单的分析，相信大家对 CountDownLatch 和 CyclicBarrier 的原理有一定的了解了。那么接下来趁热打铁，我们一起探索一下这两个同步组件的具体实现吧。

#### CountDownLatch 源码分析
CountDownLatch 的原理不是很复杂，所以在具体的实现上，也不是很复杂。当然，前面说过 CountDownLatch 是基于 AQS 实现的，AQS 的实现则要复杂的多。不过这里仅要求大家掌握 AQS 的基本原理，知道它内部维护了一个同步队列，同步队列中的线程会按照 FIFO 依次获取同步状态就行了。好了，下面我们一起去看一下 CountDownLatch 的源码吧。

###### 源码结构
CountDownLatch 的代码量不大，加上注释也不过300多行，所以它的代码结构也会比较简单。如下：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15258394284770.jpg)

如上图，CountDownLatch 源码包含一个构造方法和一个私有成员变量，以及数个普通方法和一个重要的静态内部类 Sync。CountDownLatch 的主要逻辑都是封装在 Sync 和其父类 AQS 里的。所以分析 CountDownLatch 的源码，本质上是分析 Sync 和 AQS 的原理。相关的分析，将会在下一节中展开，本节先说到这。

###### 构造方法及成员变量
本节来分析一下 CountDownLatch 的构造方法和其 Sync 类型的成员变量实现，如下：

```java
public class CountDownLatch {

    private final Sync sync;

    /** CountDownLatch 的构造方法，该方法要求传入大于0的整型数值作为计数器 */
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        // 初始化 Sync
        this.sync = new Sync(count);
    }
    
    /** CountDownLatch 的同步控制器，继承自 AQS */
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            // 设置 AQS state
            setState(count);
        }

        int getCount() {
            return getState();
        }

        /** 尝试在共享状态下获取同步状态，该方法在 AQS 中是抽象方法，这里进行了覆写 */
        protected int tryAcquireShared(int acquires) {
            /*
             * 如果 state = 0，则返回1，表明可获取同步状态，
             * 此时线程调用 await 方法时就不会被阻塞。
             */ 
            return (getState() == 0) ? 1 : -1;
        }

        /** 尝试在共享状态下释放同步状态，该方法在 AQS 中也是抽象方法 */
        protected boolean tryReleaseShared(int releases) {
            /*
             * 下面的逻辑是将 state--，state 减至0时，调用 await 等待的线程会被唤醒。
             * 这里使用循环 + CAS，表明会存在竞争的情况，也就是多个线程可能会同时调用 
             * countDown 方法。在 state 不为0的情况下，线程调用 countDown 是必须要完
             * 成 state-- 这个操作。所以这里使用了循环 + CAS，确保 countDown 方法可正
             * 常运行。
             */
            for (;;) {
                // 获取 state
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                
                // 使用 CAS 设置新的 state 值
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
}
```
需要说明的是，Sync 中的 tryAcquireShared 和 tryReleaseShared 方法并不是直接给 await 和 countDown 方法调用了的，这两个方法以“try”开头的方法最终会在 AQS 中被调用。

###### await
CountDownLatch中有两个版本的 await 方法，一个响应中断，另一个在此基础上增加了超时功能。本节将分析无超时功能的 await，如下：

```java
/** 
 * 该方法会使线程进入等待状态，直到计数器减至0，或者线程被中断。当计数器为0时，调用
 * 此方法将会立即返回，不会被阻塞住。
 */
public void await() throws InterruptedException {
    // 调用 AQS 中的 acquireSharedInterruptibly 方法
    sync.acquireSharedInterruptibly(1);
}

/** 带有超时功能的 await */
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}

+--- AbstractQueuedSynchronizer
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
    // 若线程被中断，则直接抛出中断异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 调用 Sync 中覆写的 tryAcquireShared 方法，尝试获取同步状态
    if (tryAcquireShared(arg) < 0)
        /*
         * 若 tryAcquireShared 小于0，则表示获取同步状态失败，
         * 此时将线程放入 AQS 的同步队列中进行等待。
         */ 
        doAcquireSharedInterruptibly(arg);
}
```
从上面的代码中可以看出，CountDownLatch await 方法实际上调用的是 AQS 的 acquireSharedInterruptibly 方法。该方法会在内部调用 Sync 所覆写的 tryAcquireShared 方法。在 state != 0时，tryAcquireShared 返回值 -1。此时线程将进入 doAcquireSharedInterruptibly 方法中，在此方法中，线程会被放入同步队列中进行等待。若 state = 0，此时 tryAcquireShared 返回1，acquireSharedInterruptibly 会直接返回。此时调用 await 的线程也不会被阻塞住。

###### countDown
与 await 方法一样，countDown 实际上也是对 AQS 方法的一层封装。具体的实现如下：

```java
** 该方法的作用是将计数器进行自减操作，当计数器为0时，唤醒正在同步队列中等待的线程 */
public void countDown() {
    // 调用 AQS 中的 releaseShared 方法
    sync.releaseShared(1);
}

+--- AbstractQueuedSynchronizer
public final boolean releaseShared(int arg) {
    // 调用 Sync 中的 tryReleaseShared 尝试释放同步状态
    if (tryReleaseShared(arg)) {
        /*
         * tryReleaseShared 返回 true 时，表明 state = 0，即计数器为0。此时调用
         * doReleaseShared 方法唤醒正在同步队列中等待的线程
         */ 
        doReleaseShared();
        return true;
    }
    return false;
}
```

#### CyclicBarrier 源码分析
###### 源码结构
如前面所说，CyclicBarrier 是基于重入锁 ReentrantLock 实现相关逻辑的。所以要弄懂 CyclicBarrier 的源码，仅需有 ReentrantLock 相关的背景知识即可。关于重入锁 ReentrantLock 方面的知识，有兴趣的朋友可以参考我之前写的文章 Java 重入锁 ReentrantLock 原理分析。下面看一下 CyclicBarrier 的代码结构吧，如下：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15258605203557.jpg)

从上图可以看出，CyclicBarrier 包含了一个静态内部类Generation、数个方法和一些成员变量。结构上比 CountDownLatch 略为复杂一些，但总体仍比较简单。好了，接下来进入源码分析部分吧。

###### 构造方法及成员变量
CyclicBarrier 包含两个有参构造方法，分别如下：

```java
/** 创建一个允许 parties 个线程通行的屏障 */
public CyclicBarrier(int parties) {
    this(parties, null);
}

/** 
 * 创建一个允许 parties 个线程通行的屏障，若 barrierAction 回调对象不为 null，
 * 则在最后一个线程到达屏障后，执行相应的回调逻辑
 */
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```
上面的第二个构造方法初始化了一些成员变量，下面我们就来说明一下这些成员变量的作用。

成员变量	|作用
--------|------
parties |线程数，即当 parties 个线程到达屏障后，屏障才会放行
count   |计数器，当 count > 0 时，到达屏障的线程会进入等待状态。当最后一个线程到达屏障后，count 自减至0。最后一个到达的线程会执行回调方法，并唤醒其他处于等待状态中的线程。
barrierCommand |回调对象，如果不为 null，会在第 parties 个线程到达屏障后被执行

除了上面几个成员变量，还有一个成员变量需要说明一下，如下：

```java
/** 
 * CyclicBarrier 是可循环使用的屏障，这里使用 Generation 记录当前轮次 CyclicBarrier 
 * 的运行状态。当所有线程到达屏障后，generation 将会被更新，表示 CyclicBarrier 进入新一
 * 轮的运行轮次中。
*/
private Generation generation = new Generation();

private static class Generation {
    // 用于记录屏障有没有被破坏
    boolean broken = false;
}
```
###### await
上一节所提到的几个成员变量，在 await 方法中将会悉数登场。下面就来分析一下 await 方法的试下，如下：

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        // await 的逻辑封装在 dowait 中
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
    
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException, TimeoutException {
    
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        final Generation g = generation;

        // 如果 g.broken = true，表明屏障被破坏了，这里直接抛出异常
        if (g.broken)
            throw new BrokenBarrierException();

        // 如果线程中断，则调用 breakBarrier 破坏屏障
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        /*
         * index 表示线程到达屏障的顺序，index = parties - 1 表明当前线程是第一个
         * 到达屏障的。index = 0，表明当前线程是最有一个到达屏障的。
         */ 
        int index = --count;
        // 当 index = 0 时，唤醒所有处于等待状态的线程
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                // 如果回调对象不为 null，则执行回调
                if (command != null)
                    command.run();
                ranAction = true;
                // 重置屏障状态，使其进入新一轮的运行过程中
                nextGeneration();
                return 0;
            } finally {
                // 若执行回调的过程中发生异常，此时调用 breakBarrier 破坏屏障
                if (!ranAction)
                    breakBarrier();
            }
        }

        // 线程运行到此处的线程都会被屏障挡住，并进入等待状态。
        for (;;) {
            try {
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                /*
                 * 若下面的条件成立，则表明本轮运行还未结束。此时调用 breakBarrier 
                 * 破坏屏障，唤醒其他线程，并抛出异常
                 */ 
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    /*
                     * 若上面的条件不成立，则有两种可能：
                     * 1. g != generation
                     *     此种情况下，表明循环屏障的第 g 轮次的运行已经结束，屏障已经
                     *     进入了新的一轮运行轮次中。当前线程在稍后返回 到达屏障 的顺序即可
                     *     
                     * 2. g = generation 但 g.broken = true
                     *     此种情况下，表明已经有线程执行过 breakBarrier 方法了，当前
                     *     线程则会在稍后抛出 BrokenBarrierException
                     */
                    Thread.currentThread().interrupt();
                }
            }
            
            // 屏障被破坏，则抛出 BrokenBarrierException 异常
            if (g.broken)
                throw new BrokenBarrierException();

            // 屏障进入新的运行轮次，此时返回线程在上一轮次到达屏障的顺序
            if (g != generation)
                return index;

            // 超时判断
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
    
/** 开启新的一轮运行过程 */
private void nextGeneration() {
    // 唤醒所有处于等待状态中的线程
    trip.signalAll();
    // 重置 count
    count = parties;
    // 重新创建 Generation，表明进入循环屏障进入新的一轮运行轮次中
    generation = new Generation();
}

/** 破坏屏障 */
private void breakBarrier() {
    // 设置屏障是否被破坏标志
    generation.broken = true;
    // 重置 count
    count = parties;
    // 唤醒所有处于等待状态中的线程
    trip.signalAll();
}
```
###### reset
reset 方法用于强制重置屏障，使屏障进入新一轮的运行过程中。代码如下：

```java
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 破坏屏障
        breakBarrier();   // break the current generation
        // 开启新一轮的运行过程
        nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}
```
看完上面的分析，相信大家对着两个同步组件有了更深入的认识。那么下面趁热打铁，简单对比一下两者之间的区别。这里用一个表格列举一下：

差异点	 |CountDownLatch |CyclicBarrier
------|----------------|-------
是否可循环使用	|否|是
是否可设置回调	|否|是



