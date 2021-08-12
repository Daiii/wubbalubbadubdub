## JVM内存模型

[toc]

### JDK体系结构

![0](images/jdk体系结构.png)



### Java语言跨平台特性

![0](images/java语言跨平台特性.png)



### JVM整体结构及内存模型

![0](images/JVM内存模型.png)

#### 在minor gc过程中对象挪动后，引用如何修改？

在对象堆内部挪动的过程其实复制，原有区域对象还在，一直不直接清理，JVM内部清理过程只是将对象分配指针移动到区域的偷位置即可，比如扫描s0区域，扫到gcroot引用的非垃圾对象是将这些对象**复制**到s1或老年代，最后扫描完了将s0区域的对象分配指针移动到区域的起始位置即可，s0区域之前对象并不直接清理，当有新对象分配了，原有区域里的对象也就被清楚了。

minor gc在根扫描过程中会记录所有被扫描到的对象引用(在年轻代这些引用很少，因为大部分都是垃圾对象不会扫描到)，如果引用的对象复制到新地址了，最后会议并更新引用指向新地址。

>**参考资料**
>
>https://www.zhihu.com/question/42181722/answer/145085437
>
>https://hllvm-group.iteye.com/group/topic/39376#post-257329



### JVM内存参数设置

![0](images/jvm参数.png)

* -Xss：每个线程栈的大小
* -Xms：设置堆的初始可用大小，默认物理内存的1/64
* -Xmx：设置堆的最大可用大小，默认物理内存的1/4
* -Xmn：新生代大小
* -XX:NewRatio：默认2标识新生代占老年代的1/2，占整个堆内存的1/3
* -XX:SurvivorRatio：默认8标识一个survivor占用1/8Eden内存，即1/10新生代内存
* -XX:MaxMetaspaceSize：设置元空间最大值，默认是-1，即不限制，或者说只受限于本地内存大小
* --XX:MetaspaceSzie：指定元空间触发Fullgc的初四阈值(元空间无固定大小)，以字节为单位，默认是21M左右，打到该值就触发full gc进行类型写在，同事收集器会对该值进行调整：如果释放了大量的空间，该值就适当降低；如果释放了很少的空间，那么在不超过-XX:MaxMetaspaceSize(如果设置了的话)的情况下，该值就适当提高。

**StackOverflowError**：

```java
// JVM设置  -Xss128k(默认1M)
public class StackOverflowTest {
    
    static int count = 0;
    
    static void redo() {
        count++;
        redo();
    }

    public static void main(String[] args) {
        try {
            redo();
        } catch (Throwable t) {
            t.printStackTrace();
            System.out.println(count);
        }
    }
}

// 运行结果：
// java.lang.StackOverflowError
// 	at com.tuling.jvm.StackOverflowTest.redo(StackOverflowTest.java:12)
// 	at com.tuling.jvm.StackOverflowTest.redo(StackOverflowTest.java:13)
// 	at com.tuling.jvm.StackOverflowTest.redo(StackOverflowTest.java:13)
//  ......
```

**结论**：

-Xss设置越小count值越小，说明一个线程栈里面能分配的栈帧就越少，但是对JVM整体来说开启的线程数会更多。