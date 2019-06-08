---
title: ReentantReadWriteLock(读写锁)之学习笔记
date: 2018-02-20 17:32:00
tags: java
---

### ReadWriteLock(读写锁)简介
ReentrantLock是一种排他锁, ReadWriteLock的读锁是一种共享锁, 允许多个线程同时持有该锁。
ReadWriteLock是一个接口, 内部只定义了两个方法。
<!--more-->

``` java

	public interface ReadWriteLock {
   
	    Lock readLock();
	
	    Lock writeLock();
	}

```
ReentantReadWriteLock是ReadWriteLock的一个实现类, 内部维护了两把锁, 即读锁和写锁。允许多个线程同时获取读锁, 但是读写是互斥的, 读锁是共享锁, 写锁是排它锁。总的说来就是读读共享, 读写互斥, 写读互斥。但是还有点要注意下, **如果是当前线程获取的写锁, 它还能继续获取读锁(可重入), 其它线程就不能在写锁被获取的状态下再取获取读锁。如果当前线程获取的是读锁, 它和其它线程都不能获取写锁。**

### 实现原理分析
与ReentrantLock一样, 有Sync继承AQS, 以及Sync的实现类FairSync, NonfairSync。同样也是用AQS的state变量作为同步状态的表示。state是一个int类型变量也就是共32位。将低16位表示写锁的值, 高16位表示成读锁的值。那么只有state = 0表示锁没有被获取(包括读锁和写锁)<br/>
则: <br/>
**读状态(readCount) = state >>> 16; ** <br/>
**写状态(writeCount) = state & 65535。**<br/>
**获取读锁: state = state + 65536; 高16位加1**<br/>
**释放读锁: state = state - 65536; 高16位减1**<br/>
**获取写锁: state = state + 1;** <br/>
**释放写锁: state = state - 1;** <br/>
65535 = 2^16 - 1, 二进制表示也就是16个1, 这样与运算就能得到state的低16位了。<br/>
对应源码如下:<br/>
``` java

	abstract static class Sync extends AbstractQueuedSynchronizer {

        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /** Returns the number of shared holds represented in count  */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** Returns the number of exclusive holds represented in count  */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
	}

```

继续从读锁入手, 分析下AQS的共享操作如何实现的。<br/>
``` java
	
	//读锁ReadLock: lock()调用AQS的acquireShared
	public void lock() {
		sync.acquireShared(1);
	}

	//AQS: acquireShared方法调用tryAcquireShared方法
	public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

	//子类Sync实现
	protected final int tryAcquireShared(int unused) {
		Thread current = Thread.currentThread();
		int c = getState();
		//如果不是当前线程获取到写锁, 则失败
		if (exclusiveCount(c) != 0 &&
		    getExclusiveOwnerThread() != current)
		    return -1;

		// readCount
		int r = sharedCount(c);
		//涉及公平与否: 如果不需要被阻塞, 并且还没有超过获取锁的最大数量
		if (!readerShouldBlock() &&
		    r < MAX_COUNT &&
			// 更新state, 即state = state + 65536
		    compareAndSetState(c, c + SHARED_UNIT)) {
			//走到这里说明获取读锁成功, 下面就是记录信息
		    if (r == 0) {
				//说明锁是第一次被获取, 将当前线程赋给firstReader
		        firstReader = current;
		        firstReaderHoldCount = 1;
		    } else if (firstReader == current) {
				//记录第一次获取读锁的线程获取该锁的次数
		        firstReaderHoldCount++;
		    } else {
		        HoldCounter rh = cachedHoldCounter;
		        if (rh == null || rh.tid != getThreadId(current))
		            cachedHoldCounter = rh = readHolds.get();
		        else if (rh.count == 0)
		            readHolds.set(rh);
		        rh.count++;
		    }
			//返回成功
		    return 1;
		}
		//走到这里是那些同时竞争compareAndSetState修改失败的, 自旋获取锁
		return fullTryAcquireShared(current);
	}

```
总结下tryAcquireShared方法的主要逻辑:<br/>
1. 如果其它线程获取写锁, return -1; 否则->2 <br/>
2. 写锁没有被获取或者当前线程是写锁的拥有者, 有权利去获取读锁。->3 <br/>
3. 根据公平策略判断是否要将当前线程先暂时阻塞, 不要则去竞争修改state, 抢到锁则return 1, 否则->4 <br/>
4. 调用fullTryAcquireShared自旋竞争获取锁 <br/>
对fullTryAcquireShared的理解: 当多个线程同时去竞争获取读锁, 那么只能有一个线程能够执行CAS操作成功, 但是失败的线程还是可以获取读锁(共享)的, 因此进入一个自旋的循环, 当然在循环过程中如果读锁被释放了, 并且有线程率先在这个空隙下获得写锁, 那么这个线程就返回-1, 否则那么执行CAS操作获取锁, 成功就返回1, 失败继续循环。

如果tryAcquireShared返回-1(失败), 则调用doAcquireShared方法<br/>

``` java

	private void doAcquireShared(int arg) {
		//将当前线程构造成共享模式的结点, 加入到队列尾部
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
			//自旋可能获取到锁, 可能被挂起
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
					//队列里的第一个等待的结点, 有权利获取同步状态
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
						//成功修改头结点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
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

大部分逻辑与独占模式用到的acquireQueued()方法一样, 主要区别就是修改头结点那句。

``` java
	
	private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head;
		//修改头结点
        setHead(node);
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
			//获取当前结点的下个结点, 如果是共享的, 调用doReleaseShared
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }

	//持续唤醒队列中等待的线程
	private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
				//Node.SIGNAL, 说明其后面的线程被park()了, 唤醒它
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    //唤醒线程
					unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
			// h == head, 那么说明被唤醒的线程没有抢到同步状态, 退出循环, 不再唤醒
            if (h == head)                   // loop if head changed
                break;
        }
    }

```

**通过源码可以看到独占式功能与共享式功能实现最大的区别就是处于同步队列的第一个结点线程在获取到同步状态后所做的操作不同:**<br/>
**1. 独占式: 只是将自己修改成头结点。**<br/>
**2. 共享式: 不仅将自己修改成头结点, 而且如果后继结点是共享模式还会调用doReleaseShared方法唤醒后面处于等待的线程, 让它们也去获取同步状态。**

共享式释放读锁:<br/>
``` java

	protected final boolean tryReleaseShared(int unused) {
		Thread current = Thread.currentThread();
		if (firstReader == current) {
		    // assert firstReaderHoldCount > 0;
		    if (firstReaderHoldCount == 1)
		        firstReader = null;
		    else
		        firstReaderHoldCount--;
		} else {
		    HoldCounter rh = cachedHoldCounter;
		    if (rh == null || rh.tid != getThreadId(current))
		        rh = readHolds.get();
		    int count = rh.count;
		    if (count <= 1) {
		        readHolds.remove();
		        if (count <= 0)
		            throw unmatchedUnlockException();
		    }
		    --rh.count;
		}
		for (;;) {
			//循环更新同步状态
		    int c = getState();
			// state = state - 65536;
		    int nextc = c - SHARED_UNIT;
		    if (compareAndSetState(c, nextc))
		        // Releasing the read lock has no effect on readers,
		        // but it may allow waiting writers to proceed if
		        // both read and write locks are now free.
		        return nextc == 0;
		}
	}

```
主要逻辑:<br/>
** 1. 通过CAS操作修改同步状态, 当读锁全部释放完毕返回true, 否则返回false;**<br/>
** 2. 返回成功, 则会调用doReleaseShared方法唤醒后面的一个线程** <br/>


再看独占式写锁tryAcquire()逻辑<br/>
``` java

	protected final boolean tryAcquire(int acquires) {
	
		Thread current = Thread.currentThread();
		int c = getState();
		int w = exclusiveCount(c);
		//c不为0, 即有线程获取过锁(read/write)
		if (c != 0) {
			//w == 0, 即读锁已被获取, 失败
			//写锁被获取, 但是不是本线程, 也失败
			if (w == 0 || current != getExclusiveOwnerThread())
				return false;
			//超出数量
			if (w + exclusiveCount(acquires) > MAX_COUNT)
				throw new Error("Maximum lock count exceeded");
			// Reentrant acquire
			//可重入支持, 修改state = state + 1;
			setState(c + acquires);
			return true;
		}
		//c = 0, 读和写都没有被获取过
		if (writerShouldBlock() ||!compareAndSetState(c, c + acquires))
			return false;
		setExclusiveOwnerThread(current);
		return true;
	}

```
总结下主要流程:<br/>
**1. 如果当前读锁已经被获取, 则不能获取写锁->失败** <br/>
**2. 如果当前写锁已经被获取, 但是不是当前前程->失败, 是当前线程可重入->成功** <br/>
**3. 读写都没有获取过, CAS操作修改state, 修改失败, 其它线程获取到写锁了->失败, 修改成功->成功**<br/> 

注: writerShouldBlock()不影响主要逻辑, 已忽略, 涉及公平策略, 判断是否要将当前线程放入同步队列中。

独占式释放写锁<br/>
``` java

	protected final boolean tryRelease(int releases) {
		if (!isHeldExclusively())
		    throw new IllegalMonitorStateException();
		int nextc = getState() - releases;
		boolean free = exclusiveCount(nextc) == 0;
		if (free)
		    setExclusiveOwnerThread(null);
		setState(nextc);
		return free;
	}

```
主要逻辑:<br/>
** 1. 如果不是当前写锁的拥有者, 抛异常. ** <br/>
** 2. 更新同步状态(减1), 如果等于0, 则释放锁成功, 不等于0, 说明可重入了. ** <br/>

### 总结
本文通过ReentantReadWriteLock类的实现分析了AQS共享模式的原理, 上一篇文章通过ReentrantLock类的实现分析了AQS独占模式的原理。更加深刻的理解了AQS框架的强大之处。

### 参考
1. 并发编程的艺术
2. JDK 1.8 源码