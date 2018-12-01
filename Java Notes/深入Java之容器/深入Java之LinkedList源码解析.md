### LinkedList源码解析

[TOC]

##### 0. LinkedList

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

* 继承于AbstractSequentialList抽象类的双向链表，可以作为栈、队列、双端队列使用
* 实现`List`接口，可以进行队列操作
* 实现`Deque`接口，可以进行双端队列的操作
* 实现`Cloneable`接口，可以被克隆
* 实现`Serializable`接口，可以序列化
* 非线程安全
* 可以存null

##### 1. LinkedList属性

```java
transient int size = 0;
transient Node<E> first; // Pointer to first node.
transient Node<E> last; // Pointer to last node.
```

##### 2. LinkedList内部类

```java
private static class Node<E> { // 链表节点
    E item;
    Node<E> next;
    Node<E> prev;
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

##### 3. LinkedList构造方法

```java
public LinkedList() {} // 
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}
// 在链表中的index位置开始添加集合c的元素
public boolean addAll(int index, Collection<? extends E> c) {
    checkPositionIndex(index); // 越界检查

    Object[] a = c.toArray(); // 先转换成对象数组
    int numNew = a.length;
    if (numNew == 0)
        return false;

    Node<E> pred, succ;
    if (index == size) { // 在链表末尾添加
        succ = null;
        pred = last;
    } else {
        succ = node(index); // 在index位置添加
        pred = succ.prev;
    }

    for (Object o : a) { // 遍历集合c中的每一个元素，一个一个添加
        E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        pred = newNode;
    }

    if (succ == null) {
        last = pred;
    } else {
        pred.next = succ;
        succ.prev = pred;
    }

    size += numNew;
    modCount++;
    return true;
}
Node<E> node(int index) { // 返回指定位置的节点
    // assert isElementIndex(index);
    if (index < (size >> 1)) { // 如果index在链表前半部分，从前往后找
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else { // 否则，从后往前找
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

##### 4. 实现List接口的方法

add方法：

```java
public boolean add(E e) {
    linkLast(e);
    return true;
}
void linkLast(E e) { // 把e添加到链表末尾
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

remove方法：

```java
public boolean remove(Object o) { // 删除找到的第一个元素，包括null，然后size--，modCount++
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
E unlink(Node<E> x) { // Unlinks non-null node x.
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```

indexOf方法：可以从头到尾，也可以从尾到前遍历查找

```java
public int indexOf(Object o) {
    int index = 0;
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index;
            index++;
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index;
            index++;
        }
    }
    return -1;
}
public int lastIndexOf(Object o) {
    int index = size;
    if (o == null) {
        for (Node<E> x = last; x != null; x = x.prev) {
            index--;
            if (x.item == null)
                return index;
        }
    } else {
        for (Node<E> x = last; x != null; x = x.prev) {
            index--;
            if (o.equals(x.item))
                return index;
        }
    }
    return -1;
}
```

get/set方法：获取/设置指定位置的元素

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
```

clear方法：

```java
public void clear() {
        // Clearing all of the links between nodes is "unnecessary", but:
        // - helps a generational GC if the discarded nodes inhabit
        //   more than one generation
        // - is sure to free memory even if there is a reachable Iterator
        for (Node<E> x = first; x != null; ) {
            Node<E> next = x.next;
            x.item = null;
            x.next = null;
            x.prev = null;
            x = next;
        }
        first = last = null;
        size = 0;
        modCount++;
    }
```

##### 5. 实现Deque接口的方法

offer方法：添加元素，返回是否添加成功

```java
public boolean offer(E e) { // 添加到链表尾部
    return add(e); // linkLast(e)
}
public boolean offerFirst(E e) { // 添加到链表头部
    addFirst(e);
    return true;
}
public boolean offerLast(E e) { // 添加到链表尾部
    addLast(e);
    return true;
}
public void addFirst(E e) {
    linkFirst(e);
}
public void addLast(E e) {
    linkLast(e);
}
private void linkFirst(E e) { // Links e as first element
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

poll方法：获取并删除

```java
public E poll() { // 获取链表第一个节点并删除
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
public E pollFirst() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
public E pollLast() {
    final Node<E> l = last;
    return (l == null) ? null : unlinkLast(l);
}

private E unlinkFirst(Node<E> f) { // 删除头节点
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
private E unlinkLast(Node<E> l) { // 删除尾节点
    // assert l == last && l != null;
    final E element = l.item;
    final Node<E> prev = l.prev;
    l.item = null;
    l.prev = null; // help GC
    last = prev;
    if (prev == null)
        first = null;
    else
        prev.next = null;
    size--;
    modCount++;
    return element;
}
```

peek方法：获取元素，不删除

```java
public E peek() { // 获取第一个元素
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}
public E peekFirst() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}
public E peekLast() { // 获取最后一个元素
    final Node<E> l = last;
    return (l == null) ? null : l.item;
}
```

##### 6. 作为stack使用的方法

```java
public void push(E e) { 
    addFirst(e);
}
public E pop() { // 获取并删除
    return removeFirst();
}
public E peek() { // 获取第一个元素
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}
public boolean isEmpty() {
    return size() == 0;
}
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}
```

##### 7. 遍历集合：listIterator

```java
public ListIterator<E> listIterator(int index) {
    checkPositionIndex(index);
    return new ListItr(index);
}
private class ListItr implements ListIterator<E> {
    private Node<E> lastReturned;
    private Node<E> next; // 调用next()要返回的节点
    private int nextIndex;
    private int expectedModCount = modCount;

    ListItr(int index) {
        // assert isPositionIndex(index);
        next = (index == size) ? null : node(index);
        nextIndex = index;
    }

    public boolean hasNext() {
        return nextIndex < size;
    }

    public E next() {
        checkForComodification();
        if (!hasNext())
            throw new NoSuchElementException();

        lastReturned = next;
        next = next.next;
        nextIndex++;
        return lastReturned.item;
    }

    public boolean hasPrevious() {
        return nextIndex > 0;
    }

    public E previous() {
        checkForComodification();
        if (!hasPrevious())
            throw new NoSuchElementException();

        lastReturned = next = (next == null) ? last : next.prev;
        nextIndex--;
        return lastReturned.item;
    }

    public int nextIndex() {
        return nextIndex;
    }

    public int previousIndex() {
        return nextIndex - 1;
    }

    public void remove() { // 删除上一次访问的元素
        checkForComodification();
        if (lastReturned == null)
            throw new IllegalStateException();

        Node<E> lastNext = lastReturned.next;
        unlink(lastReturned);
        if (next == lastReturned)
            next = lastNext;
        else
            nextIndex--;
        lastReturned = null;
        expectedModCount++;
    }

    public void set(E e) { // 设置上一次访问的元素为e
        if (lastReturned == null)
            throw new IllegalStateException();
        checkForComodification();
        lastReturned.item = e;
    }

    public void add(E e) { // 
        checkForComodification();
        lastReturned = null;
        if (next == null)
            linkLast(e);
        else
            linkBefore(e, next);
        nextIndex++;
        expectedModCount++;
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

##### 8. clone方法

```java
public Object clone() { // 克隆的时候要遍历一遍原list
    LinkedList<E> clone = superClone();

    // Put clone into "virgin" state
    clone.first = clone.last = null;
    clone.size = 0;
    clone.modCount = 0;

    // Initialize clone with our elements
    for (Node<E> x = first; x != null; x = x.next)
        clone.add(x.item);

    return clone;
}
```

##### 9. toArray方法

```java
public Object[] toArray() { // 返回Object数组
    Object[] result = new Object[size];
    int i = 0;
    for (Node<E> x = first; x != null; x = x.next)
        result[i++] = x.item;
    return result;
}
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        a = (T[])java.lang.reflect.Array.newInstance(
        a.getClass().getComponentType(), size);
    int i = 0;
    Object[] result = a;
    for (Node<E> x = first; x != null; x = x.next)
        result[i++] = x.item;

    if (a.length > size)
        a[size] = null;

    return a;
}
```

##### 10. 序列化

和ArrayList类似