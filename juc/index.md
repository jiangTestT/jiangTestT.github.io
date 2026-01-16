# JUC


## JUC 概述

### JUC是什么？

在 Java 5.0 提供了 `java.util.concurrent`(简称 JUC )包，在此包中增加了在并发编程中很常用的工具类。此包包括了几个小的、已标准化的可扩展框架，并提供一些功能实用的类，没有这些类，一些功能会很难实现或实现起来冗长乏味。

### 进程和线程

进程：进程是一个具有一定独立功能的程序关于某个数据集合的一次运行活动。它是操作系统动态执行的基本单元，在传统的操作系统中，进程既是基本的分配单元，也是基本的执行单元。

线程：通常在一个进程中可以包含一个或若干个线程。线程可以利用进程所拥有的资源，在引入线程的操作系统中。

通常都是把进程作为分配资源的基本单位，而把线程作为独立运行和独立调度的基本单位。

**进程（Process）**

- 进程就像是一家公司。每个公司（进程）都有自己的资源，比如办公区、文件、员工等，这些资源彼此独立。公司之间一般不共享资源，如果需要共享，会通过一定的“协议”来进行（比如进程间通信）。
- 独立性：每个进程都有自己独立的内存空间和资源，它们相互隔离。一个进程崩溃了，不会影响其他进程的运行。
- 创建开销大：启动和管理一个进程需要分配相对较多的系统资源（内存、CPU 等），就像成立一个公司需要很多准备工作。
- 例子：打开多个浏览器窗口时，每个窗口可能是一个单独的进程。

**线程（Thread）**

- 线程就像是公司里的员工。一个公司（进程）可以有多个员工（线程）同时工作，他们可以协作完成任务，并且共享公司的资源，比如办公文具、电脑等。
- 共享资源：线程属于同一个进程，多个线程可以共享该进程的内存和资源，因此它们可以更高效地协同工作。一个线程做某些任务时，另一个线程可以同时做别的事情。
- 轻量级：创建一个线程的开销比创建进程要小得多，就像雇佣一个员工比开公司容易。
- 多线程并发：线程可以同时执行多个任务，比如一边播放视频，一边接受用户输入。

区别总结：

- **进程 = 一个大公司（独立的资源空间）**。
- **线程 = 公司里的员工（共享资源，任务执行）**。

线程运行在进程中，一个进程可以有多个线程，如果一个进程崩溃了，所有线程也会崩溃；而线程崩溃了，进程可能还会继续运行。

### 并行和并发

并发（Concurrency）：并发关注的是在多个任务之间如何进行切换和协调，以使它们能够在共享的资源（如 CPU）上“同时”进行，但并不是在同一时刻真正地执行多个任务。在单核 CPU 上，通过线程或进程的时间片轮转，使得多个任务交替执行，给人的感觉是“同时”执行，但实际上每个时刻只有一个任务在运行。

并行（Parallelism）：并行指的是多个任务在同一时刻同时执行。在并行场景下，多个任务可以同时进行，每个任务拥有自己的处理单元（例如 CPU 核心），从而实现真正的并行执行。

并行通常用于同时处理大规模的任务集合，可以显著提高任务的执行速度和吞吐量。通过将任务划分为更小的子任务，并分配给不同的处理单元并行执行，可以加快整体任务的完成时间。

总结来说，并发是指多个任务在同一个时间段内交替执行，而并行则是指多个任务在同一时刻同时执行。

### 同步和异步

同步和异步是两种不同的工作方式，用于处理任务的执行方式：

同步：在同步操作中，任务按照顺序依次执行，每个任务必须等待前一个任务完成后才能开始执行。这意味着任务之间是连续的、有序的，一个任务的完成通常会阻塞后续任务的执行，直到它自己完成。

例子：想象你在餐馆点餐，每个顾客都必须等待前面的顾客点完餐并完成付款，然后才能点餐。这是一个同步的过程，每个任务（点餐和付款）按顺序执行，一个任务完成后才能开始下一个。

异步：在异步操作中，任务可以同时执行，而不需要等待前一个任务完成。任务的执行不按照固定的顺序，可以随时开始和结束，任务之间是相互独立的。

例子：想象你使用手机上的社交媒体应用，你可以同时发送消息给多个朋友，而不必等待一个朋友回复后才能给下一个朋友发送消息。这是一个异步的过程，每个任务（发送消息给不同的朋友）可以独立执行，而不会相互阻塞。

总之，同步是按照顺序执行任务，而异步是任务可以同时执行而不受顺序限制。在计算机编程和生活中的许多情况下，都可以看到这两种不同的工作方式。

### 创建线程

**创建线程常用的两种方式**：

- 继承 Thread：java 是单继承，资源宝贵，要用接口方式
- 实现 Runable 接口

**继承 Thread 抽象类**：

```java
class T1 extends Thread{
    @Override
    public void run() {
        System.out.println("Thread....");
        super.run();
    }
}
 
public class ThreadDemo {
    public static void main(String[] args) {
        
        T1 t1 = new T1();
        t1.start();
    }
}
```

**实现 Runnable 接口**的方式：

1. 新建类实现 runnable 接口：这种方法会新增类，有更好的方法

   ```java
   class T2 implements Runnable{
       @Override
       public void run() {
           System.out.println(Thread.currentThread().getName() +" runnable....");
       }
   }
    
   public class ThreadDemo {
       public static void main(String[] args) {
           new Thread(new T2(), "线程名").start();
       }
   }
   ```

2. 匿名内部类

   ```java
   new Thread(new Runnable() {
       @Override
       public void run() {
    		// 调用资源方法，完成业务逻辑
       }}, "thread name").start();
   ```

3. lambda 表达式

   ```java
   new Thread(()->{
       // 调用资源方法，完成业务逻辑
   }, "thread name").start();
   ```

注意：线程的启动方式是调用 `start()`方法，一个线程只能启动一次，多次启动会抛 IllegalThreadStateException 异常。

### 线程的状态

枚举类中定义了六种线程的状态，可以调用线程 Thread 中的 getState() 方法获取当前线程的状态

```java
public class Thread {
    public enum State {
        /* 新建 */
        NEW , 
        /* 可运行状态 */
        RUNNABLE , 
        /* 阻塞状态 */
        BLOCKED , 
        /* 无限等待状态 */
        WAITING , 
        /* 计时等待 */
        TIMED_WAITING , 
        /* 终止 */
        TERMINATED;
	}
    // 获取当前线程的状态
    public State getState() {
        return jdk.internal.misc.VM.toThreadState(threadStatus);
    }
}
```

通过源码我们可以看到Java中的线程存在6种状态，每种线程状态的含义如下:

| 线程状态      | 具体含义                                                     |
| ------------- | ------------------------------------------------------------ |
| NEW           | 线程刚被创建，但是并未启动，还没调用 start 方法。            |
| RUNNABLE      | 线程准备执行（等待调度）或正在执行。                         |
| BLOCKED       | 线程因尝试获取锁而阻塞。                                     |
| WAITING       | 等待状态。造成线程等待的原因有两种，分别是调用 Object.wait()、Object.join() 方法。处于等待状态的线程，正在等待其他线程去执行一个特定的操作。例如：因为 wait() 而等待的线程正在等待另一个线程去调用 notify() 或 notifyAll()；因为 join() 而等待的线程正在等待另一个线程结束。 |
| TIMED_WAITING | 限时等待。造成线程限时等待状态的原因有三种，分别是：Thread.sleep(long)，Object.wait(long)、Object.join(long)。 |
| TERMINATED    | 线程完成执行或因异常终止。                                   |

### synchronized隐式锁

#### 特点

- synchronized 关键字可以用来修饰静态方法、非静态方法和代码块。
- 被 synchronized 修饰的方法被称之为同步方法，被 synchronized 修饰的代码块被称之同步代码块

**同步代码块的格式**

```java
//该对象可以是任意的对象，这个对象可以简单的理解就是一把锁：专业的术语将其称之为"监视器"
synchronized (对象) {
    // 在此代码块中访问共享数据
}
```

**同步方法的格式**

```java
public synchronized void sellTicket(){...}
```

**静态同步方法的格式**

```java
public static synchronized void sellTicket(){...}
```

Java中 的每一个对象都可以作为锁。具体表现为以下3种形式：

- 对于普通同步方法，锁是当前实例对象。
- 对于静态同步方法，锁是当前类的Class对象。
- 对于同步代码块，锁是 Synchonized 括号里配置的对象
- 而静态同步方法（Class对象锁）与非静态同步方法（实例对象锁）之间是不会有竞争的。

#### sleep和wait的区别

- sleep是Thread类中方法，wait方法是Object类中的方法
- sleep方法不会释放同步锁，wait方法会释放同步锁

### 线程之间的数据交换

使用到的是 Exchanger<V> 类，在 java.util.concurrent 包中。使用的时候需要指定交换的类型。
调用 Exchanger 类对象的 exchange(V x) 方法，两个线程都传入要交换的数据 v 。
使用该类是两个线程都调用了 exchange(V v) 方法才能交换。
如果只有一个线程调用了 exchange(V v)，程序会一直等待。
代码测试:一个线程调用 exchanger(V v) 方法时。

```java
import java.util.concurrent.Exchanger;

public class Test {
    public static void main(String[] args) {
        Exchanger<Integer> change= new Exchanger<>();
        new Thread(new Runnable() {
            @Override
            public void run() {
                Thread.currentThread().setName("线程1");
                try{
                    Integer t = 1;
                    t = change.exchange(t);

                    System.out.println(Thread.currentThread().getName()+" t = " +t);
                }catch (Exception e){
					e.printStackTrace();
                }
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                Thread.currentThread().setName("线程2");
                try{
                    Integer t = 5;
                    t = change.exchange(t);

                    System.out.println(Thread.currentThread().getName() +" t = " +t);
                }catch(Exception e){
					e.printStackTrace();
                }
            }
        }).start();
    }
}
```

如果是多个线程进行数据交换，就是先来先交换。最终交换结果会不固定。

## Lock显示锁

### ReentrantLock 可重入锁

首先看一下 JUC 的重磅武器——锁（Lock）

相比同步锁，JUC 包中的 Lock 锁的功能更加强大，他是一个接口，提供了各种各样的锁（公平锁，非公平锁，共享锁，独占锁……），所以使用起来很灵活。

这里主要有三个实现：ReentrantLock、ReentrantReadWriteLock.ReadLock、ReentrantReadWriteLock.WriteLock。

ReentrantLock 是可重入的互斥锁，虽然具有与 synchronized 相同功能，但是会比 synchronized 有更多的方法，因此更加灵活。

#### 可重入性

可重入锁又名递归锁，是指在同一个线程中，外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁。Java 中 ReentrantLock 和 synchronized 都是可重入锁，可重入锁的一个优点是`可一定程度避免死锁`。

ReentrantLockDemo 类中有两个普通同步方法，都需要对象a的锁。如果是不可重入锁的话，a 方法首先获取到锁，a 方法在执行的过程中需要调用 b 方法，此时锁被 a 方法占有，b 方法无法获取到锁，这样就会导致 b 方法无法执行，a 方法也无法执行，出现了死锁情况。可重入锁可避免这种死锁的发生。

**使用 synchronized 实现可重入锁**

```java
public class ReentrantLockDemo {
 
    // 方法 a 被 synchronized 关键字修饰，意味着该方法是同步的，
    // 在执行该方法时，需要获得当前对象的锁（this 对象的锁）。
    public synchronized void a() {
        // 在 a 方法中，首先调用了 b 方法
        // 因为 a 方法和 b 方法都被 synchronized 修饰，所以当调用 b 方法时，
        // 线程会试图重新获取当前对象的锁 (this)，
        // 由于 Java 支持可重入锁 (Reentrant Lock)，
        // 当前线程已经持有了 this 对象的锁，所以可以再次获取而不被阻塞。
        this.b();
        // 打印 "a"
        System.out.println("a");
    }
 
    // 方法 b 也是一个同步方法，同样需要获取 this 对象的锁
    public synchronized void b() {
        // 打印 "b"
        System.out.println("b");
    }
 
    public static void main(String[] args) {
        // 创建一个 ReentrantLockDemo 对象，并调用其 a() 方法
        // 当调用 a() 方法时，线程首先会获得当前 ReentrantLockDemo 对象的锁。
        // 然后 a() 方法内部会调用 b() 方法，当前线程再次尝试获取锁，
        // 由于可重入锁的特性，线程可以重复获取已经持有的锁。
        new ReentrantLockDemo().a();
    }
}
```

**使用 ReentrantLock 实现可重入锁**

```java
public class ReentrantLockDemo2 {
 
    // 创建一个 ReentrantLock 实例，用于显式地控制锁机制
    private final ReentrantLock lock = new ReentrantLock();
 
    // 方法 a 使用显式锁 lock 来确保线程安全
    public void a() {
 
        // 获取锁，lock() 方法将阻塞当前线程，直到获得锁为止
        lock.lock();
        try {
            // 调用 b() 方法，b() 方法内部也使用了同一个锁
            // ReentrantLock 是可重入锁，当前线程可以多次获取同一个锁
            this.b();
            // 打印 "a"
            System.out.println("a");
        } finally {
            // 确保无论是否发生异常，锁都能被释放，避免死锁
            lock.unlock();
        }
    }
 
    // 方法 b 同样使用了 ReentrantLock，确保线程安全
    public void b() {
        // 获取锁
        lock.lock();
        try {
            // 打印 "b"
            System.out.println("b");
        } finally {
            // 确保锁能正确释放
            lock.unlock();
        }
    }
 
    public static void main(String[] args) {
        // 创建一个 ReentrantLockDemo2 对象并调用 a() 方法
        new ReentrantLockDemo2().a();
    }
}
```

#### 公平锁

ReentrantLock 还可以实现公平锁。所谓公平锁，也就是在锁上等待时间最长的线程优先获得锁的使用权。通俗的理解就是谁排队时间最长谁先获取锁。（公平锁不允许插队，非公平锁允许插队)

```java
private ReentrantLock lock = new ReentrantLock(true);
```

案例

```java
public class FairLockDemo {
 
    // 创建一个公平锁的 ReentrantLock 实例，true 参数表示启用公平性
    private final ReentrantLock lock = new ReentrantLock(true);
 
    // 模拟一个方法，多个线程竞争锁
    public void testMethod() {
        try {
            // 获取锁，保证线程以公平的顺序获取资源
            lock.lock();
            System.out.println(Thread.currentThread().getName() + " 获得了锁");
            // 模拟线程占用资源，进行操作
            Thread.sleep(200);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            // 确保锁最终被释放
            lock.unlock();
            System.out.println(Thread.currentThread().getName() + " 释放了锁");
        }
    }
 
    public static void main(String[] args) {
        FairLockDemo demo = new FairLockDemo();
 
        // 创建 5 个线程，每个线程都会调用 testMethod 方法，争夺锁
        for (int i = 1; i <= 5; i++) {
            Thread t = new Thread(() -> {
                for (int j = 0; j < 5; j++) {
                    demo.testMethod();
                }
            }, "线程-" + i);
            t.start();
        }
    }
}
```

测试结果：

```java
线程-1 获得了锁
线程-1 释放了锁
线程-2 获得了锁
线程-2 释放了锁
线程-3 获得了锁
线程-3 释放了锁
线程-4 获得了锁
线程-4 释放了锁
线程-5 获得了锁
线程-5 释放了锁
```

总结：

- 公平锁：公平锁通过一个**队列**来管理多个线程的请求顺序，确保线程按照进入等待队列的顺序获取锁。避免了某些线程被长时间“插队”导致饥饿的问题。
- 非公平锁（默认的 ReentrantLock）在极端情况下可能会导致某些线程长时间等待，因为新的线程有可能会插队直接获取锁。公平锁虽然能避免这种情况，但可能会带来性能上的开销，因为需要维护一个有序的队列。
- 使用场景：公平锁适用于对公平性要求较高的场景，比如多个线程需要公平竞争资源、避免某些线程被长期占用而“饿死”。

#### 限时等待

这个是什么意思呢？也就是通过我们的 tryLock 方法来实现，可以选择传入时间参数，表示等待指定的时间，无参则表示立即返回锁申请的结果：true 表示获取锁成功，false 表示获取锁失败。我们可以将这种方法用来解决死锁问题。

```java
public class DeadlockExample {
 
    // 两个锁对象，用于演示死锁
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();
 
    // 第一个方法，尝试获取 lock1，然后获取 lock2
    public void method1() {
        synchronized (lock1) {
            System.out.println(Thread.currentThread().getName() + " 持有 lock1，等待 lock2...");
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (lock2) {
                System.out.println(Thread.currentThread().getName() + " 持有 lock2");
            }
        }
    }
 
    // 第二个方法，尝试获取 lock2，然后获取 lock1
    public void method2() {
        synchronized (lock2) {
            System.out.println(Thread.currentThread().getName() + " 持有 lock2，等待 lock1...");
            try {
                Thread.sleep(100); 
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (lock1) {
                System.out.println(Thread.currentThread().getName() + " 持有 lock1");
            }
        }
    }
 
    public static void main(String[] args) {
        DeadlockExample deadlock = new DeadlockExample();
 
        // 创建两个线程，分别调用 method1 和 method2
        Thread thread1 = new Thread(deadlock::method1, "线程-1");
        Thread thread2 = new Thread(deadlock::method2, "线程-2");
 
        // 启动线程
        thread1.start();
        thread2.start();
    }
}
```

死锁发生过程：

1. `线程-1` 先持有 `lock1`，然后等待获取 `lock2`。
2. `线程-2` 先持有 `lock2`，然后等待获取 `lock1`。
3. 两个线程互相等待对方释放锁，导致死锁，程序卡住无法继续执行。

执行结果：

```tex
线程-1 持有 lock1，等待 lock2...
线程-2 持有 lock2，等待 lock1...
```

**使用ReentrantLock解决死锁问题：**

```java
public class DeadlockDemo2 {
 
    // 创建两个 ReentrantLock 对象，lock1 和 lock2
    private static ReentrantLock lock1 = new ReentrantLock();
    private static ReentrantLock lock2 = new ReentrantLock();
 
    public static void main(String[] args) {
        // 创建并启动线程1
        new Thread(() -> {
            // 尝试获取 lock1 锁
            boolean result1 = lock1.tryLock();
            if (result1){
                try {
                    // 成功获取 lock1 锁，输出提示信息
                    System.out.println("线程1: 持有锁1...");
                    // 模拟一些工作
                    try {
                        Thread.sleep(500); // 休眠 500 毫秒
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    // 输出提示信息，表示线程1正在等待获取 lock2 锁
                    System.out.println("线程1: 等待获取锁2...");
                    // 尝试获取 lock2 锁
                    boolean result2 = lock2.tryLock();
                    if(result2){
                        try {
                            // 成功获取 lock2 锁，输出提示信息
                            System.out.println("线程1: 成功获取到两个锁。");
                        } finally {
                            // 确保最终释放 lock2 锁
                            lock2.unlock();
                        }
                    }else {
                        // 获取 lock2 失败，输出提示信息
                        System.out.println("线程1: 未获取到锁2");
                    }
                }finally {
                    // 确保最终释放 lock1 锁
                    lock1.unlock();
                }
            }else{
                // 获取 lock1 失败，输出提示信息
                System.out.println("线程1: 未获取到锁1");
            }
        }).start(); // 启动线程1
 
        // 创建并启动线程2
        new Thread(() -> {
            // 尝试获取 lock2 锁
            boolean result2 = lock2.tryLock();
            if(result2){
                try {
                    // 成功获取 lock2 锁，输出提示信息
                    System.out.println("线程2: 持有锁2...");
                    // 模拟一些工作
                    try {
                        Thread.sleep(500); // 休眠 500 毫秒
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    // 输出提示信息，表示线程2正在等待获取 lock1 锁
                    System.out.println("线程2: 等待获取锁1...");
                    // 尝试获取 lock1 锁
                    boolean result1 = lock1.tryLock();
                    if(result1){
                        try {
                            // 成功获取 lock1 锁，输出提示信息
                            System.out.println("线程2: 成功获取到两个锁。");
                        } finally {
                            // 确保最终释放 lock1 锁
                            lock1.unlock();
                        }
                    }else {
                        // 获取 lock1 失败，输出提示信息
                        System.out.println("线程2: 未获取到锁1");
                    }
                }finally {
                    // 确保最终释放 lock2 锁
                    lock2.unlock();
                }
            }else{
                // 获取 lock2 失败，输出提示信息
                System.out.println("线程2: 未获取到锁2");
            }
        }).start(); // 启动线程2
    }
}
```

执行结果：

```tex
线程1: 持有锁1...
线程2: 持有锁2...
线程1: 等待获取锁2...
线程2: 等待获取锁1...
线程1: 未获取到锁2
线程2: 未获取到锁1
```

#### ReentrantLock和synchronized区别

- synchronized 是独占锁，加锁和解锁的过程自动进行，易于操作，但不够灵活。ReentrantLock 也是独占锁，加锁和解锁的过程需要手动进行，不易操作，但非常灵活。
- synchronized 可重入，因为加锁和解锁自动进行，不必担心最后是否释放锁；ReentrantLock 也可重入，但加锁和解锁需要手动进行，且次数需一样，否则其他线程无法获得锁。
- synchronized 不可响应中断，一个线程获取不到锁就一直等着；ReentrantLock 可以响应中断（ tryLock方法：获取不到锁则返回 false）。
- synchronized 不具备设置公平锁的特点，ReentrantLock 可以成为公平锁。

## 并发容器类

集合类是不安全的

**ConcurrentModificationException**

在多线程环境中，当一个线程对集合进行结构修改（如添加或删除元素）时，其他线程在迭代这个集合时可能会抛出 `ConcurrentModificationException`

**数据丢失**

在没有适当同步的情况下，不同线程对集合的并发修改可能会导致数据丢失。

**数据不一致**

在没有同步的情况下，一个线程读取集合的内容时，另一个线程可能正在修改这个集合，这会导致读取到的数据不一致。

**总结**

这些例子展示了集合类在多线程环境中可能面临的安全问题。为了确保线程安全，通常需要使用同步机制，如 synchronized 关键字、ReentrantLock 或使用 Java 提供的线程安全集合类（例如 ConcurrentHashMap、CopyOnWriteArrayList 等）。

### CopyOnWrite容器

CopyOnWrite 容器（简称 COW 容器）即写时复制的容器。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对 CopyOnWrite 容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以 CopyOnWrite 容器也是一种读写分离的思想，读和写不同的容器。

从 JDK1.5 开始 Java 并发包里提供了两个使用 CopyOnWrite 机制实现的并发容器,它们是 CopyOnWriteArrayList 和 CopyOnWriteArraySet。

**缺点：**

1. 内存占用问题。写的时候会创建新对象添加到新容器里，而旧容器的对象还在使用，所以有两份对象内存。
2. 数据一致性问题。CopyOnWrite 容器只能保证数据的最终一致性，不能保证数据的实时一致性。所以如果你希望写入的数据，马上能读到，请不要使用CopyOnWrite 容器。

### ConcurrentHashMap

ConcurrentHashMap 的特点
并发性：ConcurrentHashMap 允许多个线程同时访问，读操作不会被阻塞，不需要加锁。这意味着多个线程可以并发地读取其中的数据，而不会发生竞争或锁定。

分段锁：ConcurrentHashMap 是线程安全的 Map 容器

**JDK8 之前的底层原理**
在JDK8之前，ConcurrentHashMap 主要是由分段锁（Segment）和链表数组（Entry[]）组成。它将整个存储空间分成多个段，每个段都维护着一个独立的链表数组。每个段都拥有自己的锁，因此不同的线程可以同时对不同的段进行操作，从而实现并发访问。

当一个线程需要访问某个元素时，首先需要根据元素的哈希值定位到对应的段，然后获取该段的锁。获取锁后，线程会在该段对应的链表上进行插入、删除或者查找操作。这样做的好处是，不同线程对于不同段的操作不会相互阻塞，提高了并发性能。

但是，当多个线程同时修改同一个段的链表时，可能会导致链表成为热点，并发性能下降。

**JDK8 之后的底层原理**
相比于 JDK8 之前的实现方式，JDK8 使用了更加高效的数据结构来减少锁的粒度，提高了并发性能。

在 JDK8 中，ConcurrentHashMap 的底层数据结构采用了一种称为「CAS + Synchronized + Node + TreeBin」的实现方式。当链表长度超过一定阈值时，会将链表转换为红黑树，提高查询和删除的效率。

同时，JDK8 还引入了Node 和 TreeBin 来表示键值对的存储。Node 是链表节点，TreeBin 是红黑树节点。这些节点都通过 CAS（Compare and Swap） 操作来保证并发安全性。

此外，JDK8 还引入了一种新的扩容机制，使用了无锁算法来提高并发性能。与之前版本的 ConcurrentHashMap 不同，JDK8 中的 ConcurrentHashMap 在扩容时不需要对整个数据结构进行加锁，而只需对正在被操作的部分加锁。

总体而言，JDK8 后的 ConcurrentHashMap 通过优化锁的粒度、引入红黑树以及改进扩容机制，进一步提高了并发性能和吞吐量。

## JUC强大的辅助类

### CountDownLatch

CountDownLatch是一个非常实用的多线程控制工具类，应用非常广泛。

例如：在手机上安装一个应用程序，假如需要 5 个子线程检查服务授权，那么主线程会维护一个计数器，初始计数就是 5。用户每同意一个授权该计数器减 1，当计数减为 0 时，主线程才启动，否则就只有阻塞等待了。

CountDownLatch 中 count down 是倒数的意思，latch 则是门闩的含义。整体含义可以理解为倒数的门栓，似乎有一点“三二一，芝麻开门”的感觉。CountDownLatch 的作用也是如此。

常用的就下面几个方法：
```		java
new CountDownLatch(int count) //实例化一个倒计数器，count指定初始计数
countDown() // 每调用一次，计数减一
await() //等待，当计数减到0时，阻塞线程（可以是一个，也可以是多个）并行执行
```

案例：6个同学陆续离开教室后值班同学才可以关门。

```java
public class CountDownLatchDemo {

    public static void main(String[] args) throws InterruptedException {
        // 初始化计数器，初始计数为6
        CountDownLatch countDownLatch = new CountDownLatch(6);
        for (int i = 0; i < 6; i++) {
            new Thread(()->{
                try {
                    // 每个同学墨迹几秒钟
                    TimeUnit.SECONDS.sleep(new Random().nextInt(5));
                    System.out.println(Thread.currentThread().getName() + " 同学出门了");
                    // 调用countDown()计算减1
                    countDownLatch.countDown();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, String.valueOf(i)).start();
        }
 
        // 调用计算器的await方法，等待6位同学都出来
        countDownLatch.await();
        System.out.println("值班同学锁门了");
    }
}
```

执行结果：

```tex
同学3 出来了
同学1 出来了
同学0 出来了
同学2 出来了
同学5 出来了
同学4 出来了
值班同学锁门了
```

### CyclicBarrier

从字面上的意思可以知道，这个类的中文意思是“循环栅栏”。大概的意思就是一个可循环利用的屏障。是 Java 中用于多线程编程的同步工具之一，它的主要作用是在多个线程相互等待达到某个共同点之后再一起继续执行。CyclicBarrier 是一个同步辅助类，通常用于协调多个线程之间的任务分配和执行。

常用方法：

CyclicBarrier(int parties, Runnable barrierAction) 创建一个 CyclicBarrier 实例，parties 指定参与相互等待的线程数，barrierAction 一个可选的 Runnable 命令，该参数只在每个屏障点运行一次，可以在执行后续业务之前共享状态。该操作由最后一个进入屏障点的线程执行。

CyclicBarrier(int parties) 创建一个 CyclicBarrier 实例，parties 指定参与相互等待的线程数。

await() 该方法被调用时表示当前线程已经到达屏障点，当前线程阻塞进入休眠状态，直到所有线程都到达屏障点，当前线程才会被唤醒。

**案例：集齐7颗龙珠召唤神龙**

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
public class CyclicBarrierDemo {
    public static void main(String[] args) {
        // 召唤龙珠的线程
        CyclicBarrier cyclicBarrier = new CyclicBarrier(7,()->{
            System.out.println("召唤神龙成功！");
        });
        for (int i = 1; i <=7 ; i++) {
            final int temp = i;
            // lambda能操作到 i 吗
            new Thread(()->{
                System.out.println(Thread.currentThread().getName()+"收集"+temp+"个龙珠");
                try {
                    cyclicBarrier.await(); // 等待
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

### Semaphore

Semaphore 翻译成字面意思为信号量，Semaphore 可以控制同时访问的线程个数。非常适合需求量大，而资源又很紧张的情况。比如给定一个资源数目有限的资源池，假设资源数目为 N，每一个线程均可获取一个资源，但是当资源分配完毕时，后来线程需要阻塞等待，直到前面已持有资源的线程释放资源之后才能继续。

**常用方法：**

```java
public Semaphore(int permits) // 构造方法，permits指资源数目（信号量）
public void acquire() throws InterruptedException // 占用资源，当一个线程调用acquire操作时，它要么通过成功获取信号量（信号量减1），要么一直等下去，直到有线程释放信号量，或超时。
public void release() // （释放）实际上会将信号量的值加1，然后唤醒等待的线程。
```

信号量主要用于两个目的：

- 多个共享资源的互斥使用。
- 用于并发线程数的控制。保护一个关键部分不要一次输入超过N个线程。`（限流）`

 **案例：6辆车抢占3个车位**

```java
public class SemaphoreDemo {
 
    public static void main(String[] args) {
        // 初始化信号量，3个车位
        Semaphore semaphore = new Semaphore(3);
        // 6个线程，模拟6辆车
        for (int i = 0; i < 6; i++) {
            new Thread(()->{
                try {
                    // 抢占一个停车位
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName() + " 抢到了一个停车位！！");
                    // 停一会儿车
                    TimeUnit.SECONDS.sleep(new Random().nextInt(10));
                    // 开走，释放一个停车位
                    System.out.println(Thread.currentThread().getName() + " 离开停车位！！");
                    semaphore.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, String.valueOf(i)).start();
        }
    }
}
```

打印结果：

```tex
0 抢到了一个停车位！！
1 抢到了一个停车位！！
2 抢到了一个停车位！！
1 离开停车位！！
3 抢到了一个停车位！！
2 离开停车位！！
4 抢到了一个停车位！！
0 离开停车位！！
5 抢到了一个停车位！！
5 离开停车位！！
3 离开停车位！！
4 离开停车位！！
```

## Callable接口

Thread 类、Runnable 接口使得多线程编程简单直接。

但 Thread 类和 Runnable 接口都不允许声明检查型异常，也不能定义返回值。没有返回值这点稍微有点麻烦。不能声明抛出检查型异常则更麻烦一些。

以上两个问题现在都得到了解决。从 java5 开始，提供了 Callable 接口，是 Runable 接口的增强版。用 Call() 方法作为线程的执行体，增强了之前的 run() 方法。因为 call 方法可以有返回值，也可以声明抛出异常。

### Callable的使用

```java
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

/**
 * 1. 创建Callable的实现类，并重写call()方法，该方法为线程执行体，并且该方法有返回值
 */
class MyCallableThread implements Callable<Integer> {
 
    @Override
    public Integer call() throws Exception {
        int i;
        for (i = 0; i < 10; i++) {
            Thread.sleep(300);
            System.out.println(Thread.currentThread().getName() + " 执行了！" + i);
        }
        return i;
    }
}
 
public class CallableDemo {
 
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        // 创建多线程
        new Thread(new MyRunnableThread(), "threadName").start();
        // 2. 创建Callable的实例，并用FutureTask类来包装Callable对象
        FutureTask task = new FutureTask<>(new MyCallableThread());
        // 3. 创建线程
        new Thread(task, "MyCallableThread").start();
    }
}
```

小结： 

**Runnable vs Callable:**

- Runnable 接口不允许返回结果，也无法抛出检查型异常。
- Callable 接口允许返回结果（这里是 Integer 类型）并可以抛出异常。

**FutureTask:**

- FutureTask 是 Runnable 和 Future 的结合体，它既能作为一个任务执行，又能保存异步计算的结果。
- 可以使用 FutureTask 来管理 Callable 的结果，并且可以调用 get() 方法来获取执行结果。如果任务未完成，get() 方法会阻塞直到任务完

**task1.cancel(false):**

- 调用 cancel() 方法时，如果传递 true，任务会被立即中断。
- 如果传递 false，任务可以执行完当前代码段，但不会继续执行后续代码。

**join() vs FutureTask.get():**

- join() 用于等待线程结束，而 FutureTask.get() 既可以等待线程结束，还可以返回线程执行的结果。

**异步执行:**

- 使用 FutureTask 可以让任务在后台执行，同时主线程可以做其他事情，等到需要结果时再调用 get() 获取执行结果。

这个例子展示了如何使用 Runnable 和 Callable 来定义任务，如何使用 FutureTask 来异步执行和获取任务的结果，以及如何在任务执行过程中取消或等待线程完成。

注意：

- 为了防止主线程阻塞，建议 get 方法放到最后
- 只计算一次，FutureTask 会复用之前计算过的结果

**callable接口与runnable接口的区别**

- 相同点：都是接口，都可以编写多线程程序，都采用Thread.start()启动线程
- 不同点：
  - 具体方法不同：一个是run，一个是call
  - Runnable没有返回值；Callable可以返回执行结果，是个泛型
  - Callable接口的call()方法允许抛出异常；Runnable的run()方法异常只能在内部消化，不能往上继续抛

**获得多线程的方法几种？**

- 传统的是继承 thread 类和实现 runnable 接口

- java5 以后又有实现 callable 接口和 java 的线程池

## 阻塞队列（BlockingQueue）

栈与队列简单回顾：

栈：先进后出，后进先出

队列：先进先出

#### 什么是BlockingQueue

在多线程领域：所谓阻塞，在某些情况下会挂起线程（即阻塞），一旦条件满足，被挂起的线程又会自动被唤起。

BlockingQueue 即阻塞队列，是 java.util.concurrent 下的一个接口，因此不难理解，BlockingQueue 是为了解决多线程中数据高效安全传输而提出的。从阻塞这个词可以看出，在某些情况下对阻塞队列的访问可能会造成阻塞。

**被阻塞的情况主要有如下两种：**

- 当队列满了的时候，依然进行入队列操作
- 当队列空了的时候，依然进行出队列操作

因此，当一个线程试图对一个已经满了的队列进行入队列操作时，它将会被阻塞，除非有另一个线程做了出队列操作；

同样，当一个线程试图对一个空队列进行出队列操作时，它将会被阻塞，除非有另一个线程进行了入队列操作。

**阻塞队列主要用在生产者/消费者的场景**，下面这幅图展示了一个线程生产、一个线程消费的场景：

![](https://crry-assist.oss-cn-chengdu.aliyuncs.com/Screenshot/%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97-%E7%94%9F%E4%BA%A7%E8%80%85%E6%B6%88%E8%B4%B9%E8%80%85%E6%A8%A1%E5%9E%8B.png)

**为什么需要BlockingQueue**

多线程环境中，通过队列可以很容易实现数据共享，比如经典的“生产者”和“消费者”模型中，通过队列可以很便利地实现两者之间的数据共享。

假设我们有若干生产者线程，另外又有若干个消费者线程。如果生产者线程需要把准备好的数据共享给消费者线程，利用队列的方式来传递数据，就可以很方便地解决他们之间的数据共享问题。

但如果生产者和消费者在某个时间段内，万一发生数据处理速度不匹配的情况呢？理想情况下，如果生产者产出数据的速度大于消费者消费的速度，并且当生产出来的数据累积到一定程度的时候，那么生产者必须暂停等待一下（阻塞生产者线程），以便等待消费者线程把累积的数据处理完毕，反之亦然。然而，在concurrent 包发布以前，在多线程环境下，我们每个程序员都必须去自己控制这些细节，尤其还要兼顾效率和线程安全，而这会给我们的程序带来不小的复杂度。

**这也是我们在多线程环境下，为什么需要 BlockingQueue 的原因。作为 BlockingQueue 的使用者，我们再也不需要关心什么时候需要阻塞线程，什么时候需要唤醒线程，因为这一切 BlockingQueue 都给你一手包办了。**

#### 认识BlockingQueue

java.util.concurrent 包里的 BlockingQueue 是一个接口，继承 Queue 接口，Queue 接口继承 Collection。

BlockingQueue接口主要有以下7个实现类：

- ArrayBlockingQueue：由数组结构组成的有界阻塞队列。使用一个固定大小的数组来存储元素。它需要在创建时指定容量，不支持动态扩展。
- LinkedBlockingQueue：由链表结构组成的有界阻塞队列。使用一个链表来存储元素，它可以是无界的（未指定容量）或有界的（在创建时指定容量）。当使用无界队列时，它可以一直增长（最大值是 Integer.MAX_VALUE），不会导致生产者或消费者被阻塞。对于有界队列，当队列满时，生产者会被阻塞，直到有空间可用，当队列为空时，消费者会被阻塞，直到有元素可取出。
- PriorityBlockingQueue：支持优先级排序的无界阻塞队列。
- DelayQueue：使用优先级队列实现的延迟无界阻塞队列。
- SynchronousQueue：不存储元素的阻塞队列，也即单个元素的队列。
- LinkedTransferQueue：由链表组成的无界阻塞队列。
- LinkedBlockingDeque：由链表组成的双向阻塞队列。

阻塞队列提供以下**4种处理方法**

|              | 抛出异常  | 特殊值   | 阻塞   | 超时                  |
| ------------ | --------- | -------- | ------ | --------------------- |
| 插入         | add(e)    | offer(e) | put(e) | offer(e,  time, unit) |
| 移除         | remove()  | poll()   | take() | poll(time,  unit)     |
| 检查（获取） | element() | peek()   | 不可用 | 不可用                |

**抛出异常**

add 正常执行返回 true，element（不删除）和 remove（删除）返回阻塞队列中的第一个元素  当阻塞队列满时，再往队列里 add 插入元素会抛IllegalStateException:Queue full；当阻塞队列空时，再从队列里 remove 移除元素会抛 NoSuchElementException；当阻塞队列空时，再调用 element 检查元素会抛出 NoSuchElementException。

**特殊值** 

插入方法，成功 ture 失败 false 

移除方法，成功返回出队列的元素，队列里没有就返回 null 

检查方法，成功返回队列中的元素，没有返回 null

**阻塞**

如果试图的操作无法立即执行，该方法调用将会发生阻塞，直到能够执行。  当阻塞队列满时，再往队列里 put 元素，队列会一直阻塞生产者线程，直到 put 数据or 响应中断退出；当阻塞队列空时，再从队列里 take 元素，队列会一直阻塞消费者线程，直到队列可用。

**超时**

如果试图的操作无法立即执行，该方法调用将会发生阻塞，直到能够执行，但等待时间不会超过给定值。  返回一个特定值以告知该操作是否成功(典型的是 true / false)。

#### SynchronousQueue

SynchronousQueue，实际上它不是一个真正的队列，因为它不会为队列中元素维护存储空间。

就好比将文件直接交给同事，不是将文件放到她的邮箱中，这样可以尽快拿到文件。这种实现队列的方式看似很奇怪，但由于可以直接交付工作，从而降低了将数据从生产者移动到消费者的延迟。

因为 SynchronousQueue 没有存储功能，因此 put 和 take 会一直阻塞，直到有另一个线程已经准备好参与到交付过程中。仅当有足够多的消费者，并且总是有一个消费者准备好获取交付、 的工作时，才适合使用同步队列。

