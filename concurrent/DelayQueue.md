[toc]

##### 特性

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {
    
    
public interface Delayed extends Comparable<Delayed> {
    long getDelay(TimeUnit unit);
}
```

DelayQueue实现了BlockingQueue，所以它是一个阻塞队列。

另外DelayQueue还组合了一个叫Delayed的接口，DelayQueue中存储的所有元素必须实现Delayed接口。

##### 成员变量

```java
// 并发的锁
private final transient ReentrantLock lock = new ReentrantLock();
// 优先级队列,存储元素
private final PriorityQueue<E> q = new PriorityQueue<E>();
// 用于标记当前是否有线程在排队（仅用于取元素时）
private Thread leader = null;
// 条件，用于表示现在是否有可取的元素
private final Condition available = lock.newCondition();
```

使用优先级队列实现，无界

##### 构造器

```java
public DelayQueue() {}
public DelayQueue(Collection<? extends E> c) {
    this.addAll(c);
}
```

##### 入队

因为DelayQueue是阻塞队列，且优先级队列是无界的，所以入队不会阻塞不会超时，因为它的四个入队方式是一样的

```java
public boolean add(E e) {
    return offer(e);
}

public void put(E e) {
    offer(e);
}

//timeout unit都会被忽略(接口方法)
public boolean offer(E e, long timeout, TimeUnit unit) {
    return offer(e);
}

public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
      	//入队
        q.offer(e);
      	//如果添加的元素是堆顶元素，就把leader置为空，并唤醒等待在条件available上的线程
        if (q.peek() == e) {
            leader = null;
            available.signal();
        }
        return true;
    } finally {
        lock.unlock();
    }
}
```

##### 出队

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        E first = q.peek();// 取出堆顶元素
        //检查第一个元素，如果为空或者还没到期，就返回null；
        if (first == null || first.getDelay(NANOSECONDS) > 0)
            return null;
        else
            return q.poll();//弹出第一个元素
    } finally {
        lock.unlock();
    }
}

public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();// 取出堆顶元素   
            if (first == null)// 如果堆顶元素为空，说明队列中还没有元素，直接阻塞等待
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);// 堆顶元素的到期时间             
                if (delay <= 0)// 如果小于0说明已到期，直接调用poll()方法弹出堆顶元素
                    return q.poll();
                
                // 如果delay大于0 ，则下面要阻塞了
                // 将first置为空方便gc
                first = null; 
                // 如果前面有其它线程在等待，直接进入等待
                if (leader != null)
                    available.await();
                else {
                    // 如果leader为null，把当前线程赋值给它
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        // 等待delay时间后自动醒过来
                        // 醒过来后把leader置空并重新进入循环判断堆顶元素是否到期
                        // 这里即使醒过来后也不一定能获取到元素
                        // 因为有可能其它线程先一步获取了锁并弹出了堆顶元素
                        // 条件锁的唤醒分成两步，先从Condition的队列里出队
                        // 再入队到AQS的队列中，当其它线程调用LockSupport.unpark(t)的时候才会真正唤醒
                        available.awaitNanos(delay);
                    } finally {
                        // 如果leader还是当前线程就把它置为空，让其它线程有机会获取元素
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        // 成功出队后，如果leader为空且堆顶还有元素，就唤醒下一个等待的线程
        if (leader == null && q.peek() != null)
            // signal()只是把等待的线程放到AQS的队列里面，并不是真正的唤醒
            available.signal();
        // 解锁，这才是真正的唤醒
        lock.unlock();
    }
}
```

+ 加锁
+ 判断堆顶元素是否为空，为空的话直接阻塞等待
+ 判断堆顶元素是否到期，到期了直接调用优先队列的poll()弹出元素
+ 没到期，再判断是否有其他线程在等待，有则直接等待
+ 前面没有其他线程在等待，则把它自己作为一个线程等待delay实现后唤醒，再尝试获取元素
+ 获取到元素之后再唤醒下一个等待的线程
+ 解锁

java中的线程实现是直接用DelayQueue吗？

不是，ScheduledThreadPoolExecutor中使用的是它自己定义的内部类

DelayedWorkQueue，里面的实现逻辑基本是一样的，只不过DelayedWorkQueue里面没有使用线程的PriorityQueue，而是使用数组又实现了一遍优先级队列，本质上没啥区别。