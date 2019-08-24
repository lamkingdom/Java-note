# 内部类

## 成员内部类

### 概念

定义在外部类之内的成员类。

### 格式

```java
public class Outer {

    public class Inner {

    }
}
```

### 特点

内部类访问外部类的全成员（包含private），而外部类如果要访问内部类，必须先在类中实例化内部类。

```java
public class Outer {

    private String name = "lqd";
    private int age = 22;

    public void outMenthod() {
        System.out.println("out method");
    }

    // 外部类访问内部类
    public void outToIn() {

        Inner n = new Inner();
        System.out.println(n.getAge());
    }

    public class Inner {

        private int age = 21;

        // 内部类访问外部类
        public void outPrint() {

            int age = 20;

            // 访问成员变量
            System.out.println(name);    // lqd

            // 访问成员方法
            outMenthod();

            // 如果变量重名，输出顺序为就近原则（局部 -> 内部类 -> 外部类）
            System.out.println("method age is = " + age);            // 20
            System.out.println("Inner age is = " + this.age);        // 21
            System.out.println("Outter age is = " + Outer.this.age); // 22
        }

        public int getAge() {
            return age;
        }
    }

    public static void main(String[] args) {

        Outer.Inner inner = new Outer().new Inner();
        inner.outPrint();
    }
}
```



### 实例化格式

外部类.内部类 对象名称 = new 外部类().new 内部类();

也可以使用外部类里去实例化内部类，使用时只需实例化外部类。

更常见的格式是:

```java
Outter out = new Outter();
Outter.Inner inner = out.new Inner();
```

### 类修饰符

外部类的类修饰符仅有`public`或者`default`;

而内部类的类修饰符四种都可以。（内部类其实同成员变量一样，所以可以拥有所有权限的修饰符）

## 局部内部类

### 概念

定义在方法里的类。

### 格式

```java
public class MethodInner {


    public void method() {
        class Inner {
        }
    }
}
```

### 特点

同局部变量一样，不能拥有权限修饰符及static

```java
public class MethodInner {


    public void method() {

        // 同局部变量一样，不能拥有权限修饰符及static
        class Inner {

            public void innerMethod() {
                System.out.println("inner method");
            }
        }

        // 使用方法
        Inner inner = new Inner();
        inner.innerMethod();
    }


    public static void main(String[] args) {

        MethodInner methodInner = new MethodInner();
        methodInner.method();
    }
}
```

### 实例化格式

仅能在局部内进行普通实例化

### 类修饰符

不能拥有修饰符及`static`关键字。

## 匿名内部类

### 概念

匿名内部类是只能实例化一次的局部内部类。

### 格式

```java
public interface AnonymousInner {

    public abstract void method();
}

public class AnonymousInnerTest {

    public static void main(String[] args) {

        AnonymousInner anonymousInner = new AnonymousInner() {
            @Override
            public void method() {
                System.out.println("inner method");
            }
        };
    }
}
```

### 特点

```java
public class AnonymousInnerTest {

    public static void main(String[] args) {

        // AnonymousInner是个接口
        // 实际上匿名内部类也是局部内部类的一种
        // 匿名内部类有且只能实例化一次
        // 如果是匿名内部类的匿名对象有且只能使用一次
        AnonymousInner anonymousInner = new AnonymousInner() {
            @Override
            public void method() {
                System.out.println("inner method");
            }
        };

        new AnonymousInner() {
            @Override
            public void method() {
                System.out.println("inner method");
            }
        }.method();
    }
}
```

### 实例化格式

接口 对象名称 = new 接口(){ // 接口实现方法 }

### 类修饰符

同局部内部类

## 静态内部类

### 概念

定义在类里的静态类。

### 格式

```java
public class StaticInner {
    
    static class Inner {
        
    }
}
```

### 特点

静态内部类是不需要依赖于外部类的，这点和类的静态成员属性有点类似，并且它不能使用外部类的非static成员变量或者方法，这点很好理解，因为在没有外部类的对象的情况下，可以创建静态内部类的对象，如果允许访问外部类的非static成员就会产生矛盾，因为外部类的非static成员必须依附于具体的对象。

```java
public class StaticInner {

    private int num = 1;
    private static int age = 10;

    static class Inner {

        public Inner() {
            System.out.println(age);
            //不能使用非static变量
        }
    }

    public static void main(String[] args) {

        // 实例化格式
        StaticInner.Inner inner = new StaticInner.Inner();
    }
}
```

### 实例化格式

外部类.内部类 对象名称 = new 外部类内部类();

### 类修饰符

`static`默认全部可以。

