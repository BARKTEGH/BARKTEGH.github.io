---
title: HashMap详解
date: 2019-03-02 19:06:20
tags:
- HashMap
- 数据结构
categories:
- java
---

# HashMap介绍
---
## 前言
HashMap是一个经典的key-value结构，是线程不安全的。如果要使用线程安全的hashMap可以使用并发包里的ConcurrentHashMap。
jdk在1.7和1.8的具体实现稍有不同。

## HaspMap在1.7的实现


### 基本数据结构和变量：

![](https://i.imgur.com/qJFRT7H.jpg)

hashmap的内部数据结构是一个Entry数组`transient Node<K,V>[] table`（被关键citransient修饰是为了序列化时只用序列化已使用的数据）。

其中Entry是一个内部类，源码如下：

	static class Entry<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Entry<K,V> next;

        Entry(int hash, K key, V value, Node<K,V> next) {
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


Entry主要有四个成员变量：	
	
- key就是键
- value 是值
- hash存放当前key的hashcode
- next用于实现链表

HashMap还有一些核心变量如下：

1. static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 

	初始化桶的大小，默认为16。选择2的n次方是为了在扩容是计算hashcode将取模运算转为位运算。

    	static final int hash(Object key) {
		    int h;
		    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
	    }

	
2. static final int MAXIMUM_CAPACITY = 1 << 30;
	
	桶最大值
3. static final float DEFAULT_LOAD_FACTOR = 0.75f;

	默认装载因子0.75
4. transient int size;

	Map存放数量的大小

5. int threshold;

	桶大小，可在初始化时显式指定

6. final float loadFactor;

	装载因子，可在初始化时显式指定。
	其中当map的数量达到threshold*loadFactor时，就需要对map就行扩容。

### put方法

HashMap存放数据1.7源码如下：

    public V put(K key, V value) {
	    if (table == EMPTY_TABLE) {
	        inflateTable(threshold);
	    }
	    if (key == null)
	        return putForNullKey(value);
	    int hash = hash(key);
	    int i = indexFor(hash, table.length);
	    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
	        Object k;
	        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
	            V oldValue = e.value;
	            e.value = value;
	            e.recordAccess(this);
	            return oldValue;
	        }
	    }
	    modCount++;
	    addEntry(hash, key, value, i);
	    return null;
	}

流程如下：
	
1. 判断当前数组是否需要初始化。
2. 如果 key 为空，则 put 一个空值进去。
3. 根据 key 计算出 hashcode。
4. 根据计算出的 hashcode 定位出所在桶。
5. 如果桶是一个链表则需要遍历判断里面的 hashcode、key 是否和传入 key 相等，如果相等则进行覆盖，并返回原来的值。
6. 如果桶是空的或者没有找到key，说明当前位置没有数据存入；新增一个 Entry 对象写入当前位置。

		void addEntry(int hash, K key, V value, int bucketIndex) {
		    if ((size >= threshold) && (null != table[bucketIndex])) {
		        resize(2 * table.length);
		        hash = (null != key) ? hash(key) : 0;
		        bucketIndex = indexFor(hash, table.length);
		    }
		    createEntry(hash, key, value, bucketIndex);
		}
		void createEntry(int hash, K key, V value, int bucketIndex) {
		    Entry<K,V> e = table[bucketIndex];
		    table[bucketIndex] = new Entry<>(hash, key, value, e);
		    size++;
		}


### get方法
HashMap获取数据1.7源码如下：

	public V get(Object key) {
	    if (key == null)
	        return getForNullKey();
	    Entry<K,V> entry = getEntry(key);
	    return null == entry ? null : entry.getValue();
	}
	final Entry<K,V> getEntry(Object key) {
	    if (size == 0) {
	        return null;
	    }
	    int hash = (key == null) ? 0 : hash(key);
	    for (Entry<K,V> e = table[indexFor(hash, table.length)];
	         e != null;
	         e = e.next) {
	        Object k;
	        if (e.hash == hash &&
	            ((k = e.key) == key || (key != null && key.equals(k))))
	            return e;
	    }
	    return null;
	}

流程如下：

1. 根据key计算出hashcode，定位到桶的位置
2. 如果桶为空，直接返回null
3. 不然遍历该桶，比较key，value，hash值是否相等；如果相等就返回值，不然返回null


## HaspMap在1.8的实现

1.8相比较1.7的改变主要在以下几个点：

1. 将Entry更改为Node
2. 在链表长度超过8时就更改为红黑树，加快查询。
3. 添加static final int TREEIFY_THRESHOLD = 8变量。
4. 链表头插法改为尾插法（保持原来的顺序），就要为了解决并发put导致resize出现死循环。

以上这些改变都是为了解决hash冲突时链表长度过长，导致查询效率下降的问题，通过将长度超过8的链表转为红黑树来加快查询。此时的数据结构如下：
![](https://i.imgur.com/KZQGEtv.jpg)

### put方法
HashMap存放数据1.8源码：

	public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * Implements Map.put and related methods.
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }

流程如下：

1. 判断桶数组是否为空；如果为空通过resize()来初始化桶数组。
2. 否则通过hash定位到桶位置；若果桶为空，新建一个Node节点（或者说新桶）。
3. 否则先判断当前桶的hash，key与写入的hash，key是否相等；相等的话将值赋予给e；
4. 如果当前桶是红黑树，就以红黑树的方式写入；
5. 如果是个链表，如果在遍历过程中找到 key 相同时直接退出遍历。
6. 如果没有找到相同的节点，就需要将当前的 key、value 封装成一个新节点写入到当前桶的后面（形成链表）赋予给e，接着判断当前链表的大小是否大于预设的阈值，大于时就要转换为红黑树。
7. 判断，如果 e != null 就相当于存在相同的 key,那就需要将值覆盖。
8. 最后判断是否需要进行扩容。

### get方法

HashMap获取数据1.8源码如下：


	public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    /**
     * Implements Map.get and related methods.
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
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

流程如下：

1. 首先将 key hash 之后取得所定位的桶。
2. 如果桶为空则直接返回 null 。
3. 否则判断桶的第一个位置(有可能是链表、红黑树)的 key 是否为查询的 key，是就直接返回 value。
4. 如果第一个不匹配，则判断它的下一个是红黑树还是链表。
5. 红黑树就按照树的查找方式返回值。
6. 不然就按照链表的方式遍历匹配返回值。


# ConcurrentHashMap
---

## ConcurrentHashMap在1.7中的实现
### 基本数据结构
![](https://i.imgur.com/7xWGOSn.jpg)


ConcurrentHashMap 采用了分段锁技术，其中 Segment 继承于 ReentrantLock，不会像 HashTable 那样不管是 put 还是 get 操作都需要做同步处理。无论是读操作还是写操作都能保证很高的性能：在进行读操作时(几乎)不需要加锁，而在写操作时通过锁分段技术只对所操作的段加锁而不影响客户端对其它段的访问。特别地，在理想状态下，ConcurrentHashMap 可以支持 16 个线程执行并发写操作（如果并发级别设为16），及任意数量线程的读操作。

ConcurrentHashMap的高效并发机制是通过以下三方面来保证的：

1. 通过锁分段技术保证并发环境下的写操作；

2. 通过 HashEntry的不变性、Volatile变量的内存可见性和加锁重读机制保证高效、安全的读操作；

3. 通过不加锁和加锁两种方案控制跨段操作的的安全性。

主要变量如下：

	/**
     * Mask value for indexing into segments. The upper bits of a
     * key's hash code are used to choose the segment.
     */
    final int segmentMask;  // 用于定位段，大小等于segments数组的大小减 1，是不可变的

    /**
     * Shift value for indexing within segments.
     */
    final int segmentShift;    // 用于定位段，大小等于32(hash值的位数)减去对segments的大小取以2为底的对数值，是不可变的

    /**
     * The segments, each of which is a specialized hash table
     */
    final Segment<K,V>[] segments;   // ConcurrentHashMap的底层结构是一个Segment数组


Segment类主要组成：

	// 
	static final class Segment<K,V> extends ReentrantLock implements Serializable {

        /**
         * The number of elements in this segment's region.
         */
        transient volatile int count;    // Segment中元素的数量，可见的

        /**
         * Number of updates that alter the size of the table. This is
         * used during bulk-read methods to make sure they see a
         * consistent snapshot: If modCounts change during a traversal
         * of segments computing size or checking containsValue, then
         * we might have an inconsistent view of state so (usually)
         * must retry.
         */
        transient int modCount;  //对count的大小造成影响的操作的次数（比如put或者remove操作）

        /**
         * The table is rehashed when its size exceeds this threshold.
         * (The value of this field is always <tt>(int)(capacity *
         * loadFactor)</tt>.)
         */
        transient int threshold;      // 阈值，段中元素的数量超过这个值就会对Segment进行扩容

        /**
         * The per-segment table.
         */
        transient volatile HashEntry<K,V>[] table;  // 链表数组

        /**
         * The load factor for the hash table.  Even though this value
         * is same for all segments, it is replicated to avoid needing
         * links to outer object.
         * @serial
         */
        final float loadFactor;  // 段的负载因子，其值等同于ConcurrentHashMap的负载因子

        ...
    }

HashEntry类
	
	 /**
     * HashMap 中的 Entry 类
     */
    static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        final int hash;

        /**
         * Creates new entry.
         */
        Entry(int h, K k, V v, Entry<K,V> n) {
            value = v;
            next = n;
            key = k;
            hash = h;
        }
        ...
    }


### put方法


	public V put(K key, V value) {
	    Segment<K,V> s;
	    if (value == null)
	        throw new NullPointerException();
	    int hash = hash(key);
	    int j = (hash >>> segmentShift) & segmentMask;
	    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
	         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
	        s = ensureSegment(j);
	    return s.put(key, hash, value, false);
	}

	final V put(K key, int hash, V value, boolean onlyIfAbsent) {
	    HashEntry<K,V> node = tryLock() ? null :
	        scanAndLockForPut(key, hash, value);
	    V oldValue;
	    try {
	        HashEntry<K,V>[] tab = table;
	        int index = (tab.length - 1) & hash;
	        HashEntry<K,V> first = entryAt(tab, index);
	        for (HashEntry<K,V> e = first;;) {
	            if (e != null) {
	                K k;
	                if ((k = e.key) == key ||
	                    (e.hash == hash && key.equals(k))) {
	                    oldValue = e.value;
	                    if (!onlyIfAbsent) {
	                        e.value = value;
	                        ++modCount;
	                    }
	                    break;
	                }
	                e = e.next;
	            }
	            else {
	                if (node != null)
	                    node.setNext(first);
	                else
	                    node = new HashEntry<K,V>(hash, key, value, first);
	                int c = count + 1;
	                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
	                    rehash(node);
	                else
	                    setEntryAt(tab, index, node);
	                ++modCount;
	                count = c;
	                oldValue = null;
	                break;
	            }
	        }
	    } finally {
	        unlock();
	    }
	    return oldValue;
	}

流程如下：

1. 根据key的hash值定位到Segment
2. 利用 scanAndLockForPut() 获得Segment的锁

	- 尝试自旋获取锁。
	- 如果重试的次数达到了 MAX_SCAN_RETRIES 则改为阻塞锁获取，保证能获取成功。
3. 将当前 Segment 中的 table 通过 key 的 hashcode 定位到 HashEntry。
4. 遍历该 HashEntry，如果不为空则判断传入的 key 和当前遍历的 key 是否相等，相等则覆盖旧的 value。不为空则需要新建一个 HashEntry 并加入到 Segment 中，同时会先判断是否需要扩容。
5. 最后会解除所获取当前 Segment 的锁。

### get方法

	public V get(Object key) {
	    Segment<K,V> s; // manually integrate access methods to reduce overhead
	    HashEntry<K,V>[] tab;
	    int h = hash(key);
	    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
	    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
	        (tab = s.table) != null) {
	        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
	                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
	             e != null; e = e.next) {
	            K k;
	            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
	                return e.value;
	        }
	    }
	    return null;
	}


## ConcurrentHashMap在1.8中的实现


1.8相较于1.7更改主要在一下几个点：

- 抛弃了原有的 Segment 分段锁，而采用了 CAS + synchronized 来保证并发安全性
- 存放数据的 HashEntry 改为 Node，但作用都是相同的
- val next 都用了 volatile 修饰，保证了可见性
- 链表长度超过8就采用红黑树来实现









主要引用：
> [HashMap? ConcurrentHashMap? 相信看完这篇没人能难住你](https://crossoverjie.top/2018/07/23/java-senior/ConcurrentHashMap/)

> [Map 综述（三）：彻头彻尾理解 ConcurrentHashMap](https://blog.csdn.net/justloveyou_/article/details/72783008)