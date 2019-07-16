# Java多线程编程

- [对象和变量的并发访问](#一-对象和变量的并发访问)
    - [synchronized同步方法](#1.1-synchronized同步方法)
        - [方法内的变量为线程安全](#111-方法内的变量为线程安全)
        - [实例变量非线程安全](#112-实例变量非线程安全)
        - [多个对象多个锁](#113-多个对象多个锁)
        - [synchronized方法和锁对象](#114-synchronized方法和锁对象)
        - [synchronized锁重入](#115-synchronized锁重入)
        - [synchronized同步代码块](#116-synchronized同步代码块)
        - [synchronized代码块的对象监视器](#117-synchronized代码块的对象监视器)
        - [将任意对象作为对象监视器](#118-将任意对象作为对象监视器)

## 一 对象和变量的并发访问

### 1.1 synchronized同步方法
synchronized锁的几种不同情况：
* 修饰实例方法，锁是当前实例对象，进入同步代码前要获得当前实例的锁；
* 修饰`static`方法，锁是当前类的class对象，进入同步代码前要获得当前类对象的锁；
* 修饰方法块，锁是括号里面的对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。

### 1.1.1 方法内的变量为线程安全
看一下测试代码：
```java
public class SynchronizedTest1 {
    public void add(String name) throws InterruptedException {
        int num = 0;//变量定义在方法内部
        if(name.equals("a")){
            num=10;
            Thread.sleep(2000);
        }else{
            num=20;
        }
        System.out.println("name:"+name+",num:"+num);
    }


    public static void main(String[] args) {
        SynchronizedTest1 test = new SynchronizedTest1();
        new Thread(() -> {
            try {
                test.add("a");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
        new Thread(() -> {
            try {
                test.add("b");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```
输出永远都是：
```
name:b,num:20
name:a,num:10
```
证明方法种的变量不存在线程安全不安全的问题，永远是线程安全的。这是方法内部变量的私有性造成的。
### 1.1.2 实例变量非线程安全
代码改动一下，把num改成实例变量：
```java
public class SynchronizedTest1 {
    private int num = 0;//实例变量

    public void add(String name) throws InterruptedException {
        if (name.equals("a")) {
            num = 10;
            System.out.println("a set over");
            Thread.sleep(2000);
        } else {
            num = 20;
            System.out.println("b set over");
        }
        System.out.println("name:" + name + ",num:" + num);
    }


    public static void main(String[] args) {
        SynchronizedTest1 test = new SynchronizedTest1();
        new Thread(() -> {
            try {
                test.add("a");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
        new Thread(() -> {
            try {
                test.add("b");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```
可以看到产生了线程不安全问题：
```
a set over
b set over
name:b,num:20
name:a,num:20
```
说明如果多个线程同时操作对象中的实例变量，有可能会产生线程不安全的问题。要实现线程安全只需要在`add`方法前加上关键字`synchronized`即可。
代码：
```java
public class SynchronizedTest1 {
    private int num = 0; //实例变量

    synchronized public void add(String name) throws InterruptedException {
        if (name.equals("a")) {
            num = 10;
            System.out.println("a set over");
            Thread.sleep(2000);
        } else {
            num = 20;
            System.out.println("b set over");
        }
        System.out.println("name:" + name + ",num:" + num);
    }

    public static void main(String[] args) {
        SynchronizedTest1 test = new SynchronizedTest1();
        new Thread(() -> {
            try {
                test.add("a");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
        new Thread(() -> {
            try {
                test.add("b");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```
发现输出符合我们的预期：
```
a set over
name:a,num:10
b set over
name:b,num:20
```
实验结论：在多个线程访问同一个对象中的同步方法时是线程安全的。上面代码因为实现了同步，所以先打印出来a，然后在打印b。
### 1.1.3 多个对象多个锁
在上面代码的基础上在修改一下，创建2个对象实例。
```java
public class SynchronizedTest1 {
    private int num = 0; //实例变量

    synchronized public void add(String name) throws InterruptedException {
        if (name.equals("a")) {
            num = 10;
            System.out.println("a set over");
            Thread.sleep(2000);
        } else {
            num = 20;
            System.out.println("b set over");
        }
        System.out.println("name:" + name + ",num:" + num);
    }

    public static void main(String[] args) {
        SynchronizedTest1 test = new SynchronizedTest1();
        SynchronizedTest1 test2 = new SynchronizedTest1();
        new Thread(() -> {
            try {
                test.add("a");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
        new Thread(() -> {
            try {
                test2.add("b");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```
输出
```
a set over
b set over
name:b,num:20
name:a,num:10
```
发现设置的num值是没有问题的，但是打印的顺序缺乱了。这是因为`synchronized`取的锁都是对象锁，而不是一段代码或者函数当作锁，哪个线程先执行，那么这个线程就先获取到这个方法的对象锁，其他线程就处于等待状态，前提是多个线程访问同一个对象。
### 1.1.4 synchronized方法和锁对象
为了证明上面的代码锁的是对象，写了个测试用例
```java
public class MyObject {
    public void funA() throws InterruptedException {
        System.out.println("funA方法运行，线程名："+Thread.currentThread().getName());
        Thread.sleep(2000);
        System.out.println("end..");
    }
    
    public static void main(String[] args) {
        MyObject object=new MyObject();
        new Thread(()->{
            try {
                object.funA();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"A").start();
        new Thread(()->{
            try {
                object.funA();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"B").start();
    }
}

```
输出：
```
funA方法运行，线程名：A
funA方法运行，线程名：B
end..
end..
```
这时候并没有同步，两个线程同时进入方法。那么加上同步锁试试。
```java
public class MyObject {
    synchronized public void funA() throws InterruptedException {
        System.out.println("funA方法运行，线程名：" + Thread.currentThread().getName());
        Thread.sleep(2000);
        System.out.println("end..");
    }

    public static void main(String[] args) {
        MyObject object = new MyObject();
        new Thread(() -> {
            try {
                object.funA();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "A").start();
        new Thread(() -> {
            try {
                object.funA();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "B").start();
    }
}

```
输出：
```
funA方法运行，线程名：A
end..
funA方法运行，线程名：B
end..

```
这才是方法正常调用的结果，说明`synchronized`锁修饰的方法一定是排队进行的。那如果一个类有两个方法，一个实现了同步，一个普通方法，线程A访问了同步方法，线程B还可以访问另外一个普通方法吗？写个测试代码：
```java
public class MyObject {
    //funA 实现了同步
    synchronized public void funA() throws InterruptedException {
        System.out.println("funA方法运行，线程名：" + Thread.currentThread().getName());
        Thread.sleep(2000);
        System.out.println("funA end");
    }
    //funB 普通方法
    public void funB() throws InterruptedException {
        System.out.println("funB方法运行，线程名：" + Thread.currentThread().getName());
        Thread.sleep(2000);
        System.out.println("funB end.. ");
    }
    public static void main(String[] args) {
        MyObject object = new MyObject();
        new Thread(() -> {
            try {
                object.funA();//调用funA
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "A").start();
        new Thread(() -> {
            try {
                object.funB();//调用funB
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "B").start();
    }
}

```

输出：
```
funA方法运行，线程名：A
funB方法运行，线程名：B
funA end
funB end.. 
```
虽然线程A先持有了对象锁，但是线程B完全可以调用非`synchronized`的方法。如果给方法B也加上同步呢？
```java
public class MyObject {
    //funA 实现了同步
    synchronized public void funA() throws InterruptedException {
        System.out.println("funA方法运行，线程名：" + Thread.currentThread().getName());
        Thread.sleep(2000);
        System.out.println("funA end");
    }
    //funB 普通方法
    synchronized public void funB() throws InterruptedException {
        System.out.println("funB方法运行，线程名：" + Thread.currentThread().getName());
        Thread.sleep(2000);
        System.out.println("funB end.. ");
    }
    public static void main(String[] args) {
        MyObject object = new MyObject();
        new Thread(() -> {
            try {
                object.funA();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "A").start();
        new Thread(() -> {
            try {
                object.funB();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "B").start();
    }
}
```
输出：
```
funA方法运行，线程名：A
funA end
funB方法运行，线程名：B
funB end.. 
```
证明是同步调用的。结论：
* A线程先持有object对象的锁，B线程可以以异步的方式调用object对象中非`synchronized`的方法；
* A线程先持有object对象的锁，B线程如果在这时调用object中的`synchronized`方法则需要等待，也就是同步。
### 1.1.5 synchronized锁重入
关键字`synchronized`拥有锁重入功能，也就是使用`synchronized`时，当一个线程的到一个对象锁后，再次请求此对象锁时是可以再次得到该对象的锁。
```java
public class Service {
    synchronized public void fun1() {
        System.out.println("fun1");
        fun2();
    }

    synchronized public void fun2() {
        System.out.println("fun2");
        fun3();
    }

    synchronized public void fun3() {
        System.out.println("fun3");
    }

    public static void main(String[] args) {
        Service service = new Service();
        new Thread(() -> {
            service.fun1();
        }).start();
    }
}
```
输出：
```
fun1
fun2
fun3
```
可重入锁的概念是：自己可以再次获取自己的内部锁，比如一个线程获取了某个对象锁，这个锁还没有释放，当它想要再次获取这个对象锁的时候是可以获取到的。可重入锁也至此父子类继承环境。
```java
public class TTest {
    public static void main(String[] args) {
        new Thread(() -> {
            SubClass subClass = new SubClass();
            subClass.subFun();
        }).start();
    }
}

class SubClass extends SuperClass {
    synchronized public void subFun() {
        try {
            while (a > 0) {
                a--;
                System.out.println("subclass a=" + a);
                Thread.sleep(1000);
                super.superFun();
            }
        } catch (Exception e) {

        }
    }
}

class SuperClass {
    public int a = 10;

    public void superFun() throws InterruptedException {
        a--;
        System.out.println("superClass  a=" + a);
        Thread.sleep(1000);
    }
}
```
输出结果：
```
subclass a=9
superClass  a=8
subclass a=7
superClass  a=6
subclass a=5
superClass  a=4
subclass a=3
superClass  a=2
subclass a=1
superClass  a=0
```
说明父子继承关系时，子类是完全可以通过"可重入锁"调用父类的同步方法。
### 1.1.6 synchronized同步代码块
用关键字`synchronized`修饰方法时，一个线程调用这个同步方法，如果这个方法执行时间很长，那么其他线程必须等待比较长的时间，这样的情况，可以使用同步代码块来解决。
```java
public class Task {
    public void doLongTimeTask() {
        try {
            System.out.println("当前线程："+Thread.currentThread().getName()+"，正在执行一个时间较长的任务");
            Thread.sleep(2000);

            synchronized (this) {
                System.out.println("当前线程："+Thread.currentThread().getName()+"，执行同步代码块");
                Thread.sleep(1000);
            }
            System.out.println("当前线程："+Thread.currentThread().getName()+"，执行完毕");
        } catch (Exception e) {

        }
    }
    
    public static void main(String[] args) {
        Task task = new Task();
        new Thread(() -> {
            task.doLongTimeTask();
        }, "A").start();
        new Thread(() -> {
            task.doLongTimeTask();
        }, "B").start();
    }
}

```
输出：
```
当前线程：A，正在执行一个时间较长的任务
当前线程：B，正在执行一个时间较长的任务
当前线程：A，执行同步代码块
当前线程：A，执行完毕
当前线程：B，执行同步代码块
当前线程：B，执行完毕
```
结论：当一个线程访问`synchronized`同步代码块的时候，另外的线程仍然可以访问该对象中非`synchronized(this)`的代码。在用一个更直观的例子试试，证明一半同步，一半异步。
```java
public class Task {
    public void doLongTimeTask() {
        for (int i = 0; i < 10; i++) {
            System.out.println("非同步方法执行，线程名："+Thread.currentThread().getName()+",i="+i);
        }

        synchronized (this){
            for (int i = 0; i < 10; i++) {
                System.out.println("同步代码块执行，线程名："+Thread.currentThread().getName()+",i="+i);
            }
        }
    }

    public static void main(String[] args) {
        Task task = new Task();
        new Thread(() -> {
            task.doLongTimeTask();
        }, "A").start();
        new Thread(() -> {
            task.doLongTimeTask();
        }, "B").start();
    }
}
```
输出：
```
非同步方法执行，线程名：A,i=0
非同步方法执行，线程名：A,i=1
非同步方法执行，线程名：B,i=0
非同步方法执行，线程名：B,i=1
非同步方法执行，线程名：B,i=2
非同步方法执行，线程名：B,i=3
非同步方法执行，线程名：B,i=4
非同步方法执行，线程名：B,i=5
非同步方法执行，线程名：B,i=6
非同步方法执行，线程名：B,i=7
非同步方法执行，线程名：B,i=8
非同步方法执行，线程名：B,i=9
同步代码块执行，线程名：B,i=0
同步代码块执行，线程名：B,i=1
同步代码块执行，线程名：B,i=2
同步代码块执行，线程名：B,i=3
同步代码块执行，线程名：B,i=4
同步代码块执行，线程名：B,i=5
同步代码块执行，线程名：B,i=6
同步代码块执行，线程名：B,i=7
同步代码块执行，线程名：B,i=8
同步代码块执行，线程名：B,i=9
非同步方法执行，线程名：A,i=2
非同步方法执行，线程名：A,i=3
非同步方法执行，线程名：A,i=4
非同步方法执行，线程名：A,i=5
非同步方法执行，线程名：A,i=6
非同步方法执行，线程名：A,i=7
非同步方法执行，线程名：A,i=8
非同步方法执行，线程名：A,i=9
同步代码块执行，线程名：A,i=0
同步代码块执行，线程名：A,i=1
同步代码块执行，线程名：A,i=2
同步代码块执行，线程名：A,i=3
同步代码块执行，线程名：A,i=4
同步代码块执行，线程名：A,i=5
同步代码块执行，线程名：A,i=6
同步代码块执行，线程名：A,i=7
同步代码块执行，线程名：A,i=8
同步代码块执行，线程名：A,i=9
```
### 1.1.7 synchronized代码块的对象监视器
在使用`synchronized (this)`的时候，当一个线程访问`synchronized (this)`同步代码块时，其他线程对这个对象里其他的所有的`synchronized (this)`代码块的访问，是同步的、阻塞的，这说明`synchronized `是对象监视器。
这说明synchronized同步方法或synchronized(this)  同步代码块分别有两种作用。
1. synchronized同步方法
* 对其他synchronized同步方法或`synchronized (this)`同步代码块呈阻塞状态；
* 同一时间只有一个线程可以执行synchronized同步方法中的代码。
2. synchronized(this)同步代码块
* 对其他synchronized同步方法或`synchronized (this)`同步代码块呈阻塞状态。
* 同一时间只有一个线程可以执行`synchronized (this)`同步代码块中的代码。
```java
public class ObjectService {

    public void funA() {
        try {
            synchronized (this) {
                System.out.println("A 执行开始");
                Thread.sleep(2000);
                System.out.println("A 执行结束");
            }
        } catch (Exception e) {

        }
    }

    public void funB() {
        System.out.println("B 执行开始");
        System.out.println("B 执行结束");
    }

    public static void main(String[] args) {
        ObjectService objectService = new ObjectService();
        new Thread(() -> {
            objectService.funA();
        }, "A").start();//线程A调用同步代码块
        new Thread(() -> {
            objectService.funB();
        }, "B").start();//线程B调用非同步代码块
    }
}
```
输出：
```
A 执行开始
B 执行开始
B 执行结束
A 执行结束
```
发现 A线程执行funA方法的时候，其他线程依然能访问非`synchronized`的方法，不是同步的。如果把funB也加上同步代码块的话，试试
```java
public class ObjectService {

    public void funA() {
        try {
            synchronized (this) {
                System.out.println("A 执行开始");
                Thread.sleep(2000);
                System.out.println("A 执行结束");
            }
        } catch (Exception e) {

        }
    }

    public void funB() {
        synchronized (this) {
            System.out.println("B 执行开始");
            System.out.println("B 执行结束");
        }
    }

    public static void main(String[] args) {
        ObjectService objectService = new ObjectService();
        new Thread(() -> {
            objectService.funA();
        }, "A").start();//线程A调用同步代码块
        new Thread(() -> {
            objectService.funB();
        }, "B").start();//线程B调用非同步代码块
    }
}
```
输出：
```
A 执行开始
A 执行结束
B 执行开始
B 执行结束
```
可以发现他是同步的，证明了上面的观点是对的。

### 1.1.8 将任意对象作为对象监视器
