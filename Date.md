# Date

## Date

### 基本信息

用于创建日期及时间对象，位于`Java.util`包。

### 实例化

```java
// 使用无参构造
// 获取一个英文格式的日期时间
// 例如：Fri Mar 01 22:48:24 CST 2019
Date date = new Date();

// 使用new Date(long date)构造函数，参数为毫秒数（以元历起算，北京地区 + 8:00）
Date mDate = new Date(1551451872082L);	// 输出 Fri Mar 01 22:51:12 CST 2019
```

### 毫秒与日期时间

#### 毫秒转日期

```java
// 获取当前毫秒数
Date date = new Date();
long timeMillis = date.getTime();

Date mDate = new Date(timeMillis);	// 输出 Fri Mar 01 22:51:12 CST 2019


// 实际上Date.getTime()方法调用的就是 System.currentTimeMillis()
// 源码如下
public Date() {
        this(System.currentTimeMillis());
}
```

#### 日期转毫秒

```java
// 获取Calendar实例
Calendar c = Calendar.getInstance();
// 设置对应的日期，可以选择YEAR和MONTH等等
c.set(Calendar.Year, 2019);

// 获取毫秒数
// 此方法实例化了一个Date()对象,因此我们如果要获取毫秒，只需要再调用一次Date的getTime()方法即可
// public final Date getTime() {
//      return new Date(getTimeInMillis());
// }
c.getTime().getTime();

// 上面这种方法还需要多创建一个对象，所以可以使用Calendar自带的方法
c.getTimeInMillis();
```

## DateFormat

### 基本信息

日期格式化抽象类，子类一般使用`SimpleDateFormat`。

### 实例化

```java
// 构造参数为日期格式
// 参数为日期模式，解析时必须传入对应格式的字符串，否则将解析失败 ParseException
// 注意，字母不可以更改，但连接字母的符号可以更改
// 例如 yyyy-MM-dd HH:mm:ss --> yyyy/MM/dd HH时mm分ss秒
SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
```



### format（格式化，日期 --> 文本）

```java
Date date = new Date();
SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
System.out.println(simpleDateFormat.format(date)); // 2019-03-01 23:11:30
```

### parse（解析，文本 --> 日期）

```java
// 字符串格式必须与构造模式一致
String dateStr = "2019-03-01 23:11:30";
SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

try {
	System.out.println(simpleDateFormat.parse(str));	// Fri Mar 01 23:18:06 CST 2019
} catch (ParseException e) {
    e.printStackTrace();
}
```



## Calendar(📅)

### 基本信息

`Calendar`译为日历。位于`java.util`包内，替换了许多`Date`类的方法，并且将所有时间可能用到的类封装成了静态成员变量，方便使用。

### 实例化

```java
// 直接采用获取实例的方法（实际上方法根据不同条件返回不同的实现类，一般返回的是GregorianCalendar）
Calendar c = Calendar.getInstance();

// 打印Calendar实例，输出的是一串信息，指代打印那一瞬间的时间信息
```

### 常用方法

#### get()

```java
// 获取Calendar实例里的信息
// 获取方式为实例.get(Calendar.对应字段常量)
Calendar c = Calendar.getInstance();

// 获取年份
c.get(Calendar.YEAR);	// 2019
// 获取月份
// 注意，这里采用的是西方的月份，月份值为（0 ～ 11）
c.get(Calendar.MONTH);	// 2(实际上此时对应东方的月份3月)
```



#### getTime()

```java
// 返回Calendar信息生成的一个Date对象
// 源码如下
public final Date getTime() {
        return new Date(getTimeInMillis());
}

// 使用
Calendar c = Calendar.getInstance();
c.getTime();	// Fri Mar 01 23:18:06 CST 2019
```



#### set()

```java
// 设置Calendar实例里的信息
Calendar c = Calendar.getInstance();

System.out.println(c.get(Calendar.YEAR));	// 2019
c.set(Calendar.YEAR, 2020);					// 2020

// 设置月份为12月
// 这里注意，后面的传入值为int型，可以为 > 11 也可以 < 0
// 但会自动往年上修改，比如输入13，则年份 + 1，月份改为0，同理负数
c.set(Calendar.YEAR, 11);			
```



#### add()

```java
// 增加或减少实例里的信息值
Calendar c = Calendar.getInstance();

System.out.println(instance.get(Calendar.YEAR));	// 2019
c.add(Calendar.YEAR, 1);
System.out.println(instance.get(Calendar.YEAR));	// 2020
```