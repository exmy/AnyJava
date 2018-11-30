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
* 当哈希表容量达到`threshold`时，就会触发扩容。扩容前后，哈希桶的**长度一定会是2的幂**。 这样在根据key的hash值寻找对应的哈希桶时，可以**用位运算替代取余操作**，**更加高效**。
* 扩容操作时，会new一个新的`Node`数组作为哈希桶，然后将原哈希表中的所有数据(`Node`节点)移动到新的哈希桶中，相当于对原哈希表中所有的数据重新做了一个put操作。所以性能消耗很大，**可想而知，在哈希表的容量越大时，性能消耗越明显。**

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
//key的hash值，并不仅仅只是key对象的hashCode()方法的返回值，还要经过扰动函数的扰动，以使hash值更加均匀，计算方式：h^(h>>16)，得到的hash值还要对表长取模，取模方式是个亮点，hash&(n-1)
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

##### 5. putVal方法：往哈希表插入一个节点

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i; // n是表长，i是key在table中的位置
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length; // 哈希表为空，初始化
    // 如果table[i]还没有存放任何Node，没有哈希碰撞，直接构建一个新节点插入
    if ((p = tab[i = (n - 1) & hash]) == null) // (n-1)&hash，相当于对表长取模，获取在table表中的位置
        tab[i] = newNode(hash, key, value, null);
    else {
        // 如果table[i]已经存在Node，接下来要做的事情，如果key存在，则把value替换为新的，如果key不存在，则在table[i]下的链表或红黑树中找一个位置插入该(key,val)组成的新Node
        Node<K,V> e; K k;
        if (p.hash == hash && // 什么情况下p.hash会不等于hash？
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p; // 要put的key存在，并且这个key等于table[i]的第一个Node的key
        else if (p instanceof TreeNode) // key不存在或key不等于table[i]第一个节点的key，并且当前table[i]是红黑树的形式
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else { // key不存在或key不等于table[i]第一个节点的key，并且当前table[i]是链表的形式
            for (int binCount = 0; ; ++binCount) { // 遍历链表
                if ((e = p.next) == null) { // 从第二个节点开始遍历
                    p.next = newNode(hash, key, value, null); // key不存在，尾插
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash); // 链表长度达到8，树化为红黑树
                    break;
                }
                if (e.hash == hash && // key存在
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // e不为null，表示key存在，替换旧value
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // onlyIfAbsent为true表示不会覆盖相同key的value
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue; // 覆盖一个旧节点则直接返回
        }
    }
    // 新插入一个节点，表示修改了表结构，更新modCount
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

##### 6. resize方法：扩容

```java
// 初始化或者加倍哈希桶的大小，如果是当前哈希桶是null,分配符合当前阈值的初始容量目标。 
// 否则，因为我们扩容成以前的两倍。 
// 在扩容时，要注意区分以前在哈希桶相同index的节点，现在是在以前的index里，还是index+oldlength 里
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) { // 当前哈希桶容量大于0
        if (oldCap >= MAXIMUM_CAPACITY) { // 如果当前哈希桶容量已经达到上限，
            threshold = Integer.MAX_VALUE; // 设置阈值为2^31-1，无需扩容，直接返回
            return oldTab;
        } // 新的哈希桶容量为原先容量的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)// 如果原先容量大于等于默认初始容量
            newThr = oldThr << 1; // double threshold 设置新阈值为原先阈值的2倍
    }
    // 当前容量为0，即当前哈希表是空的，但是阈值不为0，这是初始化时指定了初始容量的情况，即new HashMap<>(cap)，阈值就是tableSizeFor(cap)，新的容量就是旧的阈值，新阈值在下面的if(newThr==0)设置
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // 当前容量为0，并且阈值也为0，这是初始化时没有指定初始容量的情况，即new HashMap<>()
    else {  // 指定加载因子为默认的加载因子0.75
        newCap = DEFAULT_INITIAL_CAPACITY; // 设置容量为默认初始容量16
        // 设置阈值为16 * 0.75 = 12
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 当前表空、有阈值的情况，即上面第二个if的情况，设置新阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr; // 更新阈值
    // 根据新容量构建新的哈希表
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab; // 更新哈希表引用
    if (oldTab != null) { // 将原先哈希表的节点转移到新的哈希表中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) { // e指向oldTab[j]第一个节点
                oldTab[j] = null; // 原先的节点引用置为null，以便gc
                if (e.next == null) // oldTab[j]只有一个节点
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode) // 红黑树的情况
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order 链表的情况
                    // 扩容之后容量翻倍，原哈希表oldTab[j]上的链表节点，经过哈希取模之后，在新哈希表的位置有两种情况，1.仍然是原来的位置，下面称之为lo位 2. 原来的位置+oldTab.length，下面称之为hi位，比如，长度为16的哈希表扩容之后长度为32，某个节点key的哈希值为24，在索引为24%16=8的位置，现在在新哈希表的索引为24 % 32 =24，即8+16。
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        // 遍历oldTab[j]链表，把链表中每一个节点转移到新哈希表中
                        next = e.next;
                        // 这个位运算相当妙，相当于(e.hash % oldCap == e.hash % newCap),仍然是原来的位置
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
                }
            }
        }
    }
    return newTab;
}
```

总结：

如果size大小超过了阈值需要扩容，分为两步：

* 扩容：哈希表容量扩充为原来的2倍
* 移动：哈希表中的每个节点重新计算其hash值，重新计算每个节点在新哈希表中位置，将原来的节点移动到新哈希表中
  * hash值计算方式不变，计算在新哈希表位置：`(hash & oldCap ) == 0`则仍然是原来的位置，否则是原来的位置+oldTab.length

##### 7. put方法

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
public void putAll(Map<? extends K, ? extends V> m) {
    putMapEntries(m, true);
}
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        if (table == null) { // pre-size
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
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
// 如果key对应的value存在，不会覆盖
public V putIfAbsent(K key, V value) {
    return putVal(hash(key), key, value, true, true);
}
```

总结：

* put方法内部调用putVal方法，主要传入key、value以及key的hash值
* 首先根据key计算出其hash值，方式是`(h = key.hashCode()) ^ (h >>> 16)`，然后对表长取模得到在哈希表中的存放位置，取模方式：`h&(n-1)`，假设位置是`i`
* 然后就是三部曲：
  * 如果该hash没有碰撞，也就是说table[i]上还没有任何节点，那么直接创建一个节点
  * 如果hash有碰撞，判断一下table[i]这个节点是不是红黑树节点，是的话去红黑树里put
  * 如果不是，就从table[i].next这个节点开始遍历链表，用`key.equals(k)`判断是否存在相同的key，是的话用value替换旧的value，如果key不存在，放到链表尾部
* put一个节点后，如果table[i]节点个数等于8，链表转化为红黑树
* 添加一个节点后，如果size大于阈值，扩容

##### 8. get方法

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
}
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first; // tab[i]第一个节点就是key
        if ((e = first.next) != null) {
            if (first instanceof TreeNode) // 如果tab[i]是红黑树，在树种去找
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do { // 遍历链表
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

总结：

* get方法内部调用getNode方法，传入两个参数，key以及key的hash值
* 根据hash值取模(h&(n-1))得到key在哈希表中的索引
* 判断该处索引的第一个节点是否就是要查找的节点，如果是，返回这个节点
* 如果不是，接下来判断这第一个节点是否为TreeNode，如果是TreeNode，那么从红黑树中查找
  * 注意TreeNode和Node的关系
    * TreeNode继承自LinkedHashMap.Entry，而这个Entry继承自HashMap.Node
    * 也就是说，TreeNode是Node的子类
  * 因此，(TreeNode<K,V>)first可以转型
* 如果不是TreeNode，那么遍历链表，找到为key的节点即可

##### 9. remove方法

```java
// Removes the mapping for the specified key from this map if present
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
// Removes the entry for the specified key only if it is currently mapped to the specified value.
public boolean remove(Object key, Object value) {
    return removeNode(hash(key), key, value, true, true) != null;
}
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash && // 总是检查第一个节点，第一个节点就是要删除的节点
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode) // 红黑树的情况
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do { // 遍历链表，如果找到，e就是要删除的节点，p是该节点的前面一个节点
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        // node即为要删除的节点，matchValue为true,则node.value必须等于value才可删除node
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p) // node为tab[index]第一个节点
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

##### 10. replace方法

```java
// Replaces the entry for the specified key only if it is currently mapped to some value.
public V replace(K key, V value) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) != null) {
        V oldValue = e.value;
        e.value = value;
        afterNodeAccess(e);
        return oldValue;
    }
    return null;
}
// Replaces the entry for the specified key only if currently mapped to the specified value
public boolean replace(K key, V oldValue, V newValue) {
    Node<K,V> e; V v;
    if ((e = getNode(hash(key), key)) != null &&
        ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {
        e.value = newValue;
        afterNodeAccess(e);
        return true;
    }
    return false;
}
```

##### 11. contains方法

```java
public boolean containsKey(Object key) {
    return getNode(hash(key), key) != null;
}
public boolean containsValue(Object value) {
    Node<K,V>[] tab; V v;
    if ((tab = table) != null && size > 0) {
        for (int i = 0; i < tab.length; ++i) {
            // 很有意思
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                if ((v = e.value) == value ||
                    (value != null && value.equals(v)))
                    return true;
            }
        }
    }
    return false;
}
```

##### 12. 遍历

```java
public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}    
public Set<K> keySet() {
    Set<K> ks = keySet;
    if (ks == null) {
        ks = new KeySet();
        keySet = ks;
    }
    return ks;
}
public Collection<V> values() {
    Collection<V> vs = values;
    if (vs == null) {
        vs = new Values();
        values = vs;
    }
    return vs;
}
```

EntrySet：

```java
final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<Map.Entry<K,V>> iterator() {
        return new EntryIterator();
    }
    public final boolean contains(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>) o;
        Object key = e.getKey();
        Node<K,V> candidate = getNode(hash(key), key);
        return candidate != null && candidate.equals(e);
    }
    public final boolean remove(Object o) {
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>) o;
            Object key = e.getKey();
            Object value = e.getValue();
            return removeNode(hash(key), key, value, true, true) != null;
        }
        return false;
    }
}
```

KeySet：

```java
final class KeySet extends AbstractSet<K> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<K> iterator()     { return new KeyIterator(); }
    public final boolean contains(Object o) { return containsKey(o); }
    public final boolean remove(Object key) {
        return removeNode(hash(key), key, null, false, true) != null;
    }
    public final Spliterator<K> spliterator() {
        return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
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
}
```

Values:

```java
final class Values extends AbstractCollection<V> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<V> iterator()     { return new ValueIterator(); }
    public final boolean contains(Object o) { return containsValue(o); }
    public final Spliterator<V> spliterator() {
        return new ValueSpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super V> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.value);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}
```

##### 13. clear方法

```java
public void clear() {
    Node<K,V>[] tab;
    modCount++;
    if ((tab = table) != null && size > 0) {
        size = 0;
        // 扫描一遍哈希表，每个tab[i]引用设为null即可
        for (int i = 0; i < tab.length; ++i)
            tab[i] = null;
    }
}
```

##### 14.一张图总结

![](https://github.com/exmy/CSNotes/raw/master/pics/hashmap.jpg)

##### 15. 和HashTable比较

* hashtable线程安全，不允许key、value为null
* hashtable默认容量为11
* `HashTable`是直接使用key的hashCode(`key.hashCode()`)作为hash值，不像`HashMap`内部使用`static final int hash(Object key)`扰动函数对key的hashCode进行扰动后作为hash值。
* `HashTable`取哈希桶下标是直接用模运算%.（因为其默认容量也不是2的n次方。所以也无法用位运算替代模运算）
* 扩容时，新容量是原来的2倍+1。`int newCapacity = (oldCapacity << 1) + 1;`
* `Hashtable`是`Dictionary`的子类同时也实现了`Map`接口，`HashMap`是`Map`接口的一个实现类；

##### 16. 参考

* [面试必备：HashMap源码解析(JDK8)](https://blog.csdn.net/zxt0601/article/details/77413921)