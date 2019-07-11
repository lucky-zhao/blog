# 深入理解JAVA虚拟机
- [运行时数据区域](#二-运行时数据区域)
    - [堆内存](#21-堆内存)
    - [方法区](#22-方法区)
    - [虚拟机栈](#23-虚拟机栈)
    - [本地方法栈](#24-本地方法栈)
    - [程序计数器](#25-程序计数器)
    - [直接内存](#26-直接内存)
- [HotSpot虚拟机对象探秘](#三-HotSpot虚拟机对象探秘)
    - [对象的创建](#31-对象的创建)
    - [对象的内存布局](#32-对象的内存布局)
    - [对象的访问定位](#33-对象的访问定位)
    - [代码实现OutOfMemoryError异常](#34-代码实现OutOfMemoryError异常)
        - [堆内存溢出](#341-堆内存溢出)
        - [虚拟机栈和本地方法栈内存溢出](#342-虚拟机栈和本地方法栈内存溢出)
        - [方法区和运行时常量池溢出](#343-方法区和运行时常量池溢出)
        - [本地直接内存溢出](#344-本地直接内存溢出)
- [垃圾回收机制和内存分配策略](#四-垃圾回收机制和内存分配策略)
    - [对象已死吗](#41-对象已死吗)
         - [引用计数算法](#411-引用计数算法)
         - [引用计数算法](#412-可达性分析算法)
    - [垃圾收集算法](#42-垃圾收集算法)
        - [标记-清除算法](#421-标记-清除算法)
        - [复制算法](#422-复制算法)
        - [标记整理算法](#423-标记整理算法)
        - [分代收集算法](#424-分代收集算法)
    - [看懂GC日志](#43-看懂GC日志)
- [内存分配和回收策略](#五-内存分配和回收策略)
    - [对象内存分配](#51-对象内存分配)
        
## 一 先看下JDK1.8前的Java虚拟机的内存区域。
![JVM](https://github.com/lucky-zhao/blog/blob/master/jvm/img/jvm.jpg "JVM内存区域")


## JDK1.8的内存划分
![JVM](https://github.com/lucky-zhao/blog/blob/master/jvm/img/JDK1.8.png "JVM内存区域")
## 二 运行时数据区域

### 2.1 堆内存 
对于大多数应用来说，Java堆内存(Java Heap)是Java虚拟机所管理的内存中最大的一块。堆内存是线程共享的，在虚拟机启动时创建，堆内存主要存放对象实例，也是GC主要活动区域。
堆内存还分新生代(Eden区，From Survivor区，To Survivor区)和老年代，进一步的划分内存区域是为了能更好的回收内存或者更快的分配内存。对象生成后最先放到Eden区，也就是新生代，新生代的对象很多都是"朝生夕死"的，在经过GC后会放入survivor1区，如果survivor1区放满了，就GC这个survivor1区，然后把还能存活的对象转移到survivor2区，那么此时survivor1中的对象就是可以回收的对象，然后把带有对象survivor2与没有对象的survivor1交换，因为gc survivor的时候是对第一个，应该把所有survivor中的对象都gc一次，看看对象是否可以清除。当我们多次gc的时候，survivor中仍然有对象存活，就将这些数据放到老年代中。如果老年代也存储不下的时候，会触发full gc。

### 2.2 方法区
方法区和堆一样是线程共享的内存区域，主要存储已被虚拟机加载的类信息、常量、静态变量、JIT编译器编译后的代码等信息。方法区有个名字Non-Heap，是把方法区和Java堆内存区分开。方法区也被称为永久代。方法区是规范层面的东西，规定了这个区域存放哪些东西，永久代是对方法区的实现。并不是所有jvm都有永久代的。JDK1.8已经彻底移除了这一区域，引入了一个新的内存区域元空间(metaspace)。
* 运行时常量池是方法区的一部分。Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有常量池信息.Java并不要求常量一定只有编译期才能产生，运行期间也可能将新的常量放入常量池中，比如`String`类的`intern()`方法。后面详细了解补充一下。

### 2.3 虚拟机栈
虚拟机栈是线程私有的，他的生命周期和线程相同。虚拟机栈描述Java方法执行的内存模型：每个方法在执行时都会创建一个栈帧(Stack Frame)。用于存储局部变量，操作数栈，动态链接，方法出口等信息。每个方法从调用到执行完毕的过程，都对应了一个栈帧在虚拟机栈中入栈和出栈的过程。局部变量存放了编译器可知的各种基本数据类型、对象的引用(reference类型，他不等于对象本身，引用是一个指向对象的地址的引用指针，也可以能是指向一个代表对象的句柄或者其他与此对象相关的位置)。在虚拟机栈用会有两种异常情况：
* 栈从数据结构上来说是"先进后出，后进先出"的特性。因此，死循环的递归调用或者递归很深的时候可能会造成栈内存溢出`StackOverflowError`。
* 如果虚拟机栈可以动态扩展，如果扩展时无法申请到足够的内存，就会抛出`OutOfMemoryError`异常。

### 2.4 本地方法栈
和虚拟机栈发挥的作用非常相似。区别在于虚拟机栈为虚拟机执行Java方法服务，而本地方法栈则为虚拟机用到的Native方法服务。在虚拟机规范中对本地方法栈中方法所使用的语言、方法、数据结构并没有强制规定，因此可以由具体的虚拟机实现。和虚拟机栈一样，本地方法栈也会抛出`StackOverFlowError `和`OutOfMemoryError `两种异常。

### 2.5 程序计数器
程序计数器是一块较小的内存空间，可以看做是当前线程所执行的字节码的行号指示器。首先我们要搞清楚JVM的多线程实现方式。JVM的多线程是通过CPU时间片轮转（即线程轮流切换并分配处理器执行时间）算法来实现的。也就是说，某个线程在执行过程中可能会因为时间片耗尽而被挂起，而另一个线程获取到时间片开始执行。当被挂起的线程重新获取到时间片的时候，它要想从被挂起的地方继续执行，就必须知道它上次执行到哪个位置，每个线程都有一个独立的程序计数器。
* 执行Java方法时，程序计数器记录的是正在执行的字节码指令的地址。
* 执行Native方法时，程序计数器的值为Undefined，因为Native是通过Java直接调用本地C/C++库，可以认为Native方法是C/C++库提供的接口。由于该方法实现不在java内进行，因此也就没有字节码。此内存区域是唯一一个在Java虚拟机规范中没有规定任何`OutOfMemoryError`情况的区域。

### 2.6 直接内存
直接内存(Direct Memory)并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域。但是这部分内存也被频繁地使用。而且也可能导致`OutOfMemoryError`异常出现。JDK1.4中新加入的NIO，引入了一种基于通道(Channel)与缓存区(Buffer)的IO方式，他可以使用Native函数库直接分配堆外内存，然后通过一个存储在 Java 堆中的 DirectByteBuffer 对象作为这块内存的引用进行操作。这样就能在一些场景中显著提高性能，因为避免了在 Java 堆和 Native 堆之间来回复制数据。直接内存的分配不会受到Java堆大小的限制，但是既然是内存就会受到本机总内存大小以及处理器寻址空间的限制。

## 三 HotSpot虚拟机对象探秘

### 3.1 对象的创建
* 类加载检查：
虚拟机遇到一条 new 指令时，首先将去检查这个指令的参数是否能在常量池中定位到这个类的符号引用，并且检查这个符号引用代表的类是否已被加载过、解析和初始化过。如果没有，那必须先执行相应的类加载过程。
* 分配内存：
在类加载检查通过后，就是分配内存，对象所需内存大小在类加载检查完成后就可以确定。为对象分配空间的任务等同于把一块确定大小的内存从Java堆内存中划分出来。分配的方式有两种：
   * 指针碰撞：假设堆内存是绝对规整的，用过的内存放在一边，没用过的放在另外一边，中间有个指针作为分界点的指示器，那么分配内存就是把指针向空闲空间移动一段与对象大小相等的距离。
   * 空闲列表：假设堆内存不是规整的，已使用的内存和未使用的互相交错，就不能用指针碰撞，虚拟机必须维护一个空闲空间的列表，记录哪些内存块是可以用的，在分配的时候找到一块足够大的内存划分给对象，并更新列表记录。这种方式叫 空闲列表。
* 设置对象头：
例如这个对象是那个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的 GC 分代年龄等信息。 这些信息存放在对象头中。 另外，根据虚拟机当前运行状态的不同，如是否启用偏向锁等，对象头会有不同的设置方式。
* 初始化对象：
从虚拟机的角度来讲此时对象已经产生了，但是对于Java来说，对象创建才刚开始，所有字段都为零。执行new指令后会接着执行init方法，把对象按照程序员的意愿，进行初始化，这样一个真正可用的对象才完全产生出来。

### 3.2 对象的内存布局
在 Hotspot 虚拟机中，对象在内存中的布局可以分为 3 块区域：对象头、实例数据和对齐填充。对象头包括两部分，第一部分用于存储对象自身的自身运行时数据（哈希码、GC 分代年龄、锁状态标志等），另一部分是类型指针，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是那个类的实例。如果对象是一个Java数组，那么在对象头中必须有一块用于记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定对象的大小，但是从数组的元数据中却无法确认数组的大小。对齐填充部分不是必然存在的，也没有什么特别的含义，仅仅起占位作用。 因为 Hotspot 虚拟机的自动内存管理系统要求对象起始地址必须是 8 字节的整数倍，换句话说就是对象的大小必须是 8 字节的整数倍。而对象头部分正好是 8 字节的倍数（1 倍或 2 倍），因此，当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。

### 3.3 对象的访问定位
Java程序需要通过栈内存上的引用堆内存上的对象。虚拟机规范只规定了一个对象的引用，但是怎么定位、访问这个对象，不同的虚拟机可以有不同的实现。一般访问方式有两种：
* 句柄。如果使用句柄的话，堆内存就要划出一块内存作为句柄池，栈中的引用存放的就是对象的句柄地址。
* 直接指针。如果使用直接指针访问，那么 Java 堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，而 reference 中存储的直接就是对象的地址。
两种对象访问方式各有优势。使用句柄来访问的最大好处是 reference 中存储的是稳定的句柄地址，在对象被移动时(垃圾回收时，对象移动是非常普遍的行为)只会改变句柄中的实例数据指针，而 reference 本身不需要修改。使用直接指针访问方式最大的好处就是速度快，它节省了一次指针定位的时间开销。

### 3.4 代码实现OutOfMemoryError异常

### 3.4.1 堆内存溢出

设置虚拟机启动参数：`-Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8`。`-Xms20m`:初始化堆内存大小20M；`-Xmx20m`:堆内存最大为20M。将堆最大内存和最小内存设置一样，可以避免自动扩展内存。`-Xmn10M`:新生代内存10M；`-XX:+PrintGCDetails`输出GC详细日志。`-XX:SurvivorRatio=8`：设置两个Survivor和Eden的比值，8 表示两个Survivor ： Eden = 2：8，每个Survivor占 1/10。
java测试代码：
```java
/**
 * Created by zhao
 * 堆内存溢出
 * -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
 */
public class HeapOOM {
    static class OOMObjet {
    }

    public static void main(String[] args) {
        List<OOMObjet> list = new ArrayList<>();
        while (true) {
            list.add(new OOMObjet());
        }
    }
}
```
输出：
```
[GC (Allocation Failure) [PSYoungGen: 8192K->1016K(9216K)] 8192K->4543K(19456K), 0.0066378 secs] [Times: user=0.03 sys=0.02, real=0.01 secs] 
[GC (Allocation Failure) --[PSYoungGen: 9208K->9208K(9216K)] 12735K->19445K(19456K), 0.0117774 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (Ergonomics) [PSYoungGen: 9208K->0K(9216K)] [ParOldGen: 10237K->10211K(10240K)] 19445K->10211K(19456K), [Metaspace: 3230K->3230K(1056768K)], 0.1306755 secs] [Times: user=0.25 sys=0.00, real=0.13 secs] 
[Full GC (Ergonomics) [PSYoungGen: 8192K->7167K(9216K)] [ParOldGen: 10211K->8959K(10240K)] 18403K->16126K(19456K), [Metaspace: 3231K->3231K(1056768K)], 0.1517721 secs] [Times: user=0.38 sys=0.00, real=0.15 secs] 
[Full GC (Ergonomics) [PSYoungGen: 7813K->7667K(9216K)] [ParOldGen: 8959K->8959K(10240K)] 16772K->16626K(19456K), [Metaspace: 3231K->3231K(1056768K)], 0.1060525 secs] [Times: user=0.48 sys=0.00, real=0.11 secs] 
[Full GC (Allocation Failure) Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3210)
	at java.util.Arrays.copyOf(Arrays.java:3181)
	at java.util.ArrayList.grow(ArrayList.java:265)
	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:239)
	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:231)
	at java.util.ArrayList.add(ArrayList.java:462)
	at com.miss27.record.test.HeapOOM.main(HeapOOM.java:18)
[PSYoungGen: 7667K->7667K(9216K)] [ParOldGen: 8959K->8941K(10240K)] 16626K->16608K(19456K), [Metaspace: 3231K->3231K(1056768K)], 0.1079590 secs] [Times: user=0.36 sys=0.00, real=0.11 secs] 
Heap
 PSYoungGen      total 9216K, used 7939K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 96% used [0x00000000ff600000,0x00000000ffdc0ef0,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
  to   space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
 ParOldGen       total 10240K, used 8941K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 87% used [0x00000000fec00000,0x00000000ff4bb538,0x00000000ff600000)
 Metaspace       used 3264K, capacity 4500K, committed 4864K, reserved 1056768K
  class space    used 352K, capacity 388K, committed 512K, reserved 1048576K
```
关键点：` java.lang.OutOfMemoryError: Java heap space`，堆内存溢出。

解决方案：
* 确定是内存泄漏(Memory Leak)还是内存溢出(Memory Overflow)。如果是内存泄漏可进一步用工具查看对象到GC ROOTS的引用链，就能找到对象是通过怎样的路径与GC ROOTS相关联并导致GC无法回收。掌握了泄漏对象的类型信息以及 GC ROOTS引用链就可以比较精准的定位出泄漏代码的位置；
* 如果是内存溢出，就说明内存中的对象确实都还活着，那么就应该检查堆参数设置`-Xmx`、`-Xms`，与物理内存对比看是否可以调大一点，从代码上检查某些对象的生命周期过长、持有状态过长的情况，尝试煎炒程序运行期间的内存消耗。

### 3.4.2 虚拟机栈和本地方法栈内存溢出

对于虚拟机栈和本地方法栈，在Java虚拟机规范中描述了两种异常：
* 如果线程请求的栈深度大于虚拟机所允许的最大深度，则会抛出`StackOverflowError`异常；
* 如果虚拟机栈扩展时，无法申请到内存，则会抛出`OutOfMemoryError`
其实这两种异常有些重叠的地方：第一种情况是栈内存被用完了，第二种情况是用完了申请不到空间，其实都是对同一件事的两种描述方式。

java测试代码：
```java

/**
 * Created by zhao
 * 栈溢出 StackOverflowError
 * 设置参数  -Xss128k 栈内存设置128k大小。
 */
public class JavaStackOOM {
    private int stackLength = 1;

    public void stackLeak() {
        stackLength++;
        System.out.println(stackLength);
        stackLeak();
    }

    public static void main(String[] args){
        JavaStackOOM oom = new JavaStackOOM();
        oom.stackLeak();
    }
}
```
输出：
```
...
981
982
Exception in thread "main" java.lang.StackOverflowError
```
无论是由于栈帧太大还是栈内存太小，当内存无法分配的时候就会抛出`StackOverflowError`异常。

### 3.4.3 方法区和运行时常量池溢出

java测试代码
```java
/**
 * Created by zhao
 * -XX:PermSize=10M -XX:MaxPermSize=10M 设置方法区内存大小 初始化为10M，最大容量10M
 * 而JDK1.7之前运行时常量池是在方法区里的，因此可以设置方法区的大小，从而间接限制其中的常量池大小
 */
public class ConstantPoolOOM {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        int i = 0;
        while (true) {
            list.add(String.valueOf(i++).intern());
        }
    }
}
```
这段代码在JDK1.6环境会报错。
```
Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
```
运行时常量池溢出，由于常量池属于方法区，所以后面跟着`PermGen space`。
JDK1.8常量池移到了堆内存，方法区是JVM的规范，之前是用永久代来实现，JDK1.8中改成了元空间。本质上元空间和永久代类似，都是对JVM方法区的实现，不过元空间并不存在虚拟机中，而是在直接内存(本地内存)中。
可以通过下面的例子证明JDK1.8把常量池移到了堆内存：
```java
/**
 * Created by zhao
 * -Xms5m -Xmx5m 设置堆内存容量为5M
 */
public class ConstantPoolOOM {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        int i = 0;
        while (true) {
            list.add(String.valueOf(i++).intern());
        }
    }
}
```
输出：
```
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```
intern()方法的实现不再是在常量池中创建与此String内容相同的字符串，而改为在常量池中记录Java Heap中首次出现的该字符串的引用，并返回该引用;

对这个字符串常量池String intern方法在写个例子测试。
```java
public class StringIntern {
    public static void main(String[] args) {
        String s1 = new StringBuilder().append("Hello").append("JVM").toString();
        System.out.println(s1.intern() == s1);//true

        String s2 = new StringBuilder("Hello").append("JVM").toString();
        System.out.println(s2.intern() == s2);//false

        System.out.println(s1 == s2.intern());//true
    }
}
```
在JDK1.6的结果是：`false  false`。在JDK1.7、JDK1.8是：`true  false`;
产生差异的原因是：
JDK1.6中 intern()方法是这样的：
* 如果常量池中存在相同的字符串，则返回常量池中对应字符串的引用；
* 会把首次遇到的字符串实例复制到常量池中，返回的也是常量池中对这个字符串的引用，而由`StringBuilder`创建的字符串的引用在栈中。所以返回false；
JDK1.8中 intern() 
* `s1.intern() == s1`为true，当执行`s1.intern()`的时候，JVM不再把s1对应的字面量复制一份到字符串常量池中，而是在字符串常量池中存储一份s1的引用，这个引用指向堆中的字面量，所以`s1.intern() == s1`其实是同一个引用；
* `s2.intern() == s2`为false，是因为s2.intern()发现常量池中已经存在一个指向堆中的字面量"HelloJVM"的引用，所有s2.intern()返回的引用其实和上面的s1的引用是同一个，而s2是` new StringBuilder("Hello").append("JVM").toString()`指向堆内存中的字面量"HelloJVM"，s2.intern() 和 s2指向的不是同一个对象，所以结果是false；
* `s1 == s2.intern()`为true，验证了上面的分析；


### 3.4.4 本地直接内存溢出
直接内存貌似和NIO有关，但是目前一直没有接触NIO，因此先记下书中的例子，后面遇到在细研究。
```java
/**
 * Created by zhao
 * -Xms20m -Xmx20m -XX:MaxDirectMemorySize=10m
 */
public class DirectMemoryOOM {
    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) throws IllegalAccessException {
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);
        while (true) {
            unsafe.allocateMemory(_1MB);
        }
    }
}
```
报错：
```
Exception in thread "main" java.lang.OutOfMemoryError
```
## 四 垃圾回收机制和内存分配策略

### 4.1 对象已死吗
垃圾回收机制在堆内存回收前，第一件事就是要确定哪些对象还活着，哪些对象已经死了。常用的两种方法是：引用计数算法和可达性分析算法
### 4.1.1 引用计数算法
给对象添加一个引用计数器，每当有一个地方引用它，计数器值就加一；相反的，当引用失效的时候，计数器值就减一；任何时刻计数器为0的对象就是不可能再被使用的。但是如果在对象循环引用的情况下，内存就不会回收；
### 4.1.2 可达性分析算法
通过称为"GC Roots"的对象作为起点，从这些节点开始向下搜索，搜索走过的路称为"引用链"，当一个对象到GC Roots没有任何引用链相连接的话，则证明此对象是不可用的。
在java中可以作为GC Roots的对象包括：
* 虚拟机栈(栈帧中的本地变量表)中的引用的对象；
* 方法区中静态属性引用的对象；
* 方法区中常量引用的对象；
* 本地方法栈的本地方法(native方法)

### 4.2 垃圾收集算法
### 4.2.1 标记-清除算法
首先标记出所有需要回收的对象，在标记完成后统一回收。主要有两个劣势：
* 效率问题，标记和清除效率都不是很高；
* 空间问题，标记清除后会产生大量不连续的内存碎片，太多的内存碎片空间可能会导致以后程序运行中需要分配大对象时，无法找到足够连续空间而不得不提前触发一次GC。
### 4.2.2 复制算法
将可用内存按容量划分为大小想等的两块From和To，当From被占满时GC将From中的存活对象按照顺序复制到To中，同时将From和To交换，这样就可以每次GC都对整个半区进行垃圾回收，内存分配上也就不用考虑内存碎片等情况。缺点是内存缩小了原来的一半。
### 4.2.3 标记整理算法
和标记清除算法类似，只是后续不是对内存清除，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。
### 4.2.4 分代收集算法
当前商业虚拟机的垃圾回收基本都用这种方法。它把内存分为新生代和老年代，这样就可以根据各个年代的特点采用最合适的方式。在新生代中的对象大部分都是朝生夕死的，只有少量存活，这时候就选用复制算法，只需要把较少的存活对象复制即可。而老年代因为对象存活率高，因此可以采用标记-清除或者标记-整理算法来回收。
### 4.3 看懂GC日志
这是从上面的例子copy出来的GC日志。
```
[GC (Allocation Failure) [PSYoungGen: 8192K->1016K(9216K)] 8192K->4543K(19456K), 0.0066378 secs] [Times: user=0.03 sys=0.02, real=0.01 secs] 
[GC (Allocation Failure) --[PSYoungGen: 9208K->9208K(9216K)] 12735K->19445K(19456K), 0.0117774 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (Ergonomics) [PSYoungGen: 9208K->0K(9216K)] [ParOldGen: 10237K->10211K(10240K)] 19445K->10211K(19456K), [Metaspace: 3230K->3230K(1056768K)], 0.1306755 secs] [Times: user=0.25 sys=0.00, real=0.13 secs] 
[Full GC (Ergonomics) [PSYoungGen: 8192K->7167K(9216K)] [ParOldGen: 10211K->8959K(10240K)] 18403K->16126K(19456K), [Metaspace: 3231K->3231K(1056768K)], 0.1517721 secs] [Times: user=0.38 sys=0.00, real=0.15 secs] 
[Full GC (Ergonomics) [PSYoungGen: 7813K->7667K(9216K)] [ParOldGen: 8959K->8959K(10240K)] 16772K->16626K(19456K), [Metaspace: 3231K->3231K(1056768K)], 0.1060525 secs] [Times: user=0.48 sys=0.00, real=0.11 secs] 
[Full GC (Allocation Failure) Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3210)
	at java.util.Arrays.copyOf(Arrays.java:3181)
	at java.util.ArrayList.grow(ArrayList.java:265)
	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:239)
	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:231)
	at java.util.ArrayList.add(ArrayList.java:462)
	at com.miss27.record.test.HeapOOM.main(HeapOOM.java:18)
[PSYoungGen: 7667K->7667K(9216K)] [ParOldGen: 8959K->8941K(10240K)] 16626K->16608K(19456K), [Metaspace: 3231K->3231K(1056768K)], 0.1079590 secs] [Times: user=0.36 sys=0.00, real=0.11 secs] 
Heap
 PSYoungGen      total 9216K, used 7939K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 96% used [0x00000000ff600000,0x00000000ffdc0ef0,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
  to   space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
 ParOldGen       total 10240K, used 8941K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 87% used [0x00000000fec00000,0x00000000ff4bb538,0x00000000ff600000)
 Metaspace       used 3264K, capacity 4500K, committed 4864K, reserved 1056768K
  class space    used 352K, capacity 388K, committed 512K, reserved 1048576K 
```

`[GC`和`Full GC`说明了这次垃圾回收的停顿类型，而不是用来区分新生代GC还是老年代GC的。如果有`Full`说明这次GC是发生了`Stop-The-World`的。如果是调用`System.gc()`触发的，那么会显示`[Full GC (System.gc())`。
* `PSYoungGen`：表示新生代，这个名称由收集器决定。PS是Parallel Scavenge收集器的缩写，它配套的新生代称为PSYoungGen，新生代又分化eden space、from space和to space这三部分
* `ParOldGen`：Parallel Scavenge收集器配套的老年代
* `Metaspace`： Parallel Scavenge收集器配套的永久代
* `total` & `used`：总的空间和用掉的空间
* 第一行的`[PSYoungGen: 8192K->1016K(9216K)]`表示 GC前该内存区域已使用容量->GC后该内存区域已使用容量，后面圆括号里面的9216K为该内存区域的总容量。
* `8192K->4543K(19456K), 0.0066378 secs`表示 GC前Java堆已使用容量->GC后Java堆已使用容量，后面圆括号里面的19456K为Java堆总容量。PSYoungGen耗时
* `[Times: user=0.03 sys=0.02, real=0.01 secs]`表示 用户消耗的CPU时间、内核态消耗的CPU时间、操作从开始到结束所经过的墙钟时间（Wall Clock Time）
user是用户态耗费的时间，sys是内核态耗费的时间，real是整个过程实际花费的时间。user+sys是CPU时间，每个CPU core单独计算，所以这个时间可能会是real的好几倍。
CPU时间和墙钟时间的差别是，墙钟时间包括各种非运算的等待耗时，例如等待磁盘I/O、等待线程阻塞，而CPU时间不包括这些耗时。
* 第三行的full gc`[Full GC (Ergonomics) [PSYoungGen: 9208K->0K(9216K)] [ParOldGen: 10237K->10211K(10240K)] 19445K->10211K(19456K), [Metaspace: 3230K->3230K(1056768K)], 0.1306755 secs] [Times: user=0.25 sys=0.00, real=0.13 secs]`表示 Young区GC前内存占用->GC后内存占用(新生代总内存大小)；Old老年代GC前内存占用->GC后内存占用(老年代总内存大小)；metaspace区GC前内存占用->GC后内存占用(元空间(方法区)总内存大小),GC用户耗时，Times:用户耗时 sys=系统时间, real=实际时间 。
规律：**[名称：gc前内存占用-> gc后内存占用（该区内存总大小）]**。
 
## 五 内存分配和回收策略
对象主要分配在新生代的Eden区上，少数情况也可能直接分配在老年代中，分配的规则不是百分百固定的，还取决于使用的哪一种垃圾回收器和虚拟机内存相关的参数设置。

## 5.1 对象内存分配
大多数情况下对象优先分配在Eden区，如果Eden区内存不够，虚拟机将会执行一次`Minor GC`。
* `Minor GC` 指发生在新生代的垃圾收集动作，当Eden区满时，触发`Minor GC`。新生代的对象大部分是朝生夕死的，因此`Minor GC`非常频繁，回收速度也比较快；
* `Full GC` 是清理整个堆内存空间，有几种触发情况：
   * 调用`System.gc()`时，系统建议执行`Full GC`，但是不一定会执行；
   * 老年代空间不足