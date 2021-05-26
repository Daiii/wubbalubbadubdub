##### 特性

```java
public class LinkedList<E> extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

1. 继承AbstractSequentialList，本质上面与继承AbstractList没什么区别，AbstractSequentialList完善了AbstractList中没有实现的方法。
2. Serializable：成员变量Node使用transient修饰，通过重写read/writeObject方法实现序列化。
3. Cloneable：重写clone()方法，通过创建新的LinkedList对象，遍历拷贝数据进行对象拷贝。
4. **Deque：实现了Collection中的队列接口，说明他拥有作为双向队列的功能。**

LinkedList与ArrayList最大的区别就是LinkedList中实现了Collection中的 **Queue（Deque）接口 拥有作为双向队列的功能**

##### 基本属性

链表没有长度限制，他的内存地址不需要分配固定长度进行存储，只要记录下一个节点的存储地址就能完成整个链表的连续。

```java
//当前有多少个结点，元素个数
transient int size = 0;

//第一个节点
transient Node<E> first;

//最后一个节点
transient Node<E> last;

//Node的数据结构
private static class Node<E> {
		//存储元素
  	E item;
  	//后继
    Node<E> next;
  	//前驱
    Node<E> prev;
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

LinkedList在1.6版本及以前只通过一个header头指针保存队列的头和尾。后续更新中讲头和尾节点区分了。节点类更名为Node。

为什么Node这个类是静态的？

这跟内存泄漏有关，Node类是在LinkedList类中，也就是一个内部类，如果不适用static修饰那么Node就是一个普通的内部类，在java中一个普通内部类在实例化之后默认会持有外部类的引用，这就有可能造成内存泄漏(内部类与外部类生命周期不一致时)。使用static修饰过的内部类(静态内部类)就不会有这个问题。

非静态内部类会自动生成一个构造器依赖于外部类：也是内部类可以访问外部类的实例变量的原因。

静态内部类不会生成，访问不了外部类的实例变量，只能访问类变量。

##### 构造器

```java
public boolean add(E e) {
     linkLast(e);
     return true;
 }

//目标节点创建后寻找前驱节点， 前驱节点存在就修改前驱节点的后继，指向目标节点
void linkLast(E e) {
  	//获取这个list对象内部的Node类型成员last，即末位节点，以该节点作为新插入元素的前驱节点
    final Node<E> l = last;
	  //创建新节点
    final Node<E> newNode = new Node<>(l, e, null);
		//把新节点作为该list对象的最后一个节点
  	last = newNode;
  	//处理原先的末位节点，如果这个list本来就是一个空的链表
    if (l == null)
      	//把新节点作为首节点
        first = newNode;
    else
      	//如果链表内部已经有元素，把原来的末位节点的后继指向新节点，完成链表修改
        l.next = newNode;
		//修改当前list的size，
  	size++;
  	//并记录该list对象被执行修改的次数
    modCount++;
}

public void add(int index, E element) {
		//检查下标的合法性
  	checkPositionIndex(index);
		//插入位置是末位，那还是上面末位添加的逻辑
  	if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}

private void checkPositionIndex(int index) {
    if (!isPositionIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

private boolean isPositionIndex(int index) {
    return index >= 0 && index <= size;
}

Node<E> node(int index) {
  	//二分查找   index离哪端更近 就从哪端开始找
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
						//找到index位置的元素
          	x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

//指位添加方法核心逻辑  操作新节点，紧接修改原有节点的前驱属性，最后再修改前驱节点的后继属性
void linkBefore(E e, Node<E> succ) {
    final Node<E> pred = succ.prev;//原位置节点的前驱pred
    final Node<E> newNode = new Node<>(pred, e, succ);//创建新节点,设置新节点其前驱为原位置节点的前驱pred，其后继为原位置节点succ
    succ.prev = newNode;//将新节点设置到原位置节点的前驱
    if (pred == null)//前驱如果为空，空链表，则新节点设置为first
        first = newNode;
    else
        pred.next = newNode;//将新节点设置到前驱节点的后继
    size++;//修改当前list的size
    modCount++;//记录该list对象被执行修改的次数。
}

public boolean addAll(int index, Collection<? extends E> c) {
    checkPositionIndex(index);
    //将集合转化为数组
    Object[] a = c.toArray();
    int numNew = a.length;
    if (numNew == 0)
        return false;
    Node<E> pred, succ;
    //获取插入节点的前节点（prev）和尾节点（next）
    if (index == size) {
        succ = null;
        pred = last;
    } else {
        succ = node(index);
        pred = succ.prev;
    }
    //将集合中的数据编织成链表
    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        pred = newNode;
    }
    //将 Collection 的链表插入 LinkedList 中。
    if (succ == null) {
        last = pred;
    } else {
        pred.next = succ;
        succ.prev = pred;
    }
    size += numNew;
    modCount++;
    return true;
}
```

final修饰，不希望在运行时对变量做重新赋值。

LinkedList在插入数据优于ArrayList，主要是因为他只需要修改指针的指向即可，而不需要将整个数组的数据进行转移。

由于LinkedList没有实现RandomAccess不支持索引查找，在查找元素这一操作需要消耗比较多的时间进行操作(n/2)。

##### 删除元素

1. AbstractSequentialList的remove

   ```java
   public E remove(int index) {
       checkElementIndex(index);
       //node(index)找到index位置的元素
       return unlink(node(index));
   }
   
   //remove(Object o)这个删除元素的方法的形参o是数据本身，而不是LinkedList集合中的元素（节点），所以需要先通过节点遍历的方式，找到o数据对应的元素，然后再调用unlink(Node x)方法将其删除
   public boolean remove(Object o) {
       if (o == null) {
           for (Node<E> x = first; x != null; x = x.next) {
               if (x.item == null) {
                   unlink(x);
                   return true;
               }
           }
       } else {
           for (Node<E> x = first; x != null; x = x.next) {
               if (o.equals(x.item)) {
                   unlink(x);
                   return true;
               }
           }
       }
       return false;
   }
   
   E unlink(Node<E> x) {
       //x的数据域element
       final E element = x.item;
       //x的下一个结点
       final Node<E> next = x.next;
       //x的上一个结点
       final Node<E> prev = x.prev;
       //如果x的上一个结点是空结点的话，那么说明x是头结点
       if (prev == null) {
           first = next;
       } else {
         	//将x的前后节点相连   双向链表
           prev.next = next;
         	//x的属性置空
           x.prev = null;
       }
       //如果x的下一个结点是空结点的话，那么说明x是尾结点
       if (next == null) {
           last = prev;
       } else {
         	//将x的前后节点相连   双向链表
           next.prev = prev;
           x.next = null;
       }
     	//指向null  方便GC回收
       x.item = null;
       size--;
       modCount++;
       return element;
   }
   ```

   

2. Deque中的remove

   ```java
   //将first 节点的next 设置为新的头节点，然后将 f 清空。 removeLast 操作也类似。
   private E unlinkFirst(Node<E> f) {
       final E element = f.item;
       //获取到头结点的下一个结点           
       final Node<E> next = f.next;
       f.item = null;
       f.next = null; // 方便 GC
       //头指针指向的是头结点的下一个结点
       first = next;
       //如果next为空，说明这个链表只有一个结点
       if (next == null)
           last = null;
       else
           next.prev = null;
       size--;
       modCount++;
       return element;
   }
   ```

##### 双向队列(队列Queue)

java中队列的实现就是LinkedList：我们之所以说LinkedList为双向链表是因为他实现了Deque接口，队列是先进先出的，添加元素只能从队尾添加，删除元素只能队头删除，Queue中的方法提现了这种特性。

支持队列的一些操作：

+ pop() 是栈结构的实现类的方法，返回栈顶元素，并且将栈顶元素删除。
+ poll() 是队列的数据结构，获取队投元素并且删除队头元素。
+ push() 是栈结构的实现类的方法，把元素压入到栈中。
+ peek() 获取队头元素，单是不删除队头元素。、
+ offer() 添加队尾元素。

##### 队列的增

offer() 添加队尾元素

```java
public boolean offer(E e) {
    return add(e);
}
```

具体的实现就是在尾部添加一个元素

##### 队列的删

poll() 是队列的数据结构，获取队头元素并且删除头元素

```java
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
```

具体的实现就是删除队列头部的元素。

##### 队列的查

peek() 获取队头元素，但是不删除队头的元素。

```java
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}
```

##### 栈的增

push() 是栈结构的实现方法，把元素压如栈中。

```java
public void push(E e) {
    addFirst(e);
}
```

##### 栈的删

pop() 是栈结构的实现类的方法，返回的是栈顶元素，并且将栈顶元素删除。

```java
public E pop() {
    return removeFirst();
}

public E removeFirst() {
    final Node f = first;
    if (f == null)
    throw new NoSuchElementException();
    return unlinkFirst(f);
}
```