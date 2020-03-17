# JUC

## 谈谈你对volatile的理解？

1. Volatile 是Java虚拟机提供的`轻量级的同步机制`

   1. 保证可见性
   2. 不保证原子性
   3. 禁止指令重排

2. 谈谈JMM（Java 内存模型）

   1. 首先线程1将主内存的数据拷贝到自己的工作内存中，当数据修改成功后，会将数据写入到主内存
   2. 当主内存的数据改变后，其他线程需及时的更新自己工作内存数据。这时候需要volatile的可见性
   3. <img src="/Users/apple/Desktop/JavaBasic/images/jmm.png" style="zoom: 33%;" />

   ```java
   /**
    * 验证volatile 可见性
    */
   public class VolatileDemo {
   
       public static void main(String[] args) {
   
           MyData myData = new MyData();
   
           new Thread(() -> {
               System.out.println(Thread.currentThread().getName() + "come in ");
               try {
                   TimeUnit.SECONDS.sleep(3);
               } catch (Exception e) {
                   e.printStackTrace();
               }
               myData.addTo60();
               System.out.println(Thread.currentThread().getName() + "\t update number value : " + myData.number);
           }, "AAA").start();
   
           /**
            *
            */
           while (myData.number == 0) {
   
           }
   
           System.out.println(Thread.currentThread().getName() + "\t mission is over， main get number value :\t" + myData.number);
       }
   
   }
   
   class MyData {
       volatile int number = 0;
   
       public void addTo60() {
           this.number = 60;
       }
   }
   ```

   ```java
   **
    * 2:验证volatile 不保证原子性
    * 原子性：不可分割，完整性，也就是某个线程在做某个具体业务时，中间不可以加塞或者分割。要么成功，要么失败
    */
   public class VolatileDemo {
   
       public static void main(String[] args) {
   
           MyData myData = new MyData();
   
           for (int i = 0; i < 20; i++) {
               new Thread(() -> {
   
                   for (int j = 1; j <= 1000; j++) {
                       myData.addPlusPlus();
                   }
               }, String.valueOf(i)).start();
           }
   
           while (Thread.activeCount() > 2) {
               Thread.yield();
           }
   
           System.out.println(Thread.currentThread().getName() + "判断原子性:\t" + myData.number);
       }
   
   class MyData {
       volatile int number = 0;
     
       public void addPlusPlus() {
           this.number++;
       }
   }
   ```

   ```java
   **
    * 2:验证volatile 保证原子性的方案
    * 
    */
   public class VolatileDemo {
   
       public static void main(String[] args) {
   
           MyData myData = new MyData();
   
           for (int i = 0; i < 20; i++) {
               new Thread(() -> {
   
                   for (int j = 1; j <= 1000; j++) {
                       myData.addAtomic();
                   }
               }, String.valueOf(i)).start();
           }
   
           while (Thread.activeCount() > 2) {
               Thread.yield();
           }
   
            System.out.println(Thread.currentThread().getName() + "保证原子性:\t" + myData.atomicInteger);
   
   class MyData {
     
       AtomicInteger atomicInteger = new AtomicInteger();
   
       public void addAtomic() {
           atomicInteger.getAndIncrement();
       }
   }
   ```

3. 为什么`atomicInteger` 可以保证原子性?

   1. CAS 
   2. 自旋锁
   3. unsafe.直接访问内存的类（`native`）

   ```java
   // setup to use Unsafe.compareAndSwapInt for updates
   private static final Unsafe unsafe = Unsafe.getUnsafe();
   private static final long valueOffset;
   
   static {
     try {
       valueOffset = unsafe.objectFieldOffset
         (AtomicInteger.class.getDeclaredField("value"));
     } catch (Exception ex) { throw new Error(ex); }
   }
   private volatile int value;
   ```
   
   

4. 你在哪些地方用到过volatile？

   1. 单例模式DCL
      - DCL(双端检索机制)不一定线程安全，原因是存在指令重排，加入volatile 可以禁止指令重排

## CAS（比较并交换）

1. ~~当线程A中的工作内存中的数据与主内存的数据一样，就修改主内存的数据~~
2. **期望值和更新值、主内存的值和期望值比较，相同就修改成功，不相同就修改失败**

```java
 AtomicInteger atomicInteger = new AtomicInteger(5);
 System.out.println(atomicInteger.compareAndSet(5, 2020)+ "\t" + atomicInteger.get());


/**
 * Atomically increments by one the current value.
 *
 * @return the previous value
 */
public final int getAndIncrement() {
  return unsafe.getAndAddInt(this, valueOffset, 1);
}

public final int getAndAddInt(Object var1, long var2, int var4) {
  int var5;
  do {
    var5 = this.getIntVolatile(var1, var2);
  } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

  return var5;
}
```

* 当5 和主内存的5一样，主内存就改为2020

3. unsafe + CAS （思想）: 自旋锁。

## CAS 缺点

1. 循环时间长开销很大

2. 只能保证一个共享变量的原子操作

3. 引出`ABA`问题？

   - ABA 意思是中间存在猫腻

   ```java
    integerAtomicReference.compareAndSet(100, 101)
    integerAtomicReference.compareAndSet(101, 100);
   ```

4. 解决ABA问题

   - `AtomicStampedReference` 解决ABA问题

5. 原子更新引用

```java
public static void main(String[] args) {
User z3 = new User("z3", 12);
User li4 = new User("li4", 34);
AtomicReference<User> reference = new AtomicReference<>();
reference.set(z3);
System.out.println(reference.compareAndSet(z3, li4) + "\t" + reference.get().toString());
System.out.println(reference.compareAndSet(z3, li4) + "\t" + reference.get().toString());
}
```

```java
static AtomicReference<Integer> integerAtomicReference = new AtomicReference<>(100);

static AtomicStampedReference<Integer> stampedReference = new AtomicStampedReference<>(100, 1);

public static void main(String[] args) {
  System.out.println("================ 以下是ABA问题的产生 ==============");
  new Thread(() -> {
    integerAtomicReference.compareAndSet(100, 101);
    integerAtomicReference.compareAndSet(101, 100);
  }, "t1").start();

  new Thread(() -> {
    try {
      TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
      e.printStackTrace();
    }

    System.out.println(integerAtomicReference.compareAndSet(100, 2020) + "\t" + integerAtomicReference.get());
  }, "t2").start();

  try {
    TimeUnit.SECONDS.sleep(3);
  } catch (Exception e) {
    e.printStackTrace();
  }
  System.out.println("================ 以下是ABA问题的解决 ==============");

  new Thread(() -> {
    int stamp = stampedReference.getStamp();
    System.out.println(Thread.currentThread().getName() + "\t第一次版本号：" + stamp);
    try {
      TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
      e.printStackTrace();
    }
    stampedReference.compareAndSet(100, 101, stampedReference.getStamp(), stampedReference.getStamp() + 1);
    System.out.println(Thread.currentThread().getName() + "\t第二次版本号：" + stampedReference.getStamp());
    stampedReference.compareAndSet(101, 100, stampedReference.getStamp(), stampedReference.getStamp() + 1);
    System.out.println(Thread.currentThread().getName() + "\t第三次版本号：" + stampedReference.getStamp());
  }, "t3").start();


  new Thread(() -> {
    int stamp = stampedReference.getStamp();
    System.out.println(Thread.currentThread().getName() + "\t第一次版本号：" + stamp);
    try {
      TimeUnit.SECONDS.sleep(3);
    } catch (Exception e) {
      e.printStackTrace();
    }
    boolean result = stampedReference.compareAndSet(100, 2020, stamp, stamp + 1);
    System.out.println(Thread.currentThread().getName() + "是否成功修改：" + result + "\t" + "最新版本号：" + stampedReference.getStamp());
    System.out.println(Thread.currentThread().getName() + "当前最新值：" + stampedReference.getReference());
  }, "t4").start();
```

# 集合类不安全的问题

## 单线程arrayList

1. new ArrayList 底层创建的是啥？什么类型的数据？
   - 底层创建的是new Object类型的初始值长度为10数组
2. 数组拷贝

```java
Arrays.copyof()
```

3. 扩容原址的一半，即10 的一半 5 

## 多线程下的ArrayList

1. 故障现象

```
java.util.ConcurrentModificationException
```

2. 导致原因
   * 并发争抢修改
3. 解决方案

```Java
new Vector<>()  // 底层synchronized
Collections.synchronizedList(new ArrayList<>())
new CopyOnWriteArrayList<>(); // 推荐（写实复制）
```

4. 优化建议

## 多线程下的HashSet

 ```java
Set<String> set = new CopyOnWriteArraySet<>(); // 底层用的是CopyOnWriteArrayList
 ```



# 锁

## 公平锁 / 非公平锁

- 并发包中`ReentrantLock`来创建可以指定boolean类型来得到公平锁和非公平锁，默认是非公平锁。

### 公平锁

- 是指多线程按照申请锁的顺序来获取锁，先来后到

### 非公平锁

- 是指多个线程获取锁的顺序并不是根据申请锁的顺序。有可能后申请的线程比申请 线程优先获取锁，在高并发的情况下，有可能造成优先级反转或者饥饿现象
- `Synchronized`默认也是非公平锁

## 可重入锁（递归锁）

- 指的是同一线程外层函数获得锁之后，内层递归函数仍然能获得该锁的代码，在同一个线程在外层方法获得锁的时候，在进入内层方法会自动获取锁， 也即是说:**线程可以进入任何一个它已经拥有锁的同步代码块**

- `Synchronized`, `ReentrantLock`就是典型的可重入锁
- 作用 ：防止死锁 

## 自旋锁

* 

## 独占锁（写锁） / 共享锁（读锁） / 互斥锁

### 独占锁

* 指的是锁一次只能被一个线程所持有，`Synchronized`,`ReentrantLock`就是独占锁

### 共享锁

* 指的是锁被多个线程所持有
* `ReetrantReadWriteLock` 读锁是共享锁，写锁是独占锁
* 读锁的共享锁可保证并发是非常高效的。读写，写读，读读是互斥的

```java
private volatile Map<String, Object> map = new HashMap<>();

private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

public void put(String key, String value) {
  lock.writeLock().lock();
  try {
  	System.out.println(Thread.currentThread().getName() + "\t 正在写入....\t" + key);
  try {
  	TimeUnit.MILLISECONDS.sleep(300);
  } catch (Exception e) {
  	e.printStackTrace();
  }
    map.put(key, value);
    System.out.println(Thread.currentThread().getName() + "\t 写入完成....");
  } finally {
  	lock.writeLock().unlock();
  }
}

public void get(int key) {
  lock.readLock().lock();
  try {
  	System.out.println(Thread.currentThread().getName() + "\t 正在读取....");
  try {
    TimeUnit.MILLISECONDS.sleep(300);
    } catch (Exception e) {
    e.printStackTrace();
    }
    Object value = map.get(key);
    System.out.println(Thread.currentThread().getName() + "\t 读取完成....\t" + value);

  } finally {
  	lock.readLock().unlock();
  }
}
```

## CountDownLatch

```java
 public static void main(String[] args) throws InterruptedException {
   CountDownLatch countDownLatch = new CountDownLatch(6);
   for (int i = 0; i < 6; i++) {
     new Thread(() -> {
       System.out.println(Thread.currentThread().getName() + "\t" + "离开教室");
       countDownLatch.countDown();
     }, String.valueOf(i)).start();
   }
   countDownLatch.await();
   System.out.println(Thread.currentThread().getName() + "\t班长离开教室");
 }
```

## CyclicBarrier

```java
CyclicBarrier cyclicBarrier = new CyclicBarrier(7, () -> {
  System.out.println("召唤神龙");
});

for (int i = 1; i <= 7; i++) {
  final int temp = i;
  new Thread(() -> {
    System.out.println(Thread.currentThread().getName() + "\t收集" + temp + "颗龙珠");
    try {
      cyclicBarrier.await();
    } catch (InterruptedException e) {
      e.printStackTrace();
    } catch (BrokenBarrierException e) {
      e.printStackTrace();
    }
  }, String.valueOf(i)).start();
}
```

## Semaphore

```java
Semaphore semaphore = new Semaphore(3); //模拟资源类，有3个空车位
for (int i = 0; i < 6; i++) {
  new Thread(() -> {
    try {
      semaphore.acquire();
      System.out.println(Thread.currentThread().getName() + "\t抢占了车位");
      TimeUnit.SECONDS.sleep(3);
      System.out.println(Thread.currentThread().getName() + "\t离开了车位");
    } catch (InterruptedException e) {
      e.printStackTrace();
    } finally {
      semaphore.release();
    }
  }, String.valueOf(i)).start();
}
```

# 队列

### 阻塞队列

* 当阻塞队列满时，添加元素的操作会被堵塞
* 当阻塞队列空时，获取元素的操作会被堵塞

1. 阻塞队列有没有好的一面
2. 不得不堵塞，你如何管理

### ArrayBlockingQueue:

* 由数组结构组成的有界阻塞队列                           

### LinkedBlockingQueue  

* 由链表结构组成的有界（默认大小Integer.MAX_VALUE）阻塞队列    

### priorityBlockingQueue 

* 支持优先级排序的无界阻塞队列 

### SynchronousQueue 

* 不存储元素的阻塞队列，也即单个元素的队列                     

### LinkedTransferQueue   

* 由链表结构组成的无界阻塞队列                           

### LinkedBlockingDeque  

* 由链表结构结构组成的双向阻塞队列  

### DelayQueue       

* 使用优先级队列实现的延迟无界阻塞队列     

|  A   | 抛出异常 |  特殊值  |  阻塞  |        超时        |
| :--: | :------: | :------: | :----: | :----------------: |
| 插入 |  add(e)  | offer(e) | put()  | offer(e,time,unit) |
| 移除 | remove() |  poll()  | take() |  poll(time, unit)  |
| 检查 | element  |  peek()  | 不可用 |       不可用       |



| 抛出异常 | B                                                            |
| :------- | :----------------------------------------------------------- |
| 特殊值   | 插入方法：成功返回true。失败返回false。移除方法：成功返回队列中的元素，队列没有返回null |
| 一直阻塞 |                                                              |
|          |                                                              |



# 面试

1. `Synchronized`和`lock`的区别？

* `Synchronized`属于JVM层面，底层通过monitor对象完成 **monitorenter、monitorexit**
* `lock` 具体类属于API层面
* `Synchronized`不需要手动释放锁
* `lock`需要手动释放锁，有可能出现死锁的情
* `Synchronized` 非公平锁。lock可以时公平锁也可以是非公平锁

* `Lock` 可以绑定多个condition ，synchronized不可以

# 多线程

* 实现多线程的方式？

1. `new Thread`
2. `implements Runnable`
3. `callable`
4. `FutureTask`

## Callable

```java
public class CallableDemo2 {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask<Integer> task = new FutureTask<>(new Thread2());
        new Thread(task).start();
//        while (!task.isDone()) {
////
////        }
        System.out.println(task.get()); //放在最后。因为它会等待线程的执行完、导致其他线程等待
    }
}

class Thread2 implements Callable<Integer> {

    @Override
    public Integer call() throws Exception {
        TimeUnit.SECONDS.sleep(2);
        return 1024;
    }
}

class Thread1 implements Runnable {

    @Override
    public void run() {

    }
}
```

```java
class MyResource {

    private volatile boolean flag = true;//默认开启。生产+消费
    private AtomicInteger atomicInteger = new AtomicInteger();
    BlockingQueue<String> blockingQueue = null;

    public MyResource(BlockingQueue<String> blockingQueue) {
        this.blockingQueue = blockingQueue;
        System.out.println("name: \t" + blockingQueue.getClass().getName()); //利用反射获取传过来的是哪个类
    }

    public void myProd() throws Exception {
        String data = null;
        boolean resValue;
        while (flag) {
            data = atomicInteger.incrementAndGet() + "";
            resValue = blockingQueue.offer(data, 2L, TimeUnit.SECONDS);
            if (resValue) {
                System.out.println(Thread.currentThread().getName() + "\t 插入队列 \t" + data + "\t success");
            } else {
                System.out.println(Thread.currentThread().getName() + "\t 插入队列 \t" + data + "\t fair");
            }
            TimeUnit.SECONDS.sleep(1);
        }

        System.out.println(Thread.currentThread().getName() + "\t大老板叫停了， 表示flag = false ,生产动作结束");
    }

    public void myConsumer() throws Exception {
        String data = null;
        while (flag) {
            data = blockingQueue.poll(2L, TimeUnit.SECONDS);
            if (null == data || data.equalsIgnoreCase("")) {
                flag = false;
                System.out.println(Thread.currentThread().getName() + "\t超过2秒没有取到，消费退出");
            }
            System.out.println(Thread.currentThread().getName() + "\t消费队列\t" + data + "\t success");
        }
    }

    public void stop() {
        this.flag = false;
    }

}

/**
 * <h2>validate/CAS/atomicInteger/BlockQueue/线程交互</h2>
 * <h2>v3.0版 生产者、消费者</h2>
 */
public class ProConsumer_BlockQueue {

    public static void main(String[] args) {
        MyResource myResource = new MyResource(new ArrayBlockingQueue<>(10));
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t生产线程启动");
            try {
                myResource.myProd();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }, "prod").start();

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t消费线程启动");
            try {
                myResource.myConsumer();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }, "Consumer").start();

        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println();
        System.out.println();
        System.out.println();
        System.out.println();

        myResource.stop();
    }
}
```

# 线程池

* 线程池的底层是`ThreadPoolExecutor` 
  *  ExecutorService threadPool = Executors.newFixedThreadPool(5);
  * ExecutorService threadPool = Executors.newSingleThreadExecutor();
  * ExecutorService threadPool = Executors.newCachedThreadPool();

```java
public ThreadPoolExecutor(int corePoolSize, // 常驻核心线程数
  int maximumPoolSize,					//线程池能够容纳最大的核心线程数
  long keepAliveTime,					//多余的空闲线程的存活时间
  TimeUnit unit,						// keepAliveTime 的时间单位
  BlockingQueue<Runnable> workQueue,	//任务队列
  ThreadFactory threadFactory,			//	线程工厂。创建线程的方式 一般默认
  RejectedExecutionHandler handler) {	//拒绝策略
```

* 底层工作原理？

- 拒接策略

```java
// 拒绝策略
- new ThreadPoolExecutor.AbortPolicy()
- new ThreadPoolExecutor.CallerRunsPolicy()
- new ThreadPoolExecutor.DiscardOldestPolicy()
- new ThreadPoolExecutor.DiscardPolicy() 直接丢包
```



## 面试： 怎么配置 线程池能够容纳最大的核心线程数

- CPU 密集型 	线程核心数 + 1
- IO密集型 		任务线程不是一直在执行任务	线程核心数 * 2
- IO 密集型 		大部分线程被堵塞 			CPU核数 / (1-阻塞系数)  阻塞系数0.8～0.9 如: 8核CPU /（1-0.9）= 80

# 死锁

* Jps命令定位进程号
* jStack找到死锁查看

```java
jps -l
jstack -pid //进程号
```



# JVM 体系结构

## JVM调优

* jsp -l
* jinfo -flag Metaspaces 
* -Xms -> -XX:initialHeapSIze.      -Xmx -> -XX:MaxHeapSize
* java -XX:+PrintFlagsInitial 查看参数盘点家底

