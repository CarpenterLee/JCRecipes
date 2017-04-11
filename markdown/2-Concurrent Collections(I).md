# Concurrent Collections(I)

生产者-消费者模型最简单的实现方式是什么？使用Java语言中的`BlockingQueue`是最简单有效的实现方式。本节我们将对Java并发容器给出介绍，完成我们在[《深入理解Java集合框架》](https://github.com/CarpenterLee/JCFInternals)系列文章中未竟的内容。

## BlockingQueue

`BlockingQueue`是一个阻塞队列接口，所谓阻塞队列就是在添加元素或获取元素时，线程会阻塞等待直到队列不满或者非空。该接口常见实现类有`ArrayBlockingQueue`，`LinkedBlockingQueue`和`PriorityBlockingQueue`等，前两个分别是依靠数组和链表实现的阻塞队列，后一个是阻塞的优先队列。关于[数组](https://github.com/CarpenterLee/JCFInternals/blob/master/markdown/2-ArrayList.md)、[链表](https://github.com/CarpenterLee/JCFInternals/blob/master/markdown/3-LinkedList.md)以及[优先队列](https://github.com/CarpenterLee/JCFInternals/blob/master/markdown/8-PriorityQueue.md)数据结构方面的知识，[《深入理解Java集合框架》](https://github.com/CarpenterLee/JCFInternals)系列文章已经讲解的非常清楚，不再重复。此处主要考察阻塞队列的特点和用法，阻塞队列常见的接口方法如下表，不同方法对特殊情况的处理方式不同：

<table width="600px"><tr><td></td><td>抛异常</td><td>返回特殊值</td><td>阻塞</td><td>阻塞直到超时</td></tr><tr><td>插入</td><td>add(e)</td><td>offer(e)</td><td>`put(e)`</td><td>`offer(e, time, unit)`</td></tr><tr><td>删除</td><td>remove()</td><td>poll()</td><td>`take()`</td><td>`poll(time, unit)`</td></tr><tr><td>查看头部</td><td>element()</td><td>peek()</td><td>无</td><td>无</td></tr><table>

上表中后两列方法使用较多，因为我们使用阻塞队列显然是想发挥其阻塞的特性。

BlockingQueue是线程安全的，常用于生产者-消费者模式的线程间数据共享。通常队列都会有一个固定大小，能够乘放指定个数个元素。当队列空间占满时，生产者将会挂起直到队列不满；当队列为空时，消费者将会挂起知道队列非空。一个简单而实用的例子如下：

```Java
// 使用BlockingQueue实现生产者-消费者模式
public class ProducerConsumer {
	public static void main(String[] args) {
		BlockingQueue<String> queue = new LinkedBlockingQueue<>(16);// 固定容量为16的阻塞队列
		new Producer<String>(queue).start();// Producer 1
		new Producer<String>(queue).start();// Producer 2
		new Consumer<String>(queue).start();// Consumer
	}
	static class Producer<E> extends Thread{
		private BlockingQueue<E> queue;
		public Producer(BlockingQueue<E> queue){ this.queue = queue; }
		@Override
		public void run(){//不停生产元素，添加到共享队列当中
			while(true){
				try {
					E e = produce();
					queue.put(e);// 挂起，直到队列非满
				} catch (InterruptedException e1) { /* exception */ }
			}
		}
		protected E produce(){return (E)String.valueOf(System.nanoTime());}
	}
	static class Consumer<E> extends Thread{
		private BlockingQueue<E> queue;
		public Consumer(BlockingQueue<E> queue){ this.queue = queue; }
		@Override
		public void run(){//不断从共享队列当中取出元素，并消费
			while(true){
				try {
					E e = queue.take();// 挂起，直到队列非空
					consume(e);
				} catch (InterruptedException e1) { /* exception */ }
			}
		}
		protected void consume(E e){System.out.println(e);}
	}
}
```

