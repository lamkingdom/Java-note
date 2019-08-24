# 包装类

## 概念

Java中提倡**万物皆对象**的思想，但此时矛盾就来了，基本类型并不能算是对象，故而有了各自的包装类。

## 基础类型对应的包装类

| 基本数据类型 | 包装类  |  父类  |
| :----------: | :-----: | :----: |
|     int      | Integer | Number |
|    short     |  Short  | Number |
|     long     |  Long   | Number |
|    float     |  Float  | Number |
|    double    | Double  | Number |
|     byte     |  Byte   | Number |
|   boolean    | Boolean | Object |
|     char     |  Char   | Object |

其中，Number是一个抽象类。

## 装箱与拆箱

### 概念

Java为每种基本数据类型都提供了对应的包装器类型。装箱就是将基本数据类型转换为包装器类型，拆箱就是将包装器类型转换为基本数据类型。

**注**：在 JavaSE5以上就支持自动装箱及自动拆箱，即用户不用手动去进行装箱和拆箱的操作。

### Code

```java
int num = 10;
Integer number1 = new Integer(num); //装箱举例1

Integer number2 = new Integer(10); 	//装箱举例2

Integer number3 = 10; 				//自动装箱	
----------------------------------------------------------
int i = number1.intValue();  //拆箱

int i = number3;             //自动拆箱
```



### 自动装拆箱底层原理

首先先了解两个方法

1.**valueOf():** 返回一个表示指定的**基本数据类型值**的**包装类**实例

2.**xxxValue():** 返回一个xxx**基本数据类型**。

不难看出第一个方法是在自动装箱时，自动调用的装箱方法；而第二个方法则是包装类自动拆箱时所需要调用的方法。



### 面试题

**1.下面代码会输出什么？**

```java
public class Main {
    public static void main(String[] args) {
         
        Integer i1 = 100;
        Integer i2 = 100;
        Integer i3 = 200;
        Integer i4 = 200;
         
        System.out.println(i1==i2);
        System.out.println(i3==i4);
    }
}
```

**结果**

```java
true
false
```

**解：**Java在初始化时会自动加载IntegerCache类的cache()方法，自动生成一个区间为[-128,127]区间的常量池，故

i1及i2在自动装箱时会生成两个Integer对象，但由于其数值在此区间内，系统默认直接引用常量池的值，故二者引用的是同一个对象。（非基本类型"=="比较的是地址值），而i3 和 i4不介于此区间，故在自动装箱时等于实例化了两个不同的Integer对象，其内存地址不想等，故比较结果为false。

**2.下面这段代码的输出结果是什么？**

```java
public class Main {
    
    public static void main (String[] args) {
        
        Double f1 = 100.0;
        Double f2 = 100.0;
        Double f3 = 200.0;
        Double f4 = 200.0;
        
        System.out.println(f1==f2);
        System.out.println(f3==f4);
    }
}
```

**结果**

```java
false
false
```

**解：**不同于int,double是浮点数。简单的说就是小数是无穷的，系统无法加载全部的小数作为常量，故不存在预先加载好的常量池。

**注：**

Integer、Short、Byte、Character、Long这几个类的valueOf方法的实现是类似的。

Double、Float的valueOf方法的实现是类似的。



**3.下面这段代码输出结果是什么：**

```java
public class Main {
    
    public static void main (String[] args) {
        
        Boolean b1 = true;
        Boolean b2 = true;
        Boolean b3 = false;
        Boolean b4 = false;
        
        System.out.println(b1==b2);
        System.out.println(b3==b4);
    }
}
```

**结果**

```java
true
true
```

**解：**原因是Boolean的valueOf代码又不一样了 (ヽﾐ ´∀｀ﾐノ＜)

```java
public static Boolean valueOf (boolean b) {
    
    return (b ? true : false)
}
```



**4.谈谈Integer i = new Integer(xxx)和Integer i =xxx;这两种方式的区别**

   主要有两点：

(1) 第一种不会触发自动装箱的过程；而第二种会。	

(2) 第二种在某种情况的效率优于第一种，个人理解因为第一种需要调构造器去赋值才进行valueOf()，第二种直   接使用静态方法valueOf()。（但这个效率并不是绝对的)



**5.下面程序的输出结果是什么？**

```java
public class Main {
    
    public static void main (String[] args) {
        
        Integer a = 1;
        Integer b = 2;
        Integer	c =	3;
        Integer d =	3;
        Integer e = 321;
        Integer	f = 321;
        Long g = 3L;
        Long h = 2L;
        
        System.out.println(c==d);
        System.out.println(e==f);
        System.out.println(c==(a+b));
        System.out.println(c.equals(a+b));
        System.out.println(g==(a+b));
        System.out.println(g.equals(a+b));
        System.out.println(g.equals(a+h));
    }
}
```

**结果**

```java
true
false
true 
true
true
false
true
```

**解：**

1,2没啥好说。

3.a+b进行运算时会自动拆箱，调用intValue()方法，返回int值；而Integer与int比较也会自动拆箱成int，int与int比较的是具体数值，故为true。

4.这里有两点要注意

**第一点：**我们都知道每个类如果没有继承父类，默认都会继承Object类。而Object类有equals这个默认方法，其比较的是地址的值，本质上与“==”一致，源码如下：

```java
public boolean equals (Object obj) {
    
    return (this == obj);
}
```

而Integer重写了它的equals方法，如下：

```java
public boolean equals(Object obj) {
    
    if (obj instanceof Integer) {
        
        return value == ((Integer)obj).intValue();
    }
    return false;
}
```

我们可以知道这里equal先判断是否为Integer对象，再比较的是两个对象的**值**。

**第二点：**根据重写的方法，可以知道其传入的是一个Object对象，而我们刚才说了(a+b)进行运算时会自动拆箱，得出一个int值，而将这个int传入equal()会自动向上转型为Integer，即自动装箱。然后才是上面的步骤。

5.(a+b)自动拆箱成int，Long与int比较自动拆箱为long，比较二者值。

6.(a+b)自动拆箱为int，装箱时为Integer，long值和int值不相等。

7.(a+b)自动拆箱为(int + long)，由于equal方法在装箱为Long（低精度+高精度=高精度)，equal比较值，相等。
