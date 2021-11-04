# Juc并发编程



## 线程和进程

> 线程和进程

 

进程：一个程序的集合

一个进程往往可以包含多个线程，至少包含一个；

线程：开了一个进程，之后多线程处理

```java
    public synchronized void start() {
        /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        group.add(this);

        boolean started = false;
        try {
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }
	// 调用本地的c++方法，开启线程，java不能直接开启，运行在虚拟机上
    private native void start0();
```

> 并发和并行

并发(多线程操作同一个资源)：

- 抢票
- cpu一核，模拟出来多条线程，天下武功，唯快不破，快速交替

并行(多人个一起行走)：

- cpu多核，多个线程一同执行

```java
package com.jxau.demo01;

public class Test01 {
    public static void main(String[] args) {
        /*new Thread().start();*/

        System.out.println(Runtime.getRuntime().availableProcessors());
    }
}

```



并发编程的本质：提高效率，充分利用CPU的资源



> 线程有几个状态

```java
public enum State {
       
        NEW,

        RUNNABLE,

        BLOCKED,
        
        WAITING,

        TIMED_WAITING,

        TERMINATED;
    }
```

> wait/sleep的区别



![image-20211015111710883](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211015111710883.png)







![image-20211015112631724](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211015112631724.png)



### Lock锁

> synchronized

```java
package com.jxau.demo01;

public class SynchronizedTest {

    public static void main(String[] args) {
        Ticklet ticklet=new Ticklet();
        new Thread(()->{for(int i=0;i<50;i++) ticklet.saleTicklet();},"A").start();
        new Thread(()->{for(int i=0;i<50;i++) ticklet.saleTicklet();},"B").start();
        new Thread(()->{for(int i=0;i<50;i++) ticklet.saleTicklet();},"C").start();
    }
}

class Ticklet{

    private static int nums=50;

    public synchronized void saleTicklet(){
        if(nums>0) System.out.println(Thread.currentThread().getName()+"卖出了一张还剩"+(--nums)+"张票");
    }
}

```

**lock**

![image-20211015120336486](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211015120336486.png)

![image-20211015120614869](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211015120614869.png)

公平锁:十分公平:可以先来后到

非公平锁:十分不公平:可以插队



使用onglock的步骤：

![image-20211015121244548](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211015121244548.png)





> Synchronized 和LOck 区别

1、Synchronized内置的Java关键字，Lock是一个Java类

2、Synchronized无法判断获取锁的状态，Lock可以判断是否获取到了锁

3、Synchronized 会自动释放锁，lock必须要手动释放锁!如果不释放锁，**死锁**

4、Synchronized线程1(获得锁，阻塞)、线线程2(等待，傻傻的等） ; Lock锁就不一定会等待下去;

5、Synchronized可重入锁，不可以中断的，非公平;Lock，可重入锁，可以判断锁，非公平(可以自己设置) ;

6、Synchronized适合锁少量的代码同步问题，Lock适合锁大量的同步代码!





### 生产者和消费者问题 

判断等待、业务、通知唤醒

![image-20211015125711844](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211015125711844.png)

```java
package com.jxau.pc;

public class Test01 {

    public static void main(String[] args) {
        Data data=new Data();

        new Thread(()->{
            for(int i=0;i<10;i++){
                data.doProduct();
            }
        },"product").start();
        new Thread(()->{
            for(int i=0;i<10;i++){
                data.doConsumer();
            }
        },"consumer").start();
        new Thread(()->{
            for(int i=0;i<10;i++){
                data.doProduct();
            }
        },"c").start();
        new Thread(()->{
            for(int i=0;i<10;i++){
                data.doConsumer();
            }
        },"d").start();

    }
}

/*
* 生产者模式与消费者模式三部曲
* 1、判断等待
* 2、业务功能
* 3、通知唤醒
* */

class Data{
    private int num=0;
    public synchronized void doProduct(){
        while(num!=0){
            try {
                this.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        num++;
        System.out.println(Thread.currentThread().getName()+"生产之后为"+num);
        this.notifyAll();

    }
    public synchronized void doConsumer(){
        while(num==0){
            try {
                this.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        num--;
        System.out.println(Thread.currentThread().getName()+"消费之后为"+num);
        this.notifyAll();

    }

}

```





> A、b、c、d四个线程可能出现的问题

![image-20211015130646484](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211015130646484.png)

==要使用while==



> juc版的生产者与消费者问题
>
> 使用lock condition.await() condition.singal()
>
> 通过look找到condition

```java
package com.jxau.pc;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Test02 {

    public static void main(String[] args) {
        Data1 data=new Data1();

        new Thread(()->{
            for(int i=0;i<10;i++){
                data.doProduct();
            }
        },"product").start();
        new Thread(()->{
            for(int i=0;i<10;i++){
                data.doConsumer();
            }
        },"consumer").start();
        new Thread(()->{
            for(int i=0;i<10;i++){
                data.doProduct();
            }
        },"c").start();
        new Thread(()->{
            for(int i=0;i<10;i++){
                data.doConsumer();
            }
        },"d").start();

    }
}

/*
 * 生产者模式与消费者模式三部曲
 * 1、判断等待
 * 2、业务功能
 * 3、通知唤醒
 * */

class Data1{
    // 使用lock锁的版本
    Lock lock=new ReentrantLock();
    Condition condition = lock.newCondition();
    
    private int num=0;
    public  void doProduct(){


        try{
            lock.lock();
            while(num!=0){
                condition.await();
            }
            num++;
            System.out.println(Thread.currentThread().getName()+"生产之后为"+num);
            condition.signalAll();

        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }



    }
    public  void doConsumer(){


        try{
            lock.lock();
            while(num==0){
               condition.await();
            }
            num--;
            System.out.println(Thread.currentThread().getName()+"消费之后为"+num);
            condition.signalAll();

        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }





    }


}
```

**任何一个新的技术，绝对不仅仅只是覆盖了原来的技术，优势和补充！

> Condition精准的通知和唤醒线程

```java
package com.jxau.pc;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Test03 {
    public static void main(String[] args) {
        Data3 data3=new Data3();
        new Thread(()->{
            for(int i=0;i<10;i++){
                data3.printfA();
            }
        },"A").start();
        new Thread(()->{
            for(int i=0;i<10;i++){
                data3.printfB();
            }
        },"B").start();
        new Thread(()->{
            for(int i=0;i<10;i++){
                data3.printfC();
            }
        },"C").start();
    }
}


/*
 * 生产者模式与消费者模式三部曲
 * 1、判断等待
 * 2、业务功能
 * 3、通知唤醒
 * */

class Data3{
    // 使用lock锁的版本
    Lock lock=new ReentrantLock();
    Condition condition1 = lock.newCondition();
    Condition condition2 = lock.newCondition();
    Condition condition3= lock.newCondition();
    int num=1;
   public void printfA(){
       // 判断何时等待、业务、唤醒通知
       lock.lock();
       try{
           while(num!=1){
               condition1.await();
           }
           System.out.println(Thread.currentThread().getName()+"->AAAAAAA");
           num=2;
           condition2.signal();
       }catch (Exception e){
           e.printStackTrace();
       }finally {
           lock.unlock();
       }

   }

    public void printfB(){
       lock.lock();
       try{
            while(num!=2){
                condition2.await();
            }
           System.out.println(Thread.currentThread().getName()+"->BBBBBBB");
            num=3;
            condition3.signal();
        }catch (Exception e){
            e.printStackTrace();}
        finally {
            lock.unlock();
        }
    }

    public void printfC(){
        lock.lock();
        try{
            while(num!=3){
                condition3.await();
            }
            System.out.println(Thread.currentThread().getName()+"->CCCCCC");
            num=1;
            condition1.signal();
        }catch (Exception e){
            e.printStackTrace();}
        finally {
            lock.unlock();
        }
    }


}
```



### 八锁问题

**synchronized锁的对象时方法的调用者！**



一个静态方法锁，一个普通方法锁

![image-20211015190805003](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211015190805003.png)



### 集合类不安全



![image-20211015194604443](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211015194604443.png)



hashset的底层

```java
  public HashSet() {
        map = new HashMap<>();
    }

// add
 public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
```

![image-20211015203259471](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211015203259471.png)



### Runnable

![image-20211015205746439](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211015205746439.png)



```java
package com.jxau.callable;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

public class Test01 {

    public static void main(String[] args) throws ExecutionException, InterruptedException {

        FutureTask futureTask=new FutureTask(new MyCallable());

        Thread thread = new Thread(futureTask);
        thread.start();
        Integer o =(Integer) futureTask.get();
        System.out.println(o);

    }

}

class MyCallable implements Callable<Integer>{
    @Override
    public Integer call() throws Exception {
        System.out.println("call()");
        return 1024;
    }
}

```



### 常见的辅助类（可以解决高并发限流）

#### CountDownLatch

```java
package com.jxau.count;

import java.util.concurrent.CountDownLatch;

public class CountDownLatchTest {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch=new CountDownLatch(6);
        for(int i=1;i<=8;i++){

            new Thread(()->{
                System.out.println(Thread.currentThread().getName()+"同学已经走了");
                countDownLatch.countDown();// -1
            },String.valueOf(i)).start();
        }
        countDownLatch.await();// 等待计数器归零，再往下执行
        System.out.println("同学已经走完了");
    }
}

```

原理：

` countDownLatch.countDown();`  数量-1

` countDownLatch.await(); `等待计数器归零，再往下执行

#### CyclicBarrier 

```java
package com.jxau.count;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierTest {
    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier=new CyclicBarrier(7,()->{
            System.out.println("集齐七颗龙珠，召唤神龙！");
        });

        for(int i=0;i<7;i++){
            int finalI = i;
            new Thread(()->{
                System.out.println("收集了"+ finalI +"星龙珠");
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}

```



#### Semaphore

```java
package com.jxau.count;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class SemaphoreTest {

    public static void main(String[] args) {
        Semaphore semaphore=new Semaphore(3); // 信号量，初始为3

        for(int i=1;i<=6;i++){

            new Thread(()->{
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName()+"占用车位");
                    TimeUnit.SECONDS.sleep(2);
                    System.out.println(Thread.currentThread().getName()+"离开车位");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    semaphore.release();// 释放
                }
            },String.valueOf(i)).start();


        }
    }
}

```

原理：

`semaphore.acquire();`获得，假设如果已经满了，进行等待，等待到被释放为止

`  semaphore.release();`释放，会将当前的信号量释放+1，然后唤醒等待的线程

区别与联系：

CyclicBarrier: 指定个数线程执行完毕后在执行操作

Semaphore: 同一时间只能有指定数量个得到线程



### 读写锁

`ReadWriteLock`

独占锁（写锁）’一次只能被一个线程占有

共享锁（读锁)多个线程可以同时占有

ReadwriteLock 

*读-读可以共存!*

*读-写不能共存!*

写-写不能共存!

```java
package com.jxau.rw;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteTest {

    public static void main(String[] args) {

        Data data=new Data();

        for(int i=1;i<=5;i++){
            int finalI = i;
            new Thread(()->{
                Object o=new Object();
                data.write(String.valueOf(finalI),o);
            },String.valueOf(finalI)).start();
        }


        for(int i=1;i<=5;i++){
            int finalI = i;
            new Thread(()->{

                data.read(String.valueOf(finalI));
            },String.valueOf(finalI)).start();
        }

    }
}

class Data{
    // 加上volatile保证原子性
    private volatile Map<String ,Object> map=new HashMap<>();
    private ReadWriteLock readWriteLock=new ReentrantReadWriteLock();



    public void write(String key,Object o){
        readWriteLock.writeLock().lock();
        try{
            System.out.println(Thread.currentThread().getName()+"写入了"+key);
            map.put(key,o);
            System.out.println(Thread.currentThread().getName()+"写入完毕");

        }catch (Exception e){
            e.printStackTrace();
        }finally {
            readWriteLock.writeLock().unlock();
        }

    }

    public void read(String key){
        readWriteLock.readLock().lock();
        try{
            System.out.println(Thread.currentThread().getName()+"读取"+key);
            Object o = map.get(key);
            System.out.println(Thread.currentThread().getName()+"读取完毕");
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            readWriteLock.readLock().unlock();
        }
    }

}

```



### 阻塞队列

![image-20211016123442892](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211016123442892.png)



什么情况下我们会使用 阻塞队列：多线程并发处理，线程池！

#### 四组常用API

**学会使用队列**

添加、移除

1、抛出异常

2、不会抛出异常

3、阻塞 等待

4、超时 等待

| 方式         | 抛出异常  | 有返回值，不抛出异常 | 阻塞 等待 | 超时等待  |
| ------------ | --------- | -------------------- | --------- | --------- |
| 添加         | add()     | offer()              | put()     | offer(,,) |
| 移除         | remove()  | poll()               | take()    | poll()    |
| 检测队首元素 | element() | peek()               | -         | -         |

```java
 /*
    * 抛出异常
    * */
    public static void test01(){

        ArrayBlockingQueue<Integer> arrayBlockingQueue=new ArrayBlockingQueue<Integer>(3);
        System.out.println(arrayBlockingQueue.add(1));
        System.out.println(arrayBlockingQueue.add(2));
        System.out.println(arrayBlockingQueue.add(3));
        // System.out.println(arrayBlockingQueue.add(4));
        System.out.println(arrayBlockingQueue.element());

        System.out.println(arrayBlockingQueue.remove());
        System.out.println(arrayBlockingQueue.remove());
        System.out.println(arrayBlockingQueue.remove());
        System.out.println(arrayBlockingQueue.remove());

    }
```

```java
 /*
     * 有返回值，不抛出异常
     * */
    public static void test02(){

        ArrayBlockingQueue<Integer> arrayBlockingQueue=new ArrayBlockingQueue<Integer>(3);

        System.out.println(arrayBlockingQueue.offer(1));
        System.out.println(arrayBlockingQueue.offer(2));
        System.out.println(arrayBlockingQueue.offer(3));
        System.out.println(arrayBlockingQueue.offer(4));
        System.out.println("==============");
        System.out.println(arrayBlockingQueue.peek());
        System.out.println(arrayBlockingQueue.poll());
        System.out.println(arrayBlockingQueue.poll());
        System.out.println(arrayBlockingQueue.poll());
        System.out.println(arrayBlockingQueue.poll());



    }

```

```java
    /*
     * 堵塞 等待
     * */
    public static void test03(){

        ArrayBlockingQueue<Integer> arrayBlockingQueue=new ArrayBlockingQueue<Integer>(3);

        try {
            arrayBlockingQueue.put(1);
            arrayBlockingQueue.put(2);
            arrayBlockingQueue.put(3);
            //arrayBlockingQueue.put(1);
            System.out.println("===========");
            System.out.println(arrayBlockingQueue.take());
            System.out.println(arrayBlockingQueue.take());
            System.out.println(arrayBlockingQueue.take());
            System.out.println(arrayBlockingQueue.take());

        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
    }
```

```java
    /*
     * 超时等待
     * */
    public static void test04(){

        ArrayBlockingQueue<Integer> arrayBlockingQueue=new ArrayBlockingQueue<Integer>(3);

        try {
            System.out.println(arrayBlockingQueue.offer(1, 2, TimeUnit.SECONDS));
            System.out.println(arrayBlockingQueue.offer(2, 2, TimeUnit.SECONDS));
            System.out.println(arrayBlockingQueue.offer(3, 2, TimeUnit.SECONDS));
            System.out.println(arrayBlockingQueue.offer(4, 2, TimeUnit.SECONDS));
            System.out.println("================");
            System.out.println(arrayBlockingQueue.poll(2, TimeUnit.SECONDS));
            System.out.println(arrayBlockingQueue.poll(2, TimeUnit.SECONDS));
            System.out.println(arrayBlockingQueue.poll(2, TimeUnit.SECONDS));
            System.out.println(arrayBlockingQueue.poll(2, TimeUnit.SECONDS));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
```



#### SynchronousQueue 

同步队列 和其他的`BlockQueue`不一样，`SynchronousQueue` 不存储元素put了一个元素，必须从里面先take取出来，否则不能在put进去值！

```java
package com.jxau.queue;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.TimeUnit;

/**
 * 同步队列 和其他的BlockQueue不一样，SynchronousQueue 不存储元素
 * put了一个元素，必须从里面先take取出来，否则不能在put进去值！
 */
public class SynchronousQueueTest {
    public static void main(String[] args) {

        BlockingQueue<String> synchronousQueue=new SynchronousQueue<>();



            new Thread(()->{

                try {
                    System.out.println(Thread.currentThread().getName()+"put "+ "t1");
                    synchronousQueue.put("t1");
                    System.out.println(Thread.currentThread().getName()+"put "+ "t2");
                    synchronousQueue.put("t2");
                    System.out.println(Thread.currentThread().getName()+"put "+ "t3");
                    synchronousQueue.put("t3");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            },"I").start();



            new Thread(()->{
                try {
                    TimeUnit.SECONDS.sleep(2);
                    String take = synchronousQueue.take();
                    System.out.println(Thread.currentThread().getName()+"get "+ take);
                    TimeUnit.SECONDS.sleep(2);
                     take = synchronousQueue.take();
                    System.out.println(Thread.currentThread().getName()+"get "+ take);
                    TimeUnit.SECONDS.sleep(2);
                     take = synchronousQueue.take();
                    System.out.println(Thread.currentThread().getName()+"get "+ take);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            },"You").start();

    }
}

```





### 线程池

线程池：三大方法、7大参数、四种拒绝策略

> 池化技术

程序的运行，本质：占用系统的资源！优化资源的使用！=》池化技术

线程池、连接池、内存池、对象池 ///..... 创建、销毁。十分浪费资源

池化技术：事先准备好一些资源，有人要用，就来我这里拿，用完之后还给我。



线程池的好处：

1、降低资源的消耗

2、提高响应的速度

3、方便管理

==线程复用、可以控制最大并发数、管理线程==



> 线程池：三大方法

![image-20211016162205772](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211016162205772.png)

```java
package com.jxau.ThreadPool;

import java.util.concurrent.Executor;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

// Executors 工具类、3大方法
public class Test01 {
    public static void main(String[] args) {
        //ExecutorService executorService = Executors.newSingleThreadExecutor();// 创建单一线程池
        //ExecutorService executorService = Executors.newCachedThreadPool();// 遇强则强、遇弱则弱
ExecutorService executorService = Executors.newFixedThreadPool(5);// 创建合适的线程池
        try{
            for(int i=0;i<100;i++){
                executorService.execute(()->{
                    System.out.println(Thread.currentThread().getName());
                });
            }

        }catch (Exception e){
            e.printStackTrace();
        }finally {
            executorService.shutdown();// 关闭线程池
        }

    }
}

```

源码分析：

```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }


 public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

// 七大核心参数
public ThreadPoolExecutor(int corePoolSize,// 核心线程大小
                              int maximumPoolSize,// 最大核心线程池大小
                              long keepAliveTime,// 超时了没有人调用就会释放
                              TimeUnit unit,// 超时单位
                              BlockingQueue<Runnable> workQueue,// 阻塞队列
                              ThreadFactory threadFactory,// 线程工厂：创建线程，一般不用动
                              RejectedExecutionHandler handler// 拒绝策略) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```



![image-20211016155143712](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211016155143712.png)

> 手动创建线程池

```java
package com.jxau.ThreadPool;

import java.util.concurrent.LinkedBlockingDeque;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;


public class Test02 {
    public static void main(String[] args) {
        // 手动创建一个线程池
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(2,
                                                                   5,
                                                                      3,
                                                                        TimeUnit.SECONDS,
                                                new LinkedBlockingDeque<>(3),
                                                new ThreadPoolExecutor.DiscardPolicy()// 队列满了，丢掉任务，不会抛出异常
                                                                        );
        try{
            for(int i=0;i<10;i++){
                threadPoolExecutor.execute(()->{
                    System.out.println(Thread.currentThread().getName());
                });
            }

        }catch (Exception e){
            e.printStackTrace();
        }finally {
            threadPoolExecutor.shutdown();// 关闭线程池
        }
    }
}

```



> 四种拒绝策略

![image-20211016161257429](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211016161257429.png)

```java
/*
*   new ThreadPoolExecutor.AbortPolicy() // 银行满了，还有人进来，不处理这个人的，抛出异常
*   new ThreadPoolExecutor.CallerRunsPolicy()// 哪来的去哪里
*   new ThreadPoolExecutor.DiscardOldestPolicy()// 队列满了，尝试去和最早的竞争，也不会抛出异常
*   new ThreadPoolExecutor.DiscardPolicy()// 队列满了，丢掉任务，不会抛出异常
*/
```



> 小结和扩展

池的最大的大小如何去设置

了解：IO密集型，CPU密集型：（调优）

```java
//最大线程该如何定义

//1、CPU 密集型 几核 就是几 保持CPU的效率最高

//2、IO 密集型 > 判断你程序中十分耗IO的线程

//3、程序 15个大型任务 io十分占用资源

package com.jxau.ThreadPool;

import java.util.concurrent.LinkedBlockingDeque;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/*
*   new ThreadPoolExecutor.AbortPolicy() // 银行满了，还有人进来，不处理这个人的，抛出异常
*   new ThreadPoolExecutor.CallerRunsPolicy()// 哪来的去哪里
*   new ThreadPoolExecutor.DiscardOldestPolicy()// 队列满了，尝试去和最早的竞争，也不会抛出异常
*   new ThreadPoolExecutor.DiscardPolicy()// 队列满了，丢掉任务，不会抛出异常
*/

public class Test02 {
    public static void main(String[] args) {

        int i1 = Runtime.getRuntime().availableProcessors(); // 获得运行时的机子最大cpu核数
        /*System.out.println(i1);*/
        

        // 手动创建一个线程池
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(2,
                                                                      i1,
                                                                       3,
                                                         TimeUnit.SECONDS,
                                                new LinkedBlockingDeque<>(3),
                                                new ThreadPoolExecutor.DiscardPolicy()// 队列满了，丢掉任务，不会抛出异常
                                                                        );
        try{
            for(int i=0;i<10;i++){
                threadPoolExecutor.execute(()->{
                    System.out.println(Thread.currentThread().getName());
                });
            }

        }catch (Exception e){
            e.printStackTrace();
        }finally {
            threadPoolExecutor.shutdown();// 关闭线程池
        }
    }
}

```



### 函数式接口

lambda表达式、链式编程、函数式接口、stream流式计算

> function函数式接口

```java
package com.jxau.function;

import java.util.function.Function;

public class Test01 {

    public static void main(String[] args) {
        Function<String, String> function = new Function<String, String>() {
            @Override
            public String apply(String s) {
                return s;
            }
        };
        Function<String, String> function1=(s)->{return s;}; // lamdand 表达式简化
        System.out.println(function.apply("123"));
    }
}

```

> Predicate 断定型接口

```java
package com.jxau.function;

import java.util.function.Predicate;

public class Test02 {
    public static void main(String[] args) {
        Predicate <String> predicate=new Predicate<String>() {
            @Override
            public boolean test(String s) {
                return s.isEmpty();
            }
        };
        System.out.println(predicate.test("123"));
    }
}

```

> consumer 消费型接口

![image-20211018165234875](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211018165234875.png)

```java
package com.jxau.function;

import java.util.function.Consumer;

public class Test03 {


    public static void main(String[] args) {
        // 消费性接口
/*       Consumer<Object> c= new Consumer<Object>() {
           @Override
            public void accept(Object o) {
                System.out.println(o);
            }
        };*/

        Consumer<String> c=(res)->{ System.out.println(res);};
       c.accept("123");

    }
}

```



> supply 供给型接口

![image-20211018165746097](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211018165746097.png)

```java
package com.jxau.function;

import java.util.function.Supplier;

public class Test04 {

    public static void main(String[] args) {
     /*   Supplier<String> ss= new Supplier<String>(){
            @Override
            public String get() {
                return "123";
            }
        };*/

        Supplier<String> ss=()->{return "123";};
        System.out.println(ss.get());
    }
}

```



### 流式计算

```java
package com.jxau.stream;

import java.util.Arrays;
import java.util.List;

public class Test01 {

    /*
    * stream流式计算
    *   只用一行代码完成下列要求：
    * 1、倒序输出名字
    * 3、名字字母大写
    * 4、id为偶数的
    * 5、年龄大于23的
    *
    * */
    public static void main(String[] args) {

        User u1=new User(1,"a",20);
        User u2=new User(2,"b",21);
        User u3=new User(3,"c",22);
        User u4=new User(4,"d",23);
        User u5=new User(5,"e",24);
        User u6=new User(6,"f",25);
        List<User> users = Arrays.asList(u1, u2, u3, u4, u5, u6);
        users.stream()
                .filter((u)->{ return u.getId()%2==0;})
                .filter((u)->{return u.getAge()>23;})
                .map((u)->{return u.getName().toUpperCase();})// 函数式接口，可输入可输出
                .sorted((uu1,uu2)->{return uu1.compareTo(uu2);})
                .limit(1)
                .forEach(System.out::println);


    }

}

```



### ForkJoin

> 什么是ForkJoin

ForkJoin在JDK1.7，并行执行任务！提高效率。大数据量！

大数据：Map Reduce(把大任务拆分为小任务)



> ForkJoin 特点：工作窃取

![image-20211018191734652](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211018191734652.png)







### 异步回调

![image-20211018195300724](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211018195300724.png)



没有返回值的异步回调

![image-20211018200929979](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211018200929979.png)

```java
package com.jxau.async;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;

public class Test01 {

    public static void main(String[] args) {

        CompletableFuture<Void> completableFuture= CompletableFuture.runAsync(()->{

            System.out.println(Thread.currentThread().getName()+"runAsync");
            System.out.println("123");

        });
        try {
            completableFuture.get();// 执行方法获得返回值
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        System.out.println("1111");
    }
}

```



有返回值的异步回调

![image-20211018201140539](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211018201140539.png)

```java
package com.jxau.async;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;

public class Test02 {
    public static void main(String[] args) {
        CompletableFuture<String> completableFuture=CompletableFuture.supplyAsync(()->{

            System.out.println(Thread.currentThread().getName()+"supplyAsync");
            //int i=10/0;
            return "123";
        });

        try {
            System.out.println(completableFuture.whenCompleteAsync((u1, u2) -> {

                System.out.println("u1=" + u1);// 输出正常的返回值
                System.out.println("u2=" + u2);// 输出错误信息 java.util.concurrent.CompletionException: java.lang.ArithmeticException: / by zero

            }).exceptionally((e)->{
                System.out.println(e);
                return e.getMessage();
            }).get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}

```

### JMM

> 谈谈你对Volatile的理解

volatile是java虚拟机提供**轻量级的同步机制**

1、保证可见性

2、不保证原子性

3、禁止指令重排



> 什么是JMM

JMM：java内存模型，不存在的东西，概念！约定！



**关于JMM的一些同步的约定**：

1、线程解锁前，必须把共享变量**立刻**刷回主存。

2、线程加锁前，必须读取主存中的最新值到工作内存中！

3、加锁和解锁是同一把锁



![image-20211019092047757](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211019092047757.png)

**内存交互操作有8种，虚拟机实现必须保证每一个操作都是原子的，不可在分的（对于double和long类型的变量来说，load、store、read和write操作在某些平台上允许例外）**

- lock   （锁定）：作用于主内存的变量，把一个变量标识为线程独占状态

- unlock （解锁）：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定
- read  （读取）：作用于主内存变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的load动作使用
- load   （载入）：作用于工作内存的变量，它把read操作从主存中变量放入工作内存中
- use   （使用）：作用于工作内存中的变量，它把工作内存中的变量传输给执行引擎，每当虚拟机遇到一个需要使用到变量的值，就会使用到这个指令
- assign （赋值）：作用于工作内存中的变量，它把一个从执行引擎中接受到的值放入工作内存的变量副本中
- store  （存储）：作用于主内存中的变量，它把一个从工作内存中一个变量的值传送到主内存中，以便后续的write使用
- write 　（写入）：作用于主内存中的变量，它把store操作从工作内存中得到的变量的值放入主内存的变量中

　　**JMM对这八种指令的使用，制定了如下规则：**

- 不允许read和load、store和write操作之一单独出现。即使用了read必须load，使用了store必须write

- 不允许线程丢弃他最近的assign操作，即工作变量的数据改变了之后，必须告知主存
- 不允许一个线程将没有assign的数据从工作内存同步回主内存
- 一个新的变量必须在主内存中诞生，不允许工作内存直接使用一个未被初始化的变量。就是怼变量实施use、store操作之前，必须经过assign和load操作
- 一个变量同一时间只有一个线程能对其进行lock。多次lock后，必须执行相同次数的unlock才能解锁
- 如果对一个变量进行lock操作，会清空所有工作内存中此变量的值，在执行引擎使用这个变量前，必须重新load或assign操作初始化变量的值
- 如果一个变量没有被lock，就不能对其进行unlock操作。也不能unlock一个被其他线程锁住的变量
- 对一个变量进行unlock操作之前，必须把此变量同步回主内存



#### Volatile

> 保证可见性

```java
package com.jxau.tvolatile;

import java.util.concurrent.TimeUnit;

public class tvolatile {
     public volatile static int num=0;
    public static void main(String[] args) {


        new Thread(()->{
            while(num==0){

            }
        }).start();
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        num=1;
        System.out.println(num);
    }
}

```

> 2、不保证原子性

原子性：不可分割

线程A在执行任务的时候，不能被打扰的，也不能被分割。要么同时成功，要么同时失败。

```java
package com.jxau.tvolatile;

public class Test01 {

    private  volatile   static  int num=0;
    public  static void add(){
        num++;
    }
    public  static void main(String[] args) {

        for(int i=0;i<20;i++){
            new Thread(()->{

                for(int j=0;j<1000;j++){

                    add();
                }

            }).start();
        }

        while(Thread.activeCount()>2){// main、gc线程
            Thread.yield();// 线程礼让

        }
        System.out.println(num);

    }
}

```

**如果不加lock和synchronized，怎么样保证原子性**

![image-20211020123238658](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211020123238658.png)



使用原子类解决原子性问题

![image-20211020123356088](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211020123356088.png)

这些类的底层和操作系统挂钩！

```java
package com.jxau.tvolatile;

import java.util.concurrent.atomic.AtomicInteger;

public class Test03 {
    private  static AtomicInteger num=new AtomicInteger();// 原子类
    public  static void add(){
        num.incrementAndGet();
    }
    public  static void main(String[] args) {

        for(int i=0;i<20;i++){
            new Thread(()->{

                for(int j=0;j<1000;j++){

                    add();
                }

            }).start();
        }

        while(Thread.activeCount()>2){// main、gc线程
            Thread.yield();// 线程礼让

        }
        System.out.println(num);

    }
}

```



> 指令重排

什么是指令重排：你写的程序，计算机并不是按照你写的那样去执行。

源代码-->编译器优化的重排-->指令并行也可能会重排-->内存系统也会重排-->执行。

处理器在进行指令重排的时候，考虑：数据之间的依赖性！

```java
int x=1;// 1
int y=2;// 2
x=x+5;// 3
y=x*x;// 4

执行顺序不一样
```

可能造成影响的结果：a b x v这四个值默认都是 0；

| 线程A | 线程B |
| ----- | ----- |
| x=a   | y=b   |
| b=1   | a=2   |

正常的结果：x=0,y=0;

| 线程A | 线程B |      |
| ----- | ----- | ---- |
| b=1   | a=2   |      |
| x=a   | y=b   |      |

指令重排导致的诡异结果:x=2;y=1;



> 非计算机专业

**volatile可以避免指令重排**

内存屏障。CPU指令。作用：
1、保证特定的操作的执行顺序！

2、可以保证某些变量的内存可见性（利用这些特性volatile实现了可见性）

![image-20211020192830126](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211020192830126.png)

volatile是可以保持可见性。不能保证原子性，由于内存屏障，可以保证指令重排的现象产生！



### 彻底玩转单例模式

> 饿汉式单例

构造器私有，别人无法去new出对象

```java
package com.jxau.signal;

public class Hungry {
    // 饿汉式 可能会浪费空间
    private Hungry(){}
    public static final Hungry hungry=new Hungry();
    public static Hungry getInstance(){
        return hungry;
    }
}

```





> 懒汉式单例

![image-20211020203810835](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211020203810835.png)

```java
package com.jxau.signal;

public class LazyMan {

    private LazyMan(){
        System.out.println(Thread.currentThread().getName()+" "+"OK");
    }

    private volatile static  LazyMan lazyMan;// volatile 防止指令重排

    public static LazyMan getInstance(){
        // 双重检测锁检测+volatile避免指令重排
        if(lazyMan==null){
            synchronized (LazyMan.class){// synchronized保证原子性
                if(lazyMan==null) lazyMan=new LazyMan(); // 单线程下是没事的
                /*
                * 1、分配内存空间
                * 2、执行构造方法，初始化对象
                * 3、把这个对象指向这个空间
                * */
            }

        }
        return lazyMan;
    }


}

class Test{

    public static void main(String[] args) {
        for(int i=0;i<10;i++){
            new Thread(()->{
                LazyMan.getInstance();//  多线程下出现不安全的情况
            }).start();
        }
    }

        }

```

反射破坏单例

```java
package com.jxau.signal;

import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;

public class LazyMan {

    private LazyMan(){
        System.out.println(Thread.currentThread().getName()+" "+"OK");
    }

    private volatile static  LazyMan lazyMan;// volatile 防止指令重排

    public static LazyMan getInstance(){
        // 双重检测锁检测+volatile避免指令重排
        if(lazyMan==null){
            synchronized (LazyMan.class){// synchronized保证原子性
                if(lazyMan==null) lazyMan=new LazyMan(); // 单线程下是没事的
                /*
                * 1、分配内存空间
                * 2、执行构造方法，初始化对象
                * 3、把这个对象指向这个空间
                * */
            }

        }
        return lazyMan;
    }


}

class Test{

    public static void main(String[] args) throws Exception {

       // 用反射破坏单例模式
        LazyMan lazyMan=LazyMan.getInstance();

        Constructor<LazyMan> declaredConstructor = LazyMan.class.getDeclaredConstructor(null);
        declaredConstructor.setAccessible(true);
        LazyMan lazyMan1 = declaredConstructor.newInstance();
        System.out.println(lazyMan);
        System.out.println(lazyMan1);

    }

}

```

三重检测锁---------反射new出两个对象

```jaVA
package com.jxau.signal;

import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;

public class LazyMan {

    private LazyMan(){

        synchronized (LazyMan.class){
            if(lazyMan!=null){
                System.out.println(Thread.currentThread().getName()+" "+"OK");
                throw new RuntimeException("不要试图使用反射破坏单例模式");
            }
        }

    }

    private volatile static  LazyMan lazyMan;// volatile 防止指令重排

    public static LazyMan getInstance(){
        // 双重检测锁检测+volatile避免指令重排
        if(lazyMan==null){
            synchronized (LazyMan.class){// synchronized保证原子性
                if(lazyMan==null) lazyMan=new LazyMan(); // 单线程下是没事的
                /*
                * 1、分配内存空间
                * 2、执行构造方法，初始化对象
                * 3、把这个对象指向这个空间
                * */
            }

        }
        return lazyMan;
    }


}

class Test{

    public static void main(String[] args) throws Exception {

       // 用反射破坏单例模式
        //LazyMan lazyMan=LazyMan.getInstance();

        Constructor<LazyMan> declaredConstructor = LazyMan.class.getDeclaredConstructor(null);
        declaredConstructor.setAccessible(true);
        LazyMan lazyMan1 = declaredConstructor.newInstance();
        LazyMan lazyMan2 = declaredConstructor.newInstance();
        System.out.println(lazyMan2);
        System.out.println(lazyMan1);

    }

}

```

设置红绿灯-----反射发i百年关键字

```java
package com.jxau.signal;

import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;

public class LazyMan {

    private static boolean flag=false;
    private LazyMan(){

        synchronized (LazyMan.class){

            if(flag==false){
                flag=true;
            }else{
                throw new RuntimeException("不要试图使用反射破坏单例模式");
            }

        }

    }

    private volatile static  LazyMan lazyMan;// volatile 防止指令重排
    public static LazyMan getInstance(){
        // 双重检测锁检测+volatile避免指令重排
        if(lazyMan==null){
            synchronized (LazyMan.class){// synchronized保证原子性
                if(lazyMan==null) lazyMan=new LazyMan(); // 单线程下是没事的
                /*
                * 1、分配内存空间
                * 2、执行构造方法，初始化对象
                * 3、把这个对象指向这个空间
                * */
            }

        }
        return lazyMan;
    }


}

class Test{

    public static void main(String[] args) throws Exception {

        // 用反射破坏单例模式
        //LazyMan lazyMan=LazyMan.getInstance();

        Constructor<LazyMan> declaredConstructor = LazyMan.class.getDeclaredConstructor(null);
        declaredConstructor.setAccessible(true);
        LazyMan lazyMan1 = declaredConstructor.newInstance();
        LazyMan lazyMan2 = declaredConstructor.newInstance();
        System.out.println(lazyMan2);
        System.out.println(lazyMan1);

    }

}



 // 用反射破坏单例模式
        //LazyMan lazyMan=LazyMan.getInstance();
        Field flag = LazyMan.class.getDeclaredField("flag");
        flag.setAccessible(true);

        Constructor<LazyMan> declaredConstructor = LazyMan.class.getDeclaredConstructor(null);

        declaredConstructor.setAccessible(true);
        LazyMan lazyMan1 = declaredConstructor.newInstance();
        flag.set(lazyMan1,false);
        LazyMan lazyMan2 = declaredConstructor.newInstance();
        System.out.println(lazyMan2);
        System.out.println(lazyMan1);
```

> 内部类版

```java
package com.jxau.signal;

public class Holer {

    private static Holer holer;
    private Holer(){}

    public static Holer getInstance(){
        return InnerClass.HOLDER;
    }

    public static class InnerClass{
        private static final Holer HOLDER=new Holer();
    }
}

```

> 枚举类  反射无法破解

```java
package com.jxau.signal;

import java.lang.reflect.Constructor;

public enum EnumSignalTest {

    ENUM_SIGNAL_TEST;

    public static EnumSignalTest getEnumSignalTest(){
        return EnumSignalTest.ENUM_SIGNAL_TEST;
    }

}

class Test02{


    public static void main(String[] args) throws Exception {
        EnumSignalTest enumSignalTest = EnumSignalTest.ENUM_SIGNAL_TEST;

        System.out.println(EnumSignalTest.getEnumSignalTest());
        System.out.println(EnumSignalTest.getEnumSignalTest());
        Constructor<EnumSignalTest> constructor = EnumSignalTest.class.getDeclaredConstructor(String.class,int.class);
        constructor.setAccessible(true);
        System.out.println(constructor.newInstance());

    }
}

```

![image-20211020211733818](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211020211733818.png)





### CAS

>什么是CAS

CAS：比较当前工作内存中的值和主存中的值，如果这个值是期望的，那么则者自行操作！如果不是就不更新

**缺点**：

1、你会耗时

2、一次性只能保证一个你共享变量的原子性

3、ABA问题



```java
package com.jxau.cas;

import java.util.concurrent.atomic.AtomicInteger;

public class Test {

    public static void main(String[] args) {

        AtomicInteger atomicInteger=new AtomicInteger(2020);
        System.out.println(atomicInteger.compareAndSet(2020, 2021));
        System.out.println(atomicInteger.get());
        atomicInteger.getAndIncrement();
        System.out.println(atomicInteger.compareAndSet(2020, 2021));
        System.out.println(atomicInteger.get());


    }
}

```





> Unsafe

![image-20211020213959530](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211020213959530.png)

![image-20211020215629809](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211020215629809.png)



![image-20211020215825265](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211020215825265.png)

对应的地址上如果是这个var5，那就加上var4（1）

内存操作，效率很高！



### 原子引用解决ABA问题

> 什么是ABA问题

 在atomiinteger这个类中,他用cas 保证原子性问题,但同时也引发了新的问题;

ABA,一句话,狸猫换太子,举个例子,

(V,内存值,A旧的预期值,B,要求个更新值);

举例:

有两个线程,同时操作一个变量,线程1执行时间比线程2执行时间长,线程2执行快

线程1读取值,此时读到的值是A,这时候线程被挂起,

线程2也读到值,并将A修改为X,然后又做了操作,X又改为Z,最后又将Z改为A;线程2交出执行权;

线程1此时拿到执行权了,此时进行compareAndSwap,发现内存值和期望值是一样,于是正常执行,

但是内存值在这期间已经被操作过;


> ABA问题带来的影响

aba不解决资源会被提前挪用，这不是我们所希望的





![image-20211021122629133](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211021122629133.png)



> 解决ABA问题，引入原子引用！对应的思想：乐观锁！

带版本号的原子操作！

```java
package com.jxau.abaproblem;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicStampedReference;

public class Test01 {

    public static void main(String[] args) {
        // 通过原子引用解决ABA问题  乐观锁
        AtomicStampedReference<Integer> atomicStampedReference=new AtomicStampedReference<>(123,1);// 指定初始数据，指定时间戳（版本号）

        new Thread(()->{
            // 引出ABA问题

            System.out.println(Thread.currentThread().getName()+"A1-->"+atomicStampedReference.getStamp());
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            atomicStampedReference.compareAndSet(123,
                                    124,
                                    atomicStampedReference.getStamp(),
                                    atomicStampedReference.getStamp()+1);

            System.out.println(Thread.currentThread().getName()+"A2-->"+atomicStampedReference.getStamp());
            atomicStampedReference.compareAndSet(124,
                    123,
                    atomicStampedReference.getStamp(),
                    atomicStampedReference.getStamp()+1);

            System.out.println(Thread.currentThread().getName()+"A3-->"+atomicStampedReference.getStamp());

        }).start();

        new Thread(()->{
            int stamp=atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName()+"B1-->"+atomicStampedReference.getStamp());
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(atomicStampedReference.compareAndSet(123,
                    124,
                    stamp,
                    stamp+ 1));

            System.out.println(Thread.currentThread().getName()+"B2-->"+atomicStampedReference.getStamp());

        }).start();

    }


}

```



### 各种锁的理解

#### 公平锁和非公平锁

公平锁：非常公平：需要排队等待其他线程执行完毕

非公平锁：非常不公平，可以插队（默认都是公平的）

```java
 public ReentrantLock() {
        sync = new NonfairSync();
    }

public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```



#### 可重入锁

```java
package com.jxau.locks;

public class Test01 {

    public static void main(String[] args) {

        Phone phone=new Phone();
        new Thread(()->{
            phone.call();
        },"A").start();

        new Thread(()->{
            phone.call();
        },"B").start();
    }
}


class Phone{
    public synchronized void call(){

        System.out.println(Thread.currentThread().getName()+" "+"call");
        email();
    }

    public synchronized void email(){

        System.out.println(Thread.currentThread().getName()+" "+"email");
    }
}

```



```java
package com.jxau.locks;

import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Test02 {
    public static void main(String[] args) {
        Phone1 phone=new Phone1();
        new Thread(()->{
            phone.call();
        },"A").start();

        new Thread(()->{
            phone.call();
        },"B").start();
    }

}

class Phone1{

    Lock lock=new ReentrantLock();
    public  void call(){
        lock.lock();
        try{
            System.out.println(Thread.currentThread().getName()+" "+"call");
            email();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }

    public  void email(){


        lock.lock();
        try{
            System.out.println(Thread.currentThread().getName()+" "+"email");
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
}

```

#### 自旋锁

![image-20211021143130800](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211021143130800.png)

```java
package com.jxau.locks;

import java.util.concurrent.atomic.AtomicReference;

public class SpinLock {

    AtomicReference<Thread> atomicReference=new AtomicReference<>();

    public void myLock(){
        Thread thread=Thread.currentThread();
        System.out.println(Thread.currentThread().getName()+" "+"-->Lock");
        while(!atomicReference.compareAndSet(null,thread)){

        }

    }

    public void myUnlock(){
        Thread thread=Thread.currentThread();
        System.out.println(Thread.currentThread().getName()+" "+"-->UnLock");
        while(!atomicReference.compareAndSet(thread,null)){

        }
    }
}

```



```java
package com.jxau.locks;

import java.util.concurrent.TimeUnit;

public class SpinLockTest {

    public static void main(String[] args) {

        SpinLock spinLock=new SpinLock();

        new Thread(()->{

            spinLock.myLock(); // 使用CAS自旋锁

            try{
                TimeUnit.SECONDS.sleep(5);

            }catch (Exception e){
                e.printStackTrace();
            }finally {
                spinLock.myUnlock();
            }

        },"A").start();
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(()->{

            spinLock.myLock();
            try{
                TimeUnit.SECONDS.sleep(1);
            }catch (Exception e){
                e.printStackTrace();
            }finally {
                spinLock.myUnlock();
            }

        },"B").start();

    }
}

```



#### 死锁

![image-20211021144935577](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211021144935577.png)

```java
package com.jxau.locks;

import java.util.concurrent.TimeUnit;

public class DeadLock {
    public static void main(String[] args) {

        Object o1=new Object();
        Object o2=new Object();
        new Thread(new Test(o1,o2),"T1").start();
        new Thread(new Test(o2,o1),"T2").start();
    }
}

class Test implements Runnable{

    private Object object1;
    private Object object2;

    public Test(Object object1, Object object2) {
        this.object1 = object1;
        this.object2 = object2;
    }

    @Override
    public void run() {

        synchronized (object1){

            System.out.println(Thread.currentThread().getName()+" get Object1 ==> Object2");

            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (object2){
                System.out.println(Thread.currentThread().getName()+" get Object2 ==> Object1");
            }
        }
    }
}


```

![image-20211021145539861](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211021145539861.png)

![image-20211021145521104](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20211021145521104.png)



面试，工作中！排查问题：

1、日志 

2、**堆栈**

