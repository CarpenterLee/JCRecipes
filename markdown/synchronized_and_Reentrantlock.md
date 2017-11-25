# synchronized and Reentrantlock

多线程编程中，当代码需要同步时我们会用到锁。Java为我们提供了内置锁(`synchronized`)和显式锁(`ReentrantLock`)两种同步方式。显式锁是JDK1.5引入的，这两种锁有什么异同呢？是仅仅增加了一种选择还是另有其因？本文为您一探究竟。

## 内置锁

Java内置锁通过synchronized关键字使用，使用其修饰方法或者代码块，就能保证*方法*或者*代码块*以同步方式执行。使用起来非常近简单，就像下面这样：

```Java
// synchronized关键字用法示例
public synchronized void add(int t){// 同步方法
    this.v += t;
}

public static synchronized void sub(int t){// 同步静态方法
    value -= t;
}
public int decrementAndGet(){
    synchronized(obj){// 同步代码块
        return --v;
    }
}
```

这就是内置锁的全部用法，你已经学会了。

内置锁使用起来非常方便，不需要显式的获取和释放，任何一个对象都能作为一把内置锁。使用内置锁能够解决大部分的同步场景。*“任何一个对象都能作为一把内置锁”*也意味着出现synchronized关键字的地方，都有一个对象与之关联，具体说来：
 
- 当synchronized作用于普通方法是，锁对象是this；
- 当synchronized作用于静态方法是，锁对象是当前类的Class对象；
- 当synchronized作用于代码块时，锁对象是synchronized(obj)中的这个obj。

## 显式锁

内置锁这么好用，为什么还需多出一个显式锁呢？因为有些事情内置锁是做不了的，比如：

1. 我们想给锁加个等待时间超时时间，超时还未获得锁就放弃，不至于无限等下去；
2. 我们想以可中断的方式获取锁，这样外部线程给我们发一个中断信号就能唤起等待锁的线程；
3. 我们想为锁维持多个等待队列，比如一个生产者队列，一个消费者队列，一边提高锁的效率。

显式锁(ReentrantLock)正式为了解决这些灵活需求而生。ReentrantLock的字面意思是*可重入锁*，可重入的意思是*线程可以同时多次请求同一把锁，而不会自己导致自己死锁*。下面是内置锁和显式锁的区别：


- 可定时：`RenentrantLock.tryLock(long timeout, TimeUnit unit)`提供了一种以定时结束等待的方式，如果线程在指定的时间内没有获得锁，该方法就会返回false并结束线程等待。

- 可中断：你一定见过InterruptedException，很多跟多线程相关的方法会抛出该异常，这个异常并不是一个缺陷导致的负担，而是一种必须，或者说是一件好事。可中断性给我们提供了一种让线程提前结束的方式（而不是非得等到线程执行结束），这对于要取消耗时的任务非常有用。对于内置锁，线程拿不到内置锁就会一直等待，除了获取锁没有其他办法能够让其结束等待。`RenentrantLock.lockInterruptibly()`给我们提供了一种以中断结束等待的方式。

- 条件队列(condition queue)：线程在获取锁之后，可能会由于等待某个条件发生而进入等待状态（内置锁通过`Object.wait()`方法，显式锁通过`Condition.await()`方法），进入等待状态的线程会挂起并自动释放锁，这些线程会被放入到条件队列当中。synchronized对应的只有一个条件队列，而ReentrantLock可以有多个条件队列，多个队列有什么好处呢？请往下看。

- 条件谓词：线程在获取锁之后，有时候还需要等待某个条件满足才能做事情，比如生产者需要等到“缓存不满”才能往队列里放入消息，而消费者需要等到“缓存非空”才能从队列里取出消息。这些条件被称作条件谓词，线程需要先获取锁，然后判断条件谓词是否满足，如果不满足就不往下执行，相应的线程就会放弃执行权并自动释放锁。使用同一把锁的不同的线程可能有不同的条件谓词，如果只有一个条件队列，当某个条件谓词满足时就无法判断该唤醒条件队列里的哪一个线程；但是如果每个条件谓词都有一个单独的条件队列，当某个条件满足时我们就知道应该唤醒对应队列上的线程（内置锁通过`Object.notify()`或者`Object.notifyAll()`方法唤醒，显式锁通过`Condition.signal()`或者`Condition.signalAll()`方法唤醒）。这就是多个条件队列的好处。

使用内置锁时，对象本身既是一把锁又是一个条件队列；使用显式锁时，RenentrantLock的对象是锁，条件队列通过`RenentrantLock.newCondition()`方法获取，多次调用该方法可以得到多个条件队列。

一个使用显式锁的典型示例如下：

```Java
// 显式锁的使用示例
ReentrantLock lock = new ReentrantLock();

// 获取锁，这是跟synchronized关键字对应的用法。
lock.lock();
try{
    // your code
}finally{
    lock.unlock();
}

// 可定时，超过指定时间为得到锁就放弃
try {
    lock.tryLock(10, TimeUnit.SECONDS);
    try {
        // your code
    }finally {
        lock.unlock();
    }
} catch (InterruptedException e1) {
    // exception handling
}

// 可中断，等待获取锁的过程中线程线程可被中断
try {
    lock.lockInterruptibly();
    try {
        // your code
    }finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    // exception handling
}

// 多个等待队列，具体参考[ArrayBlockingQueue](https://github.com/CarpenterLee/JCRecipes/blob/master/markdown/ArrayBlockingQueue.md)
/** Condition for waiting takes */
private final Condition notEmpty = lock.newCondition();
/** Condition for waiting puts */
private final Condition notFull = lock.newCondition();

```

注意，上述代码将`unlock()`放在finally块里，这么做是必需的。<strong>显式锁不像内置锁那样会自动释放，使用显式锁一定要在finally块中手动释放</strong>，如果获取锁后由于异常的原因没有释放锁，那么这把锁将永远得不到释放！将unlock()放在finally块中，保证无论发生什么都能够正常释放。

## 结论

内置锁能够解决大部分需要同步的场景，只有在需要额外灵活性是才需要考虑显式锁，比如可定时、可中断、多等待队列等特性。

显式锁虽然灵活，但是需要显式的申请和释放，并且<strong>释放一定要放到finally块中，否则可能会因为异常导致锁永远无法释放！</strong>这是显式锁最明显的缺点。

综上，当需要同步时请优先考虑更安全的更易用的隐式锁。

## 参考文献

https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantLock.html