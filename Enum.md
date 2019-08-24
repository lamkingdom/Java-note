# 枚举类

## 基本用法

创建一个简单的枚举类

```java
public enum Enmu {
    
    Sun,Mon,Tues,Wed,THU,FRI,STA;           //必须定义在类的第一行
}
```



上述类的实现过程及结果



```java
public class EnumTest{

	public static void main(String[] args) {
		
		
		System.out.println(Enum.Mon);              //相当于调用toString()方法
		System.out.println(Enum.FRI);              //相当于调用toString()方法
	}
}

//输出
Mon
FRI
```



**注：定义泛型时，采用全大写，单词直接用"_"隔开。**



## 相关方法

先展示一下各方法的功能（还是基于刚才定义的枚举类）

```java
public class EnumTest{

	public static void main(String[] args) {
		
		System.out.println(Enum.FRI);                      // = toString()
		System.out.println(Enum.FRI.name());               // = toString()
		System.out.println(Enum.FRI.toString());
		System.out.println(Enum.FRI.ordinal());            // 输出在枚举类定义的位置，从0开始，类似数组下标
		System.out.println(Enum.FRI.compareTo(Enum.Mon));  // 5 - 1 = 4
		for (Enum item : Enum.values()) {
			
			System.out.println(item);
		}
	}
}

//输出
FRI
FRI
FRI
5
4
Sun
Mon
Tues
Wed
THU
FRI
STA
```



### name()

```java
它和toString返回值一样，二者在源码中都是直接 return name；即定义的枚举类的值。
```



### toString()

```java
protected Enum(String name, int ordinal) {
        this.name = name;
        this.ordinal = ordinal;
}

public String toString() {
        return name;
}

//我们所定义的枚举类其实隐藏继承Enum类，即我们在加载我们的类时会去先调用父类的构造函数，初始化参数。
```



### ordinal()

```java
默认情况下，枚举类会给每个枚举变量一个次序，从0开始，类似数组下标。
```



### getDeclaringClass()

```java
public final Class<E> getDeclaringClass() {
        Class<?> clazz = getClass();
        Class<?> zuper = clazz.getSuperclass();
        return (zuper == Enum.class) ? (Class<E>)clazz : (Class<E>)zuper;
}

从源码能看出getDeclaringClass()在无实现接口的情况下是返回正常的类，但在接口实现类中返回的是接口的父类
```



### compareTo()

```java
public final int compareTo(E o) {
        Enum<?> other = (Enum<?>)o;
        Enum<E> self = this;
        if (self.getClass() != other.getClass() && // optimization
            self.getDeclaringClass() != other.getDeclaringClass())
            throw new ClassCastException();
        return self.ordinal - other.ordinal;
}

从源码可以看出，枚举类必须传入对象而不是基本数据类型（泛型的原因）。功能是返回二者位置差
```



### values()

```java
返回Enum[]，方便循环。
```



### valueOf()

```java
它的作用是传来一个字符串，然后将它转变为对应的枚举变量。前提是你传的字符串和定义枚举变量的字符串一抹一样，区分大小写。如果你传了一个不存在的字符串，那么会抛出异常。
```



## 进阶用法

### 添加方法及重载构造函数

我们知道枚举类是无法实例化的，因为它的构造函数默认是private，也只能用private来修饰。但实际上我们可以在类里去重载默认的无参构造，并声明对应的参数。然后通过定义枚举参数来实例化，最后方法调用到该枚举类就可以得到我们拓展的内容了。

```java
public enum EnumSuper {
    
    MON(1, "Monday"),TUE(2, "Tuesday");   //必须在第一行定义 
    
    private int value;        //定义变量用于自定义内容
    private String completeName;
    
    private EnumSuper (int value, String completeName) {    //就算不指定私有域默认也是私有的
        
        this.value = value;
        this.completeName = completeName;
    }
    
    public int getValue() {
		return value;
	}
	
	public String getCompleteName() {
		return completeName;
	}
}
```



反编译得到其源码

```java
public final class org.study.enumuse.EnumSuper extends java.lang.Enum<org.study.enumuse.EnumSuper> {
  public static final org.study.enumuse.EnumSuper MON;

  public static final org.study.enumuse.EnumSuper TUE;

```



可以看到，其实枚举类的声明方式只是一种简写，它其实已经继承了java枚举类的基类，所以无法再继承自其它类，同时它使用final修饰，无法被继承。

我们声明的枚举参数也是用static final声明的。



### 添加抽象方法

我们知道，含有抽象方法的类一定是抽象类。但我们也从上面的反编译看到了，我们所定义的枚举类是被final修饰的不可继承类。此时由于枚举类会定义静态枚举参数，会直接加载实例化所在类的对象，所以抽象方法就会要求实现在实例化的过程中，具体过程及写法如下：

```java
public enum EnumSuper {
 

	MON(1, "Monday"){                //我们知道枚举参数加载时即实例化，所以要实现抽象方法
		                             //写法就是{ 抽象方法 }
		@Override
		public void print() {
			
			System.out.println("1");
		}
	}, 
	TUE(2, "Tuesday"){
		
		@Override
		public void print() {
			
			System.out.println("2");
		}
	};
	
	private int value;
	private String completeName;
	
	EnumSuper (int value, String completeName) {      //默认private，也只能是private
		
		this.value = value;
		this.completeName = completeName;
	}
	
	public int getValue() {
		return value;
	}
	
	public String getCompleteName() {
		return completeName;
	}
	
	public abstract void print();           //定义一个抽象方法
}
```



反编译：

```java
public abstract class org.study.enumuse.EnumSuper extends    java.lang.Enum<org.study.enumuse.EnumSuper> {  //可以看到前面多了一个abstract关键字修饰
  public static final org.study.enumuse.EnumSuper MON;

  public static final org.study.enumuse.EnumSuper TUE;
    ......
}
```



### Switch

```java
EnumSuper e;
e = EnumSuper.MON;
switch (e) {
		
		case MON:
			System.out.println("Today is Monday, work.");
			break;
		case TUE:
			System.out.println("Today is Tuesday, work.");
			break;
			default:
				System.out.println("Today is Weekends, sleep.");
				break;
}

//输出
Today is Monday, work.
```



### 接口

```java
public enum InterfaceCase implements Food {
   
	RED("红色"),
	GREEN("绿色"),
	BLUE("蓝色");
	
	private String colorName;
	
    private InterfaceCase(String colorName) {
		
    	this.colorName = colorName;
	}
	
	@Override
	public void action() {
		
		System.out.println("chose color :" + colorName);
	}

	public static void main(String[] args) {
		
		InterfaceCase.GREEN.action();
	}
}
```



### 接口组织枚举

```java
public interface Food {

	enum Coffee implements Food {
		
		BLACK_COFFEE, DECAF_COFFEE, LATTE, CAPPUCCINO;

		@Override
		public void action() {
			
			System.out.println("drink coffee");
		}
	}
	
	enum Dessert implements Food {
		
		FRUIT, CAKE, GELATO;
		
		@Override
		public void action() {
			
			System.out.println("eat dessert");
		}
	}
	
	void action ();
}

public class EnumTest{

	public static void main(String[] args) {
        Food f = Coffee.BLACK_COFFEE;
		f.action();
}
    
//输出
drink coffee
```



其实也就是枚举类和接口的组合使用，一般都用于枚举区分。

## 应用场景

1.需要定义一组有关系常量

2.单例实现

```java
public enum  SingletonEnum {
    INSTANCE;
    private String name;
    public String getName(){
        return name;
    }
    public void setName(String name){
        this.name = name;
    }
}
```



非常简单，实现其实只需要一句话。完美的利用枚举的特性实现。