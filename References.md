# 引用

## 引入

Java中存在四种类型的引用，分别是：强引用，软引用，弱引用和虚引用。被GC回收的等级从低到高，一个强引用如果还被其他对象所使用，是永远不会被回收的，JVM宁可抛出OOM异常。

## 强引用

默认的引用方式就是强引用。

强引用被依赖时，不能被GC回收，如果当强引用对象占用的内存足够多时，JVM就会抛出OutOfMemory。

### 引用

```java
public class References {

    private String name;

    public References(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

public class ReferencesDemo {

    public static void main(String[] args) {

        // 强引用
        References strong = new References("strong");
        // GC回收方式
        strong = null;
    }
}
```

### 释放

必须用户主动释放，`strong = null`。

## 软引用

### 引用

```java
public class ReferencesDemo {

    public static void main(String[] args) {

        // 软引用
        References soft = new References("soft");
        SoftReference<References> softReference = new SoftReference<>(soft);
        System.out.println("对象强引用: " + soft);
        soft = null;
        System.out.println("软引用: " + softReference.get());
        // 未被GC回收时，还可以转为强引用
        soft = softReference.get();
        System.out.println("软引用 --> 强引用: " + soft);
        // 当内存不足时，GC会回收软引用的对象，前提是该对象已经不再被其他强引用引用
        // 反之，内存足够时，并不会去回收软引用对象
        soft = null;
        // 强制GC回收
        System.gc();
        System.out.println("GC回收后soft中的软引用: " + softReference.get());
    }
}

// 输出
对象强引用: cn.invain.references.References@610455d6
软引用: cn.invain.references.References@610455d6
软引用 --> 强引用: cn.invain.references.References@610455d6
GC回收后soft中的软引用: cn.invain.references.References@610455d6
```

### 释放

当内存不足时，且强引用已经删除。

### 场景

缓存数据。

## 弱引用

### 引用

```java
public class WeakReferencesDemo {
    public static void main(String[] args) {

        // 弱引用
        References weak = new References("weak");
        // 创建一个弱引用对象
        WeakReference<References> wf = new WeakReference<>(weak);
        // 清空强引用
        weak = null;
        System.out.println("GC回收前,weak对象的地址: " + wf.get());
        System.gc();
        System.out.println("GC回收后,weak对象的地址: " + wf.get());
    }
}

// 输出
GC回收前,weak对象的地址: cn.invain.references.References@610455d6
GC回收后,weak对象的地址: null
```

### 释放

一个引用**仅**被标记为弱引用时，不管GC内存是否充足，都会对其进行回收。如果还有弱引用或强引用存在，则不会被回收。

### 场景

操作或依附该对象，却不能影响其正常被回收时

## 虚引用

虚引用与软引用和弱引用有一个区别，它必须与引用队列（`ReferenceQueue`）一起使用，而另外两种则是可选的。

虚引用顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收。

### 引用

```java
public class PhantomReferenceDemo {

    public static void main(String[] args) {
		
        // 创建对象
        References phantom = new References("phantom");
        // 创建虚引用的必要条件
        ReferenceQueue queue = new ReferenceQueue();
        // 创建虚引用
        PhantomReference<References> phantomReference = new PhantomReference<>(phantom, queue);
        System.out.println(phantomReference.get());	// null
    }
}
```

`PhantomReference`类的源码

```java
public class PhantomReference<T> extends Reference<T> {

    // 无论get什么都返回null
    public T get() {
        return null;
    }

    // 构造函数，必须持有一个引用队列
    public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```

### 释放

当对象只有虚引用指向它时

### 场景

结合ReferenceQueue追踪对象是否已经被回收。

>当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会把这个虚引用加入到与之 关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

## ReferenceQueue

软引用及弱引用的构造器都有`ReferenceQueue`的重载。

```java
public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
}

public SoftReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
        this.timestamp = clock;
}
```

>ReferenceQueue主要是用于监听Reference所指向的对象是否已经被垃圾回收。
>
>当大量使用Reference时，虽然Reference指向的对象可能被回收了，但Reference本身也是个对象，所以也需要回收。这时就需要使用ReferenceQueue了。
>
>当SoftReference或WeakReference的get()加入ReferenceQueue或get()返回null时，仅是表明其指示的对象已经进入垃圾回收流程，此时对象不一定已经被垃圾回收。
>
>当PhantomReference加入ReferenceQueue时，则表明对象已经被回收。