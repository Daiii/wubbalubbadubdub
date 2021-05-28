[toc]

##### 成员变量

底层是数组，平衡二叉树实现

```java
// 默认容量为11
private static final int DEFAULT_INITIAL_CAPACITY = 11;
// 最大数组大小
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
// 存储元素的数组
private transient Object[] queue;
// 元素个数
private transient int size;
// 排序比较器
private transient Comparator<? super E> comparator;
// 重入锁
private final ReentrantLock lock;
// 非空条件
private final Condition notEmpty;
// 扩容的时候使用的控制变量，CAS更新这个值，谁更新成功了谁扩容，其它线程让出CPU,自旋锁
private transient volatile int allocationSpinLock;
// 数组实现的最小堆，writeObject和readObject用到。 为了兼容之前的版本，只有在序列化和反序列化才非空
private PriorityQueue<E> q;
```

##### 构造器

```java
public PriorityBlockingQueue() {
  	// 默认容量为11
    this(DEFAULT_INITIAL_CAPACITY, null);
}

// 传入初始容量
public PriorityBlockingQueue(int initialCapacity) {
    this(initialCapacity, null);
}

// 传入初始容量和比较器，初始化各变量
public PriorityBlockingQueue(int initialCapacity, Comparator<? super E> comparator) {
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    this.comparator = comparator;
    this.queue = new Object[initialCapacity];
}

public PriorityBlockingQueue(Collection<? extends E> c) {
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    boolean heapify = true; 
    boolean screen = true;  
  	//如果是SortedSet类型，无须进行堆有序化
    if (c instanceof SortedSet<?>) {
        SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
        this.comparator = (Comparator<? super E>) ss.comparator();
				//不需要重建堆
      	heapify = false;
    }
  	//如果是PriorityBlockingQueue类型,无须进行堆有序化
    else if (c instanceof PriorityBlockingQueue<?>) {
        PriorityBlockingQueue<? extends E> pq = (PriorityBlockingQueue<? extends E>) c;
        this.comparator = (Comparator<? super E>) pq.comparator();
        screen = false;
      	//严格匹配，非子类
        if (pq.getClass() == PriorityBlockingQueue.class) 
          	//不需要重建堆
            heapify = false;
    }
    Object[] a = c.toArray();
    int n = a.length;
    if (c.getClass() != java.util.ArrayList.class)
        a = Arrays.copyOf(a, n, Object[].class);
    if (screen && (n == 1 || this.comparator != null)) {
        for (int i = 0; i < n; ++i)
          	//不接受null元素
            if (a[i] == null)
                throw new NullPointerException();
    }
    this.queue = a;
    this.size = n;
    if (heapify)
        heapify();
}
```

##### 入队

永远返回true，无界

```java
public boolean offer(E e) {
  	//不支持null
    if (e == null)
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();
    int n, cap;
    Object[] array;
   
  	//如果当前元素个数>=队列容量，则扩容
    while ((n = size) >= (cap = (array = queue).length)) 
        tryGrow(array, cap);
    try {
        Comparator<? super E> cmp = comparator;
      	//默认比较器为null,自下而上的堆化
        if (cmp == null)
            siftUpComparable(n, e, array);
        else  
            siftUpUsingComparator(n, e, array, cmp);
        //队列元素增加1，并且激活notEmpty的条件队列里面的一个阻塞线程
        size = n + 1;
      	//激活调用take（）方法被阻塞的线程
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    return true;
}
```

默认的插入规则中，新加入的元素可能会破坏小顶堆的性质，因此需要进行调整。

调整的过程为：从尾部下标的位置开始，将加入的元素逐层与当前的父节点的内容进行比较并交换，知道满足父节点内容都小于子节点的内容位置。

默认的删除调整中，时候获取顶部下标和最尾部的元素内容，从顶部的位置开始，将尾部的元素内容逐层向下与当前的左右子节点中较小的那个交换，直到判断元素小于或者等于左右子节点的任何一个为止。

##### 扩容

```java
private void tryGrow(Object[] array, int oldCap) {
  	// 释放锁，防止阻塞的线程过多
    lock.unlock(); 
    Object[] newArray = null;
    //CAS更新allocationSpinLock变量为1的线程获得扩容资格
    if (allocationSpinLock == 0 &&   
        UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,0, 1)) {
        try {
          	//oldGap<64则扩容新增oldcap+2,否者扩容50%，并且最大为MAX_ARRAY_SIZE
            int newCap = oldCap + ((oldCap < 64) ?
                                   (oldCap + 2) : 
                                   (oldCap >> 1));
            if (newCap - MAX_ARRAY_SIZE > 0) {    //扩容后超过最大容量处理
                int minCap = oldCap + 1;
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)//整数溢出
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;
            }
            if (newCap > oldCap && queue == array)//queue是公共变量
                newArray = new Object[newCap];
        } finally {// 解锁，因为只有一个线程到此，因而不需要CAS操作；
            allocationSpinLock = 0;
        }
    }
    //扩容失败的线程newArray == null，调用Thread.yield()让出cpu， 让扩容成功线程优先调用lock.lock重新获取锁，
    //但是这得不到一定的保证，有可能调用Thread.yield()的线程先获取了锁（进入自旋）
    if (newArray == null) 
        Thread.yield();//让出cpu
    lock.lock();//有可能扩容成功的线程先走到这里，也有可能扩容失败的线程先走到这里。
    //准备赋值给共有变量queue，要加锁，
    //扩容成功的线程newArray != null ，扩容失败的线程newArray == null(再次进入while循环去扩容) 
    if (newArray != null && queue == array) {
        queue = newArray;// 队列赋值为新数组
        System.arraycopy(array, 0, newArray, 0, oldCap);// 并拷贝旧数组元素到新数组中
    }
}
```

**ryGrow 目的是扩容，这里要思考下为啥在扩容前要先释放锁，然后使用 cas 控制只有一个线程可以扩容成功呢？**

其实这里不先释放锁也是可以的，也就是在整个扩容期间一直持有锁，但是扩容是需要花时间的，如果扩容的时候还占用锁，那么其他线程在这个时候是不能进行出队和入队操作的，

这大大降低了并发性。所以为了提高性能，使用CAS控制只有一个线程可以进行扩容，并且在扩容前释放了锁，让其他线程可以进行入队和出队操作。

spinlock锁使用CAS控制只有一个线程可以进行扩容，CAS失败的线程会调用Thread.yield() 让出 cpu，目的是为了让扩容线程扩容后优先调用 lock.lock 重新获取锁，

但是这得不到一定的保证。有可能yield的线程在扩容线程扩容完成前已经退出，并执行了代码lock.lock()获取到了锁。如果当前数组扩容还没完毕，当前线程会再次调用tryGrow方法，

然后释放锁，这又给扩容线程获取锁提供了机会，如果这时候扩容线程还没扩容完毕，则当前线程释放锁后又调用yield方法让出CPU。可知当扩容线程进行扩容期间，

其他线程是原地自旋通过while检查当前扩容是否完毕，等扩容完毕后才退出while的循环。

当扩容线程扩容完毕后会重置自旋锁变量allocationSpinLock 为 0，这里并没有使用UNSAFE方法的CAS进行设置是因为同时只可能有一个线程获取了该锁，并且 allocationSpinLock 被修饰为了 volatile。

当扩容线程扩容完毕后会执行代码lock.lock()获取锁，获取锁后复制当前 queue 里面的元素到新数组。

##### 出队

```java
public E take() throws InterruptedException {
    //获取锁，可被中断
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        //如果队列为空，则阻塞，把当前线程放入notEmpty的条件队列
        while ( (result = dequeue()) == null)
            notEmpty.await();//阻塞当前线程
    } finally {
        lock.unlock();//释放锁
    }
    return result;
}

private E dequeue() {
    int n = size - 1;
    if (n < 0)
        return null;
    else {
        Object[] array = queue;
        E result = (E) array[0];// 弹出堆顶元素
        E x = (E) array[n];// 把堆尾元素拿到堆顶
        array[n] = null;
        Comparator<? super E> cmp = comparator;
        if (cmp == null)//自上而下的堆化
            siftDownComparable(0, x, array, n);
        else
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;
        return result;
    }
}

private static <T> void siftDownComparable(int k, T x, Object[] array,
                                           int n) {
    if (n > 0) {
        Comparable<? super T> key = (Comparable<? super T>)x;
        int half = n >>> 1;           
        // 只需要遍历到叶子节点就够了
        while (k < half) {
            // 左子节点
            int child = (k << 1) + 1; 
            // 左子节点的值
            Object c = array[child];
            // 右子节点
            int right = child + 1;
            // 取左右子节点中最小的值
            if (right < n &&
                ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                c = array[child = right];
            // key如果比左右子节点都小，则堆化结束
            if (key.compareTo((T) c) <= 0)
                break;
            // 否则，交换key与左右子节点中最小的节点的位置
            array[k] = c;
            k = child;
        }
        // 找到了放元素的位置，放置元素
        array[k] = key;
    }
}
```

PriorityBlockingQueue在入队的时候如果没有空间了是会自动扩容的，也就不存在队列满了的状态，也就是不需要等待通知队列不满了可以放元素了，所以也就不需要notFull条件了

