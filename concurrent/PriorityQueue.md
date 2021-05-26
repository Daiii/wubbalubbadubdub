优先级队列：一般使用堆数据结构实现，堆一般使用数组存储

集合中的每个元素都有一个权重值，每次出队都弹出优先级最大或最小的元素。

- PriorityQueue是一个小顶堆；
- PriorityQueue是非线程安全的；
- PriorityQueue不是有序的，只有堆顶存储着最小的元素；
- 入队就是堆的插入元素的实现；
- 出队就是堆的删除元素的实现

##### 成员变量

```java
// 默认容量
private static final int DEFAULT_INITIAL_CAPACITY = 11;
// 存储元素的数组
transient Object[] queue; 
// 元素个数
private int size = 0;
// 比较器   保证优先级，为空则是自然顺序
private final Comparator<? super E> comparator;
// 操作次数
transient int modCount = 0; 
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;//容量限制
```

##### 构造器

```java
//默认容量11
public PriorityQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}

//指定容量    
public PriorityQueue(int initialCapacity) {
    this(initialCapacity, null);
}
//默认容量，指定比较器
public PriorityQueue(Comparator<? super E> comparator) {
    this(DEFAULT_INITIAL_CAPACITY, comparator);
}

//指定容量，指定比较器，判断传入的容量是否合法，初始化存储元素的数组及比较器
public PriorityQueue(int initialCapacity, Comparator<? super E> comparator) {
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.queue = new Object[initialCapacity];
    this.comparator = comparator;
}

//根据传入的集合构造队列
public PriorityQueue(Collection<? extends E> c) {
    if (c instanceof SortedSet<?>) {//SortedSet类型的集合
        SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
        this.comparator = (Comparator<? super E>) ss.comparator();//取c的比较器
        initElementsFromCollection(ss);//元素入队
    }
    else if (c instanceof PriorityQueue<?>) {//同上
        PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
        this.comparator = (Comparator<? super E>) pq.comparator();
        initFromPriorityQueue(pq);
    }
    else {
        this.comparator = null;//自然顺序
        initFromCollection(c);
    }
}
@SuppressWarnings("unchecked")
public PriorityQueue(PriorityQueue<? extends E> c) {
    this.comparator = (Comparator<? super E>) c.comparator();
    initFromPriorityQueue(c);
}

@SuppressWarnings("unchecked")
public PriorityQueue(SortedSet<? extends E> c) {
    this.comparator = (Comparator<? super E>) c.comparator();
    initElementsFromCollection(c);
}

private void initFromPriorityQueue(PriorityQueue<? extends E> c) {
    if (c.getClass() == PriorityQueue.class) {//说明是相同的类型，直接赋值
        this.queue = c.toArray();
        this.size = c.size();
    } else {
        initFromCollection(c);
    }
}

private void initElementsFromCollection(Collection<? extends E> c) {
    Object[] a = c.toArray();
    if (c.getClass() != ArrayList.class)//底层是数组，进行数组拷贝
        a = Arrays.copyOf(a, a.length, Object[].class);
    int len = a.length;
    if (len == 1 || this.comparator != null)
        for (int i = 0; i < len; i++)
            if (a[i] == null)
                throw new NullPointerException();//不支持null元素
    this.queue = a;
    this.size = a.length;
}

private void initFromCollection(Collection<? extends E> c) {
    initElementsFromCollection(c);
    heapify();//堆化
}
```

##### 入队

```java
public boolean add(E e) {
    return offer(e);
}

public boolean offer(E e) {
    if (e == null)
      	// 不支持null元素
        throw new NullPointerException();
  	//记录操作数
    modCount++;
    int i = size;
  	// 元素个数大于数组长度，需要扩容
    if (i >= queue.length)
        grow(i + 1);
  	// 元素个数加1
    size = i + 1;
    // 如果还没有元素，直接插入到数组第一个位置
    if (i == 0)
        queue[0] = e;
    else
        // 否则，插入元素到数组的size位置，自下而上堆化
        siftUp(i, e);
    return true;
}

private void siftUp(int k, E x) {
    // 根据是否有比较器，使用不同的方法
    if (comparator != null)
        siftUpUsingComparator(k, x);
    else
        siftUpComparable(k, x);
}

@SuppressWarnings("unchecked")
private void siftUpComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    while (k > 0) {
        // 找到父节点的位置
        // 因为元素是从0开始的，所以减1之后再除以2
        int parent = (k - 1) >>> 1;
        // 父节点的值
        Object e = queue[parent];
        // 比较插入的元素与父节点的值
        // 如果比父节点大，则跳出循环
        // 否则交换位置
        if (key.compareTo((E) e) >= 0)
            break;
        // 与父节点交换位置
        queue[k] = e;
        // 现在插入的元素位置移到了父节点的位置
        // 继续与父节点再比较
        k = parent;
    }
    // 最后找到应该插入的位置，放入元素
    queue[k] = key;
}
```

##### 出队

```java
public E remove() {
    E x = poll();// 调用poll弹出队首元素
    if (x != null)
        return x;// 有元素就返回弹出的元素
    else
        throw new NoSuchElementException();// 否则抛出异常
}

@SuppressWarnings("unchecked")
public E poll() {
    if (size == 0)// 如果size为0，说明没有元素
        return null;
    int s = --size;// 弹出元素，元素个数减1
    modCount++; 
    E result = (E) queue[0];// 队列首元素
    E x = (E) queue[s];// 队尾元素
    queue[s] = null; // 将队列末元素删除
    if (s != 0)// 如果弹出元素后还有元素
        // 将队列末元素移到队列首，再做自上而下的堆化
        siftDown(0, x);
    return result;// 返回弹出的元素
}

private void siftDown(int k, E x) {
    // 根据是否有比较器，选择不同的方法
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}

@SuppressWarnings("unchecked")
private void siftDownComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>)x;
    // 只需要比较一半就行了，因为叶子节点占了一半的元素
    int half = size >>> 1;        // loop while a non-leaf
    while (k < half) {
        // 寻找子节点的位置，这里加1是因为元素从0号位置开始
        int child = (k << 1) + 1; // assume left child is least
        // 左子节点的值
        Object c = queue[child];
        // 右子节点的位置
        int right = child + 1;
        if (right < size &&
            ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            // 左右节点取其小者
            c = queue[child = right];
        // 如果比子节点都小，则结束
        if (key.compareTo((E) c) <= 0)
            break;
        // 如果比最小的子节点大，则交换位置
        queue[k] = c;
        // 指针移到最小子节点的位置继续往下比较
        k = child;
    }
    // 找到正确的位置，放入元素
    queue[k] = key;
}
```

+ 将队列首元素弹出
+ 将队列末元素移动到队头
+ 自上而下堆化，一直往下与最小的节点比较
+ 如果逼最小的子节点大，就交换位置，再继续与最小的子节点比较
+ 如果比最小的子节点小，就不用交换位置了，堆化结束
+ 这就是堆中的删除堆顶元素

##### 扩容

```java
private void grow(int minCapacity) {
    // 旧容量
    int oldCapacity = queue.length;
    
    // 旧容量小于64时，容量翻倍，旧容量大于等于64，容量只增加旧容量的一半
    int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                     (oldCapacity + 2) :
                                     (oldCapacity >> 1));
    // 检查是否溢出
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
        
    // 创建出一个新容量大小的新数组并把旧数组元素拷贝过去
    queue = Arrays.copyOf(queue, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```