# Atomics

`java.util.concurrent.atomic`包定义了一些常见类型的原子变量。这些原子变量为我们提供了一种操作单一变量无锁(*lock-free*)的线程安全(*thread-safe*)方式。实际上该包下面的类为我们提供了类似`volatile`变量的特性，同时还提供了诸如`boolean compareAndSet(expectedValue, updateValue)`的功能。不使用锁实现线程安全，这听起来似乎很不可思议，这其实是通过CPU的compare and swap指令实现的，由于硬件指令支持，当然不需要加锁了。

先不去讨论这些细节，我们来看一下原子变量的用法。一个典型的用法是可以使用原子变量轻松实现全局自增id，就像下面这样：

```Java
AtomicLong atomicLong = new AtomicLong();
long id = atomicLong.getAndIncrement();// 线程安全，可在多线程下使用。
```

用起来非常简单直观，下面我们给出每种原子变量类型的用法说明。

## AtomicInteger and AtomicLong

*AtomicInteger*和*AtomicLong*分别代表原子类型的整型和长整型，这两个类提供十分相似的功能，仅仅是位宽不同。如上例所示，原子整型可用于多线程下全局自增id，除此之外还提供了原子*比较-赋值*等操作，诸如`compareAndSet(expect, update)`， `decrementAndGet()`，`getAndDecrement()`，`getAndSet(newValue)`等等，更全面的接口描述可参考JDK文档。需要提醒的是这些函数都是通过原子CPU指令实现，执行效率较高。

原子整型看似跟普通整型(*Integer, Long*)类型相似，但不能使用原子整型替代普通整型，因为原子整型是可变的，而普通整型不可变。由于这个原因，使用原子整型作为Map的key并不是个好主意。

你可能会想当然的以为应该有*AtomicFloat*和*AtomicDouble*，遗憾的是类库里并没有这两个类型，*AtomicByte*和*AtomicShort*也没有。如果需要替代方案是使用*AtomicInteger*和*AtomicLong*。可通过`Float.floatToRawIntBits(float)`和`Float.intBitsToFloat(int)`将Float存储到*AtomicInteger*中，类似的Double类型也可以存储到*AtomicLong*中。

## AtomicReference




