# HashMap源码解析
<!--- [HashMap数据结构](#一-HashMap数据结构)-->

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
* HashMap(int initialCapacity):该构造函数的参数是初始容量，可以调用该构造函数自定义初始容量;
* HashMap(int initialCapacity, float loadFactor):该构造函数的参数是初始容量和加载因子，可以调用该构造函数自定义初始容量和加载因子;
* HashMap(Map<? extends K, ? extends V> m):传入一个map。
其实最常用的就是直接`new HashMap()`，但是其他的构造器还是有合适的使用场景的，比如在写代码的时候，你能非常确定这个Map长度一定大于200，那么就可以直接`new HashMap(200)`，这样的好处是，如果你是使用默认构造器，默认容量是16，那么会进行好多次的扩容，才能达到你的200的长度。也算是性能优化的一种吧。
`put()`，put方法先是调用hash函数，计算key的hash值，然后在调用putVal方法。
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
        @SuppressWarnings({"rawtypes", "unchecked"})
        Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
        //修改hashMap的table为新建的newTab
        table = newTab;
        //如果旧table不为空，将旧table中的元素复制到新的table中
        if (oldTab != null) {
            //遍历旧哈希表的每个桶，将旧哈希表中的桶复制到新的哈希表中
            for (int j = 0; j < oldCap; ++j) {
                Node<K, V> e;
                //如果旧桶不为null
                if ((e = oldTab[j]) != null) {
                    //将旧桶置为null
                    oldTab[j] = null;
                    //如果旧桶中只有一个node
                    if (e.next == null) {
                        //将e也就是oldTab[j]放入newTab中e.hash & (newCap - 1)的位置
                        newTab[e.hash & (newCap - 1)] = e;
                    } else if (e instanceof TreeNode)//如果旧桶中的结构为红黑树
                        ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);//将树中的node分离
                    else {//如果旧桶中的结构为链表,链表重排，jdk1.8做的一系列优化
                        Node<K, V> loHead = null, loTail = null;
                        Node<K, V> hiHead = null, hiTail = null;
                        Node<K, V> next;
                        do {//遍历整个链表
                            next = e.next;
                            // 原索引
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            //原索引+oldCap
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 原索引放到bucket里
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        // 原索引+oldCap放到bucket里
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
resize基本流程：
* 先判断是扩容还是初始化
* 计算扩容后的临界值、扩容量
* 将临界值修改为扩容后的临界值
* 根据扩容后的容量新建数组，然后将HashMap的table的引用指向新数组
* 将旧的数组元素复制到table中

明天再看其他的方法
