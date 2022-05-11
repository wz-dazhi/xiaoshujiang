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

``` javascript
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

## Sync

