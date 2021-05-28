```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // key和value不能为NULL
    if (key == null || value == null) throw new NullPointerException();
    
    // key所对应的hashcode
    int hash = spread(key.hashCode());
    int binCount = 0;
    
    // 通过自旋的方式来插入数据
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 如果数组为空，则初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 算出数组下标，然后获取数组上对应下标的元素，如果为null，则通过cas来赋值
        // 如果赋值成功，则退出自旋，否则是因为数组上当前位置已经被其他线程赋值了，
        // 所以失败，所以进入下一次循环后就不会再符合这个判断了
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 如果数组当前位置的元素的hash值等于MOVED，表示正在进行扩容，当前线程也进行扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 对数组当前位置的元素进行加锁
            synchronized (f) {
                // 加锁后检查一下tab[i]上的元素是否发生了变化，如果发生了变化则直接进入下一次循环
                // 如果没有发生变化，则开始插入新key,value
                if (tabAt(tab, i) == f) {
                    // 如果tab[i]的hashcode是大于等于0的，那么就将元素插入到链表尾部
                    if (fh >= 0) {
                        binCount = 1; // binCount表示当前链表上节点的个数，不包括新节点
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 遍历链表的过程中比较key是否存在一样的
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 插入到尾节点
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 如果tab[i]是TreeBin类型，表示tab[i]位置是一颗红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 在新插入元素的时候，如果不算这个新元素链表上的个数大于等于8了，那么就要进行树化
                // 比如binCount为8，那么此时tab[i]上的链表长度为9，因为包括了新元素
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                // 存在key相同的元素
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```



```java
// 默认通过baseCount去计算size
// 如果conterCells是空的直接返回baseCount
// 如果counterCells不会空累加返回
final long sumCount() {
	CounterCell[] cs = counterCells;
	long sum = baseCount;
	if (cs != null) {
    for (CounterCell c : cs)
    if (c != null)
    sum += c.value;
	}
	return sum;
}
```



```java
// 从 putVal 传入的参数是 1， binCount，binCount 默认是0，只有 hash 冲突了才会大于 1.且他的大小是链表的长度（如果不是红黑数结构的话）。
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 如果计数盒子不是空 或者
    // 如果修改 baseCount 失败
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        // 如果计数盒子是空（尚未出现并发）
        // 如果随机取余一个数组位置为空 或者
        // 修改这个槽位的变量失败（出现并发了）
        // 执行 fullAddCount 方法。并结束
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    // 如果需要检查,检查是否需要扩容，在 putVal 方法调用时，默认就是要检查的。
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 如果map.size() 大于 sizeCtl（达到扩容阈值需要扩容） 且
        // table 不是空；且 table 的长度小于 1 << 30。（可以扩容）
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            // 根据 length 得到一个标识
            int rs = resizeStamp(n);
            // 如果正在扩容
            if (sc < 0) {
                // 如果 sc 的低 16 位不等于 标识符（校验异常 sizeCtl 变化了）
                // 如果 sc == 标识符 + 1 （扩容结束了，不再有线程进行扩容）（默认第一个线程设置 sc ==rs 左移 16 位 + 2，当第一个线程结束扩容了，就会将 sc 减一。这个时候，sc 就等于 rs + 1）
                // 如果 sc == 标识符 + 65535（帮助线程数已经达到最大）
                // 如果 nextTable == null（结束扩容了）
                // 如果 transferIndex <= 0 (转移状态变化了)
                // 结束循环 
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 如果可以帮助扩容，那么将 sc 加 1. 表示多了一个线程在帮助扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    // 扩容
                    transfer(tab, nt);
            }
            // 如果不在扩容，将 sc 更新：标识符左移 16 位 然后 + 2. 也就是变成一个负数。高 16 位是标识符，低 16 位初始是 2.
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                // 更新 sizeCtl 为负数后，开始扩容。
                transfer(tab, null);
            s = sumCount();
        }
    }
```

