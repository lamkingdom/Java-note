# String

`String`是一个特殊的引用类型，它的地位在JVM中跟基本数据类型不相上下。

## String的本质

我们都知道`String`是个不可变对象。其结构如下：

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
    
    ……
 }
```



​	可以看到`String`底层其实是使用一个`final char[]`来存储字符内容。由于声明了`final`关键字并且使用`private`关键字修饰且不提供给外部`set`方法，使得这个成员变量不可变。并且`String`类也是被`final`所修饰，故不能被继承，则不存在方法的重写。



### 不可变的好处

>**1. 可以缓存 hash 值**
>
>因为 `String` 的 `hash `值经常被使用，例如` String `用做` HashMap` 的 key。不可变的特性可以使得` hash `值也不可变，因此只需要进行一次计算。
>
>**2. String Pool 的需要**
>
>如果一个 `String` 对象已经被创建过了，那么就会从 String Pool 中取得引用。只有 `String `是不可变的，才可能使用 String Pool。
>
>![](./pictures/Stringfinal.jpg)
>
>**3. 安全性**
>
>`String` 经常作为参数，`String` 不可变性可以保证参数不可变。例如在作为网络连接参数的情况下如果 `String` 是可变的，那么在网络连接过程中，`String `被改变，改变 `String` 对象的那一方以为现在连接的是其它主机，而实际情况却不一定是。
>
>**4. 线程安全**
>
>`String` 不可变性天生具备线程安全，可以在多个线程中安全地使用。



## 创建String对象

一般我们常用的创建String的办法有两种，一种是字符串字面量直接创建，另一种是使用`new`关键字创建。

第一种创建形式如下：

```java
private String str = "abc";
```

它直接在`String Pool`（字符串常量池）中创建，JDK1.7及以后把`String Pool`也放到了堆中，这使得`intern()`方法与之前有了一定的不同。

第二种创建形式如下：

```java
private String str = new String("abc");

// 实际上等同于
String str1 = "abc";  		  		// 常量池创建
String str2 = new String(str1);		// 堆中创建
```

经常被问及这种方法创建了几个对象，实际上得先判断常量池中是否已经创建了`"abc"`字符串常量，如果没有则是两个，如果有则只创建了`str2`对象。



还有一种创建方法其实比较特别，但其实本质上是上面两种。

```java
private String str1 = "a" + "b";							// (1)
private String str2 = str1 + "c"							// (2)
private final String str3 = "ab";							// (3)
private final String method = getStr();						// 假设getStr()返回"ab"
private String str4 = str3 + "c";	
private String str5 = method + "c";
private String str6 = new String("a") + new String("b");	// (4)


// 第一处的"+"实际上在编译的时候会自动转换为以下形式,因为此时都是字面量常量
// 实际上这里只在常量池中创建了"ab"这个常量对象，"a"和"b"并没有被创建,一会验证
private String str1 = "ab";

// 第二处由于str1是个变量所以"+"运算符对String来说其实等同于StringBuffer.append()
// 而append()方法返回的是new String();
// 所以以下代码输出的是false
System.out.println(str2 == "abc");

// 但是如果是final关键字修饰的String字面常量（注意是字面常量）,则在编译时会自动将str3转为"ab"
// 像第二处实际上就因为是变量所以无法在编译时确定
// 只有在编译期间能确切知道final变量值的情况下，编译器才会进行这样的优化
// 所以实际上str4 = "ab" + "c",故以下输出true
System.out.println(str4 == "abc");
System.out.println(str5 == "abc");		//false


// 第四处实际上创建了常量池中的"a"和"b",堆中的两个对象"a","b",以及最后组合的对象"ab"也在堆中
// 但"ab"并不在常量池中，一会验证
```



## String的方法

​	首先很容易发现返回值是`String`的方法最后都是重新`new String()`返回了一个新的`String`对象，例如以下方法

```java
    public String substring(int beginIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        int subLen = value.length - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
        return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
    }


    public String concat(String str) {
        int otherLen = str.length();
        if (otherLen == 0) {
            return this;
        }
        int len = value.length;
        char buf[] = Arrays.copyOf(value, len + otherLen);
        str.getChars(buf, len);
        return new String(buf, true);
    }

	//..........
```

​	

​	具体的不展开来说，但这里重点讲`inter()`方法。

### String.intern()

```java
	public native String intern();
```



>`String#intern`方法中看到，这个方法是一个 native 的方法，但注释写的非常明了。“如果常量池中存在当前字符串, 就会直接返回当前字符串. 如果常量池中没有此字符串, 会将此字符串放入常量池中后, 再返回”。

```java
// JDK1.6及以前与JDK1.7之后
```

```java
public static void main(String[] args) {
    String s = new String("1"); // 生成两个对象，一个在heap，一个在常量池，s指向的是heap里的对象
    s.intern();	// 此时常量池已经存在“1”
    String s2 = "1";	// s2指向常量池
    System.out.println(s == s2);

    String s3 = new String("1") + new String("1");// 此时常量池存在一个1，但不存在11，s3指向heap中的11
    s3.intern();	// 这里是在常量池生成一个“11”，而这个“11”的引用为s3的引用（JDK1.7）
    String s4 = "11";	// 此时常量池的“11”都是指向s3
    System.out.println(s3 == s4);
}
// 1.6输出
false false
// 1.7输出
false true
```

1.6中常量池是存在Perm区中的，Perm 区和正常的 JAVA Heap 区域是完全分开的。上面说过如果是使用引号声明的字符串都是会直接在字符串常量池中生成，而 new 出来的 String 对象是放在 JAVA Heap 区域。所以拿一个 JAVA Heap 区域的对象地址和字符串常量池的对象地址进行比较肯定是不相同的，即使调用`String.intern`方法也是没有任何关系的。

而1.7，将常量池移到了Heap中，所以是可以比较的。

> https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html

### length()

获取字符串长度。

```java
		String str = "abc";
        System.out.println(str.length());	// 3
```

**拓展：**

- 字符串获取长度：length()
- 数组获取长度：.length
- 集合获取长度：size()

## 常见题目

#### 字符串相加

```java
int a = 1, b = 2, c =3, d = 4;
System.out.println(a + b + "" + c + d);		// 334

byte b1 = 65;
System.out.println(b1 + 1 + "" + 2);		// 662
```

任何字符与字符串相加都会得到一个字符串，但在字符串之前的运算按原类型运算。

## StringBuilder

### 为什么要使用StringBuilder

​	我们都知道如果使用多个`String`对象相加，会产生许多多余的对象，但由于`String`的不可变性，这又是无可避免的。为了解决在需要操作多个字符串时产生大量的无效对象，这里我们就需使用`StringBuilder`类，注意`StringBuilder`是线程不安全的。

### 类源码

```java
public final class StringBuilder
    extends AbstractStringBuilder					// 继承自AbstractStringBuilder
    implements java.io.Serializable, CharSequence	// 序列化标识，字符序列标识
{
	// StringBuilder构造器
	// 无参构造器，会调用父类的构造器，生成长度为16的char[]数组存放字符
	public StringBuilder() {
        super(16);
    }
    
    // 给定初始容量构造器
    // 因为默认字符数组容量默认是预留长度只有16，不够时会在内部进行扩容
    // 如果能提前预知大概的长度，则可以提高效率，减少数组扩容的损耗
    // 但一般其实很难去预知大概的长度
    public StringBuilder(int capacity) {
        super(capacity);
    }
    
    // 初始化容量为传入的字符串长度 + 16
    // 将字符串内容添加到字符数组中
    public StringBuilder(String str) {
        super(str.length() + 16);
        append(str);
    }
    
    // 类似上面的构造器，传入的是CharSequence的实现类
    public StringBuilder(CharSequence seq) {
        this(seq.length() + 16);
        append(seq);
    }
}
```

我们再深入的了解一下它的父类`AbstractStringBuilder`

```java
// 首先它是一个包级访问权限（java.lang包下）
// 其次它实现了Appendable和CharSequence接口
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    
    // 成员变量
    char[] value;	// 字符数组，对比String中的字符数组，它并不是final对象，证明可以引用其他数组
    int count;		// 已使用的数组容量，也就是当前内容的长度，即.length()方法的实际返回值
    
    // 构造器
    // 无参构造器
    AbstractStringBuilder() {
    }
    
    // 指定容量的构造器
    AbstractStringBuilder(int capacity) {
        value = new char[capacity];	// 直接指定数组长度，而非默认的16
    }
    
    // 成员方法
    // 返回数组使用长度，即里面字符串内容
    public int length() {
        return count;
    }
    
    // 返回数组容量
    public int capacity() {
        return value.length;
    }
    
    // 类似于集合中判断是否需要进行扩容
    public void ensureCapacity(int minimumCapacity) {
        if (minimumCapacity > 0)
            ensureCapacityInternal(minimumCapacity);
    }
    
    private void ensureCapacityInternal(int minimumCapacity) {
        // overflow-conscious code
        // 当前最小容量 > 数组长度则需要进入扩容
        if (minimumCapacity - value.length > 0) {
            // Arrays.copyOf()方法见下
            value = Arrays.copyOf(value,
                    newCapacity(minimumCapacity));
        }
    }
    
    // 计算出新容量
    private int newCapacity(int minCapacity) {
        // overflow-conscious code
        // 新容量为数组长度的2倍 + 2
        int newCapacity = (value.length << 1) + 2;
        // 防止位运算变成负数
        if (newCapacity - minCapacity < 0) {
            newCapacity = minCapacity;
        }
        // MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8
        // hugeCapacity()让minCapacity与MAX_ARRAY_SIZE对比大小（int范围内）
        return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
            ? hugeCapacity(minCapacity)
            : newCapacity;
    }
}

// Arrays.copyOf()
public static char[] copyOf(char[] original, int newLength) {
    	// 创建新数组
        char[] copy = new char[newLength];
    	// 复制旧数组original的元素到新数组copy
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
}
```



### 常用的方法

#### append()

`StringBuilder`中`append()`方法重载了很多次，主要参数种类有`String`、`Object`以及基本类型。实际上方法的实现最后还是调用了对应父类中不同的`append()`方法。而`append()`方法实际上到最后都是去操作或生成新数组，并把内容追加到数组上。

```java
public StringBuilder append(StringBuffer sb) {
        super.append(sb);
        return this;
}
@Override
public StringBuilder append(Object obj) {
    return append(String.valueOf(obj));
}

@Override
public StringBuilder append(String str) {
    super.append(str);
    return this;
}
```

```java
public AbstractStringBuilder append(String str) {
        if (str == null)
            return appendNull();
        int len = str.length();
    	// 检查数组容量是否够用，长度为旧 + str长度
    	// 如果不够，则创建一个新数组，并把旧数组的内容cory到新数组（注意：这里并没有追加新内容str）
        ensureCapacityInternal(count + len);
    	// 追加新内容str到数组中
        str.getChars(0, len, value, count);
        count += len;
        return this;
}

// 其他重载方法对应AbstractStringBuilder相应的append()
// ....
```

但最后都是调用`String.getChar()`

```java
public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {
        if (srcBegin < 0) {
            throw new StringIndexOutOfBoundsException(srcBegin);
        }
        if (srcEnd > value.length) {
            throw new StringIndexOutOfBoundsException(srcEnd);
        }
        if (srcBegin > srcEnd) {
            throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
        }
    	// 拷贝内容到数组
        System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
 }
```

#### toString()

```java
// StringBuilder --> String
@Override
public String toString() {
    // Create a copy, don't share the array
    return new String(value, 0, count);
}

// String --> StringBuilder
String str = "abc";
StringBuilder sb = new StringBuilder(str);
```

#### trimToSize()

将数组长度缩减至与内容等长。

```java
StringBuilder stringBuilder =  new StringBuilder("a");
System.out.println(stringBuilder.length());		// 1
System.out.println(stringBuilder.capacity());	// 16
stringBuilder.trimToSize();
System.out.println(stringBuilder.length());		// 1
System.out.println(stringBuilder.capacity());	// 1

// 源码
public void trimToSize() {
    if (count < value.length) {
        value = Arrays.copyOf(value, count);
    }
}
```

#### setLength()

将内容长度设置为指定长度。

```java
StringBuilder stringBuilder =  new StringBuilder("abcdefg");
//设置长度为小于当前长度
stringBuilder.setLength(5);
System.out.println(stringBuilder.toString());	// "abcde"
System.out.println(stringBuilder.length());		// 5
System.out.println(stringBuilder.capacity());	// 23 = 7("abcdefg"长度) + 16

// 设置为大于当前内容长度
// 实际上多出来的填充了char的'\0'字符，但不显示
stringBuilder.setLength(6);
System.out.println(stringBuilder.toString());	// "abcde"
System.out.println(stringBuilder.length());		// 6
System.out.println(stringBuilder.capacity());	// 23 = 7("abcdefg"长度) + 16

// 注意，此处setLength()参数 < 23 * 2 + 2才会按规则扩容，这就是扩容机制
// 否则直接使用参数 = count = value.length
stringBuilder.setLength(25);
System.out.println(stringBuilder.length());		// 25
System.out.println(stringBuilder.capacity());	// 48 = 23 * 2 + 2

// 此时容量为48，按照上面说法，如果设置超过100的长度
// 则使用设置的长度，如下
// count = value.length
stringBuilder.setLength(101);
System.out.println(stringBuilder.length());		// 101
System.out.println(stringBuilder.capacity())	// 101

// 源码
public void setLength(int newLength) {
        if (newLength < 0)
            throw new StringIndexOutOfBoundsException(newLength);
        ensureCapacityInternal(newLength);

    	// 填充'\0'
        if (count < newLength) {
            Arrays.fill(value, count, newLength, '\0');
        }

        count = newLength;
}
```



### StringBuffer

`StringBuffer`实际上的整体类似于`StringBuilder`，但由于它是线程安全的，所以在涉及并发的方法上加了`synchronized`关键字保证同步。所以效率会比`StringBuilder`来得低。