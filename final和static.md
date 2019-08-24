# Final关键字

## 基础概念

`final`译为最终的，不可改变的。

- 使用`final`修饰类，意味着该不可被继承，即不能被`extends`。

- 使用`final`修饰方法，意味着方法不可被重写，即不能被`@Override`。如果超类的方法修饰符是`private final`，则子类其实本身就不带有父类的该方法，也就不存在重写，所以子类的同名方法实则是全新的方法。(注意修饰方法时与abstract关键字冲突）

- 使用`final`修饰局部变量，意味着值不可变（与`final`修饰成员变量作用一致）有且只能赋值一次，不一定要在定义变量时赋值，只要在局部内有定义即可。

- 使用`final`修饰成员变量

  - 修饰基本数据类型：值不可变

    ```java
    public final int num = 1;
    
    // 编辑时异常
    num = 2;
    ```

  - 修饰对象：地址不可变，如果对象能设置成员变量值，值可变。

    ```java
    public final Person person = new Person();
    
    // 编辑时异常
    person = new Person();
    
    // 正常
    person.setName("lqd");
    ```

    

## final变量

### 初始化成员变量

一共有三种方式，只能三选一。

```java
public class StudyFinal {

    // 方式一:直接赋值
    private final int num = 1;
    
    // 方式二:使用构造器初始化
    private final String name;
    public StudyFinal(String name) {
        this.name = name;
    }
    
    // 方式三:使用代码块
    private final int age;
    
    {
        this.age = 10;
    }
}
```

**注意：**使用构造器赋值时，要确保所有构造器都对`final`成员变量进行赋值。



### final变量的特性

在一些情况下，JVM会优化final变量。

```java
byte b1 = 1, b2 = 2, b3, b6;
final byte b4 = 4, b5 = 6;
b6 = b4 + b5;				// b4 + b5 = (byte -> int) + (byte -> int) = int + int
							// 但是b4, b5都是final变量, 当2个final修饰相加时候会转换为左边变量的类型
b3 = (b1 + b2);				// 出错，左边为byte，右边为int。
System.out.println(b3 + b6);
```

### final修饰方法

表示方法不可被重写，超类用`private`关键字修饰默认就是加上了`final`。

### final修饰类

表示类不可被继承。

## Static

`static`意为静态的，可以修饰**类**的**成员变量及成员方法**（不可修饰局部变量）使之对应成为静态变量和静态方法。

使用：静态变量及静态方法由于依赖的是类而不是对象，所以在内存中有且只有一份。

### 静态变量

#### 初始化静态变量

一共两种方式。

```java
public class Test {

    private static int a = 1;	// 方式一，声明时设置初始值
    private static int b;			

    static {
        b = 1;	// 方式二，使用静态代码块
    }
}  
```

#### 静态变量在继承中的表现

```java
// 超类
public class Parent {
    public static int a = 1;
}

// 子类
public class Child extends Parent {

    public static int a = 2;

    public static void main(String[] args) {
        Child child = new Child();
        Parent parent = new Parent();
        System.out.println(a);	// 2
        System.out.println(parent.a);	// 1
        System.out.println(child.a);// 2
        System.out.println(Child.a);// 2
    }
}
```

如果子类不设置一个同名变量，输出的会是超类的静态变量。

### 静态方法

静态方法同样，子类能够继承得到，但不能覆盖重写，只能定义重名方法。

```java
public class Parent {

    public static void test(){
        System.out.println("parent");
    }
}

public class Child extends Parent {

    public static void main(String[] args) {
        Child child = new Child();
        Parent parent = new Parent();
        System.out.println(a);
        System.out.println(parent.a);
        System.out.println(child.a);
        System.out.println(Child.a);

        test();	// 此时调用的是父类，输出parent
        Parent.test();	// parent
    }
}
```

