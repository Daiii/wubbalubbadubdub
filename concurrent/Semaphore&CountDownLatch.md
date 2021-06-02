[toc]

## 1. Semaphore是什么?

Semaphore 字面意思是信号量的意思，它的作用是控制访问特定资源的线程数目，底层依赖AQS的状态State，是在生产当中比较常用的一个工具类。

## 2. 怎么使用Semaphore?

### 2.1 构造方法

```java
public Semaphore(int permits)
public Semaphore(int permits, boolean fair)
```

* permits表示允许的线程数
* fair表示公平性，如果这个设为true的话，下次执行的线程会是等待最久的线程。

### 2.2 重要方法

```java
public void acquire()throws InterruptedException
public void release()
tryAcquire(int args, long timeout, TimeUnit unit)
```

* acquire()表示阻塞并获取许可
* release()表示释放许可

### 2.3 基本使用

#### 2.3.1 需求场景

> 资源访问，服务限流(Hystrix里限流就有基于信号量的方式)

#### 2.3.2 代码实现

```java
public class SemaphoreRunner {
    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(2);
        for (int i=0;i<5;i++){
            new Thread(new Task(semaphore,"yangguo+"+i)).start();
        }
    }

    static class Task extends Thread{
        Semaphore semaphore;

        public Task(Semaphore semaphore,String tname){
            this.semaphore = semaphore;
            this.setName(tname);
        }

        public void run() {
            try {
                semaphore.acquire();               
                System.out.println(Thread.currentThread().getName()+":aquire() at time:"+System.currentTimeMillis());
                Thread.sleep(1000);
                semaphore.release();               
                System.out.println(Thread.currentThread().getName()+":aquire() at time:"+System.currentTimeMillis());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }
    }
}
```

**打印结果**

```java
Thread-3:aquire() at time:1563096128901
Thread-1:aquire() at time:1563096128901
Thread-1:aquire() at time:1563096129903
Thread-7:aquire() at time:1563096129903
Thread-5:aquire() at time:1563096129903
Thread-3:aquire() at time:1563096129903
Thread-7:aquire() at time:1563096130903
Thread-5:aquire() at time:1563096130903
Thread-9:aquire() at time:1563096130903
Thread-9:aquire() at time:1563096131903
```



## CountDownLatch使用及应用场景列子

### CountDownLatch是什么？

CountDownLatch这个类能够使一个线程等待其他线程完成各自的工作后再执行。例如，应用程序的主线程希望在负责启动框架服务的线程已经启动所有的框架服务之后再执行。

**使用场景：**

> Zookeeper分布式锁，Jmeter模拟高并发等

### CountDownLatch如果工作?

CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就会减1。当计数器值到达0时，它表示所有的线程已经完成了任务，然后在闭锁上等待的线程就可以恢复执行任务。

**API**

```java
CountDownLatch.countDown();
CountDownLatch.await();
```

## CyclicBarrier

栅栏屏障，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。

CyclicBarrier默认的构造方法是CyclicBarrier（int parties），其参数表示屏障拦截的线程数量，每个线程调用await方法告CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。

**API**

```java
cyclicBarrier.await();
```

**应用场景**

可以用于多线程计算数据，最后合并计算结果的场景。例如，用一个Excel保存了用户所有银行流水，每个Sheet保存一个账户近一年的每笔银行流水，现在需要统计用户的日均银行流水，先用多线程处理每个sheet里的银行流水，都执行完之后，得到每个sheet的日均银行流水，最后，再用barrierAction用这些线程的计算结果，计算出整个Excel的日均银行流水。

```java
public class CyclicBarrierRunner implements Runnable {
    private CyclicBarrier cyclicBarrier;
    private int index ;

    public CyclicBarrierTest(CyclicBarrier cyclicBarrier, int index) {
        this.cyclicBarrier = cyclicBarrier;
        this.index = index;
    }

    public void run() {
        try {
            System.out.println("index: " + index);
            index--;
            cyclicBarrier.await();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws Exception {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(11, new Runnable() 					          {
            public void run() {
                System.out.println("所有特工到达屏障，准备开始执行秘密任务");
            }
        });
        for (int i = 0; i < 10; i++) {
            new Thread(new CyclicBarrierTest(cyclicBarrier, i)).start();
        }
        cyclicBarrier.await();
        System.out.println("全部到达屏障....");
    }
}
```

## Exchanger

当一个线程运行到exchange()方法时会阻塞，另一个线程运行到exchange()时，二者交换数据，然后执行后面的程序。

```java
public class ExchangerTest {

    public static void main(String []args) {
        final Exchanger<Integer> exchanger = new Exchanger<Integer>();
        for(int i = 0 ; i < 10 ; i++) {
            final Integer num = i;
            new Thread() {
                public void run() {
                    System.out.println("我是线程：Thread_" + this.getName() + "我的数据是：" + num);
                    try {
                        Integer exchangeNum = exchanger.exchange(num);
                        Thread.sleep(1000);
                        System.out.println("我是线程：Thread_" + this.getName() + "我原先的数据为：" + num + " , 交换后的数据为：" + exchangeNum);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }.start();
        }
    }
}
```

