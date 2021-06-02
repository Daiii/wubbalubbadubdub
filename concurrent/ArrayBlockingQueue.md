[toc]

##### 特性

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable
```

BlockingQueue：定义了队列的入队和出队方法。

AbstractQueue：入队和出队的基本操作。

##### 成员变量

组成：一个对象数组+1把锁ReentrantLock+2个条件Condition

```java
// 使用数组存储元素
final Object[] items;
// 取元素的指针  记录下一次操作的位置
int takeIndex;
// 放元素的指针   记录下一次操作的位置
int putIndex;
// 元素数量
int count;
// 保证并发访问的锁
final ReentrantLock lock;
//等待出队的条件 消费者监视器
private final Condition notEmpty;
//等待入队的条件   生产者监视器
private final Condition notFull;
```

##### 构造器

```java
//必须传入容量，可以控制重入锁是公平还是非公平
public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
}

public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    // 初始化数组
    this.items = new Object[capacity];
    // 创建重入锁及两个条件
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}

public ArrayBlockingQueue(int capacity, boolean fair, Collection<? extends E> c) {
  	//final修饰的变量不会发生指令重排
  	this(capacity, fair);
    final ReentrantLock lock = this.lock;
  	//保证可见性 不是为了互斥  防止指令重排 保证item的安全
  	lock.lock(); 
    try {
        int i = 0;
        try {
            for (E e : c) {
                checkNotNull(e);
                items[i++] = e;
            }
        } catch (ArrayIndexOutOfBoundsException ex) {
            throw new IllegalArgumentException();
        }
        count = i;
        putIndex = (i == capacity) ? 0 : i;
    } finally {
        lock.unlock();
    }
}
```

对象创建三步：2和3可能会发生重排(互相不依赖)

1. 分配内存空间
2. 初始化对象
3. 将内存空间的地址赋值给对应的引用

##### 入队

```java
public boolean add(E e) {
    return super.add(e);
}

// AbstractQueue 调用offer(e)如果成功返回true，如果失败抛出异常
public boolean add(E e) {   
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}

public boolean offer(E e) {    
  	// 元素不可为空
  	checkNotNull(e);
    final ReentrantLock lock = this.lock;    
    lock.lock();// 加锁
    try {
      	// 如果数组满了就返回false            
        if (count == items.length)
            return false;
        else {         
          	// 如果数组没满就调用入队方法并返回true
            enqueue(e);
            return true;
        }
    } finally {        
        lock.unlock();
    }
}

public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;    
  	// 加锁，如果线程中断了抛出异常
    lock.lockInterruptibly();
    try {             
      	// 这里之所以使用while而不是if,是因为有可能多个线程阻塞在lock上,即使唤醒了可能其它线程先一步修改了队列又变成满的了,因此必须重新判断，再次等待
      	// 如果数组满了，使用notFull等待
      	while (count == items.length)
          	// notFull等待表示现在队列满了,等待被唤醒(只有取走一个元素后，队列才不满)        
          	notFull.await();
        enqueue(e);// 入队
    } finally {
        lock.unlock();
    }
}

public boolean offer(E e, long timeout, TimeUnit unit)throws InterruptedException {
    checkNotNull(e);
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
      	// 如果数组满了，就阻塞nanos纳秒，如果唤醒这个线程时依然没有空间且时间到了就返回false
        while (count == items.length) {
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
      	//入队
        enqueue(e);
        return true;
    } finally {
        lock.unlock();
    }
}

private void enqueue(E x) {
    final Object[] items = this.items;    
  	// 把元素直接放在放指针的位置上    
    items[putIndex] = x;
  	// 如果放指针到数组尽头了，就返回头部
    if (++putIndex == items.length)
        putIndex = 0;  
  	// 数量加1  
    count++;
  	// 唤醒notEmpty，因为入队了一个元素，所以肯定不为空了
    notEmpty.signal();
}
```

+ add(e)时如果队列满了抛出异常
+ offer(e)时如果队列满了则返回false
+ put(e)是如果队列满了则使用notFull等待
+ offer(e, timeout, unit)时如果队列满了则等待一段时间后如果队列依然满就返回false
+ 利用放指针循环使用数组来存储元素

##### 出列

```java
public E remove() {    
  	// 调用poll()方法出队
    E x = poll();
  	// 如果有元素出队就返回这个元素     
    if (x != null)
        return x;
    else 
      	// 如果没有元素出队就抛出异常
        throw new NoSuchElementException();
}

public E poll() {
    final ReentrantLock lock = this.lock;    
    lock.lock();
    try {       
      	// 如果队列没有元素则返回null，否则出队
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}

public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;   
    lock.lockInterruptibly();
    try {  
      	// 如果队列无元素，则阻塞等待在条件notEmpty上
        while (count == 0) 
            notEmpty.await();
      	// 有元素了再出队
        return dequeue();
    } finally {
        lock.unlock();
    }
}

public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
      	// 如果队列无元素，则阻塞等待nanos纳秒
      	// 如果下一次这个线程获得了锁但队列依然无元素且已超时就返回null
        while (count == 0) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
  	// 取出指针位置的元素
    E x = (E) items[takeIndex];
  	// 把取指针位置设为null
    items[takeIndex] = null;
  	// 将指针前移，如果数组到头了就返回数组前端循环利用
    if (++takeIndex == items.length)
        takeIndex = 0;  
  	// 元素数量减1
  	count--;
    if (itrs != null)
        itrs.elementDequeued();
  	// 唤醒notFull条件
    notFull.signal();
    return x;
}
```

+ remove()时如果队列为空抛出异常
+ poll()时如果队列为空则返回null
+ take()时如果队列空则祖师等待在条件notEmpty上
+ poll(time, unit)时如果队列为空则阻塞等待一段时间后如果还为空就返回null
+ 利用取指针循环从数组中取元素

##### 指定元素删除

```java
public boolean remove(Object o) {
    if (o == null) return false;
    final Object[] items = this.items;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {  
      	//如果此时队列不为null
        if (count > 0) {
          	//获取下一个要添加元素时的索引      
            final int putIndex = this.putIndex;
          	//从出队索引位置开始查找     
            int i = takeIndex;
            do {            
              	//查找要删除的元素，直接删除
                if (o.equals(items[i])) {
                    removeAt(i);
                    return true;
                }
              	//若为true，说明已到数组尽头，将i设置为0,继续查找
                if (++i == items.length)
                    i = 0; 
            } while (i != putIndex);//不等继续查找，如果相等，说明队列已经查找完毕
        }
        return false;
    } finally {
        lock.unlock();
    }
}

//把删除索引之后的元素往前移动一个位置
void removeAt(final int removeIndex) {
    final Object[] items = this.items;
  	//先判断要删除的元素是否为当前队列头元素,如果是直接删除
    if (removeIndex == takeIndex) { 
        items[takeIndex] = null;
      	//判断出队索引已经到达数组最末，出队索引设置为0
        if (++takeIndex == items.length)
            takeIndex = 0;
      	//队列元素减1
        count--;
        if (itrs != null)
          	//更新迭代器中的数据
            itrs.elementDequeued();
    } else {
      	//如果要删除的元素不在队列头部，
      	//那么只需循环迭代把删除元素后面的所有元素往前移动一个位置
      	//获取下一个要被添加的元素的索引，作为循环判断结束条件
        final int putIndex = this.putIndex;
      	//执行循环
        for (int i = removeIndex;;) {
          	//获取要删除节点索引的下一个索引
            int next = i + 1;
          	//判断是否已为数组长度，如果是从数组头部（索引为0）开始找
            if (next == items.length)
                next = 0;
          	//如果查找的索引不等于要添加元素的索引，说明元素可以再移动
            if (next != putIndex) {
                items[i] = items[next];//把后一个元素前移覆盖要删除的元
                i = next;
            } else {
              	//在removeIndex索引之后的元素都往前移动完毕后清空最后一个元素
                items[i] = null;
                this.putIndex = i;
                break;//结束循环
            }
        }
      	//队列元素减1
        count--;
        if (itrs != null)
          	//更新迭代器数据
            itrs.removedAt(removeIndex);
    }
  	//唤醒添加线程
    notFull.signal();
}
```

##### 缺点

+ 队列长度固定且必须在初始化时指定，所以使用之前一定要考虑好容量
+ 如果消费速度跟不上入队速度，则会导致提供者线程一直阻塞，且越阻塞越多。
+ 只用了一个锁来控制入队出队，效率偏低。