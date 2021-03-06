#JVM内存模型

## 运行时数据区

大概分为五块：

1. 程序计数器
2. JVM虚拟栈(stack)
3. 本地方法栈
4. 方法区
5. 堆(heap)

以下为JDK1.6与1.8版本的区别，先大致了解一下。

### 1.6版本模型

![](./pic/HotSpot1.6.png)

### 1.8版本模型

![](./pic/HotSpot1.8.png)

可以看到，虚拟机栈、本地方法栈以及程序计数器是线程私有的。而堆和方法区是线程共享的。

## 程序计数器

又叫PC计数器。当JVM执行字节码指令时需要使用字节码引擎进行解析，由于线程之间存在切换，所以需要记录当前线程执行到了哪条字节码指令。当下次获取CPU时继续从这条指令往下执行。

这样就很好的说明了，为什么PC计数器是线程私有的。

## 虚拟机栈

### 栈的基础结构

#### 数据结构

栈和堆是我们所熟知的，那么什么是栈呢？

其实栈是一种逻辑结构，是一种FILO（后进先出）的执行方式。类似叠罗汉，从无开始叠加（进栈），但是想要获得最底层的那块，需要从当前最上面那块（栈顶）开始移除（出栈）。

![](./pic/stack.jpeg)

#### 栈帧

栈的基本单位是栈帧，当一个方法入栈就创建一个栈帧。我们知道方法里有局部变量，这些局部变量的范围仅在该方法，故需要单独存储，所以栈帧里需要有局部变量表，除此之外还有操作数栈、动态链接以及方法出口信息。

#### 局部变量表

局部变量表是一个以字长为单位的数组。**局部变量表主要存放了编译器可知的各种数据类型**（boolean、byte、char、short、int、float、long、double）、**对象引用**（reference 类型，它不同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置）。

#### 操作数栈

和局部变量表结构一致，也是一个以字长为单位的数组。但逻辑结构是栈，当程序在方法内执行运算时，要先将数据压栈到操作数栈，需要用到时再取出，计算完成后再压栈。

### 异常

**Java 虚拟机栈会出现两种异常：StackOverFlowError 和 OutOfMemoryError。**

- **StackOverFlowError：** 若 Java 虚拟机栈的内存大小不允许动态扩展，那么当线程请求栈的深度超过当前 Java 虚拟机栈的最大深度的时候，就抛出 StackOverFlowError 异常。
- **OutOfMemoryError：** 若 Java 虚拟机栈的内存大小允许动态扩展，且当线程请求栈时内存用完了，无法再动态扩展了，此时抛出 OutOfMemoryError 异常。

## 本地方法栈

和虚拟机栈所发挥的作用非常相似，区别是： **虚拟机栈为虚拟机执行 Java 方法 （也就是字节码）服务，而本地方法栈则为虚拟机使用到的 Native 方法服务。** 在 HotSpot 虚拟机中和 Java 虚拟机栈合二为一。

本地方法被执行的时候，在本地方法栈也会创建一个栈帧，用于存放该本地方法的局部变量表、操作数栈、动态链接、出口信息。

方法执行完毕后相应的栈帧也会出栈并释放内存空间，也会出现 StackOverFlowError 和 OutOfMemoryError 两种异常。

## 堆

### 基本概念

堆是JVM所管理的内存最大的一块，也是垃圾收集器管理的主要区域，因此也被称作**GC 堆（Garbage Collected Heap）。**

### 结构

从垃圾回收的角度，由于现在收集器基本都采用分代垃圾收集算法，所以 Java 堆还可以细分为：

- 新生代（Young Generation）

- 老年代（Old Generation）

二者比例大概为1:2。

再细致一点有：Eden 空间、From Survivor、To Survivor 空间，永久代（1.8已移除）。**进一步划分的目的是更好地回收内存，或者更快地分配内存。**

### 参数设置

可以通过 -Xms 和 -Xmx 这两个虚拟机参数来指定一个程序的堆内存大小，第一个参数设置初始值，第二个参数设置最大值。

> java -Xms1M -Xmx2M HackTheJava

## 方法区

### 基本概念

方法区与 Java 堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。虽然 Java 虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做 **Non-Heap（非堆）**，目的应该是与 Java 堆区分开来。

### 关于永久代以及方法区

关于方法区和永久代的说法：

> 《Java虚拟机规范》只是规定了有方法区这么个概念和它的作用，并没有规定如何去实现它。那么，在不同的 JVM 上方法区的实现肯定是不同的了。 同时大多数用的JVM都是Sun公司的HotSpot。在HotSpot上把GC分代收集扩展至方法区，或者说使用永久代来实现方法区。因此，我们得到了结论，永久代是HotSpot的概念，方法区是Java虚拟机规范中的定义，是一种规范，而永久代是一种实现，一个是标准一个是实现。其他的虚拟机实现并没有永久带这一说法。
> https://blog.csdn.net/u011635492/article/details/81046174

### 参数设置

JDK1.8之前

```
-XX:PermSize=N //方法区 (永久代) 初始大小
-XX:MaxPermSize=N //方法区 (永久代) 最大大小,超过这个值将会抛出 OutOfMemoryError 异常:java.lang.OutOfMemoryError: PermGen
```

JDK 1.8 的时候，方法区（HotSpot 的永久代）被彻底移除了（JDK1.7 就已经开始了），取而代之是元空间，元空间使用的是直接内存。

```
-XX:MetaspaceSize=N //设置 Metaspace 的初始（和最小大小）
-XX:MaxMetaspaceSize=N //设置 Metaspace 的最大大小
```

### 运行时常量池

运行时常量池是方法区的一部分。Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有常量池信息（用于存放编译期生成的各种字面量和符号引用）

既然运行时常量池时方法区的一部分，自然受到方法区内存的限制，当常量池无法再申请到内存时会抛出 OutOfMemoryError 异常。

**JDK1.7 及之后版本的 JVM 已经将运行时常量池从方法区中移了出来，在 Java 堆（Heap）中开辟了一块区域存放运行时常量池。**

