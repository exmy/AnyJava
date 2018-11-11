#### 深入Java之类型信息（RTTI）

##### 1. 概念

RTTI：Run-Time Type Identification，运行时类型识别，顾名思义，在运行时识别一个对象的类型和类的信息。

RTTI分为两种:

* 传统的RTTI，假定在编译时就已经知道了所有的类型
* 反射机制：允许我们在运行时发现和使用类的信息

在Java中用来表示运行时类型信息的就是**Class类**，这个类存在于`java.lang`包中。

```Java
public final class Class<T> implements java.io.Serializable,
                              GenericDeclaration,
                              Type,
                              AnnotatedElement {
    private static final int ANNOTATION= 0x00002000;
    private static final int ENUM      = 0x00004000;
    private static final int SYNTHETIC = 0x00001000;

    private static native void registerNatives();
    static {
        registerNatives();
    }

    /*
     * Private constructor. Only the Java Virtual Machine creates Class objects.
     * This constructor is not used and prevents the default constructor being
     * generated.
     */
    private Class(ClassLoader loader) {
        // Initialize final field for classLoader.  The initialization value of non-null
        // prevents future JIT optimizations from assuming this final field is null.
        classLoader = loader;
    }

```

##### 2. Class对象

* 每个类都有一个Class对象，每当编写并编译一个新类就会产生一个Class对象（保存在一个同名的.class文件中，实际上，**编译后的字节码文件保存的就是Class对象**），这个对象就保存了与该类有关的所有信息，类的实例对象创建时就是依据Class对象中的类型信息完成的
* 当new一个对象或者引用类的静态成员变量时，JVM中的类加载子系统会将该类的字节码（即该类的Class对象）加载到JVM中，载入之后，就可以用来创建这个类的所有对象。
* Class类只存私有构造函数，因此对应Class对象只能由有JVM创建和加载。
* 每个通过关键字class标识的类，在内存中有且只有一个与之对应的Class对象来描述其类型信息，无论创建多少个实例对象，其依据的都是用一个Class对象

##### 3. Class对象的加载和获取

* 加载时机：使用new创建对象实例或引用类的静态成员的时候

* 加载过程：

  * 类加载器首先检查该类的Class对象是否已被加载，如果还没加载，默认的类加载器就根据类名查找.class文件（因为Class对象就保存在该文件中）
  * 加载字节码文件时，需要进行验证，以确保其没有被破坏并且不包含不良Java代码
  * 加载到内存后，相当于Class对象也就被载入内存了，就可以用来创建这个类的所有实例对象

* 获取Class对象：

  * `Class.forName`方法：返回一个对应类的Class对象，无需通过持有该类的实例对象引用就能获取

    ```java
    static Class<?> forName(String className)  
    static Class<?> forName(String name, boolean initialize, ClassLoader loader) 
    ```

  * `obj.getClass()`：通过一个实例对象获取一个类的Class对象，`getClass`方法继承自顶级类`Object`

  * **Class字面常量**：比如，`int.class`, `Integer.TYPE`（TYPE是一个引用，指向基本数据类型的Class对象）

    * 这种方法相比前两种更简单，也更安全
    * 享有编译期检查
    * 通过字面常量获取Class对象的引用不会自动初始化该类
    * 字面常量获取Class对象引用的方式不仅可以应用于普通的类，也可以应用于接口、数组以及基本数据类型，这点在反射技术应用传递参数时很有帮助
    * 获取字面常量的Class引用时，触发的只是类的加载阶段

  * 总结：

    * 实例类的getClass方法和Class类的静态方法forName都将会触发类的初始化阶段，而字面常量获取Class对象的方式则不会触发初始化
    * 初始化是类加载的最后一个阶段，也就是说完成这个阶段后类也就加载到内存中(Class对象在加载阶段已被创建)，此时可以对类进行各种必要的操作了（如new对象，调用静态成员等），注意在这个阶段，才真正开始执行类中定义的Java程序代码或者字节码。

##### 4. 类加载的过程

* 加载：通过一个类的完全限定名查找此类的字节码文件，并利用字节码文件**创建一个Class对象**。

* 链接：验证字节码的安全性和完整性，**为静态域分配存储空间**，注意此时只是分配静态成员变量的存储空间，不包含实例成员变量，如果必要的话，解析这个类创建的对其他类的所有引用。
* 初始化：类加载最后阶段，若该类具有超类，则对其进行初始化（**初始化该存储空间**），执行静态初始化器和静态初始化成员变量。

##### 5. 泛型的Class引用

* 如果不加泛型，Class应用表示的就是它所指向的对象的确切类型

  ```java
  public static void main(String[] args) {
          Class intClass = int.class;
          intClass = double.class;
          System.out.println(intClass.getName()); // 输出double 
      }
  ```

  如果是`Class<Integer>`，编译就会出错，使用泛型语法，就可以让编译器强制执行额外的类型检查

* 通配符`?`表示“任何事物”，`Class<?>`优于直接使用`Class`，这样做的好处是告诉编译器，我们是确实是采用任意类型的泛型，而非忘记使用泛型约束，而且`?`于`extends`相结合可以告诉编译器接收某个类型的子类

  ```java
  public static void main(String[] args) {
          Class<? extends Number> bounded = int.class;
          bounded = double.class;
          System.out.println(bounded.getName()); // 输出double
      }
  ```

* 时刻记住向Class引用添加泛型约束仅仅是为了提供编译期类型的检查从而避免将错误延续到运行时期。

* `Class`中的实例方法`newInstance`可返回该对象的确切类型，`getSuperClass`方法返回的是`Class<? super T>`

##### 5. 关于类型转换

* 传统的类型转换，即**直接强制转换类型**，如`(Shape)`，由RTTI确保类型转换的正确性，如果类型转换失败，将会抛出类型转换异常`ClassCastException`

  * 在Java中，编译器允许自由的做向上转型的赋值操作，但是向下转型赋值需要显式的类型转换
  * 编译器将检查向下转型是否合理，不允许向下转型到实际上不是待转型类的子类的类型上

* 利用Class对象的`cast`方法，其参数接收一个参数对象并将其转换为Class引用的类型

  ```java
  public T cast(Object obj) {
      if (obj != null && !isInstance(obj))
           throw new ClassCastException(cannotCastMsg(obj));
       return (T) obj;
    }
  ```

* 关键字`instanceof`，返回一个布尔值，告诉我们对象是不是某个特定的类型实例

* Class类的`isInstance`方法和`instanceof`作用相同

  ```java
  public boolean isInstance(Object obj)
  ```


##### 参考资料

* 《java编程思想》第4版
* [深入理解Java类型信息(Class对象)与反射机制](https://blog.csdn.net/javazejian/article/details/70768369)