# ArrayBlockingQueue

本节我们介绍ArrayBlockingQueue的实现原理，学完本节你将学会如何手动实现一个生产者-消费者队列，并对Java显式锁(ReentrantLock)的使用有深入理解。

## 前言

`ArrayBlockingQueue`是一种有界的`BlockingQueue`，常用于生产者-消费者模式，能够容纳指定个数的元素，当队列满时不能再放如新的元素，当队列为空时不能取出元素，此时相应的生产者或者消费者线程往往会挂起。本类是一种先进先出(FIFO)的队列结构，内部通过数组实现。来回顾一下`BlockingQueue`的常见接口：

<table width="600px"><tr><td></td><td>抛异常</td><td>返回特殊值</td><td>阻塞</td><td>阻塞直到超时</td></tr><tr><td>插入</td><td>add(e)</td><td>offer(e)</td><td>put(e)</td><td>offer(e, time, unit)</td></tr><tr><td>删除</td><td>remove()</td><td>poll()</td><td>take()</td><td>poll(time, unit)</td></tr><tr><td>查看头部</td><td>element()</td><td>peek()</td><td>无</td><td>无</td></tr><table>

我们着重关注阻塞的`put()`和`take()`方法的实现。

## 显式锁和内置锁

解读ArrayBlockingQueue源码之前我们有必要区分一下Java中的显式锁和内置锁。

### 内置锁

我们都知道java中的synchronized关键字，用该关键字修饰方法可以保证方法是同步的，用该关键字和一个对象来包裹一个代码块，可以保证该代码块是同步的，出现synchronized的地方就是使用内置锁的地方，内置锁使用起来非常方便，不需要显式的获取和释放，任何一个对象都能作为一把内置锁。使用内置锁能够解决大部分的同步场景。

```Java
// synchronized关键字用法示例
public synchronized void add(int t){// 同步方法
    this.v += t;
}
public int decrementAndGet(){
    synchronized(this){// 同步代码块
        return --v;
    }
}
```

### 显式锁

内置锁虽然好用，但它不可中断，不可定时，并且只有一个条件队列，有时候我们需要更灵活的获取锁机制，显式锁(RenentrantLock)应运而生。ReentrantLock是可重入的（线程可以同时多次请求同一把锁，而不会自己导致自己死锁）可定时、可中断并且支持多个条件队列。我们来分别解释这些概念。

- 可中断：你一定见过InterruptedException，很多跟多线程相关的方法会抛出该异常，这个异常并不是一个缺陷导致的负担，而是一种必须，或者说是一件好事。可中断性给我们提供了一种让线程提前结束的方式（而不是非得等到线程执行结束），这对于要取消耗时的任务非常有用。对于内置锁，线程拿不到内置锁就会一直等待，除了获取锁没有其他办法能够让其结束等待。`RenentrantLock.lockInterruptibly()`给我们提供了一种以中断结束等待的方式。

- 可定时：`RenentrantLock.tryLock(long timeout, TimeUnit unit)`提供了一种以定时结束等待的方式，如果线程在指定的时间内没有获得锁，该方法就会返回false并结束线程等待。

- 条件队列：线程在获取锁之后，可能会由于等待某个条件发生而进入等待状态（内置锁通过`Object.wait()`方法，显式锁通过`Condition.await()`方法），进入等待状态的线程会挂起并自动释放锁，这些线程会被放入到条件队列当中。synchronized对应的只有一个条件队列，而ReentrantLock可以有多个条件队列，多个队列有什么好处呢？请往下看。

- 条件谓词：线程在获取锁之后，有时候还需要等待某个条件满足才能做事情，比如生产者需要等到“缓存不满”才能往队列里放入消息，而消费者需要等到“缓存非空”才能从队列里取出消息。这些条件被称作条件谓词，线程需要先获取锁，然后判断条件谓词是否满足，如果不满足就不往下执行，相应的线程就会放弃执行权并自动释放锁。使用同一把锁的不同的线程可能有不同的条件谓词，如果只有一个条件队列，当某个条件谓词满足时就无法判断该唤醒条件队列里的哪一个线程；但是如果每个条件谓词都有一个单独的条件队列，当某个条件满足时我们就知道应该唤醒对应队列上的线程（内置锁通过`Object.notify()`或者`Object.notifyAll()`方法唤醒，显式锁通过`Condition.signal()`或者`Condition.signalAll()`方法唤醒）。这就是多个条件队列的好处。

使用内置锁时，对象本身既是一把锁又是一个条件队列；使用显式锁时，RenentrantLock的对象是锁，条件队列通过`RenentrantLock.newCondition()`方法获取，多次调用该方法可以得到多个条件队列。

一个使用显式锁的典型示例如下：

```Java
// 显式锁的使用示例
ReentrantLock lock = new ReentrantLock();

lock.lock();
try{
    // do something
}finally{
    lock.unlock();
}
```

注意，上述代码将`unlock()`放在finally块里，这么做是必需的。显式锁不像内置锁那样会自动释放，使用显式锁一定要手动释放，如果获取锁后由于异常的原因没有释放锁，那么这把锁将永远得不到释放！所以要将unlock()放在finally块中，保证无论发生什么都能够正常释放。

## ArrayBlockingQueue.put()

有了上面的知识，理解阻塞队列的代码就变的很简单。

`put(E e)`方法会以阻塞的方式向队列尾部放入元素，如果队列缓存不满就立即放入，否则挂起等待直到缓存不满，这里的谓词就是“缓存不满”，这是生产者要调用的方法。该方法的具体代码如下：

```Java
final ReentrantLock lock = new ReentrantLock();// 显式锁对象
private final Condition notFull = lock.newCondition();// put()方法的条件队列
private final Condition notEmpty = lock.newCondition();// take()方法的条件队列
...

public void put(E e) throws InterruptedException {
	...
    lock.lockInterruptibly();
    try {
        while (count == items.length)// 条件谓词“缓存不满”
            notFull.await();// 挂起等待，直到缓存非满
        items[putIndex] = e;// 将元素放入缓存
        putIndex = inc(putIndex);
        ++count;
        notEmpty.signal();// 唤醒消费者线程
    } finally {
        lock.unlock();
    }
}
```

上述代码首先创建了一个可重入锁，并通过调用两次`newCondition()`方法得到两个跟这把锁相关的条件队列，这两个条件队列分别对用生产者队列和消费者队列。

put()方法的代码中首先以可中断的方式获取锁，之后在谓词“缓存不满”上等待，如果队列满了，就调用`notFull.await()`挂起当前线程并释放锁，这里说的释放锁是`await()`方法带来的效果，不是指最后finally代码块中的`unlock()`。当缓存不满的条件满足时，会将元素放到缓存当中，并调用`notEmpty.signal()`方法唤醒一个消费者线程。

代码中`notFull.await()`被放在了一个`while`循环而不是`if`语句中，这么做也是必需的。因为线程从`await()`语句中倍唤醒时，不一定意味着自己的条件谓词一定成立，有很多原因可以导致一个等待的线程被唤醒，条件谓词被满足只是其中一个。

## ArrayBlockingQueue.take()

`take()`方法是以阻塞的方式获取队列首部的元素，不弱队列缓存非空就立即取出，否则挂起等待直到队列非空，这里的谓词是“缓存非空”，这是消费者调用的方法。该方法具体代码如下：


```Java
final ReentrantLock lock = new ReentrantLock();// 显式锁对象
private final Condition notEmpty = lock.newCondition();// take()方法的条件队列
private final Condition notFull = lock.newCondition();// put()方法的条件队列
...

public E take() throws InterruptedException {
    lock.lockInterruptibly();
    try {
        while (count == 0)// 条件谓词“缓存非空”
            notEmpty.await();// 挂起等待，直到缓存非空
        E x = (E)items[takeIndex];// 取出元素
        items[takeIndex] = null;
        takeIndex = inc(takeIndex);
        --count;
        notFull.signal();// 唤醒生产者线程
        return x;
    } finally {
        lock.unlock();
    }
}
```

上述`take()`方法首先以可中断的方式获取锁，之后在谓词“缓存非空”上等待，如果队列为空，就掉用`notEmpty.await()`挂起当前线程并释放锁，当等待条件满足时，会从缓存中取出一个元素，并调用`notFull.signal()`唤醒一个生产者线程。

理解了`put()`和`take()`方法的实现，也就理解了实现生产者-消费者模式的精髓。

## 源码说明

本文采用的是JDK 1.7u79的源码，[下载地址](http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html)。[这里复制了一份](../source/src.zip)。
