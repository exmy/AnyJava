### HashMap源码解析

基于java 1.8

##### 0. HashMap

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
```

* HashMap是一个实现了`Map`接口的哈希表，允许`key`为`null`和`value`为`null`
* 线程不安全，遍历无序
* 底层数据结构采用数组+链表，数组称为**哈希桶**，每个桶里面存放一个链表（单链表），链表的每个节点才是HashMap要存放的元素
* 在JDK8中，当链表长度达到**8**时，链表转化为红黑树，以提升查询和插入效率

##### 1. HashMap属性

```javascript
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16 默认初始容量
static final int MAXIMUM_CAPACITY = 1 << 30; // 最大容量，capacity是指桶的数量
static final float DEFAULT_LOAD_FACTOR = 0.75f; // 默认的加载因子
static final int TREEIFY_THRESHOLD = 8;	// 树化阈值，桶内bin数量达到这个值，链表就转化为树
static final int UNTREEIFY_THRESHOLD = 6; // 非树化阈值，桶内bin数量小于等于这个值时，树转化为链表
static final int MIN_TREEIFY_CAPACITY = 64; // 每个桶？？

transient Node<K,V>[] table; // 底层数组，数组存放的是一个链表或者树
transient Set<Map.Entry<K,V>> entrySet; // 
transient int size; // hashmap包含的key-value数量
transient int modCount; // 结构性修改次数，用于fast-fail
int threshold; // threshold=capacity * load factor，哈希表内元素数量的阈值，当哈希表内元素数量超过阈值时，会发生扩容resize()。
final float loadFactor; // 哈希表的加载因子
```

##### 2. HashMap内部链表节点

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }
    // 节点的hashcode由key的hashcode和value的hashcode异或得到
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

##### 4. HashMap构造方法

```java
// 构造一个指定初始容量和加载因子的空HashMap
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY) // 初始容量不能大于设定的最大容量
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor)) // 加载因子不能为负数
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor; 
    // 设定阈值为大于等于initialCapacity的最小2的幂
    this.threshold = tableSizeFor(initialCapacity);
}
public HashMap(int initialCapacity) { // 一般只需指定初始容量，加载因子采用默认的
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}

static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
// 添加另一个HashMap的元素，evict为false时表示初始化时调用
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        if (table == null) { // pre-size
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ? // 确定table的容量
                     (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        else if (s > threshold)
            resize();
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

