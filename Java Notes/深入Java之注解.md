#### 深入Java之注解(Annotation)

###### 1. 概念

* 注解，也成为元数据，用来完整的描述程序所需要的而无法用Java来表达的信息，简单来说，就是存储有关程序的额外信息。
* 注解是语言层级的概念，由编译器来测试和验证，享有编译器的类型检查
* 注解可以用来生成描述符文件（比如xml），新的类定义，减少样板代码，让代码更加干净易读
* 注解不能继承

###### 2. JavaSE内置的几个注解

* @Override：表示Override修饰的方法将覆盖超类的方法
* @Deprecated：表示该注解修饰的方法已不建议使用，如果使用了，编译器会给出警告信息
* @SuppressWarnings：关闭不当的编译器警告信息
* @SafeVarargs：java1.7引入，参数安全类型注解，提醒开发者不要用参数做一些不安全的操作,它的存在会阻止编译器产生 unchecked 这样的警告 
* @FunctionalInterface：java1.8引入，函数式接口注解，所谓函数式接口，就是一个只具有一个方法的接口

###### 3. 定义注解

* 示例：

  ```java
  @Target(ElementType.METHOD)
  @Retention(RetentionPolicy.RUNTIME)
  public @interface UseCase {
      public int id();
      public String desc() default "no desc";
  }
  ```

* 注解也会编译成一个class文件

* 定义注解需要用到一些元注解，如@Target，@Retention，@Documented ，@Inherited ，@Repeatable 

  * **@Target**：指定注解应用的地方，可以在类上、方法上、方法参数上 

    | ElementType.ANNOTATION_TYPE | 给一个注解进行注解                          |
    | --------------------------- | ------------------------------------------- |
    | ElementType.CONSTRUCTOR     | 可以给构造方法进行注解                      |
    | ElementType.FIELD           | 可以给属性进行注解                          |
    | ElementType.LOCAL_VARIABLE  | 可以局部变量进行注解                        |
    | ElementType.METHOD          | 可以给方法进行注解                          |
    | ElementType.PACKAGE         | 可以给一个包进行注解                        |
    | ElementType.PARAMETER       | 可以给一个方法内的参数进行注解              |
    | ElementType.TYPE            | 可以 给一个类型进行注解，比如类、接口、枚举|

    * **@Retention**：定义该注解需要在什么级别保存该注解，取值如下：
    	* RetentionPolicy.SOURCE 注解只在源码阶段保留，在编译器进行编译时它将被丢弃。 
    	- RetentionPolicy.CLASS 注解在class文件中可用，但会被VM丢弃。 
    	- RetentionPolicy.RUNTIME 注解将在运行期保留，它会被加载进入到 JVM 中，可以通过反射机制读取注解的信息。

  * **@Documented**： 能够将注解中的元素包含到 Javadoc 中。 

  * **@Inherited** ：允许子类继承父类的注解

  * **@Repeatable**：1.8才加入的注解，表示可重复使用

###### 4. 注解的属性（元素、成员变量）

* 可用类型：基本类型、String、Class、enum、Annotation、以上类型的数组
* 属性不能有不确定的值，要么是默认值，要么在使用注解时提供元素的值
* 对于非基本类型的属性，不能以null作为其值

###### 5. 注解的获取

注解通过反射获取。

```java
public boolean isAnnotationPresent(Class<? extends Annotation> annotationClass) {}
public <A extends Annotation> A getAnnotation(Class<A> annotationClass) {}
public Annotation[] getAnnotations() {}
```

###### 6. 注解的用处

- 提供信息给编译器： 编译器可以利用注解来探测错误和警告信息  
- 编译阶段时的处理： 软件工具可以用来利用注解信息来生成代码、Html文档或者做其它相应处理。  
- 运行时的处理： 某些注解可以在程序运行的时候接受代码的提取 

###### 参考资料

* 《Java编程思想-第4版》
* [ 秒懂，Java 注解 （Annotation）你可以这样学 ](https://blog.csdn.net/briblue/article/details/73824058)

