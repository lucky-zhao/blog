# Java多线程基础

- [线程的几种状态](#一-线程的几种状态)
- [synchronized的几个例子](#二-synchronized的几个例子)
    - [同一个对象调用两个普通的同步方法](#21-同一个对象调用两个普通的同步方法)
    - [同一个对象调用个一个非静态的同步方法和一个非静态非同步方法](#22-同一个对象调用个一个非静态的同步方法和一个非静态非同步方法)
    - [两个对象调用两个同步方法](#23-两个对象调用两个同步方法)
    - [一个对象调用两个静态同步方法](#24-一个对象调用两个静态同步方法)
- [线程虚假唤醒](#二-线程虚假唤醒)
- [线程上下文切换](#三-线程上下文切换)
- [锁](#四-锁)
    - [几种锁](#41-几种锁)
    - [怎么避免死锁](#42-怎么避免死锁)
- [并发和并行](#五-并发和并行)
- [JAVA内存模型](#六-JAVA内存模型)
    - [多线程带来的三个问题](#61-多线程带来的三个问题)
    - [JAVA内存模型怎么解决上面的问题的](#62-JAVA内存模型怎么解决上面的问题的)


## 一 线程的几种状态

线程的几种状态其实在JAVA里已经有枚举定义了：
```java
 public enum State {
        /**
         * 新建状态、线程刚new出来，还没有启动
         */
        NEW,

        /**
         * 运行状态(这里把就绪和运行中统一叫成：RUNNABLE 状态)
         */
        RUNNABLE,

        /**
         * 阻塞状态
         */
        BLOCKED,

        /**
         * 等待状态
         */
        WAITING,

        /**
         * 超时等待状态，跟上一个的区别是：上一个状态是一直在等待中，除非有异常或者被唤醒，超时等待是等待一定时间后，就不等了
         */
        TIMED_WAITING,

        /**
         * 死亡状态，线程已经执行完毕
         */
        TERMINATED;
    }

```

## 二 synchronized的几个例子

### 2.1 同一个对象调用两个普通的同步方法

提示：要知道`synchronized`锁的是什么？

```java
/**
 * @author zhao
 * 同一个对象调用两个普通的同步方法
 */
public class SyncTest01 {

    public static void main(String[] args) {
        Print print = new Print();
        new Thread(() -> {
            print.printA();
        }, "A").start();
        TimeUnit.SECONDS.sleep(2);//让main线程睡2s
        new Thread(() -> {
            print.printB();
        }, "B").start();
    }
}

class Print {
    public synchronized void printA() {
        System.out.println("A");
    }

    public synchronized void printB() {
        System.out.println("B");
    }
}
```
第一个例子输出结果是：先打印A 在打印B。因为只new了一个`Print`对象，`synchronized`锁的是同一个对象实例，
并且让`main`线程睡了2秒，让A线程先获取锁(并不是因为是先new的A线程)，所以A线程走完了，B线程才会走。

### 2.2 同一个对象调用个一个非静态的同步方法和一个非静态非同步方法
把上一个例子稍微修改一下，去掉`printB()`方法前的`synchronized`。

```java
/**
 * @author zhao
 * 同一个对象调用个一个非静态的同步方法和一个非静态非同步方法
 */
public class SyncTest02 {

    public static void main(String[] args) throws InterruptedException {
        Print2 print = new Print2();
        new Thread(() -> {
            try {
                print.printA();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "A").start();

        TimeUnit.SECONDS.sleep(2);

        new Thread(() -> {
            try {
                print.printB();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "B").start();

    }
}

class Print2 {
    public synchronized void printA() throws InterruptedException {
        TimeUnit.SECONDS.sleep(4);//模拟业务代码，假设业务需要执行4秒
        System.out.println("A");
    }

    public void printB() throws InterruptedException {
        System.out.println("B");
    }
}
```
按照第一个例子可以知道A线程先获取了对象锁，本来应该是先输出A，在输出B，但是由于`printB()`是一个非同步方法，简单来说就是，你有没有获取对象锁跟我没关系，因此，`printA()`和`printB()`可以看出是异步的，所以A线程先获取额对象锁，但是B线程也可以访问`printB()`，而我们在`printA()`中设置睡眠3秒，所有执行过程大概就是这样：
* main线程启动
* 创建A线程并且调用`start()`启动
* main线程睡眠2秒
* A线程执行`printA()`方法，执行业务代码4秒(在`printA()`中的业务代码还没执行完的时候，main线程睡眠时间到了，继续往下执行，创建B线程，并且启动，调用`printB()`方法)
* 所以先打印B，在打印A。

### 2.3 两个对象调用两个同步方法

```java
/**
 * @author zhao
 * 两个对象调用两个同步方法
 */
public class SyncTest03 {

    public static void main(String[] args) throws InterruptedException {
        //创建两个对象实例
        Print3 print = new Print3();
        Print3 pt = new Print3();
        new Thread(() -> {
            try {
                print.printA();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "A").start();


        new Thread(() -> {
            try {
                pt.printB();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "B").start();
    }
}

class Print3 {
    public synchronized void printA() throws InterruptedException {
        TimeUnit.SECONDS.sleep(4);//模拟业务代码，假设业务需要执行3秒
        System.out.println("A");
    }

    public synchronized void printB() throws InterruptedException {
        System.out.println("B");
    }
}

```

`printA()`和`printB()`竞争的不是同一把监视器锁，所有不需要考虑锁的问题，因为在`printA()`里设了睡眠，所以先打印B，在打印A。

### 2.4 一个对象调用两个静态同步方法

把`printA()`和`printB()`方法都加上`static`。

```java
/**
 * @author zhao
 * 一个对象调用两个静态同步方法
 */
public class SyncTest04 {

    public static void main(String[] args) throws InterruptedException {
        Print4 print = new Print4();
        new Thread(() -> {
            try {
                print.printA();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "A").start();

        TimeUnit.SECONDS.sleep(2);//休眠main线程，让A线程先获取锁
        new Thread(() -> {
            try {
                print.printB();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "B").start();
    }
}

class Print4 {
    public static synchronized void printA() throws InterruptedException {
        System.out.println("A");
    }

    public static synchronized void printB() throws InterruptedException {
        System.out.println("B");
    }
}

```
由于`printA()`和`printB()`方法都加上`static`，静态方法和静态变量会随着类的加载而加载，`synchronized`锁的不是对象实例，而是类模版`Print4.class`,也就是它们在竞争同一把锁，例子里让main线程休眠了，让A线程先抢到锁，所以结果是：先打印A，在打印B。

* 补充：这个的类模版可能概念上有点模糊，还是弄个例子看看把：
```java
/**
 * @author zhao
 */
public class Test {
    public static void main(String[] args) {
        ArrayList<Integer> a=new ArrayList<>();
        ArrayList<Integer> b=new ArrayList<>();
        System.out.println(a == b);
        System.out.println(a.getClass()==b.getClass());
    }
}
```
先打印`false`，然后是`true`，`a==b`比较的对象地址的引用，很明显这是两个new的操作，所以返回`false`，第二个获取的是`ArrayList`的类模版，一个类在JVM中只有一个类模版，所有返回`true`。

通过上面几个例子应该可以对`synchronized`的使用有一定的了解。主要是看到底锁的是哪个锁。


## 二 线程虚假唤醒

举个例子，简单的生产者消费者。

````java
/**
 * @author zhao
 * 车票资源类，简单模拟 生产者 生产出一张票，消费者就卖掉
 */
public class Ticket {
    private int number = 0;

    /**
     * 生产票
     *
     * @throws InterruptedException
     */
    public synchronized void incr() throws InterruptedException {
        if (number != 0) {//当票的数量不等于0，说明还有票，就不用生产票
            this.wait();
        }
        number++;//如果票==0，则生产一张票
        System.out.println(Thread.currentThread().getName() + "生产了：" + number + "张票");
        this.notifyAll();//唤醒其他线程
    }

    /**
     * 卖票
     *
     * @throws InterruptedException
     */
    public synchronized void decr() throws InterruptedException {
        if (number == 0) {//当票数量==0的时候，不能卖票了，进入wait状态
            this.wait();
        }
        number--;//当票数量不等于0，则卖出去一张票
        System.out.println(Thread.currentThread().getName() + "卖出后剩余：" + number + "张票");
        this.notifyAll();//唤醒其他线程
    }
}

````

测试类：创建两个线程，A线程生产，B线程消费

```java
/**
 * @author zhao 线程虚假唤醒
 */
public class TestTicket {
    public static void main(String[] args) {
        Ticket ticket=new Ticket();
        //A线程生产票
        new Thread(()->{
            for (int i = 0; i < 10; i++) {
                try {
                    ticket.incr();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"A").start();

        //B线程卖票
        new Thread(()->{
            for (int i = 0; i < 10; i++) {
                try {
                    ticket.decr();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"B").start();

    }
}

```
执行结果：

`
A生产了：1张票
B卖出后剩余：0张票
A生产了：1张票
B卖出后剩余：0张票
A生产了：1张票
B卖出后剩余：0张票
A生产了：1张票
B卖出后剩余：0张票
A生产了：1张票
B卖出后剩余：0张票
A生产了：1张票
B卖出后剩余：0张票
A生产了：1张票
B卖出后剩余：0张票
A生产了：1张票
B卖出后剩余：0张票
A生产了：1张票
B卖出后剩余：0张票
A生产了：1张票
B卖出后剩余：0张票
`
看起来没啥问题，但是如果多个线程生产，多个线程消费呢？修改测试类，多加了两个线程，A和B生产，C和D线程消费。

```java
/**
 * @author zhao 线程虚假唤醒
 */
public class TestTicket {
    public static void main(String[] args) {
        Ticket ticket=new Ticket();
        //A线程生产票
        new Thread(()->{
            for (int i = 0; i < 10; i++) {
                try {
                    ticket.incr();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"A").start();

        //B线程生产票
        new Thread(()->{
            for (int i = 0; i < 10; i++) {
                try {
                    ticket.incr();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"B").start();

        //C线程卖票
        new Thread(()->{
            for (int i = 0; i < 10; i++) {
                try {
                    ticket.decr();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"C").start();

        //D线程卖票
        new Thread(()->{
            for (int i = 0; i < 10; i++) {
                try {
                    ticket.decr();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"D").start();

    }
}
```

结果：

`
A生产了：1张票
C卖出后剩余：0张票
B生产了：1张票
A生产了：2张票
B生产了：3张票
C卖出后剩余：2张票
C卖出后剩余：1张票
C卖出后剩余：0张票
B生产了：1张票
A生产了：2张票
D卖出后剩余：1张票
D卖出后剩余：0张票
B生产了：1张票
C卖出后剩余：0张票
B生产了：1张票
D卖出后剩余：0张票
A生产了：1张票
D卖出后剩余：0张票
B生产了：1张票
C卖出后剩余：0张票
B生产了：1张票
D卖出后剩余：0张票
A生产了：1张票
D卖出后剩余：0张票
B生产了：1张票
C卖出后剩余：0张票
B生产了：1张票
D卖出后剩余：0张票
A生产了：1张票
D卖出后剩余：0张票
B生产了：1张票
C卖出后剩余：0张票
D卖出后剩余：-1张票
D卖出后剩余：-2张票
A生产了：-1张票
C卖出后剩余：-2张票
C卖出后剩余：-3张票
A生产了：-2张票
`
出现了问题，按照我们的想法应该是生产一张票然后卖出一张票的。但是数据出现问题了。明明在代码里加了if判断的
` 
if (number != 0) {//当票的数量不等于0，说明还有票，就不用生产票
  this.wait();
}

if (number == 0) {//当票数量==0的时候，不能卖票了，进入wait状态
  this.wait();
}

`

解决方案就是把if判断换成while，这个在javaAPI里也确实写到了：
`
当前的线程必须拥有该对象的显示器。

此方法使当前线程（称为T ）将其放置在该对象的等待集中，然后放弃对该对象的任何和所有同步声明。 线程T变得禁用线程调度目的，并且休眠，直到发生四件事情之一：

一些其他线程调用该对象的notify方法，并且线程T恰好被任意选择为被唤醒的线程。
某些其他线程调用此对象的notifyAll方法。
一些其他线程interrupts线程T。
指定的实时数量已经过去，或多或少。 然而，如果timeout为零，则不考虑实时，线程等待直到通知。
然后从该对象的等待集中删除线程T ，并重新启用线程调度。 然后它以通常的方式与其他线程竞争在对象上进行同步的权限; 一旦获得了对象的控制，其对对象的所有同步声明就恢复到现状 - 也就是在调用wait方法之后的情况。 线程T然后从调用wait方法返回。 因此，从返回wait方法，对象和线程的同步状态T正是因为它是当wait被调用的方法。
线程也可以唤醒，而不会被通知，中断或超时，即所谓的虚假唤醒 。 虽然这在实践中很少会发生，但应用程序必须通过测试应该使线程被唤醒的条件来防范，并且如果条件不满足则继续等待。 换句话说，等待应该总是出现在循环中，就像这样：

  synchronized (obj) {
         while (<condition does not hold>)
             obj.wait(timeout);
         ... // Perform action appropriate to condition
     } 
（有关此主题的更多信息，请参阅Doug Lea的“Java并行编程（第二版）”（Addison-Wesley，2000）中的第3.2.3节或Joshua Bloch的“有效Java编程语言指南”（Addison- Wesley，2001）。
`

也就是说，线程有可能没有通过`notifyAll`方法就被唤醒，解决方案就是把if换成while用循环去判断，具体为什么会造成虚假唤醒？网上都是说的不清不楚的，javaAPI里提到道格李的书里有，我过两天买个看看，这里先记得有这么个事，以及如果解决就好。

## 三 线程上下文切换

使用多线程是为了充分利用cpu资源，但是线程数不是越多越好，因为创建、销毁多线程以及上下文切换也是会消耗资源的。每个cpu核心同一时刻只能被一个线程使用，为了让用户感觉是多线程同时执行，cpu资源分配采用时间片轮转的策略。(时间片指：cpu给各个程序分配的时间，每个线程被分配一个时间段，也就是所谓的时间片)。线程在时间片内占用cpu资源执行任务，当前线程使用完时间片后，就会处于就绪状态并让出cpu给其他线程占用，这就是上下文切换。JVM中的程序计数器会记录线程执行到什么位置，等该线程持有cpu资源的时候，可以从上次断开的地方继续执行下去。上下文切换一般有两种情况：

* 让步式上下文切换：锁竞争越激烈，上下文切换越多。简单来说，如果有5个线程，cpu分5个时间片，假设每个线程可以分配200毫秒，但是由于高并发，线程数很多，时间片也会变多，每个线程分配到的时间片就会更少，造成更加频繁的上下文切换；
* cpu给线程分配的时间片用完了或者线程被中断

怎么避免或者尽可能少的造成上下文切换呢？

* 无锁：既然锁的争抢会造成性能消耗，那尽量不要让资源变成争抢的对象不就行了，比如把要操作的数据按照一定的方式分块，每个线程处理各自数据块，最后在整合，比如`LongAdder`类将对单一变量的CAS操作分散为对数组cells中多个元素的CAS操作，取值时进行求和。
* CAS算法(compare and swap)：

它包含3个参数CAS(V,E,N)。V表示要更新的变量，E表示预期值，N表示新值。当且仅当V值等于E值时，才会将V的值设为N，如果V值和E值不同，则说明已经有其他线程做了更新；
 
* 合理控制线程数；

## 四 锁

JAVA里根据通途和种类的不同可以分成很多种锁。

### 4.1 几种锁

1. 乐观锁和悲观锁

数据库层面的东西，比较基础的东西就不说了，在juc包里也有类似的思想；

2.独占锁和共享锁

常用的`ReentrantLock`和`synchronized`就是独占锁，即：只有一个线程持有锁；
`ReadWriteLock`读写锁，读是共享的，写是独占的；

3. 可重入锁

可重入锁的意思是一个线程获取了某个锁，计数器+1，在没释放这个资源之前，可以继续获取这个锁，计数器再+1，不需要和其他线程竞争，这就是可重入锁。 `ReentrantLock`和`synchronized`这种都是可重入锁；

4. 公平锁和非公平锁

公平锁表示线程获取资源的顺序是按照线程请求锁的时间早晚来决定的，也就是最早请求的线程会最早获得锁。
非公平锁则相反，不一定先来的线程就会获得锁。

5. 分段锁 segment

JDK1.7之前`ConcurrentHashMap`就是用分段锁，将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问，能够实现真正的并发访问。。

6. 自旋锁

自旋锁是指尝试获取锁的线程，如果没有抢到锁，不会立刻阻塞，而且采用循环的方式，不断的尝试去获取锁，这样的好处是一定程度上可以减少CPU上下文切换，缺点是循环会消耗CPU资源。

7. 无锁、偏向锁、轻量级锁、重量级锁

`synchronized`在JDK1.6之前是直接调用操作系统的内核，给对象加锁，所以它是重量级锁。1.6之后对`synchronized`做了优化，使用它不会一开始就是重量级锁，它有一个升级的过程。一个资源，锁升级的过程大概就是这样：无锁-偏向锁-轻量级锁-重量级锁。

* 无锁 就是对象刚创建出来，或者说 没有对对象做任何加锁的操作的时候；
* 偏向锁是同步代码一直被一个线程访问，那么线程就会自动获取该锁，降低资源消耗。
* 轻量级锁
* 重量级锁

### 4.2 怎么避免死锁

死锁是指两个或者两个以上的线程在执行过程中，因争抢资源而造成的互相等待的现象，在没有外力作用下，这些线程会一直互相等待而无法继续运行下去。

产生死锁的四个条件：

* 互斥条件：指资源已经被一个线程占用，如果此时还有其他线程请求该资源，则请求者只能等待，直到占用资源的线程释放该资源；
* 请求并持有条件：指一个线程已经持有了至少一个资源，但是又提出新的资源请求，而新资源已被其他线程占用，所以当前线程会阻塞，而且阻塞的同时并不能释放自己已经持有的资源；
* 不可剥夺条件：进程已获得的资源，在末使用完之前，不能被其他线程抢占；
* 环路等待条件：比如线程t0 t1 t2，t0在等待t1占用的资源，而t1在等待t2占用的资源，t2在等待t0占用的资源。

死锁示例：

```java
/**
 * @author zhao 死锁
 */
public class DeadLockTest {
    private static Object objectA = new Object();
    private static Object objectB = new Object();

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (objectA) {
                System.out.println("获取了A资源");
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("正在等待获取B资源");
                synchronized (objectB) {
                    System.out.println("获取了B资源");
                }
            }

        },"t1").start();

        new Thread(() -> {
            synchronized (objectB) {
                System.out.println("获取了B资源");
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("正在等待获取A资源");
                synchronized (objectA) {
                    System.out.println("获取了A资源");
                }
            }
        },"t2").start();
    }
}

```

控制台打印：
`
获取了A资源
获取了B资源
正在等待获取A资源
正在等待获取B资源

`

发现程序一直都没有退出，两个线程都在等待对象释放资源，造成死锁，看看这个例子是怎么满足上面的4个条件的。

1. `objectA`和`objectB`是互斥资源，当t1线程获取了`objectA`后，t2线程必须等他释放，否则阻塞；
2. t1线程首先获得了`objectA`锁然后在请求获取`objectB`锁，满足请求并持有条件；
3. `objectA`被t1持有后不会被其他线程掠夺走，资源的不可剥夺性；
4. t1持有`objectA`，并等待`objectB`，而持有`objectB`的t2线程在等待持有`objectA`的t1线程，构成环形条件。

1. 保证加锁顺序

上面的例子，t1线程是要先获得`objectA`锁，然后在获取`objectB`锁，而t2线程是先获得`objectB`在获取`objectA`，这样就会死锁，因此只要保证所有的线程都是按照相同的顺序加锁，就不会出现死锁。修改t2线程：

```
new Thread(() -> {
    synchronized (objectA) {
        System.out.println("获取了B资源");
        try {
            TimeUnit.SECONDS.sleep(1);
            System.out.println("正在等待获取A资源");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        synchronized (objectB) {
            System.out.println("获取了A资源");
        }
    }
}).start();
```

这样两个线程获取锁的顺序相同，都先抢`objectA`，只有抢到了`objectA`才有抢`objectB`的资格，而其他没有抢到`objectA`的线程，则阻塞，等t1执行完之后在尝试获取`objectA`。这种方式可以有效的避免死锁，但是需要提前知道所有的锁，并对锁做适当的排序，所以具体还是看业务情况，只能说这是一种方案。

2. 加锁时限

这个比较简单，也好理解，就是在尝试获取锁的时候加一个超时时间，这也就意味着在尝试获取锁的过程中若超过了这个时限该线程则放弃对该锁请求，加锁超时后可以先继续运行干点其它事情，再回头来重复之前加锁的逻辑，比如Lock接口里的`tryLock(long time, TimeUnit unit)`。

3. 死锁检测

每当一个线程获得了锁，会在线程和锁相关的数据结构中比如map将其记下。除此之外，每当有线程请求锁，也需要记录在这个数据结构中，当一个线程请求锁失败时，这个线程可以遍历锁的关系图看看是否有死锁发生。例如，线程A请求锁7，但是锁7这个时候被线程B持有，这时线程A就可以检查一下线程B是否已经请求了线程A当前所持有的锁。如果线程B确实有这样的请求，那么就是发生了死锁。

## 五 并发和并行

并发：多个任务同时在执行，这里的"同时"是cpu上下文切换，让用户看起来程序是在同时执行的，在单核CPU上还是有先后顺序的；
并行：多个任务同时在执行；这里的"同时"是真正的同时，同一时刻两个线程都是出于running状态，比如3个线程在3个CPU核心上跑，这才是真正的同时做某件事；

在单核CPU时代，多个任务都是并发执行的，这是因为一个CPU核心同一时间只能执行一个任务，当一个任务占用CPU时，其他任务就会被挂起，当占用CPU的线程时间片用完后，就会把CPU让出来给其他线程继续竞争，这就是上下文切换，前面也提到过。在单核CPU上多线程编程没什么意义，因为只是一个CPU核心在切换而已，而且还会造成额外的开销。
多核时代打破了单核CPU对多线程的限制，多核CPU意味着线程可以使用自己的CPU运行，不需要上下文切换，这才是真正的并行，我们在代码里写的多线程，只是叫并发，它可能是并行的(线程分到不同的CPU核心上)，也有可能不是并行的(多个线程在一个CPU上跑，上下文切换看起来在同时执行)。

## 六 JAVA内存模型

java内存模型(Java Memory Model，JMM)是java虚拟机规范定义的，用来屏蔽掉java程序在各种不同的硬件和操作系统对内存的访问的差异。

### 6.1 多线程带来的三个问题

1. 可见性问题

现在都是多核CPU，每个核心都有自己独立一级缓存，二级缓存，共享三级缓存(具体要看CPU架构)，每个核都至少有一个一级缓存。多个线程访问进程中的某个共享内存，且这多个线程分别在不同的核心上执行，则每个核心都会在各自的caehe中保留一份共享内存的缓冲。由于多核是可以并行的，可能会出现多个线程同时写各自的缓存的情况，而各自的cache之间的数据就有可能不同。
在CPU和主存之间增加缓存，解决了CPU和内存之间的速度差异，但是在多线程场景下就可能存在缓存一致性问题。，多线程情况下，每个核心计算之后都要把值给一级缓存，在刷新到主内存，但是每个核心的缓存是独立的，互相不可见的，也就是说：假设有个变量 a = 10, t1线程把变量改a改成了20，但是其他线程并不知道。这就是可见性问题。

2. 原子性问题

指在一个操作中cpu不可以在中途暂停然后再调度，要么全部执行完毕，要么就不执行。

3. 指令重排序问题(有序性)

为了使处理器内部的运算单元能够尽量的被充分利用，处理器可能会对输入代码进行乱序执行处理。也就是说，cpu执行指令的顺序并不是按照我们写代码的顺序来的。但是指令重排序是需要遵守`as-if-serial`的。
* as-if-serial语义的意思是：不管怎么进行指令重排序，单线程内程序的执行结果不能被改变。编译器，处理器进行指令重排序都必须要遵守as-if-serial语义规则。
* 为了遵守as-if-serial语义，编译器和处理器对存在依赖关系的操作，都不会对其进行重排序，因为这样的重排序很可能会改变执行的结果，但是对不存在依赖关系的操作，就有可能进行重排序。

Happens-Before原则
    happens-before可以理解为“先于”，是用来指定两个操作之间的执行顺序，由于这个两个操作可以在一个线程之内，也可以在不同线程之间。因此，JMM可以通过happens-before关系来保证跨线程的内存可见性（如果A线程是对变量进行写操作，而B线程是对变量进行读操作，那么如果A线程是先于B线程的操作，那么A线程写操作之后的数据对B线程也是可见的）
具体的定义：
* 如果一个操作“先于”另一个操作，那么第一操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前；
* 两个操作是happens-before的关系，也并不意味着JVM会按照这个关系去顺序执行，因为会存在重排序的可能，但是进行了重排序的执行结果，与此happens-before的关系顺序执行的结果一致的话，那就说明这个重排序是合法的（也就是JVM允许这样的重排序）。
    
 例如：
 
```
int a = 1;
int b = 2;
int c = a + b;
```
这种情况可以重排序成:
```
int b = 2;
int a = 1;
int c = a + b;
```
为了保证并发编程中保证数据的原子性、可见性、有序性，内存模型定义了多线程操作规范。<font color=red>Java内存模型规定了所有的变量都存储在主内存中，每条线程还有自己的工作内存，线程的工作内存中保存了该线程中是用到的变量的主内存副本拷贝，线程对变量的所有操作都必须在工作内存中进行，而不能直接读写主内存。不同的线程之间也无法直接访问对方工作内存中的变量，线程间变量的传递均需要自己的工作内存和主存之间进行数据同步进行。</font>

## 6.2 JAVA内存模型怎么解决上面的问题的

内存屏障

* 读内存屏障`Load Barrier`，在指令前插入`Load Barrier`，可以让高速缓存中的数据失效，强制从新从主内存加载数据；
* 写内存屏障`Store Barrier`，在指令后插入`Store Barrier`，能让写入缓存中的最新数据更新写入主内存，让其他线程可见。
* 内存屏障有两个作用：
1. 阻止屏障两侧的指令重排序；
2. 强制把写缓冲区/高速缓存中的数据写回主内存，让缓存中相应的数据失效。
