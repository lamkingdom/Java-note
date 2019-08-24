# ThreadLocal

## 理解

`ThreadLocal`主要的功能是提供一个方便的方式，根据不同的线程存放一些不同的特征属性（例如session），可以方便的在线程中进行存取。

注意：`ThreadLocal`并不是用来解决共享对象的对线程访问竞争问题的，如果将一个对象在不同线程的里添加到`ThreadLocal`，实际上是线程不安全的，也并不是`ThreadLocal`正确的用法。

## 解析

下面来对`ThreadLocal`类进行解析

### Demo

```java
public class Person {

    private String name;
    private Integer age;

    public Person(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    // get
    // set

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

public class ThreadLocalDemo {

    // 创建一个存放String对象的ThreadLocal对象
    private static final ThreadLocal<String> stringThreadLocal = new InheritableThreadLocal<>();
    // 创建一个存放Person对象的ThreadLocal对象
    private static final ThreadLocal<Person> personThreadLocal = new InheritableThreadLocal<>();
    public static void main(String[] args) {

        // 新建一个Person对象
        Person person = new Person("invain", 10);
        // 将main线程的线程名称放入stringThreadLocal中
        stringThreadLocal.set(Thread.currentThread().getName());
        // 将person对象放入stringThreadLocal中，对应main线程
        personThreadLocal.set(person);

        // 新建一个匿名线程
        new Thread(new Runnable() {
            @Override
            public void run() {
                // 将thread0线程的线程名称放入stringThreadLocal中
                stringThreadLocal.set(Thread.currentThread().getName());
                // 将person对象放入stringThreadLocal中，对应thread0线程
                personThreadLocal.set(person);
                try {
                    System.out.println("thread0线程未修改值之前: " + stringThreadLocal.get());
                    System.out.println("thread0线程未修改值之前: " + personThreadLocal.get());
                    Thread.sleep(5000);
                    System.out.println("thread0线程未修改值之后: " + stringThreadLocal.get());
                    System.out.println("thread0线程未修改值之前: " + personThreadLocal.get());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();

        System.out.println("main线程未修改值之前: " + stringThreadLocal.get());
        System.out.println("main线程未修改值之前: " + personThreadLocal.get());
        // 修改person对象
        person.setName("lamp");
        person.setAge(20);
        System.out.println("main线程未修改值之后: " + stringThreadLocal.get());
        System.out.println("main线程未修改值之后: " + personThreadLocal.get());
    }
}

// 输出
main线程未修改值之前: main
main线程未修改值之前: Person{name='invain', age=10}
thread0线程未修改值之前: Thread-0
main线程未修改值之后: main
thread0线程未修改值之前: Person{name='lamp', age=20}
main线程未修改值之后: Person{name='lamp', age=20}
thread0线程未修改值之后: Thread-0
thread0线程未修改值之前: Person{name='lamp', age=20}
```

从这个Demo可以证明，`ThreadLocal`并不是用来解决共享对象的对线程访问竞争问题的，如果将一个对象在不同线程的里添加到`ThreadLocal`，实际上是线程不安全的，也并不是ThreadLocal正确的用法。因为`ThreadLocal`类是将对象的地址值传入`ThreadLocal`对象中存储起来，所以直接操作对象的内容，那么所有的线程的`ThreadLocal`类中，只要含有同一个对象，就会改变。

### ThreadLocal源码分析

主要介绍几个常用的API

```java
public class ThreadLocal<T> {
    
    // initialValue()
    // 初始化时使用
    // protected修饰，可以进行重写，默认返回null
    protected T initialValue() {
        return null;
    }
    
    // get()
    // 返回当前线程存储的特殊属性的值
    public T get() {
        // 获取当前线程
        Thread t = Thread.currentThread();
        // 获取当前线程的ThreadLocalMap对象（注意：每个线程其实都有一个ThreadLocalMap属性）
        ThreadLocalMap map = getMap(t);
        // 如果该线程的ThreadLocalMap属性不为null
        if (map != null) {
            // 通过ThreadLocal对象获取存储在Map中的值，如果不为空则返回value
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        // 否则调用setInitialValue()初始化
        return setInitialValue();
	}
    
    // getMap()
    // ThreadLocal.ThreadLocalMap threadLocals = null; 这是定义在Thread类中的成员变量
    // ThreadLocalMap是ThreadLocal的内部类
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    
    // setInitialValue()
    private T setInitialValue() {
        // 如果没有重写则value = null
        T value = initialValue();
        // 获取当前线程
        Thread t = Thread.currentThread();
        // 获取当前线程的ThreadLocalMap对象
        ThreadLocalMap map = getMap(t);
        // 如果map不为空，则set(ThreadLocal对象, null); 如果重写了initialValue则不为null
        if (map != null)
            map.set(this, value);
        else
            // 否则创建map，并设值
            createMap(t, value);
        return value;
    }
    
    // createMap()
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    
    // set()
    // 将特殊属性的值存储到以ThreadLocal为key的map中
    // 与setInitialValue()方法类似,少了返回值以及初始化value
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            // 调用ThreadLocalMap内部的set方法
            map.set(this, value);
        else
            createMap(t, value);
    }
    
    // remove()
    public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             // 调用ThreadLocalMap内部的remove方法
             m.remove(this);
     }
}
```

可以看到`ThreadLocal`里方法的实现，大多是基于`ThreadLocalMap`，大致看一下里面的内容

```java
// ThreadLocalMap是ThreadLocal的一个静态内部类
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;
			// 以ThreadLocal对象为key,传入的值为value
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
    }
    
    private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                // 清理key为null的entry
                return getEntryAfterMiss(key, i, e);
    }
}
```

## 知识点及面试题

### 使用ThreadLocal为什么可能会造成内存泄露？

首先记住`ThreadLocalMap`的生命周期跟`Thread`一样长

因为`ThreadLocalMap`中的`Entry`继承了`WeakReference<>`，其中泛型类型就是`ThreadLocal<?>`，所以一旦主动将`ThreadLocal`对象释放之后，由于`Entry`中的key使用的是`ThreadLocal`对象，会导致无法访问value。而此时Thread --> ThreadLocalMap-->Entry-->Value，这条强引用链会导致Entry不会回收，Value也不会回收，但Entry中的Key却已经被回收的情况，造成内存泄漏。

但是JVM团队已经考虑到这样的情况，并做了一些措施来保证ThreadLocal**尽量**不会内存泄漏：在ThreadLocal的get()、set()、remove()方法调用的时候会清除掉线程ThreadLocalMap中所有Entry中Key为null的Value，并将整个Entry设置为null，利于下次内存回收。

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
}

// 在调用map.getEntry(this)时，内部会判断key是否为null，继续看map.getEntry(this)源码
private Entry getEntry(ThreadLocal key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

// 在getEntry方法中，如果Entry中的key发现是null，会继续调用getEntryAfterMiss(key, i, e)方法
private Entry getEntryAfterMiss(ThreadLocal key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}

// 注意k == null这里，继续调用了expungeStaleEntry(i)方法，expunge的意思是擦除，删除的意思
// expungeStaleEntry方法
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot（意思是，删除value，设置为null便于下次回收）
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

注意这里，将当前Entry删除后，会继续循环往下检查是否有key为null的节点，如果有则一并删除，防止内存泄漏。

但这样也并不能保证ThreadLocal不会发生内存泄漏，例如：

- 使用static的ThreadLocal，延长了ThreadLocal的生命周期，可能导致的内存泄漏。
- 分配使用了ThreadLocal又不再调用get()、set()、remove()方法，那么就会导致内存泄漏。

### 为什么使用弱引用？

从表面上看，发生内存泄漏，是因为Key使用了弱引用类型。但其实是因为整个Entry的key为null后，没有主动清除value导致。很多文章大多分析ThreadLocal使用了弱引用会导致内存泄漏，但为什么使用弱引用而不是强引用？

下面我们分两种情况讨论：

- key 使用强引用：引用的ThreadLocal的对象被回收了，但是ThreadLocalMap还持有ThreadLocal的强引用，如果没有手动删除，ThreadLocal不会被回收，导致Entry内存泄漏。
- key 使用弱引用：引用的ThreadLocal的对象被回收了，由于ThreadLocalMap持有ThreadLocal的弱引用，即使没有手动删除，ThreadLocal也会被回收。value在下一次ThreadLocalMap调用set,get，remove的时候会被清除。
   比较两种情况，我们可以发现：由于ThreadLocalMap的生命周期跟Thread一样长，如果都没有手动删除对应key，都会导致内存泄漏，但是使用弱引用可以多一层保障：弱引用ThreadLocal不会内存泄漏，对应的value在下一次ThreadLocalMap调用set,get,remove的时候会被清除。

因此，ThreadLocal内存泄漏的根源是：由于ThreadLocalMap的生命周期跟Thread一样长，如果没有手动删除对应key的value以及Entry就会导致内存泄漏，而不是因为弱引用。

### Hash冲突怎么解决

和HashMap的最大的不同在于，ThreadLocalMap结构非常简单，没有next引用，也就是说ThreadLocalMap中解决Hash冲突的方式并非链表的方式，而是采用线性探测的方式，所谓线性探测，就是根据初始key的hashcode值确定元素在table数组中的位置，如果发现这个位置上已经有其他key值的元素被占用，则利用固定的算法寻找一定步长的下个位置，依次判断，直至找到能够存放的位置。

ThreadLocalMap解决Hash冲突的方式就是简单的步长加1或减1，寻找下一个相邻的位置。

```java
/**
 * Increment i modulo len.
 */
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}

/**
 * Decrement i modulo len.
 */
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```

### 使用场景

Hibernate的session获取场景

```java
private static ThreadLocal < Connection > connectionHolder = new ThreadLocal < Connection > () {
    public Connection initialValue() {
        return DriverManager.getConnection(DB_URL);
    }
};

public static Connection getConnection() {
    return connectionHolder.get();
}
private static final ThreadLocal threadSession = new ThreadLocal();

public static Session getSession() throws InfrastructureException {
    Session s = (Session) threadSession.get();
    try {
        if (s == null) {
            s = getSessionFactory().openSession();
            threadSession.set(s);
        }
    } catch (HibernateException ex) {
        throw new InfrastructureException(ex);
    }
    return s;
} 
```

## 总结

每个线程中有一个`ThreadLocalMap`，里面存储着不同的Entry，以`ThreadLocal`为key，存储的特殊值为value。

使用完之后一定要主动调用`remove()`方法。

Threadlocal是什么？在堆内存中，每个线程对应一块工作内存，threadlocal就是工作内存的一小块内存。

 Threadlocal有什么用？threadlocal用于存取线程独享数据，提高访问效率。

## 推荐阅读

https://www.jianshu.com/p/a1cd61fa22da

https://www.jianshu.com/p/98b68c97df9b

https://www.cnblogs.com/chenkeyu/p/7623653.html