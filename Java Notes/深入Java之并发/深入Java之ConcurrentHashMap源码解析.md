#### ConcurrentHashMap源码解析

基于JDK 1.8

##### 0. ConcurrentHashMap

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable
```

* HashMap线程安全的实现
* 底层数据结构和HashMap一样，仍然是 数组+链表+红黑树

##### 1. 属性

主要介绍与HashMap不同的属性

```java
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
// 默认并发级别，在1.8中这是为了与之前的实现兼容
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
// 
private static final int MIN_TRANSFER_STRIDE = 16;
private static int RESIZE_STAMP_BITS = 16;
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

static final int MOVED     = -1; // hash for forwarding nodes
static final int TREEBIN   = -2; // hash for roots of trees
static final int RESERVED  = -3; // hash for transient reservations
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
static final int NCPU = Runtime.getRuntime().availableProcessors();

private static final ObjectStreamField[] serialPersistentFields = {
    new ObjectStreamField("segments", Segment[].class),
    new ObjectStreamField("segmentMask", Integer.TYPE),
    new ObjectStreamField("segmentShift", Integer.TYPE)
};
// 控制哈希表的初始化和扩容。
// -1，表示table正在初始化或扩容；-N 表示有N-1个线程正在进行扩容
// 大于0相当于hashMap 中的 threshold，表示阈值
private transient volatile int sizeCtl;

// 直到第一次插入才会初始化，数组大小总是2的幂
transient volatile Node<K,V>[] table;

// 只有当扩容的时候才不为null
private transient volatile Node<K,V>[] nextTable;
private transient volatile long baseCount;
```

##### 2. 重要内部类

Node与HashMap中的Node不同之处在于：

* val和next字段都是volatile的
* 不支持setValue操作
* 增加了find方法辅助map.get()

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }

    public final K getKey()       { return key; }
    public final V getValue()     { return val; }
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
    public final String toString(){ return key + "=" + val; }
    public final V setValue(V value) {
        throw new UnsupportedOperationException();
    }

    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = val) || v.equals(u)));
    }

    /**
         * Virtualized support for map.get(); overridden in subclasses.
         */
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```

TreeBin：

```java

```

##### 3. 构造方法

```java
public ConcurrentHashMap() {}

public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}

public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

##### 4. put方法

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode()); // 计算hash值，(h ^ (h >>> 16)) & HASH_BITS
    int binCount = 0;
    // for死循环，直到node插入成功
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0) 
            tab = initTable(); // table初始化
        // (n - 1) & hash，和hashmap一样，获取key在table中的位置，table[i]
        // f即为key在table中定位出的Node，(f = table[i]) f为null表示第一次插入没有冲突
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 第一次可以直接插入,table[i] = new Node(...)
            // CAS尝试插入，如果插入失败，自旋直到插入成功
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }// f=table[i]不为null,意味着发生了冲突，如果table[i]节点的hash等于MOVED(-1)
        // 表示table正在扩容，那么助其扩容转移元素
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(txab, f);
        else {// table[i]节点的hash不等于MOVED的情况，插入一个新节点
            V oldVal = null;
            synchronized (f) { // synchronized锁住桶头节点
                if (tabAt(tab, i) == f) { // 再次判断f是否为table[i]
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 如果key存在，onlyIfAbsent为false则更新value
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // key不存在，插到链表尾部
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // fh < 0，向红黑树中添加元素，TreeBin 结点的hash值为TREEBIN（-2）
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // binCount != 0 说明向链表或者红黑树中添加或修改一个节点成功
            // binCount == 0 说明 put 操作将一个新节点添加成为某个桶的首节点
            if (binCount != 0) {
                // 链表长度大于8转换为红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                // oldVal != null 说明此次操作是更新操作，直接返回旧值，无需做下面的扩容边界检查
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // CAS方式更新binCount，并判断是否需要扩容
    addCount(1L, binCount);
    return null;
}
```

initTable方法核心思想：只允许一个线程对表进行初始化，如果不巧有其他线程进来了，那么会让其他线程交出 CPU 等待下次系统调度。这样，保证了table同时只会被一个线程初始化。

```java
// 初始化过程
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // 用while循环判断，
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl<0 说明已经有线程正在进行初始化操作
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // sizeCtl以CAS的方式更新为-1，表示本线程正在初始化table
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                // 保险起见，再次判断下表是否为空
                if ((tab = table) == null || tab.length == 0) {
                    // sc 大于零说明容量已经初始化了，如果有指定初始容量，table大小即为初始容量大小，否则为默认容量大小
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    // 据容量构建table
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2); // ？？
                }
            } finally {
                sizeCtl = sc; // 设置阈值
            }
            break;
        }
    }
    return tab;
}

static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i, Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
// f即位table[i]
// 在多线程情况下，如果发现其它线程正在扩容，则帮助转移元素。
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // ForwardingNode：A node inserted at head of bins during transfer operations.
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                // Moves and/or copies the nodes in each bin to new table
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

