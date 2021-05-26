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

