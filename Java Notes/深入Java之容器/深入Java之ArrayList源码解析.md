### ArrayList源码解析

##### 0. ArrayList

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

* ArrayList是Java集合框架中一个很重要的类，它继承于AbstractList，实现了List接口，是一个长度可变的集合
* 集合中允许null的存在，底层采用数组实现，中间插入和删除元素效率不高
* 实现了RandomAccess接口，可以对元素进行快速访问
* 实现了Serializable接口，可以被序列化
* 实现了Cloneable接口，可以被复制
* 非线程安全

##### 1. ArrayList属性

```java
private static final int DEFAULT_CAPACITY = 10;
private static final Object[] EMPTY_ELEMENTDATA = {};
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}; //当第一次添加元素进入ArrayList中时，数组将扩容值DEFAULT_CAPACITY
transient Object[] elementData; // 内部使用Object对象数组存储,访问级别为包内私有，使内部类能够访问到其中的元素，被标记为transient，在对象被序列化的时候不会被序列化。
private int size; // ArrayList的实际大小（数组包含的元素个数）
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8; //数组能分配的最大长度
```

* **EMPTY_ELEMENTDATA**和**DEFAULTCAPACITY_EMPTY_ELEMENTDATA**区别
  * 都是为了初始化elementData
  * 无参构造使用DEFAULTCAPACITY_EMPTY_ELEMENTDATA，数组长度为10，带参构造但参数为0使用EMPTY_ELEMENTDATA，数组长度为0

##### 2. 构造函数

默认构造方法

```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

带有initialCapacity构造方法

```java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```

带有另一个Collection的构造方法：首先将集合c转化为数组，然后检查转化的类型，如果不是Object[]类型，使用Arrays类中的copyOf方法进行复制；同时，如果c中没有元素，使用EMPTY_ELEMENTDATA初始化

```java
 public ArrayList(Collection<? extends E> c) {
     elementData = c.toArray();
     if ((size = elementData.length) != 0) {
         // c.toArray might (incorrectly) not return Object[] (see 6260652)
         if (elementData.getClass() != Object[].class)
             elementData = Arrays.copyOf(elementData, size, Object[].class);
     } else {
         // replace with empty array.
         this.elementData = EMPTY_ELEMENTDATA;
     }
 }
```

##### 3. add方法

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
// 在指定位置插入元素
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1, size - index);
    elementData[index] = element;
    size++;
}

public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;
}
```

add之前先确保内部容量足够，minCapacity为添加新元素后总元素长度，即数组应该保证的最小容量

```java
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {// 无参构造且还是空数组时，默认容量为10
        return Math.max(DEFAULT_CAPACITY, minCapacity); 
    }
    return minCapacity;
}
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);//新容量计算方法：size / 2 + size
    if (newCapacity - minCapacity < 0) // 因为add也能添加另一个集合，可能导致newCapacity还不够，那就用给定的minCapacity
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0) // 新容量如果超出了数组能分配的最大长度
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);//使用Arrays.copyOf生成新的数组对象
}
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}
```

##### 4. remove方法

```java
public E remove(int index) { // 删除指定位置的元素
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0) // remove底层也是使用数组拷贝的方式实现
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
// 删除指定的对象，注意ArrayList可以存储null对象，只删找到的第一个
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
private void fastRemove(int index) { //删除相应位置的对象
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

##### 5. 遍历对象：iterator()

调用iterator方法，创建一个新的内部类**Itr**实例，内部类Itr对AbstractList.Itr做了优化。

```java
public Iterator<E> iterator() {
    return new Itr();
}
private class Itr implements Iterator<E> {
    int cursor;       // index of next element to return
    int lastRet = -1; // index of last element returned; -1 if no such
    int expectedModCount = modCount;

    Itr() {}

    public boolean hasNext() {
        return cursor != size;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        checkForComodification(); // 比如，单线程环境下，正在迭代时，调用了list的add或remove方法，modCount就会改变，导致expectedModCount和modCount不相等
        int i = cursor;
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length) // ？
            throw new ConcurrentModificationException();
        cursor = i + 1;
        return (E) elementData[lastRet = i];
    }
	// 删除上次返回的元素
    public void remove() {
        if (lastRet < 0) throw new IllegalStateException();
        checkForComodification();
        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;
            lastRet = -1; // 不能连续remove
            expectedModCount = modCount;//迭代器内部remove，expectedModCount和modCount一致
        } catch (IndexOutOfBoundsException ex) { // 多线程环境下？
            throw new ConcurrentModificationException();
        }
    }
    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

两种情况下会出现fail-fast（遍历一个容器对象时容器结构被修改），导致抛出ConcurrentModificationException：

* 单线程环境：用迭代器遍历一个集合过程中，集合结构被修改（迭代器内部的remove方法不会抛出这个异常）
* 多线程环境：当一个线程遍历集合过程中，而另一个线程对集合结构进行了修改。

##### 6. 遍历对象：listIterator()

ArrayList也支持双向遍历

```java
public ListIterator<E> listIterator() {
    return new ListItr(0);
}
private class ListItr extends Itr implements ListIterator<E> {
    ListItr(int index) {
        super();
        cursor = index;
    }
    public boolean hasPrevious() {
        return cursor != 0;
    }
    public int nextIndex() {
        return cursor;
    }
    public int previousIndex() {
        return cursor - 1;
    }
    @SuppressWarnings("unchecked")
    public E previous() {
        checkForComodification();
        int i = cursor - 1;
        if (i < 0)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i;
        return (E) elementData[lastRet = i];
    }
    public void set(E e) {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();
        try {
            ArrayList.this.set(lastRet, e);
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
    public void add(E e) {
        checkForComodification();
        try {
            int i = cursor;
            ArrayList.this.add(i, e);
            cursor = i + 1;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
}
```

##### 7. trimToSize方法

减小elementData的大小

```java
public void trimToSize() {
    modCount++;
    if (size < elementData.length) {
        elementData = (size == 0)
            ? EMPTY_ELEMENTDATA
            : Arrays.copyOf(elementData, size);
    }
}
```

##### 8. indexOf方法

返回找到的第一个索引，

```java
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
public int lastIndexOf(Object o) {
    if (o == null) {
        for (int i = size-1; i >= 0; i--)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = size-1; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}

```

##### 9. clone方法

使用Arrays.copyOf拷贝

```java
public Object clone() {
    try {
        ArrayList<?> v = (ArrayList<?>) super.clone();
        v.elementData = Arrays.copyOf(elementData, size);
        v.modCount = 0;
        return v;
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError(e);
    }
}
// Arrays.java
public static <T> T[] copyOf(T[] original, int newLength) { // 复制一份数组
    return (T[]) copyOf(original, newLength, original.getClass());
}
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked") // 如果是同类型数组的复制，直接new Object[newLength]
    T[] copy = ((Object)newType == (Object)Object[].class) 
        ? (T[]) new Object[newLength] // 否则Array.newInstance创建一个该类型的数组
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0, // 实现数组之间的复制
                     Math.min(original.length, newLength));
    return copy;
}
```

##### 10. toArray方法

toArray(T[] a)要注意，a本是list要转化成array的数组，但如果a的长度小于List的size，这个a是没有用的

```java
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        // Make a new array of a's runtime type, but my contents:
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}
```

##### 11. clear方法

clear只是让elementData每个元素引用都为null

```java
public void clear() {
    modCount++;
    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;
    size = 0;
}
```

##### 12. 问题：关于ArrayList序列化

ArrayList内部的数组elementData是transient的，表示在序列化对象的时候，这个属性不会被序列化，那ArrayList是如何解决这个问题呢？

* 一般而言，对象实现了Serializable接口，直接调用`ObjectOutputStream`和`ObjectInputStram`的相应方法就可以实现序列化和反序列化

* ArrayList的实现方案：注意到，ArrayList内部有两个方法`writeObject`和`readObject`，看样子和序列化有关

* ```java
  private void writeObject(java.io.ObjectOutputStream s)
      throws java.io.IOException{
      // Write out element count, and any hidden stuff
      int expectedModCount = modCount;
      s.defaultWriteObject();
  
      // Write out size as capacity for behavioural compatibility with clone()
      s.writeInt(size);
  
      // Write out all elements in the proper order.
      for (int i=0; i<size; i++) {
          s.writeObject(elementData[i]); //虽然elementData被transient修饰，不能被序列化，但是我们可以将它的值取出来，然后将该值写入输出流。
      }
  
      if (modCount != expectedModCount) {
          throw new ConcurrentModificationException();
      }
  }
  private void readObject(java.io.ObjectInputStream s)
      throws java.io.IOException, ClassNotFoundException {
      elementData = EMPTY_ELEMENTDATA;
  
      // Read in size, and any hidden stuff
      s.defaultReadObject();
  
      // Read in capacity
      s.readInt(); // ignored
  
      if (size > 0) {
          // be like clone(), allocate array based upon size not capacity
          int capacity = calculateCapacity(elementData, size);
          SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, capacity);
          ensureCapacityInternal(size);
  
          Object[] a = elementData;
          // Read in all elements in the proper order.
          for (int i=0; i<size; i++) {
              a[i] = s.readObject();
          }
      }
  }
  ```

* 这两个方法都定义为`private`，什么时候调用，由谁调用呢？

* 如果一个类不仅实现了`Serializable`接口，而且定义了` readObject（ObjectInputStream in）`和 `writeObject(ObjectOutputStream out)`方法，那么将按照如下的方式进行序列化和反序列化：

  *ObjectOutputStream会调用这个类的writeObject方法进行序列化，ObjectInputStream会调用相应的readObject方法进行反序列化。*

* `ObjectOutputStream`通过反射可以知道一个类是否实现了这两个方法。

* 为什么使用transient修饰elementData？

  * 因为ArrayList会自动扩容，elementData数组相当于容器，当容器不足时就会再扩充容量，但是容器的容量往往都是大于或者等于ArrayList所存元素的个数。
  * 如果直接序列化elementData，会浪费空间
  * 因此，ArrayList在writeObject方法中手动将其序列化，并且只序列化了实际存储的那些元素，而不是整个数组。

##### 13. 参考资料

1. [ArrayList序列化技术细节详解](https://www.cnblogs.com/aoguren/p/4767309.html)