---
title: HashMap源码浅析(JDK1.8)
date: 2018-4-5
tags: Java
---
------

### 一. HashMap的定义
#### 概述
HashMap基于HashTable的Map接口实现（实现了Map接口，继承AbstractMap），此实现提供所有可选的映射操作，并且允许NULL值和NULL键（HashMap类大致相当于HashTable类，但它是非同步的，而且允许空值。）此类不保证映射顺序。由于HashMap不是线程安全的，如果想要线程安全，可以使用ConcurrentHashMap代替。

一个HashMap的实例有两个影响其性能的参数：初始容量（capacity默认是16）和加载因子（load factor默认是0.75），通常缺省的加载因子较好地实现了时间和空间的均衡，增大加载因子虽然说可以节省空间但相对地会增加对应的查找成本，这样会影响HashMap的get和put操作。加载因子表示Hash表中的元素的填满程度，填充得越多对应的查找的时候发生冲突的机率就越大，查找的成本就越高。
<!--more-->
#### 主要属性
```java
public class HashMap<K,V> extends AbstractMap<K,V>
            implements Map<K,V>, Cloneable, Serializable {
        //存储数据的Node数组
        transient Node<K,V>[] table;
        //返回Map中所包含的Map.Entry<K,V>的Set视图。
        transient Set<Map.Entry<K,V>> entrySet;
        //当前存储元素的总个数
        transient int size;
        //HashMap内部结构发生变化的次数，主要用于迭代的快速失败（下面代码有分析此变量的作用）
        transient int modCount;
        //下次扩容的临界值，size>=threshold就会扩容，threshold等于capacity*load factor
        int threshold;
        //装载因子
        final float loadFactor;

        //默认装载因子
        static final float DEFAULT_LOAD_FACTOR = 0.75f;
        //由链表转换成红黑树的阈值TREEIFY_THRESHOLD
        static final int TREEIFY_THRESHOLD = 8;
        //由红黑树的阈值转换链表成UNTREEIFY_THRESHOLD
        static final int UNTREEIFY_THRESHOLD = 6;
        //默认容量（16）
        static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
         //数组的最大容量 （1073741824）
        static final int MAXIMUM_CAPACITY = 1 << 30;
        //当桶中的bin(链表中的元素)被树化时最小的hash表容量。（如果没有达到这个阈值，即hash表容量小于MIN_TREEIFY_CAPACITY，当桶中bin的数量太多时会执行resize扩容操作）这个MIN_TREEIFY_CAPACITY的值至少是TREEIFY_THRESHOLD的4倍。
        static final int MIN_TREEIFY_CAPACITY = 64;
        略...
```

#### 存储查找原理
存储：首先获取key的hashcode，然后取模数组的长度，这样可以快速定位到要存储到数组中的坐标，然后判断数组中是否存储元素，如果没有存储则，新构建Node节点，把Node节点存储到数组中，如果有元素，则迭代链表(红黑二叉树)，如果存在此key,默认更新value,不存在则把新构建的Node存储到链表的尾部。

查找：同上，获取key的hashcode，通过hashcode取模数组的长度，获取要定位元素的坐标，然后迭代链表，进行每一个元素的key的equals对比，如果相同则返回该元素。

### 二. HashMap的数据结构
JDK1.6实现hashmap的方式是采用位桶（数组）+链表的方式，即散列链表方式。JDK1.8则是采用位桶+链表/红黑树的方式，即当某个位桶的链表长度达到某个阈值（8）的时候，这个链表就转化成红黑树，这样大大减少了查找时间。
![hashmap](http://autism-tl.cn/picture/HashMap.png)
下面是HashMap的几个构造函数：
```java
 /**
     * 根据初始化容量和加载因子构建一个空的HashMap.
     */
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

    /**
     * 使用初始化容量和默认加载因子(0.75).
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * 使用默认初始化大小(16)和默认加载因子(0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    /**
     * 用已有的Map构造一个新的HashMap.
     */
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

#### 链表的结构
```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash; // 哈希值
        final K key;
        V value;
        Node<K,V> next; // 指向下一个节点

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

#### 红黑二叉树的结构
```java
 static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
    }
```

### 三. HashMap的数据存取
#### HashMap.put（key, value）
```java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        //p：链表节点  n:数组长度   i：链表所在数组中的索引坐标
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //判断tab[]数组是否为空或长度等于0，进行初始化扩容
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //判断tab指定索引位置是否有元素，没有则直接newNode赋值给tab[i]
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        //如果该数组位置存在Node
        else {
            //首先先去查找与待插入键值对key相同的Node，存储在e中，k是那个节点的key
            Node<K,V> e; K k;
             // 如果hash、key都相等，直接覆盖
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //如果Node是红黑二叉树，则执行树的插入操作
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //否则执行链表的插入操作（说明Hash值碰撞了，把Node加入到链表中）
            else {
                for (int binCount = 0; ; ++binCount) {
                    //如果该节点是尾节点，则进行添加操作
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //判断链表长度，如果链表长度大于8则调用treeifyBin方法，判断是扩容还是把链表转换成红黑二叉树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    // hash、key都相等，此时即要更新节点并退出循环
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    //把p执行p的子节点，开始下一次循环（p = e = p.next）
                    p = e;
                }
            }
            //判断e是否为null，如果为null则表示加了一个新节点，不是null则表示找到了hash、key都一致的Node。
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //判断是否更新value值。（map提供putIfAbsent方法，如果key存在，不更新value，但是如果value==null任何情况下都更改此值）
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                //此方法是空方法，什么都没实现，用户可以根据需要进行覆盖
                afterNodeAccess(e);
                return oldValue;链表中原本存在相同的key，则返回oldValue
            }
        }
        //只有插入了新节点才进行++modCount；
        ++modCount;
        //如果size>threshold则开始扩容（每次扩容原来的1倍）
        if (++size > threshold)
            resize();
        //此方法是空方法，什么都没实现，用户可以根据需要进行覆盖
        afterNodeInsertion(evict);
        return null; // 原HashMap中不存在相同的key，插入键值对后返回null
    }
    
```
1. 判断键值对数组tab[i]是否为空或为null，否则执行resize()进行扩容；
2. 根据键值key计算hash值得到插入的数组索引i，如果table[i]==null，直接新建节点添加，转向6，如果table[i]不为空，转向3；
3. 判断链表（或二叉树）的首个元素是否和key一样，不一样转向4，相同则覆盖后转向6；
4. 判断链表（或二叉树）的首节点 是否为treeNode，即是否是红黑树，如果是红黑树，则直接在树中插入键值对，不是则执行5；
5. 遍历链表，判断链表长度是否大于8，大于8的话把链表转换为红黑树（还判断数组长度是否小于64，如果小于只是扩容，不进行转换二叉树），在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value即可；如果调用putIfAbsent方法插入，则不更新值（只更新值为null的元素）。
6. 插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
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
首先判断数组的长度是否小于64，如果小于64则进行扩容，否则把链表结构转换成红黑二叉树结构。

modCount 变量的作用：
```java
public final void forEach(Consumer<? super K> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e.key);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }

```
从forEach循环中可以发现 modCount 参数的作用。就是在迭代器迭代输出Map中的元素时，不能编辑（增加，删除，修改）Map中的元素。如果在迭代时修改，则抛出ConcurrentModificationException异常。

#### HashMap.get（key）
```java
 public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode) // 红黑树
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 链表
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

    // 遍历红黑树搜索节点
    /**
     * Calls find for root node.
     */
    final TreeNode<K,V> getTreeNode(int h, Object k) {
        return ((parent != null) ? root() : this).find(h, k, null);
    }

    /**
     * Returns root of tree containing this node.
     */
    final TreeNode<K,V> root() {
        for (TreeNode<K,V> r = this, p;;) {
            if ((p = r.parent) == null)
                return r;
            r = p;
        }
    }

    /**
     * Finds the node starting at root p with the given hash and key.
     * The kc argument caches comparableClassFor(key) upon first use
     * comparing keys.
     */
    final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
        TreeNode<K,V> p = this;
        do {
            int ph, dir; K pk;
            TreeNode<K,V> pl = p.left, pr = p.right, q;
            if ((ph = p.hash) > h) // 当前节点hash大
                p = pl; // 查左子树
            else if (ph < h) // 当前节点hash小
                p = pr; // 查右子树
            else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                return p; // hash、key都相等，即找到，返回当前节点
            else if (pl == null) // hash相等，key不等，左子树为null，查右子树
                p = pr;
            else if (pr == null)
                p = pl;
            else if ((kc != null ||
                      (kc = comparableClassFor(k)) != null) &&
                     (dir = compareComparables(kc, k, pk)) != 0)
                p = (dir < 0) ? pl : pr;
            else if ((q = pr.find(h, k, kc)) != null)
                return q;
            else
                p = pl;
        } while (p != null);
        return null;
    }
```

### 四.细节解析
#### hash取余数，为什么不用取模操作呢，而用tab[i = (n - 1) & hash]？
它通过 (n - 1) & hash来得到该对象的保存位，而HashMap底层数组的长度总是2的n次方，这是HashMap在速度上的优化。当length总是2的n次方时， (n - 1) & hash运算等价于对length取模，也就是h%length，但是&比%具有更高的效率。  
用位来保存对象的位置同时也省去了重新计算hash值的时间。

#### hashmap查询的时间复杂度？
最好O（1），最差O（n）， 如果是红黑O（logn）

