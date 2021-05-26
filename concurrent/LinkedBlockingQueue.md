以单链表实现的阻塞队列，它是线程安全的，借助分段的思想把入队和出队分裂成两个锁。

##### 成员变量

```java
// 容量,有界队列
private final int capacity;
// 元素数量
private final AtomicInteger count = new AtomicInteger();
// 链表头  本身是不存储任何元素的，初始化时item指向null
transient Node<E> head;
// 链表尾  last.next == null
private transient Node<E> last;
// take锁   锁分离，提高效率
private final ReentrantLock takeLock = new ReentrantLock();
// notEmpty条件
// 当队列无元素时，take锁会阻塞在notEmpty条件上，等待其它线程唤醒
private final Condition notEmpty = takeLock.newCondition();
// put锁
private final ReentrantLock putLock = new ReentrantLock();
// notFull条件
// 当队列满了时，put锁会会阻塞在notFull上，等待其它线程唤醒
private final Condition notFull = putLock.newCondition();
static class Node<E> {
    E item;//存储元素
    Node<E> next;//后继元素    单链表结构
    Node(E x) { item = x; }
}
```

##### 构造器

```java
public LinkedBlockingQueue() {    
  	// 一定是有界的
    this(Integer.MAX_VALUE);
}

public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
  	// 初始化head和last指针为空值节点,head和last指向同一个对象
    last = head = new Node<E>(null);
}

public LinkedBlockingQueue(Collection<? extends E> c) {
    this(Integer.MAX_VALUE);
    final ReentrantLock putLock = this.putLock;
    putLock.lock(); 
    try {
        int n = 0;
        for (E e : c) {
            if (e == null)
                throw new NullPointerException();
            if (n == capacity)
                throw new IllegalStateException("Queue full");
            enqueue(new Node<E>(e));
            ++n;
        }
        count.set(n);
    } finally {
        putLock.unlock();
    }
}
```

##### 入队

```java
public void put(E e) throws InterruptedException {  
  	// 不允许null元素
    if (e == null) throw new NullPointerException();
    int c = -1;
  	// 新建一个节点
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
  	// 使用put锁加锁
    putLock.lockInterruptibly();
    try {
        // 如果队列满了，就阻塞在notFull条件上等待被其它线程唤醒
        while (count.get() == capacity) {
            notFull.await();
        }  
        enqueue(node);// 队列不满了，就入队
        c = count.getAndIncrement();// 队列长度加1，返回原值
        // 如果现队列长度如果小于容量，就唤醒一个阻塞在notFull条件上的线程(可以继续入队)
        // 这里为啥要唤醒一下呢？
        // 因为可能有很多线程阻塞在notFull这个条件上的,而取元素时只有取之前队列是满的才会唤醒notFull,不用等到取元素时才唤醒
        
        // 为什么队列满的才唤醒notFull呢？
        // 因为唤醒是需要加putLock的，这是为了减少锁的次数,所以，这里索性在放完元素就检测一下，未满就唤醒其它notFull上的线程,说白了，这也是锁分离带来的代价
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }    
    if (c == 0)// 如果原队列长度为0，现在加了一个元素后立即唤醒notEmpty条件
        signalNotEmpty();
}

private void enqueue(Node<E> node) { 
    last = last.next = node;// 直接加到last后面,last指向入队元素，如果是第一次入队head.next也会指向入队元素
}    

private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock; 
    takeLock.lock();// 加take锁
    try {  
        notEmpty.signal();// 唤醒notEmpty条件
    } finally {
        takeLock.unlock();
    }
}
```

+ 使用putLock加锁
+ 使用队列满了就阻塞在notFull条件上，等待被唤醒继续执行
+ 否则就入队
+ 如果入队后元素数量小于容量，唤醒其他阻塞在notFull条件上的线程
+ 释放锁
+ 如果放元素之前队列长度为0就幻想notEmpty条件

##### 对队

```java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
  	// 使用takeLock加锁
    takeLock.lockInterruptibly();
    try {
      	// 如果队列无元素，则阻塞在notEmpty条件上
        while (count.get() == 0) {
            notEmpty.await();
        }
        
      	// 否则，出队
        x = dequeue();
      	//长度-1，返回原值
        c = count.getAndDecrement();
      	// 如果取之前队列长度大于1，则唤醒notEmpty   原因与入队同理
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    // 如果取之前队列长度等于容量（已满），则唤醒notFull
    if (c == capacity)
        signalNotFull();
    return x;
}

private E dequeue() {
    // 这里把head删除，并把head.next作为出队元素
    // 并把其值置空，返回原来的值
    Node<E> h = head;
    Node<E> first = h.next;
  	// 方便GC
    h.next = h; 
    head = first;
    E x = first.item;
  	//head.item还是指向null,head.next指向了新的出队元素
    first.item = null;
    return x;
}

private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}
```

+ 使用takeLock加锁
+ 如果队列空了就阻塞在notEmpty条件上
+ 否则就出队
+ 如果出队钱元素数量大于1，唤醒其他阻塞在notEmpty条件上的线程
+ 释放锁
+ 如果取元素之前队列长度等于容量，唤醒notFull条件

##### 删除元素

```java
public boolean remove(Object o) {
    if (o == null) return false;
  	//此时将入队锁和出队锁全部锁住来保证线程安全
    fullyLock(); 
    try {
        for (Node<E> trail = head, p = trail.next;
             p != null;
             // 循环遍历查找值相等的元素
             trail = p, p = p.next) {
            if (o.equals(p.item)) {
              	//调用unlink删除此节点
                unlink(p, trail);
              	//操作成功返回true
                return true;
            }
        }
        return false;
    } finally {
        fullyUnlock();
    }
}

//p为要删除节点，trail为删除节点的前一个节点
void unlink(Node<E> p, Node<E> trail) {
    p.item = null;
  	// 改变指针将前一节点的后继节点指向删除节点的后一个节点
    trail.next = p.next; 
    if (last == p)
        last = trail;
    if (count.getAndDecrement() == capacity)
        notFull.signal();
}
```

