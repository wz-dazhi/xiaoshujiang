---
title: ReentrantLock原理
tags: java,lock,
category: /java/J.U.C /2022-05
renderNumberedHeading: true
grammar_cjkRuby: true
---
2022-05-11 13:55

# 定义
ReentrantLock是一种可重入的互斥锁（也叫排它锁）， 在多个线程的情况下， 同一时刻只有一个线程可以获取锁并执行操作。ReentrantLock主要实现是由内部Sync继承AQS实现加锁逻辑，Sync又分为非公平锁和公平锁。非公平锁和公平锁主要的区别在于要不要看。假如多个线程同一时刻都在竞争锁，非公平锁上来就直接CAS抢锁，如果抢不到则加入AQS实现的阻塞队列中等待被唤醒。公平锁上来如果别的线程持有锁，那么当前线程直接加入AQS阻塞队列。总结：非公平锁不看有没有别的线程，直接抢锁。公平锁先看看有没有别的线程，如果有就排队。

# 食用方法

```java
class X {
    // 创建一个ReentrantLock对象
    private final ReentrantLock lock = new ReentrantLock();
    
    public void m() {
        // 获取锁， 如果获取不到一直阻塞
        lock.lock();
        try {
          // 业务逻辑
        } finally {
          // 解锁
          lock.unlock()
        }
    }
}
```

# 类结构
## 整体定义
```java
public class ReentrantLock implements Lock, java.io.Serializable {
    // 同步器Sync
    private final Sync sync;
    
    // 内部同步器Sync继承了AQS
    abstract static class Sync extends AbstractQueuedSynchronizer {}

    // 非公平同步器的定义
    static final class NonfairSync extends Sync {}
    
    // 公平同步器的定义
    static final class FairSync extends Sync {}
}
```

## 构造器
```java
// 默认构造器，创建非公平锁实现
public ReentrantLock() {
    sync = new NonfairSync();
}

// fair为true创建公平锁实现， fair为false创建非公平锁实现
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

## Sync核心原理
### lock方法
由FairSync和NonfairSync实现具体加锁逻辑
```java
abstract void lock();
```
### tryRelease释放锁逻辑
公共释放锁逻辑。当调用unlock()时，会调用AQS release方法，release方法会调用当前tryRelease方法
```java
public void unlock() {
    sync.release(1);
}

protected final boolean tryRelease(int releases) {
    // 获取state变量减去释放数量
    int c = getState() - releases;
    // 判断当前线程不持有锁状态，抛出异常（没获取到锁，释放什么锁）
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    // c等于0表示释放成功并且表示当前线程没有锁重入，将排它锁线程设置为null
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    // 设置state变量，由AQS调用
    setState(c);
    // 返回free, 告诉AQS是否需要唤醒排队线程。如果返回true，AQS会唤醒后面阻塞的线程
    return free;
}
```
### nonfairTryAcquire非公平锁获取逻辑
nonfairTryAcquire方法由NonfairSync调用，但是也属于Sync里面的方法（Doug Lea大爷的写法琢磨不透）
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // 获取state
    int c = getState();
    // 如果c=0,直接CAS抢锁操作，抢锁成功将当前线程设置为持有排它锁
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 判断当前线程是否等于持有排它锁线程（线程重入）
    else if (current == getExclusiveOwnerThread()) {
        // 重入次数++
        int nextc = c + acquires;
        // 重入次数溢出抛出异常
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        // 设置重入次数
        setState(nextc);
        return true;
    }
    // 返回false，获取锁失败。告诉AQS去排队
    return false;
}
```

## FairSync公平锁
### lock方法
直接调用AQS acquire方法，acquire方法会调用tryAcquire
```java
final void lock() {
    acquire(1);
}
```
### tryAcquire方法
尝试获取锁，如果获取失败加入到AQS阻塞队列中进行排队等待。
```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // 获取state
    int c = getState();
    if (c == 0) {
        // c等于0。判断队列中是否存在排队的线程（如果在这一步之前已经有别的线程竞争锁成功了，并且队列中存在等待线程），如果没有排队的线程，直接CAS state，如果设置成功将当前线程设置为持有锁线程
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 判断当前线程是否等于持有线程（锁重入）
    else if (current == getExclusiveOwnerThread()) {
        // 锁重入++
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        // 重新设置state
        setState(nextc);
        return true;
    }
    // 告诉AQS获取锁失败
    return false;
}
```

## NonfairSync非公众平锁
### lock方法
上来直接CAS抢锁，抢锁成功，将当前线程设置为锁持有线程。获取失败调用AQS acquire方法，acquire方法会调用tryAcquire
```java
final void lock() {
    // 直接CAS抢锁
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        // 调用AQS acquire
        acquire(1);
}
```
### tryAcquire方法
tryAcquire直接调用Sync父类方法nonfairTryAcquire
```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

# 总结
ReentrantLock实现了Lock接口，实现了lock unlock等方法。具体的加锁释放锁逻辑又由内部Sync同步器继承AQS实现，AQS提供加锁解锁模板方法，Sync实现了模板方法的加锁解锁逻辑，大部分复用了AQS的相关代码。