## Java中的Lock接口

### Synchronized & Lock

* synchronized 是Java语言中的关键字，由monitorenter，monitorexit两个指令实现。JVM会将monitorenter指定插在同步代码块开始的地方，将monitorexit指定插在同步代码快结束和出现异常的地方。

* Lock是JUC包下的组件, 是基于AQS(队列同步器)实现的。

* synchronized功能与ReentrantLock类相对应, 都是可重入的锁。

* Lock与synchronized关键字相比，实现了公平锁和非公平锁，synchronized关键字是非公平的。同时Lock接口提供了在获取锁被阻塞时可响应中断以及超时获取锁的API。

* Lock接口可以绑定多个条件，即绑定多个 Condition 对象，这样唤醒时可以唤醒指定条件上的线程。如生产者消费者例子，使用 synchronized 关键字时，当生产者生产消息时，需要唤醒消费者线程，我们只能调用notifyAll 方法，这样不仅会唤醒消费者端的线程而且还会唤醒生产者自己这一端的线程。

**注： **

1. 公平锁是指多个线程等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁，而非公平锁不保证这一点。在锁被释放时，任何一个等待锁的线程都有机会获取锁。
2. ReentrantLock实现的非公平锁只能保证阻塞队列里最早等待的线程与新来的线程竞争抢锁, 对于阻塞队列其它的线程依然需要等待，因为队列是先进先出的，只有该线程获取锁然后释放锁后后续节点才有资格竞争锁。

### Lock 接口的 API

```java
public interface Lock {
    
    // 获取锁直到获取锁成功后返回，否则被阻塞，即使被中断也不会返回
    void lock();
    
    // 获取锁并且响应中断，注意如果t线程因为竞争锁失败而被阻塞，另外一个线程中断了t线程
    // 那么t线程会被唤醒，并且抛出中断异常，清除中断状态, 阻塞线程的方法是调用LockSupport.park()
    // 当调用了LockSupport.unpark或者中断了线程，线程会从park方法中唤醒。
    void lockInterruptibly() throws InterruptedException;
    
    // 尝试获取锁，不管成功与否都返回
    boolean tryLock();
    
    // 超时获取锁，获取锁失败而返回的原因如下：
    // 1. 重新竞争到锁
    // 2. 超时时间过了
    // 3. 线程被中断，抛出中断异常，清除中断状态
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    
    // 释放锁，如果当前线程没有获取到锁，调用此方法将会抛出IllegalMonitorStateException
    void unlock();
    
    // 创建Condition对象，提供了类似synchronized 所对象的wait, notify， notifyAll方法
    Condition newCondition();
}

public interface Condition {
    
    // 使获取锁的线程阻塞，并且释放锁，直到另一个线程中断了此线程或者调用了signal，signalAll方法
    // 没有获取锁的线程调用此方法会抛出IllegalMonitorStateException
    void await() throws InterruptedException;
    
    // 使获取的线程阻塞，并且释放锁，但是不响应中断请求，只有调用了signal，signalAll才会被唤醒
    // 其实是线程中断后醒过来，再次被阻塞-LockSupport.park()
    void awaitUninterruptibly();
    
    // 使获取的线程阻塞，并且释放锁, 被唤醒的原因如下:
    // 1. 超时时间已过  2. 线程被中断   3. 有线程调用了signal，signalAll方法
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    
    // 唤醒最开始等待在条件队列上的线程
    void signal();
    
    // 唤醒所有等待在条件队列上的线程 
    void signalAll();
    
}

```

**注: 所有从await中醒过来的线程只有重新获取到锁才能往下执行，否则依然会在同步队列中等待获取锁**

### Lock的使用

下面将使用ReentrantLock实现一个简单的阻塞队列。

```java
public class BlockQueue {
    private Lock lock = new ReentrantLock();

    private Condition full = lock.newCondition();

    private Condition empty = lock.newCondition();

    private Queue<String> queue;
    
    private int capacity;

    public BlockQueue(int capacity) {
        queue = new ArrayDeque<>(capacity);
        this.capacity = capacity;
    }
    
    public void put(String element) {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                try {
                    full.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            queue.add(element);
            empty.signalAll();
        } finally {
            lock.unlock();
        }
    }
    
    public String take() {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                try {
                    empty.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            String element = queue.remove();
            full.signalAll();
            return element;
        } finally {
            lock.unlock();
        }
    }
}
```

### 测试Lock接口的方法

```java
public class LockTest {

    private final Object lock = new Object();

    /**
     * 测试synchronized关键字
     * synchronized: 当线程在获取锁时, 竞争失败的线程会处于阻塞, 并且不响应中断直至获取到锁
     * 
     * 程序结果:
     * --------------主线程准备释放锁---------------------
     * 是否被中断: true
     * block-thread获取锁成功
     */
    @Test
    public void testSynchronized() {
        // 当前线程持有锁
        synchronized (lock) {
            // 开启另外一个线程尝试获取锁
            Thread t = new Thread(()->{
                //阻塞
                synchronized (lock) {
                    System.out.println("是否被中断: " + Thread.currentThread()
                                       .isInterrupted());
                    System.out.println(Thread.currentThread().getName() + "获取锁成功");
                }
            }, "block-thread");
            t.start();

            //休眠一秒, 让开启的线程充分运行, 接着进行中断
            sleep(1);
            t.interrupt();

            //长久睡眠不释放锁, 用来观察结果, 看看线程t是否会响应中断
            sleep(10);
            System.out.println("--------------主线程准备释放锁---------------------");
        }
    }

    /**
     * 测试lock.lock()同synchronized关键字
     *
     * 程序结果:
     * --------------主线程准备释放锁---------------------
     * 是否被中断: true
     * block-thread获取锁成功
     */
    @Test
    public void testLock() {
        //当前线程持有锁
        Lock lock = new ReentrantLock();
        lock.lock();
        try {
            // 开启另外一个线程尝试获取锁
            Thread t = new Thread(()->{
                //阻塞
                lock.lock();
                try {
                    System.out.println("是否被中断: " + Thread.currentThread()
                                       .isInterrupted());
                    System.out.println(Thread.currentThread().getName() + "获取锁成功");
                } finally {
                    lock.unlock();
                }

            }, "block-thread");
            t.start();

            //休眠一秒, 让开启的线程充分运行, 接着进行中断
            sleep(1);
            t.interrupt();

            //长久睡眠不释放锁, 用来观察结果, 看看线程t是否会响应中断
            sleep(10);
            System.out.println("--------------主线程准备释放锁---------------------");
        } finally {
            lock.unlock();
        }
    }

    /**
     * 测试lock.lockInterruptibly(): 抛出中断异常
     * 当线程被中断后，线程会从LockSupport.park()中醒过来，然后会检查自己是否被中断，如果被中断过
     * 则抛出中断异常，清除中断状态。
     * 
     * 程序结果: block-thread is interrupted! exit
     */
    @Test
    public void testlockInterruptibly() {
        // 当前线程持有锁
        ReentrantLock lock = new ReentrantLock();
        try {
            lock.lock();
            // 开启另外一个线程尝试获取锁
            Thread t = new Thread(()->{
                //阻塞
                try {
                    lock.lockInterruptibly();
                    System.out.println(Thread.currentThread().getName() + "获取锁成功");
                } catch (InterruptedException e) {
                    System.out.println(Thread.currentThread().getName() + 
                                       "is interrupted! exit");
                } finally {
                    if(lock.isHeldByCurrentThread())
                        lock.unlock();
                }

            }, "block-thread");
            t.start();

            //休眠一秒, 让开启的线程充分运行, 接着进行中断
            sleep(1);
            t.interrupt();

            //长久睡眠不释放锁, 用来观察结果, 看看线程t是否会相应中断
            sleep(10);
        } finally {
            lock.unlock();
        }
    }

    private void sleep(int second) {
        try {
            TimeUnit.SECONDS.sleep(second);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

