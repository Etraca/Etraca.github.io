---
layout:     post
title:      ReentrantLock 原理分析
subtitle:   ReentrantLock 原理分析
date:       2019-01-09
author:     WPF
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- ReentrantLock
- java
---

## 简介
可重入锁ReentrantLock自 JDK 1.5 被引入，功能上与synchronized关键字类似。所谓的可重入是指，线程可对同一把锁进行重复加锁，而不会被阻塞住，这样可避免死锁的产生。ReentrantLock 的主要功能和 synchronized 关键字一致，均是用于多线程的同步。但除此之外，ReentrantLock 在功能上比 synchronized 更为丰富。比如 ReentrantLock 在加锁期间，可响应中断，可设置超时等。

## 原理
#### 可重入
可重入这个概念并不难理解，本节通过一个例子简单说明一下。

现在有方法 m1 和 m2，两个方法均使用了同一把锁对方法进行同步控制，同时方法 m1 会调用 m2。线程 t 进入方法 m1 成功获得了锁，此时线程 t 要在没有释放锁的情况下，调用 m2 方法。由于 m1 和 m2 使用的是同一把可重入锁，所以线程 t 可以进入方法 m2，并再次获得锁，而不会被阻塞住。示例代码大致如下：

```java
void m1() {
    lock.lock();
    try {
        // 调用 m2，因为可重入，所以并不会被阻塞
        m2();
    } finally {
        lock.unlock()
    }
}

void m2() {
    lock.lock();
    try {
        // do something
    } finally {
        lock.unlock()
    }
}
```
假如 lock 是不可重入锁，那么上面的示例代码必然会引起死锁情况的发生。这里请大家思考一个问题，ReentrantLock 的可重入特性是怎样实现的呢？简单说一下，ReentrantLock 内部是通过 AQS 实现同步控制的，AQS 有一个变量 state 用于记录同步状态。初始情况下，state = 0，表示 ReentrantLock 目前处于解锁状态。如果有线程调用 lock 方法进行加锁，state 就由0变为1，如果该线程再次调用 lock 方法加锁，就让其自增，即 state++。线程每调用一次 unlock 方法释放锁，会让 state–。通过查询 state 的数值，即可知道 ReentrantLock 被重入的次数了。这就是可重复特性的大致实现流程。

#### 公平与非公平
公平与非公平指的是线程获取锁的方式。公平模式下，线程在同步队列中通过 FIFO 的方式获取锁，每个线程最终都能获取锁。在非公平模式下，线程会通过“插队”的方式去抢占锁，抢不到的则进入同步队列进行排队。默认情况下，ReentrantLock 使用的是非公平模式获取锁，而不是公平模式。不过我们也可通过 ReentrantLock 构造方法ReentrantLock(boolean fair)调整加锁的模式。

既然既然有两种不同的加锁模式，那么他们有什么优缺点呢？答案如下：

公平模式下，可保证每个线程最终都能获得锁，但效率相对比较较低。非公平模式下，效率比较高，但可能会导致线程出现饥饿的情况。即一些线程迟迟得不到锁，每次即将到手的锁都有可能被其他线程抢了。这里再提个问题，为啥非公平模式抢了其他线程获取锁的机会，而整个程序的运行效率会更高呢？说实话，开始我也不明白。不过好在《Java并发编程实战》在第13.3节 公平性（p232）说明了具体的原因，这里引用一下：

```text
在激烈竞争的情况下，非公平锁的性能高于公平锁的性能的一个原因是：
在恢复一个被挂起的线程与该线程真正开始运行之间存在着严重的延迟。假设线程 A 持有一个锁，并且线程 B 请求这个锁。
由于这个线程已经被线程 A 持有，因此 B 将被挂起。当 A 释放锁时，B 将被唤醒，因此会再次尝试获取锁。
与此同时，如果 C 也请求这个锁，那么 C 很有可能会在 B 被完全唤醒前获得、使用以及释放这个锁。这样的情况时一种“双赢”的局面：
B 获得锁的时刻并没有推迟，C 更早的获得了锁，并且吞吐量也获得了提高。
```

上面的原因大家看懂了吗？下面配个图辅助说明一下：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15255931517868.jpg)

如上图，线程 C 在线程 B 苏醒阶段内获取和使用锁，并在线程 B 获取锁前释放了锁，所以线程 B 可以顺利获得锁。线程 C 在抢占锁的情况下，仍未影响线程 B 获取锁，因此是个“双赢”的局面。

除了上面的原因外，《Java并发编程的艺术》在其5.3.2 公平与非公平锁的区别（p137）分析了另一个可能的原因。即公平锁线程切换次数要比非公平锁线程切换次数多得多，因此效率上要低一些。更多的细节，可以参考作者的论述，这里不展开说明了。

本节最后说一下公平锁和非公平锁的使用场景。如果线程持锁时间短，则应使用非公平锁，可通过“插队”提升效率。如果线程持锁时间长，“插队”带来的效率提升可能会比较小，此时应使用公平锁。

## 源码分析
####代码结构
前面说到 ReentrantLock 是基于 AQS 实现的，AQS 很好的封装了同步队列的管理，线程的阻塞与唤醒等基础操作。基于 AQS 的同步组件，推荐的使用方式是通过内部非 public 静态类继承 AQS，并重写部分抽象方法。其代码结构大致如下：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15256891997562.jpg)

上图中，Sync是一个静态抽象类，继承了 AbstractQueuedSynchronizer。公平和非公平锁的实现类NonfairSync和FairSync则继承自 Sync 。至于 ReentrantLock 中的其他一些方法，主要逻辑基本上都在几个内部类中实现的。

#### 获取锁
在分析 ReentrantLock 加锁的代码前，下来简单介绍一下 AQS 同步队列的一些知识。AQS 维护了一个基于双向链表的同步队列，线程在获取同步状态失败的情况下，都会被封装成节点，然后加入队列中。同步队列大致示意图如下：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15256142424566.jpg)

在同步队列中，头结点是获取同步状态的节点。其他节点在尝试获取同步状态失败后，会被阻塞住，暂停运行。当头结点释放同步状态后，会唤醒其后继节点。后继节点会将自己设为头节点，并将原头节点从队列中移除。大致示意图如下：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15256143084131.jpg)

介绍完 AQS 同步队列，以及节点线程获取同步状态的过程。下面来分析一下 ReentrantLock 中获取锁方法的源码，如下：

```java
public void lock() {
    sync.lock();
}

abstract static class Sync extends AbstractQueuedSynchronizer {
    // 这里的 lock 是抽象方法，具体的实现在两个子类中
    abstract void lock();
    
    // 省略其他无关代码
}
```
lock 方法的实现很简单，不过这里的 lock 方法只是一个壳子而已。由于获取锁的方式有公平和非公平之分，所以具体的实现是在NonfairSync和FairSync两个类中。那么我们继续往下分析一下这两个类的实现。

###### 公平锁
公平锁对应的逻辑是 ReentrantLock 内部静态类 FairSync，我们沿着上面的 lock 方法往下分析，如下：

```java
+--- ReentrantLock.FairSync.java
final void lock() {
    // 调用 AQS acquire 获取锁
    acquire(1);
}

+--- AbstractQueuedSynchronizer.java
/**
 * 该方法主要做了三件事情：
 * 1. 调用 tryAcquire 尝试获取锁，该方法需由 AQS 的继承类实现，获取成功直接返回
 * 2. 若 tryAcquire 返回 false，则调用 addWaiter 方法，将当前线程封装成节点，
 *    并将节点放入同步队列尾部
 * 3. 调用 acquireQueued 方法让同步队列中的节点循环尝试获取锁
 */
public final void acquire(int arg) {
    // acquireQueued 和 addWaiter 属于 AQS 中的方法，这里不展开分析了
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

+--- ReentrantLock.FairSync.java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // 获取同步状态
    int c = getState();
    // 如果同步状态 c 为0，表示锁暂时没被其他线程获取
    if (c == 0) {
        /*
         * 判断是否有其他线程等待的时间更长。如果有，应该先让等待时间更长的节点先获取锁。
         * 如果没有，调用 compareAndSetState 尝试设置同步状态。
         */ 
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            // 将当前线程设置为持有锁的线程
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果当前线程为持有锁的线程，则执行重入逻辑
    else if (current == getExclusiveOwnerThread()) {
        // 计算重入后的同步状态，acquires 一般为1
        int nextc = c + acquires;
        // 如果重入次数超过限制，这里会抛出异常
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        // 设置重入后的同步状态
        setState(nextc);
        return true;
    }
    return false;
}

+--- AbstractQueuedSynchronizer.java
/** 该方法用于判断同步队列中有比当前线程等待时间更长的线程 */
public final boolean hasQueuedPredecessors() {
    Node t = tail;
    Node h = head;
    Node s;
    /*
     * 在同步队列中，头结点是已经获取了锁的节点，头结点的后继节点则是即将获取锁的节点。
     * 如果有节点对应的线程等待的时间比当前线程长，则返回 true，否则返回 false
     */
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

ReentrantLock 中获取锁的流程并不是很复杂，上面的代码执行流程如下：
1. 调用 acquire 方法，将线程放入同步队列中进行等待
2. 线程在同步队列中成功获取锁，则将自己设为持锁线程后返回
3. 若同步状态不为0，且当前线程为持锁线程，则执行重入逻辑

###### 非公平锁
分析完公平锁相关代码，下面再来看看非公平锁的源码分析，如下：

```java
+--- ReentrantLock.NonfairSync
final void lock() {
    /*
     * 这里调用直接 CAS 设置 state 变量，如果设置成功，表明加锁成功。这里并没有像公平锁
     * 那样调用 acquire 方法让线程进入同步队列进行排队，而是直接调用 CAS 抢占锁。抢占失败
     * 再调用 acquire 方法将线程置于队列尾部排队。
     */
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}

+--- AbstractQueuedSynchronizer
/** 参考上一节的分析 */
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

+--- ReentrantLock.NonfairSync
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}

+--- ReentrantLock.Sync
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // 获取同步状态
    int c = getState();
    
    // 如果同步状态 c = 0，表明锁当前没有线程获得，此时可加锁。
    if (c == 0) {
        // 调用 CAS 加锁，如果失败，则说明有其他线程在竞争获取锁
        if (compareAndSetState(0, acquires)) {
            // 设置当前线程为锁的持有线程
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果当前线程已经持有锁，此处条件为 true，表明线程需再次获取锁，也就是重入
    else if (current == getExclusiveOwnerThread()) {
        // 计算重入后的同步状态值，acquires 一般为1
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        // 设置新的同步状态值
        setState(nextc);
        return true;
    }
    return false;
}
```

非公平锁的实现也不是很复杂，其加锁的步骤大致如下：
1. 调用 compareAndSetState 方法抢占式加锁，加锁成功则将自己设为持锁线程，并返回
2. 若加锁失败，则调用 acquire 方法，将线程置于同步队列尾部进行等待
3. 线程在同步队列中成功获取锁，则将自己设为持锁线程后返回
4. 若同步状态不为0，且当前线程为持锁线程，则执行重入逻辑

###### 公平和非公平细节对比
如果大家之前阅读过公平锁和非公平锁的源码，会发现两者之间的差别不是很大。为了找出它们之间的差异，这里我将两者的对比代码放在一起，大家可以比较一下，如下：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15256177988159.jpg)

从上面的源码对比图中，可以看出两种的差异并不大。那么现在请大家思考一个问题：在代码差异不大情况下，是什么差异导致了公平锁和非公平锁的产生呢？大家先思考一下，答案将会在下面展开说明。

在上面的源码对比图中，左边是非公平锁的实现，右边是公平锁的实现。从对比图中可看出，两者的 lock 方法有明显区别。非公平锁的 lock 方法会首先尝试去抢占设置同步状态，而不是直接调用 acquire 将线程放入同步队列中等待获取锁。除此之外，tryAcquire 方法实现上也有差异。由于非公平锁的 tryAcquire 逻辑主要封装在 Sync 中的 nonfairTryAcquire 方法里，所以我们直接对比这个方法即可。由上图可以看出，Sync 中的 nonfairTryAcquire 与公平锁中的 tryAcquire 实现上差异并不大，唯一的差异在第18行，这里我用一条红线标注了出来。公平锁的 tryAcquire 在第18行多出了一个条件，即!hasQueuedPredecessors()。这个方法的目的是判断是否有其他线程比当前线程在同步队列中等待的时间更长。有的话，返回 true，否则返回 false。比如下图：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15256186886587.jpg)

node1 对应的线程比 node2 对应的线程在队列中等待的时间更长，如果 node2 线程调用 hasQueuedPredecessors 方法，则会返回 true。如果 node1 调用此方法，则会返回 false。因为 node1 前面只有一个头结点，但头结点已经获取同步状态，不处于等待状态。所以在所有处于等待状态的节点中，没有节点比它等待的更长了。理解了 hasQueuedPredecessors 方法的用途后，那么现在请大家思考个问题，假如把条件去掉对公平锁会有什么影响呢？答案在 lock 所调用的 acquire 方法中，再来看一遍 acquire 方法源码：

```java
    public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

acquire 方法先调用子类实现的 tryAcquire 方法，用于尝试获取同步状态，调用成功则直接返回。若调用失败，则应将线程插入到同步队列尾部，按照 FIFO 原则获取锁。如果我们把 tryAcquire 中的条件!hasQueuedPredecessors()去掉，公平锁将不再那么“谦让”，它将会像非公平锁那样抢占获取锁，抢占失败才会入队。若如此，公平锁将不再公平。

#### 释放锁
分析完了获取锁的相关逻辑，接下来再来分析一下释放锁的逻辑。与获取锁相比，释放锁的逻辑会简单一些，因为释放锁的过程没有公平和非公平之分。好了，下面开始分析 unlock 的逻辑：

```java
+--- ReentrantLock
public void unlock() {
    // 调用 AQS 中的 release 方法
    sync.release(1);
}

+--- AbstractQueuedSynchronizer
public final boolean release(int arg) {
    // 调用 ReentrantLock.Sync 中的 tryRelease 尝试释放锁
    if (tryRelease(arg)) {
        Node h = head;
        /*
         * 如果头结点的等待状态不为0，则应该唤醒头结点的后继节点。
         * 这里简单说个结论：
         *     头结点的等待状态为0，表示头节点的后继节点线程还是活跃的，无需唤醒
         */
        if (h != null && h.waitStatus != 0)
            // 唤醒头结点的后继节点，该方法的分析请参考我写的关于 AQS 的文章
            unparkSuccessor(h);
        return true;
    }
    return false;
}

+--- ReentrantLock.Sync
protected final boolean tryRelease(int releases) {
    /*
     * 用同步状态量 state 减去释放量 releases，得到本次释放锁后的同步状态量。
     * 当将 state 为 0，锁才能被完全释放
     */ 
    int c = getState() - releases;
    // 检测当前线程是否已经持有锁，仅允许持有锁的线程执行锁释放逻辑
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
        
    boolean free = false;
    // 如果 c 为0，则表示完全释放锁了，此时将持锁线程设为 null
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    
    // 设置新的同步状态
    setState(c);
    return free;
}
```

