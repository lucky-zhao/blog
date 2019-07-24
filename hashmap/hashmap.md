# HashMap源码解析
- [HashMap数据结构](#一-HashMap数据结构)
    - [数组长度越大，哈希碰撞几率越低](#11-数组长度越大，哈希碰撞几率越低)
    - [put方法](#12-put方法)
    - [get方法](#13-get方法)
    - [resize方法](#14-resize方法)
    
面试中经常会遇到hashmap，是时候来总结一波了。
首先说数组和链表的特点，也是面试常问的ArrayList和LinkedList区别。
* 数组存储空间是连续的，占用内存严重
   * 因为数组存储空间是连续的，特点是随机查找容易，插入和删除困难;
* 链表存储空间分散，占用空间比较宽松，特点是查找慢，插入和删除容易。
能不能综合两种数据结构的优势，做出一个寻址容易，插入、删除也容易的数据结构呢？这就是哈希表。HashMap中的哈希表主要用链表式数组，也就是所谓的"拉链法"。

## 一 HashMap数据结构
HashMap底层由数组+链表+红黑树的结构构成(JDK1.8新加的红黑树)，数组里存的是`Node`对象
```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;//hash值(key经过hash算法计算出来的值)
    final K key;//key
    V value;//value
    Node<K,V> next;//指向下一个节点

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```
可以看到属性，hash、key、value、next。如果put的key计算出来的hash值是一样的，那么就会发生哈希碰撞，此时会判断如果hash值相等并且key的equals方法也相等说明是同一个key，那么新的value会覆盖掉以前的value，如果不相等，说明只是hash值相同，这时候遍历这个链表或者红黑树，找到next节点为null的那个节点，然后把要put的对象插到next节点。当链表长度大于阀值8<font color=red>并且哈希表中总元素数量大于64</font>(这一点很多百度上找的源码分析都没提到，其实是错误的)的时候就转成红黑树结构。下面还是通过源码来学习吧。
首先看一下主要参数：
```java
public class HashMap<K,V> extends AbstractMap<K,V>
        implements Map<K,V>, Cloneable, Serializable{
    //默认初始化容量 16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    //容量上限
    static final int MAXIMUM_CAPACITY = 1 << 30;
    //默认的加载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    //链表结构转化为红黑树的阈值，大于8时链表结构 树化
    static final int TREEIFY_THRESHOLD = 8;
    //红黑树转化为链表结构的阈值，桶内元素个数小于6时转化为链表结构
    static final int UNTREEIFY_THRESHOLD = 6;
    //树形化最小hash表元素个数，如果桶内元素已经达到转化红黑树阈值，但是表元素总数未达到阈值，则值进行扩容，不进行树形化
    //也就是说哪怕链表长度大于8了，但是数组的长度没有大于64，并不会转成红黑树，而是扩容，这一步后面源码分析的时候在细看
    static final int MIN_TREEIFY_CAPACITY = 64;
    //存储元素的数组，总是2的幂次倍
    transient Node<k,v>[] table; 
    //存放元素的个数，注意这个不等于数组的长度。
    transient int size;
    //临界值 当实际大小(容量*填充因子)超过临界值时，会进行扩容
    int threshold;
}
```
在看一下构造器：
```java
public class HashMap<K,V> extends AbstractMap<K,V>
        implements Map<K,V>, Cloneable, Serializable{
  public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
  }
    
  public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
  }
  
  public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
}
```
* HashMap()：这是一个没有传参的构造函数，构造函数中定义了加载因子为默认加载因子，其余值都是默认的;
* HashMap(int initialCapacity):该构造函数的参数是初始容量，可以调用该构造函数自定义初始容量，其实最终调用的还是`HashMap(int initialCapacity, float loadFactor)`这个构造器，只不过`loadFactor`是默认值罢了;
* HashMap(int initialCapacity, float loadFactor):该构造函数的参数是初始容量和加载因子，可以调用该构造函数自定义初始容量和加载因子;
* HashMap(Map<? extends K, ? extends V> m):传入一个map。
其实最常用的就是直接`new HashMap()`，但是其他的构造器还是有合适的使用场景的，比如在写代码的时候，你能非常确定这个Map长度一定大于200，那么就可以直接`new HashMap(1000)`，这样的好处是，如果你是使用默认构造器，默认容量是16，那么会进行好多次的扩容，才能达到你需要的1000的长度。并且数组长度越大，哈希碰撞几率也越小，也算是性能优化的一种吧。

写个例子证明一下，数组越大，哈希碰撞越小。先把HashMap源码里的一两个方法拿出来，一会要用到。
* 计算hash值
```
  static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
* 传入initialCapacity，实际调用的是这个方法，打断点就可以看到，实际改的是threshold临界值。
```
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
## 1.1 数组长度越大，哈希碰撞几率越低
```java
/**
 * Created by zhao
 */
public class MyHashMap {
    public static void main(String[] args) {
        int tabSize = 16;//指定HashMap大小，默认是16
        Map<Integer, String> map = new HashMap<>(tabSize);
        Integer key1 = 1;
        Integer key2 = 17;
        int capacity = tableSizeFor(tabSize);//计算出数组实际大小
        map.put(key1, "aaa");
        map.put(key2, "bbb");
        System.out.println("key1的下标：" + (capacity - 1 & hash(key1)));
        System.out.println("key2的下标：" + (capacity - 1 & hash(key2)));
    }

    //计算key的哈希值
    public static int hash(Integer key) {
        int h;
        return (h = key.hashCode()) ^ (h >>> 16);
    }

    //根据new HashMap(1000) 指定的大小，计算出数组的实际容量
    public static int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= 1 << 30) ? 1 << 30 : n + 1;
    }
}
```
里面的`hash`方法就是从源码里拷出来的，还有`tableSizeFor`方法也一样，只不过把常量替换成具体的值。这一段代码输出的结果是：
```
key1的下标：1
key2的下标：1
```
说明发生了哈希碰撞，打断点跟着源码进去，也确实发现走的代码是发生碰撞后的方法体。

改造一下这个测试类，指定HashMap的容量，即`new HashMap(1000)`。
```java
/**
 * Created by zhao
 */
public class MyHashMap {
    public static void main(String[] args) {
        int tabSize = 1000;//指定HashMap大小，默认是16
        Map<Integer, String> map = new HashMap<>(tabSize);
        Integer key1 = 1;
        Integer key2 = 17;
        int capacity = tableSizeFor(tabSize);//计算出数组实际大小
        map.put(key1, "aaa");
        map.put(key2, "bbb");
        System.out.println("key1的下标：" + (capacity - 1 & hash(key1)));
        System.out.println("key2的下标：" + (capacity - 1 & hash(key2)));
    }

    //计算key的哈希值
    public static int hash(Integer key) {
        int h;
        return (h = key.hashCode()) ^ (h >>> 16);
    }

    //根据new HashMap(1000) 指定的大小，计算出数组的实际容量
    public static int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= 1 << 30) ? 1 << 30 : n + 1;
    }
}
```
看输出结果：
```
key1的下标：1
key2的下标：17
```
发现当数组的容量越大,发生哈希碰撞的概率越低。也就是说同样的key在不同大小的HashMap里经过计算后得到的位置相同的概率很低，数组越大，概率越低。

## 1.2 put方法

看一下最重要的`put()`，put方法先是调用hash函数，计算key的hash值，然后在调用putVal方法。
```
   public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
     final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K, V>[] tab;
        Node<K, V> p;
        int n, i;
        //如果哈希表为null 或者 长度为0，则resize进行初始化，并用变量 n 记录长度，变量 tab 就是初始化后的对象
        if ((tab = table) == null || (n = tab.length) == 0) {
            n = (tab = resize()).length;
        }
        //首先 (n - 1) & hash 计算key将被放置的位置，如果tab(计算后的位置) 上为null，说明没有发生哈希碰撞，变量 p 表示 tab(计算后的位置) 里的Node对象
        if ((p = tab[i = (n - 1) & hash]) == null) {
            //则直接将键值对插入到这个节点即可。
            tab[i] = newNode(hash, key, value, null);
        } else { //进入这个方法体，说明发生了哈希碰撞
            Node<K, V> e;
            K k;
            //p 变量代表该tab已经存在的Node对象，下面直接用 原有的Node对象 说明
            //如果原有的Node对象的hash==本次要put的hash 并且 原有的Node对象的key == 本次要put的key 或者 原有的Node对象的key equals 本次要put的key
            //如果这些条件都满足，说明key一模一样
            if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k)))) {
                //将 原有的Node对象 赋值给 e  变量 ，先记录下来
                e = p;
            } else if (p instanceof TreeNode) {//如果是红黑树结构，就调用树的插入方法
                e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value);
            } else {
                //这个循环的意义就是遍历链表，找到最后一个节点，插入数据
                for (int binCount = 0; ; ++binCount) {
                    //p 是原有的Node对象，如果 p.next为null，说明是链表的最后一个节点，p.next用变量e记录
                    if ((e = p.next) == null) {
                        //到这里已经找到该链表的最后一个节点，即next对象为null，此时把需要插入的数据放入这个尾节点
                        p.next = newNode(hash, key, value, null);
                        //如果循环次数>临界值，
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        {
                            //转化成红黑树(这里并没有把链表直接转成红黑树，在treeifyBin方法里还有一层判断，后面会分析到)
                            treeifyBin(tab, hash);
                        }
                        break;
                    }
                    //如果 原有的Node对象 的hash==要插入的hash 并且 key .equals 要保存的key，不做重复操作，跳出循环
                    if (e.hash == hash &&  ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    //这里的 e 是p.next 也就是 原有的Node对象的next
                    //把 e 赋值给 p，这样就可以循环整个链表
                    p = e;
                }
            }
            //如果 e 不为null，说明该key已经存在(原来的key和要插入的key相同，equals也相等)
            if (e != null) { // existing mapping for key
                //oldValue 表示原来Node对象的value
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null) {
                    //新value 代替原来Node对象的value
                    e.value = value;
                }
                //回调
                afterNodeAccess(e);
                //返回原Node对象的值
                return oldValue;
            }
        }
        //更新结构化修改信息
        ++modCount;
        //如果键值对的数量>阀值 进行reHash
        if (++size > threshold)
            resize();
        //回调
        afterNodeInsertion(evict);
        return null;
    }
```
大致流程图如下：
![HashMap_put](https://github.com/lucky-zhao/blog/blob/master/hashmap/img/HashMap_put.png "HashMap_put")
看一下put里调用的`treeifyBin`方法：
```
  final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        //这里的if说明了，并不是说链表的长度大于阀值就会转换成红黑树，还有个条件：数组的长度要不小于`MIN_TREEIFY_CAPACITY`，也就是说，不小于64才行,如果小于64，就扩容，不会转成红黑树。
        //之所以这么设计，我觉得是因为 引入红黑树是为了在发生哈希碰撞时，减少查询时间，提高效率，但是这种提升相对于直接从数组中取值还是比较低的，当数组的长度小于64时，就进行扩容，这样会重新分布数据，
        //使数据更加分散，减少碰撞几率。假设数组长度本来就只有2，直接在某个节点中使用红黑树，还不如直接扩大数组容量，这样索引的效率反而会更高。
        //上面的例子，证明数组的长度越大，哈希碰撞几率越低。
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
```
## 1.3 get方法
看完put，在来看看get方法：
```
 public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
   final Node<K, V> getNode(int hash, Object key) {
        Node<K, V>[] tab;
        Node<K, V> first, e;
        int n;
        K k;
        //如果table不为null并且table中有值
        if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
            //如果第一个元素的hash和传进来的hash相等 并且 equals也相等，说明找的就是这个元素，直接return出去
            if (first.hash == hash && // always check first node 
                    ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //如果不仅仅只有一个Node
            if ((e = first.next) != null) {
                //是否是红黑树
                if (first instanceof TreeNode)
                    //红黑树的方式查找
                    return ((TreeNode<K, V>) first).getTreeNode(hash, key);
                do {
                    //遍历链表，找到这个key 返回这个节点
                    if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```
get方法就简单了很多。
* 首先计算索引，找到这个table；
* 判断这个table里的第一个Node节点是否就是我们需要的，如果是直接返回，如果不是走下一步；
* 判断是否有next节点，如果是红黑树就到红黑树里查找，如果是链表，则遍历链表找到这个key所在的位置，返回Node。

## 1.4 resize方法
在来看resize方法：
```
   final Node<K, V>[] resize() {
        //用变量 oldTab 保存扩容前的table
        Node<K, V>[] oldTab = table;
        //oldCap 原数组的长度
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        //oldThr 原数组扩容的临界值
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 超过最大值就不再扩容了
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            } else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY) {// 如果原数组的容量的两倍小于 MAXIMUM_CAPACITY 并且 原数组的容量 >= 16
                //临界值变为原来的2倍
                newThr = oldThr << 1; // double threshold
            }
        } else if (oldThr > 0)//如果旧临界值 > 0
            //数组的新容量设置为老数组扩容的临界值
            newCap = oldThr;
        else {
            //原临界值<0，新容量为默认初始容量
            newCap = DEFAULT_INITIAL_CAPACITY;
            //新临界值为 DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY
            newThr = (int) (DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 计算新的resize上限
        if (newThr == 0) {
            float ft = (float) newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float) MAXIMUM_CAPACITY ? (int) ft : Integer.MAX_VALUE);
        }
        //将扩容后hashMap的临界值设置为newThr
        threshold = newThr;

        //从下面开始, 初始化table或者扩容, 实际上都是通过新建一个table来完成的
        @SuppressWarnings({"rawtypes", "unchecked"})
        Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
        //修改hashMap的table为新建的newTab
        table = newTab;
        // 下面这段就是把原来table里面的值全部搬到新的table里面
        if (oldTab != null) {
            //遍历旧哈希表的每个桶，将旧哈希表中的桶复制到新的哈希表中
            for (int j = 0; j < oldCap; ++j) {
                Node<K, V> e;
                //如果旧桶不为null
                if ((e = oldTab[j]) != null) {
                    //这里注意, table中存放的只是Node的引用, 这里将oldTab[j]=null只是清除旧表的引用, 但是真正的node节点还在, 只是现在由e指向它
                    oldTab[j] = null;
                    //如果旧桶中只有一个node
                    if (e.next == null) {
                        //将e也就是oldTab[j]放入newTab中e.hash & (newCap - 1)的位置
                        newTab[e.hash & (newCap - 1)] = e;
                    } else if (e instanceof TreeNode) {//如果旧桶中的结构为红黑树
                        ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
                    } else {//如果旧桶中的结构为链表
                        //下面这段看起来有点繁琐，还是拆开看吧
                        Node<K, V> loHead = null, loTail = null;
                        Node<K, V> hiHead = null, hiTail = null;
                        Node<K, V> next;
                        do {//遍历整个链表
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null) {
                                    loHead = e;
                                } else {
                                    loTail.next = e;
                                }
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
单独拆开这段链表的代码：
```
//首先定义了4个Node引用，可以看成是2个链表 lo链表和hi链表，他们分别都有头节点和尾节点
Node<K,V> loHead = null, loTail = null;
Node<K,V> hiHead = null, hiTail = null;
Node<K,V> next;
//下面的do while循环，去掉循环体其实就是这一部分，每次拿到e的next节点，如果不为空就继续循环，达到了遍历整个链表的目的
/*
do {
    next = e.next;
} while ((e = next) != null);
*/
do {
    next = e.next;
    //判断节点在resize之后是否需要改变在数组中的位置
    if ((e.hash & oldCap) == 0) {
        if (loTail == null)
            loHead = e;
        else
            loTail.next = e;
        loTail = e;
    }
    else {
        if (hiTail == null)
            hiHead = e;
        else
            hiTail.next = e;
        hiTail = e;
    }
} while ((e = next) != null);
if (loTail != null) {
    loTail.next = null;
    newTab[j] = loHead;
}
if (hiTail != null) {
    hiTail.next = null;
    newTab[j + oldCap] = hiHead;
}
```
在拆开，看do while循环里的逻辑:
```
//如果(e.hash & oldCap) == 0 为true就插到lo链表，否则插到hi链表
 if ((e.hash & oldCap) == 0) {
 //该节点在新表的下标位置与旧表一致都为 j 
        //插入lo链表
        if (loTail == null) {
            loHead = e;
        } else {
            loTail.next = e;
        }
        loTail = e;
    } else {//插入hi链表 该节点在新表的下标位置 j + oldCap
        if (hiTail == null) {
            hiHead = e;
        } else {
            hiTail.next = e;
        }
        hiTail = e;
}
```
在看下面的if-else：
```
//将某节点中的链表分割重组为两个链表：一个需要改变位置，另一个不需要改变位置
//如果lo链表非空, 我们就把整个lo链表放到新table的j位置上 
if (loTail != null) {
    loTail.next = null;
    newTab[j] = loHead;
}
//如果hi链表非空, 我们就把整个hi链表放到新table的j+oldCap位置上
if (hiTail != null) {
    hiTail.next = null;
    newTab[j + oldCap] = hiHead;
}
```
根据上面的拆分可以知道，这段代码就是把原来的链表拆分成2个链表，并将这两个链表分别放在新的table的`j`位置和`j+oldCap`上，`j`位置就是原链表在原table中的位置，拆分的标准就是`(e.hash & oldCap) == 0`。根据这个条件, 我们将原位置的链表拆分成两个链表, 然后一次性将整个链表放到新的Table对应的位置上。
resize特点：
* resize发生在table初始化, 或者table中的节点数超过threshold值的时候, threshold的值一般为负载因子乘以容量大小；’
* 每次扩容都会新建一个table, 新建的table的大小为原大小的2倍；
* 扩容时,会将原table中的节点re-hash到新的table中, 但节点在新旧table中的位置存在一定联系: 要么下标相同, 要么相差一个oldCap(原table的大小)。

里面的具体算法，暂时不研究。