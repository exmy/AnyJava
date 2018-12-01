#### HashSet源码解析

基于 jdk 1.8

[TOC]

##### 0. HashSet

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
```

* 基于HashMap实现，底层使用HashMap来保存所有元素，相关HashSet的操作，基本上都是直接调用底层HashMap的相关方法来完成
* 线程不安全，允许null值

##### 1. HashSet属性

```java
private transient HashMap<E,Object> map; // 底层使用HashMap来保存所有元素
// 定义一个虚拟的Object对象作为HashMap的value，将此对象定义为static final
private static final Object PRESENT = new Object();
```

##### 2. HashSet构造方法

```java
public HashSet() { // 构造一个空的HashSet，底层初始化一个空的HashMap
    map = new HashMap<>();
}
// 构造一个包含指定Collection元素的HashSet
public HashSet(Collection<? extends E> c) {
    // 底层使用默认的加载因子和足以包含指定collection中所有元素的初始容量来创建一个HashMap
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}
// 指定初始容量和加载因子
public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}
// 指定初始容量
public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}
//以指定的initialCapacity和loadFactor构造一个新的LinkedHashMap。此构造函数为包访问权限，不对外公开，实际只是对LinkedHashSet的支持。
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

##### 3. add方法

```java
// 以e作为key，PRESENT作为value放进Hashmap
// 显然，如果是同样的key，用PRESENT覆盖，还是没有任何改变，保证Set中元素不重复
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

##### 4. remove方法

```java
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```

##### 5. contains方法

```java
public boolean contains(Object o) {
    return map.containsKey(o);
}
```

##### 6. 集合操作

都是继承自父类`AbstractCollection`

交集：

```java
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    boolean modified = false;
    Iterator<E> it = iterator();
    while (it.hasNext()) {
        if (!c.contains(it.next())) {
            it.remove();
            modified = true;
        }
    }
    return modified;
}
```

并集：

```java
public boolean addAll(Collection<? extends E> c) {
    boolean modified = false;
    for (E e : c)
        if (add(e))
            modified = true;
    return modified;
}
```

差集：

```java
public boolean removeAll(Collection<?> c) {
    Objects.requireNonNull(c);
    boolean modified = false;
    Iterator<?> it = iterator();
    while (it.hasNext()) {
        if (c.contains(it.next())) {
            it.remove();
            modified = true;
        }
    }
    return modified;
}
```

##### 7. 遍历

```java
public Iterator<E> iterator() {
    return map.keySet().iterator();
}
```

