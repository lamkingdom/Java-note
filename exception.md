---
typora-root-url: pictures
---

# 异常

## 异常类关系图

![](./pictures/Exception.jpg)

**异常：**异常是**导致程序中断运行**的一种指令流。java中所有的异常类都是Throwable的子类，其中异常主要分为两大块：一类是error异常，其父类为Error类，但由于这种异常是无法处理的，一般发生这种异常，JVM会选择终止程序，故不讨论。另一类则是exception类异常，其父类为Exception类，而它底下又分为两个子类，分别是**非运行时异常**以及**运行时异常**。



## 非运行时异常（又称检查异常）

即指在代码编译阶段，调用某个方法时，编译器要求必须对其强制捕获异常。常见的非运行时异常在JSON转换或者I/O流操作。

```java
JSONObject jsonObject = new JSONObject();
try {
    jsonObject.put("hello", "world");
} catch (JSONException e) {
			
    e.printStackTrace();
}
```



## 运行时异常(RuntimeException，非检查异常)

对于运行时异常，编译器不会强制要求进行异常捕获。但程序员必须清楚的知道哪些情况下哪些地方可能会产生异常，并对其进行相应的异常捕获，防止程序终止。常见的运行时异常有NullPointerException（空指针 )，内存溢出、数组越界等等。

```java
try {
    
    int i = 10 / 0; // ArithmeticException: / by zero
} catch (ArithmeticException e) {
    
    e.printStackTrace(); //catch 里对应的异常操作
} finally {
    
    System.out,println("finished");
}
```



## 异常处理

### 关键字

异常处理的关键字有以下几个

**try:** try无法单独使用，必须与catch块或finally连用。其功能是包含可能出现异常的代码块。若try块中发生异常，则异常行的后续代码不会执行。

**catch**: catch块可以有多个，且必须与try块连用，try块只有一个。其功能是捕获相应的类型的异常，并进行异常操作。异常捕捉上，只会按顺序匹配捕获一次。

**finally**：finally块是可选的，但只要出现，除了**特殊情况**就会执行，不单独出现。**切记在勿在finally块里return，会覆盖前面不管是try块还是catch块的return值。**特殊情况有以下几种：

1. 在finally语句块第一行发生了异常，如果发生在其他行，依然会执行。
2. 在前面的代码中使用了System.exit(0)退出程序。
3. 关闭CPU。
4. 程序所在的线程死亡。

以上三个关键字的组合为：

```java
try{
    
} catch (Exeception e) { 	// 可以有多个catch快
    
} 
---------------------------------------------------
try{
    
} catch (Exeception e) {  	// 可以有多个catch快
    
} finally {
    
}
---------------------------------------------------// 这种不常用，因为没有意义,但可以用
try{
    
} finally {
    
}
---------------------------------------------------// jdk8之后可以使用表达式关闭资源
try (InputStream inputStream = new ByteArrayInputStream("a".getBytes())) {
  int i = inputStream.read();
} catch (IOException e) {
  e.printStackTrace();
}
```

三个块执行顺序为：try—>catch—>finally。

**但注意，return前系统会现将返回值保存起来，然后再去执行finally块，执行完后返回return的值。这也是为什么finally中return会覆盖返回值的原因。也就意味着如果finally里面没有return,并不会影响try里面的返回值。**

**再来谈论下return值，如果return值是基本类型的话，那么假如在try块或catch块中去return，finally块中对这个值进行操作，并不会影响return的结果。因为return的是栈中的对象值，而基本数据类型中存放的就是数值。反观如果return的是一个对象，栈中存放的是这个对象的引用，所以如果在try块或catch块中去return，finally块中对这个对象进行操做，是会生效的。**==(注意：这里指的是finally块没有return的情况，如果有，则返回return的值）==

**另外，finally块中如果出现异常会覆盖前面的异常，故在finally块中一般只进行一些关闭操作，避免产生异常。**

**throw**：一般会用于程序出现某种逻辑时程序员主动抛出某种特定类型的异常。throw只会出现在方法体中，当方法在执行过程中遇到异常情况时，将异常信息封装为异常对象，然后throw出去。用法为 throw new Exception(); 

**throws**：throws出现在方法的声明中，表示该方法可能会抛出的异常，然后交给上层调用它的方法程序处理，允许throws后面跟着多个异常类型；

```java
public void test () throws Exception {}
```

如果上层没有处理，继续抛给上层，则最后由JVM进行捕获。



**在类继承的时候，方法覆盖时异常抛出声明的三点：**

1.父类的方法没有声明异常，子类重写的方法也不能声明；

2.父类的方法声明异常为exception1，子类重写的方法不能声明为exception1；

3.父类的方法如果声明的是检查时异常（运行时异常），则子类的方法只能声明检查时异常（运行时异常），而不能含有运行时异常（检查时异常）。

### 异常处理和设计的几个建议  

1.不要使用异常去控制程序，慎用异常。过多的异常处理代码会影响程序效率，如果能使用if语句和Boolean语句来进行判断的尽量使用。

2.不要使用空的catch块，这样做既没有意义，又将你的异常信息给隐藏了。

3.尽量减少检查异常的使用，因为检查异常通常try块里的东西并不多，而捕获起来又麻烦，不利于代码阅读，最好是将检查异常转变为非检查异常交由上层处理。

4.catch块的顺序要注意，上层类最好不要放在上面，因为上层类捕获到的异常可能是多种的，会导致下层异常无法去catch到异常，始终无法执行。

5.不要将异常信息返回给用户，而是集中起来将错误信息提示统一放在一个配置文件中管理。

6.避免多次日志信息记录同一个异常，尽量只在异常发生出打印相应日志。

7.异常处理尽量放到上层去处理，尽量将异常统一抛给上层调用者，再由上层调用者统一进行处理。

8.finally块中释放资源

**注意**：第一点其实如果异常没有发生时，效率并不会有影响。类会跟随一张异常表，里面存放四个字段，异常声明的开始行，异常声明的结束行，异常捕获后跳转到的代码计数器(PC)所指向的行数，异常类名称。

```java
Exception table:
   from   to  target type
     0     4     4   <Class java.lang.ArithmeticException>
```

原理：

如果在执行方法时有一个异常被抛出, JVM就会从异常表中按照条目所出现的顺序查找对应的条目. 如果异常抛出时PC计数器所指向的行数正好落在异常表中某一条目包含的范围内, 并且所抛出的异常正好是异常表中type列所指定的异常(或者所指定异常的子类), 那么JVM就会将PC计数器指向Target偏移量所指向的地址, (进入catch块)继续执行.

 

如果没有在异常表中找到异常, JVM就会将当前栈帧弹出并重新抛出这个异常. 当JVM弹出当前栈帧的时候, 它就会中止当前方法的执行, 返回到调用当前方法的外部方法中, 不过并不会像正常没有异常发生时那样继续执行外部方法, 而是在外部方法中抛出相同的异常, 这样将会导致JVM会在外部方法中重复查询异常表并处理异常的过程.

## 自定义异常类

```java
public class MyException extends Exception{
    
}
```

