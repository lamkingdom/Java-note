# 容器

## 基本概念

Java集合的框架主要分为两大类，一类为Collection接口及其实现类，Queue、Set、List。另一类是以Map接口为主及其实现类，HashMap、SortedMap等等。



## 共通实现

### HashCode

#### 意义

hashCode()和equal()都是Object自带的方法，经常有人认为hashCode返回的就是地址，因此拿来比较两个对象。其实这样的想法是不正确的。

正常情况下我们如果要插入一个元素（不能重复），我们一般会构建一个循环利用equal去比较。事实上，当元素非常多的情况下，这种方法效率极低，这时就轮到hashCode()出场了。



#### 原理

**hashCode本身就是一个通过hash函数计算出来的数值**。实质上，对象调用这个方法，就是将对象的物理地址根据一定的算法散列成一个数值。



#### 实现

假设我们存储一个对象，就采用hashCode()根据这个对象的物理地址散列出一个hash值，存入hash表。当然了，也许不同的对象会散列出相同的hash值，这时我们就引出了另一个概念“桶”，我们将hash值相同的放在一个桶里，而每个hash值对应一个桶。这样当我们数据量够大时，需要查找数据只需要利用hashCode()方法得到hash值，拿着这个值（相当于数组下标，因为这里用的是f(x) = table[i])也就意味着时间复杂度为O(1)，而后我们找到这个桶，然后再使用equal方法获取我们想要的结果，这样效率是大大的提高了。



#### 作用

一般都是Hash类需要存储对象时，调用该方法获取散列后的地址存入hash表，供查询及插入时进行查重。



#### 定理

1. 如果两个对象equal返回true，则它们的hashCode一定相等。
2. 如果两个对象equal返回false，**则它们的hashCode不一定等**。
3. 如果两个对象hashCode()值不等，则它们equal一定不等。
4. 如果两个对象hashCode()值相等，**则它们equal不一定等**。
5. 在程序执行期间，只要equals方法的比较操作用到的信息没有被修改，那么对这同一个对象调用多次，hashCode方法必须始终如一地返回同一个整数。
6. 如果重写了equal()方法，请连hashCode()一起重写。

这里前四条都好理解，这里根据第六点来解释第五点。请看代码



```java
public class HashTest {

	private String name;
	private Integer age;
	
	public HashTest (String name, Integer age) {
		
		this.name = name;
		this.age = age;
	}
	
	public static void main(String[] args) {
		
		HashTest h1 = new HashTest("lam", 22);
		HashTest h2 = new HashTest("lamp", 21);
		
		System.out.println("h1 hashCode = " + h1.hashCode());   //2018699554
		System.out.println("h2 hashCode = " + h2.hashCode());   //1311053135
	}
}
```



这里我们新建了两个对象，很明显他们是不同对象，散列出来的hashcode一般也不会一样，这没有任何问题，接下来我们重写这个类的equal方法，假如两个对象name及age相等则判定两个对象相等。



```java
public class HashTest {

	private String name;
	private Integer age;
	
	public HashTest (String name, Integer age) {
		
		this.name = name;
		this.age = age;
	}
	
	public static void main(String[] args) {
		
		HashTest h1 = new HashTest("lam", 22);
		HashTest h2 = new HashTest("lam", 22);
		
		System.out.println(h1.equals(h2));                     //true
		
		Map<HashTest, Integer> map = new HashMap<>();
		map.put(h1, 1);
		
		System.out.println(map.get(h2));                       //null
		System.out.println("h1 hashCode = " + h1.hashCode());  //2018699554
		System.out.println("h2 hashCode = " + h2.hashCode());  //1311053135
	}
	
	@Override
	public boolean equals(Object obj) {
		
		if (this == obj)
			return true;
		
		if (obj == null || !(obj instanceof HashTest))
			return false;
		
		HashTest other = (HashTest)obj;
		return this.name.equals(other.name)
				&& this.age == other.age;
	}
}

//输出
true
null
h1 hashCode = 2018699554
h2 hashCode = 1311053135
```



明明h1和h2都判断相等了，这里问题就来了Map的key传入的是一个对象。按理来说两个对象相等，散列后hash值一致，应该能get到值而不是null。其实因为默认hashCode散列的地址值并且算法也不一致，所以我们无法根据我们自己equal的逻辑来得到hashCode一样，所以必须重写hashCode()，尽管这在没有用到Hash的集合里不需要重写，但建议都重写比较合理。下面给出重写后的代码，算法可以自定义：



```java
public class HashTest {

	private String name;
	private Integer age;
	
	public HashTest (String name, Integer age) {
		
		this.name = name;
		this.age = age;
	}
	
	public static void main(String[] args) {
		
		HashTest h1 = new HashTest("lam", 22);
		HashTest h2 = new HashTest("lam", 22);
		
		System.out.println(h1.equals(h2));
		
		Map<HashTest, Integer> map = new HashMap<>();
		map.put(h1, 1);
		
		System.out.println(map.get(h2));
		System.out.println("h1 hashCode = " + h1.hashCode());
		System.out.println("h2 hashCode = " + h2.hashCode());
	}
	
	@Override
	public boolean equals(Object obj) {
		
		if (this == obj)
			return true;
		
		if (obj == null || !(obj instanceof HashTest))
			return false;
		
		HashTest other = (HashTest)obj;
		return this.name.equals(other.name)
				&& this.age == other.age;
	}
	
	@Override
	public int hashCode() {
		
		return name.hashCode()*37+age;
	}
}

//输出
true
1
h1 hashCode = 3955470
h2 hashCode = 3955470
```



这里要注意一点，这块代码我省略了对象的get及set方法。但如果实际情况中存在set方法，可以更改对象的成员变量，那么假如往Map里put了一个对象作为key，此时调用set方法修改其成员变量，那么hashCode会发生变化，换而言之无法再根据这个对象get到这个value。

所以如果你的hashCode方法依赖于对象中易变的数据，用户就要当心了，因为此数据发生变化时，hashCode()方法就会生成一个不同的散列码。

### Iterator

中文译为迭代器，Collection依赖于该Iterable接口，而接口中存在一个返回Iterator接口的抽象方法。

```java
public interface Iterable<T> {
   
    Iterator<T> iterator();
    .....
}
```



接口内含四个基础抽象方法

```java
public interface Iterator<E> {
    
    boolean hasNext();
    
    E next();
    
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }
    
    default void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
    }
```



Collection接口继承了iterator接口，声明一个变量，让其子类来实现返回一个迭代器。

```java
public interface Collection<E> extends Iterable<E> {

Iterator<E> iterator();
    .....
}
```



同理List接口，直至实现类(注意，List多声明了一个ListIterator<E> 迭代器)。这里以ArrayList类的实现方法来作参考：

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    public ListIterator<E> listIterator(int index) {       //实现类就不贴了
        if (index < 0 || index > size)
            throw new IndexOutOfBoundsException("Index: "+index);
        return new ListItr(index);
    }
    
    public ListIterator<E> listIterator() {    //重载
        return new ListItr(0);
    }
    
    public Iterator<E> iterator() {            //返回的是接口的实现如下
        return new Itr();
    }
    
    private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor != size;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;
            checkForComodification();
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
}
```



我们发现在进行next()和remove()操作时，都调用了checkForComodification()方法，目的是确保集合的线程安全。

要理解这个概念先明白modCount，expectedModCount二者是什么，以及二者在什么时候会发生改变。

1.modCount 即修改的次数，在进行add(), remove(),clear()时会自增一次

2.而expectedModCount，我们在Itr类实现迭代器时声明，将modCount 的值赋予expectedModCount引用。

而如果在next()和remove()的过程中，其他线程对此集合进行修改操作，modCount 发生了改变，与expectedModCount值不一致，就会抛出ConcurrentModificationException异常，从这也可以看出ArrayList确实线程不安全。举个例子：



```java
List<String> list = new ArrayList<>(10);
        list.add("1");
        list.add("2");
        list.iterator();

        new Thread(new Runnable() {
			
			@Override
			public void run() {
				list.add("3");            //modCount发生变化
			}
		}).start();
      
        
        ListIterator<String> it =  list.listIterator(); //此时获取modCount后进行sleep
        try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
        it.next();                      //抛出异常，ConcurrentModificationException

```



还有一种情况是我们经常使用的"for-each"，我们知道foreach的实现就是迭代器遍历collection，所以也会产生以下错误：

```java
        for (String str: list) {
        	
        	System.out.println(str);
        	list.add("a");            //ConcurrentModificationException
        }
```

同时，使用foreach不能指定特殊元素，只能全部遍历。

**但是，如果是同一个迭代器，是可以在遍历时add()或remove()的，因为其add方法（remove()）会在完成时更新expectedModCount的值，所以前后值一致，不会报错。**



#### 原理图

要把迭代器理解成在两个元素中间，next返回的是跨过的元素。

![](.\pictures\Iterator.png)



参见下述代码：

```java
List<String> list = new ArrayList<>(10);
        list.add("1");
        list.add("2");
ListIterator<String> it =  list.listIterator();
it.remove();    //出错，因为此时游标的位置在第一个元素之前，IllegalStateException

//正确写法
it.next();
it.remove();
```



### ListIterator

#### 来源

继承自Iterator，并拓展了几个方法，如下：

```java
public interface ListIterator<E> extends Iterator<E> {
    
    //省略Iterator的基本四个方法
    
    boolean hasPrevious();
    E previous();
    int nextIndex();
    int previousIndex();
    void set(E e);        //取代下一个next或下一个previous的元素
    void add(E e);        //增加至当前游标的左侧位置，相当于添加并越过
}
```



ListIterator()只存在实现List接口的子类中，一般都是直接子类AbstractList去实现接口中的方法，往后的类继承自这个类，但每个子类又都有自己的内部实现。

# Collection

## 接口图

![](.\pictures\collectionInterface.png)



## 结构图

![](.\pictures\collection.png)

Collection继承自Iterable<E>类，故实现了迭代器的方法，并进行了拓展。对于实现Interable<E>的子类，都能使用"for-each"进行迭代，故图上的对象，均可直接使用"for-each"迭代。



# List

## 结构图

![](.\pictures\List.png)



## ArrayList

参考：https://www.jianshu.com/p/ccddd869d651

### 特点

1.底层基于数组实现

2.线程不安全

3.查询效率快，但增删慢

4.初始默认容量是10，每次扩容50%（如果还不够，则扩容至传入值）

5.由于扩容非常浪费资源且效率低，尽量确定需要数组的大小

6.允许存放null（不止一个）



### 继承与实现

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

从 `ArrayList` 的继承关系来看， `ArrayList` 继承自 `AbstractList`，实现了`List<E>, RandomAccess, Cloneable, java.io.Serializable` 接口。

- 其中`AbstractList<E>`和`List<E>`是用来规定集合框架的一些基本必备方法。而ArrayList本身覆写了大量的接口中的方法，包含了基本的增删改查。
- `ArrayList` 实现 `RandomAccess` 接口标识着其支持随机快速访问，查看源码可以知道`RandomAccess` 其实只是一个标识，标识某个类拥有随机快速访问的能力，针对 ArrayList 而言通过 `get(index)`去访问元素可以达到 O(1) 的时间复杂度。有些集合类不拥有这种随机快速访问的能力，比如 `LinkedList` 就没有实现这个接口。
- `ArrayList` 实现 `Cloneable` 接口标识着他可以被克隆/复制，其内部实现了 clone 方法供使用者调用来对 ArrayList 进行克隆，但其实现只通过 `Arrays.copyOf` 完成了对 ArrayList 进行「浅复制」，也就是你改变 `ArrayList clone`后的集合中的元素，源集合中的元素也会改变，对于深浅复制我以后会单独整理一篇文章来讲述这里不再过多的说。
- 对于 `java.io.Serializable` 标识着集合可被被序列化。

### 成员变量

```java
    //序列化
    private static final long serialVersionUID = 8683452581122892189L;
    
    //默认初始容量
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 这是一个共享的空的数组实例，当使用 ArrayList(0) 或者 ArrayList(Collection<? extends E> c) 
     * 并且 c.size() = 0 的时候讲 elementData 数组讲指向这个实例对象。
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * 另一个共享空数组实例，再第一次 add 元素的时候将使用它来判断数组大小是否设置为 DEFAULT_CAPACITY
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 真正装载集合元素的底层数组 
     * 至于 transient 关键字这里简单说一句，被它修饰的成员变量无法被 Serializable 序列化 
     */
    transient Object[] elementData; 

	
    private int size;
```



### 构造器

ArrayList一共有三个构造器

#### 无参

```java
public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

在上面我们看到DEFAULTCAPACITY_EMPTY_ELEMENTDATA是一个空的对象数组，这里只是将elementData指向它，而且此处并没有默认容量=10，这是因为采用懒加载在add()方法触发时，才根据数组名判定来设置数组长度。



#### 指定初始容量的构造器

```java
public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity]; //大于零则使用new一个对象数组
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;           //等于零则使用成员变量的数组
        } else {
            //如果传入值<0，抛出IllegalArgumentException异常
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
}
```



#### 使用另一个Collection的构造方法

```java
public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();               //转为数组
        if ((size = elementData.length) != 0) {  //数组长度不为0
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // 使用空数组替代
            this.elementData = EMPTY_ELEMENTDATA;
        }
}
```



### 扩容

当对象创建后，并不会马上创建空间（懒加载？），而是等到第一次add的时候才去创建所需要的空间。

```java
public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;           // size使用后自增
        return true;
}

private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) { //根据名称确定对象
            //初次判定时，DEFAULT_CAPACITY = 10，minCapacity = size+1 =1
            //之后传入的都是size + 1
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity); 
        }

        ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
        modCount++; //因为add，次数自增

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
 }

//扩容函数
private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1); //左移1位相当于/2，也就是50%
        if (newCapacity - minCapacity < 0)          //如果还不够，则使用minCapacity当新容量
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)       //如果大于int的最大值，这里不考虑。。。
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity); //开始扩容拷贝
}
```



### 添加元素相关方法

#### 在指定角标位置添加元素的方法 

add(int index, E element)

```java
 public void add(int index, E element) {
        rangeCheckForAdd(index);         //检测角标是否越界

        ensureCapacityInternal(size + 1);  // 确保容量够用，不够则进行扩容
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);    //将当前index位置之后的元素进行后移1位
        elementData[index] = element;      //将传入对象放置至index位置
        size++;
}

private void rangeCheckForAdd(int index) {   //检测是否越界
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

/**
  * @param      src      要加入的原始数组
  * @param      srcPos   从该数组的这个起始位置开始
  * @param      dest     目标数组
  * @param      destPos  目标数组的目的位置
  * @param      length   复制的长度
**/  
public static native void arraycopy(Object src,  int  srcPos,
                                    Object dest, int destPos,
                                    int length);
```

将一个数组 `src` 起始 `srcPos` 角标之后 length 长度间的元素，赋值到 `dest` 数组中 `destPos` 到 `destPos + length -1`长度角标位置上。只是在 add 方法中 `src` 和 `destPos` 为同一个数组而已。



#### 批量添加元素

在数组末尾添加

```java
public boolean addAll(Collection<? extends E> c) {
        // 调用 c.toArray 将集合转化数组
        Object[] a = c.toArray();
        // 要添加的元素的个数
        int numNew = a.length;
        //扩容检查以及扩容
        ensureCapacityInternal(size + numNew);  // Increments modCount
        //将参数集合中的元素添加到原来数组 [size，size + numNew -1] 的角标位置上。
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        //与单一添加的 add 方法不同的是批量添加有返回值，如果 numNew == 0 表示没有要添加的元素则需要返回 false 
        return numNew != 0;
}
```



在指定位置添加

```java
public boolean addAll(int index, Collection<? extends E> c) {
        //同样检查要插入的位置是否会导致角标越界
        rangeCheckForAdd(index);
        
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew); 
        //这里做了判断，如果要numMoved > 0 代表插入的位置在集合中间位置，和在 numMoved == 0最后位置 则表示要在数组末尾添加 如果 < 0  rangeCheckForAdd 就跑出了角标越界
        int numMoved = size - index;
        if (numMoved > 0)    //移动numMoved个位置
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);

        System.arraycopy(a, 0, elementData, index, numNew); //进行插入新数组
        size += numNew;
        return numNew != 0;
    }
    
private void rangeCheckForAdd(int index) {
   if (index > size || index < 0)
       throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

两个方法不同的地方在于如果移动角标即之后的元素，`addAll(int index, Collection<? extends E> c)`里做了判断，如果要 `numMoved > 0` 代表插入的位置在集合中间位置，和在 `numMoved == 0` 最后位置 则表示要在数组末尾添加 如果 `numMoved < 0` ，`rangeCheckForAdd` 就抛出了角标越界异常了。

与单一添加的 add 方法不同的是批量添加有返回值，如果 numNew == 0 表示没有要添加的元素则需要返回 false



### 移除元素相关方法

根据角标移除元素

```java
public E remove(int index) {
        rangeCheck(index);               //越界检查

        modCount++;                      //并发保证
        E oldValue = elementData(index); //获取元素

        int numMoved = size - index - 1; //需要移动的元素数量
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);  //向前移动元素
        elementData[--size] = null; // clear to let GC do its work，置空，GC回收

        return oldValue;
}


 E elementData(int index) {
        return (E) elementData[index];
}
```



移除指定元素

```java
public boolean remove(Object o) {
        if (o == null) {             //要移除的对象是否为 null
            for (int index = 0; index < size; index++) //遍历
                if (elementData[index] == null) {      //查看该角标指向是否为null
                    fastRemove(index);                 //调用角标删除法
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                //这里要注意，Object o = E；所以o.equal()等于是调用 E.equal()
                if (o.equals(elementData[index])) {    //区别在此
                    fastRemove(index);
                    return true;
                }
        }
        return false;
}

private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
}
```

可以看出移除的是根据传入对象equal角标相等的第一个对象。



### 批量移除/保留 removeAll/retainAll

```java
public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);             //检验非空
        return batchRemove(c, false);          //与retainAll区别在false	
}

public boolean retainAll(Collection<?> c) {
        Objects.requireNonNull(c);             //检验非空
        return batchRemove(c, true);           //与removeAll区别在true
}

private boolean batchRemove(Collection<?> c, boolean complement) { //batch译为部分
        final Object[] elementData = this.elementData;    //获取实际存储对象
        int r = 0, w = 0;                                 //r循环角标，w为删除后元素长度
        boolean modified = false;                         //是否修改的标识
        try {
            for (; r < size; r++)
               if (c.contains(elementData[r]) == complement) //false删除含有，true保留含有
                   elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            // 如果c.contains（o）可能会抛出异常，如果抛出异常后 r!=size 则将 r 之后的元素不在比较直接放入数组
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;     //w长度为 w + (size -r)
            }
            if (w != size) {       //如果删除了元素，则要进行资源回收，因为是在原数组上移除的
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;  //w为不变的元素个数， size - w 为修改次数
                size = w;
                modified = true;
            }
        }
        return modified;
}

public static <T> T requireNonNull(T obj) {
        if (obj == null)
            throw new NullPointerException();
        return obj;
}
```



### 数组与集合互转

#### 数组转集合

方法一：

```java
    String[] strArray = {"one", "two", "three"};
    List<String> stringArrayList = Arrays.asList(strArray);
    stringArrayList.set(0, "1");
    System.out.println(strArray[0]);	// 输出1

		// 调用以下三个ArrayList的API均会抛出UnsupportedOperationException
		stringArrayList.add("four");
		stringArrayList.remove(2);
		stringArrayList.clear();
```

注意事项：

使用这种方法转换，除了`set()`方法可以正常使用，其余三种编译时正常，运行时出现`UnsupportedOperationException`的错误。这是因为`asList()`方法实际上返回的是一个`Arrays`内部的`ArrayList`对象，这个对象并非我们所熟知的`ArrayList`，它的源码如下：

```java
    public static <T> List<T> asList(T... a) {
        return new ArrayList<>(a);
    }

    private static class ArrayList<E> extends AbstractList<E>
        implements RandomAccess, java.io.Serializable
    {
        private static final long serialVersionUID = -2764017481108945198L;
        private final E[] a;				// 这里的final关键字使得ArrayList还是指向数组

        ArrayList(E[] array) {
            a = Objects.requireNonNull(array);
        }
        
        @Override
        public E set(int index, E element) {
            E oldValue = a[index];
            a[index] = element;
            return oldValue;
        }
    }
```

而这个内部类并没有实现上面的`add/remove/clear`方法，故会抛出异常。但这个异常是哪来的？我们可以看到这个`ArrayList`内部类也是继承自`AbstractList`，而`AbstractList`抽象类本身的抽象方法并没有具体的实现，只是抛出了`UnsupportedOperationException`异常，最后`clear`方法能抛出这个异常是因为其内部实现就是调用`remove`方法。

```java
// AbstractList部分源码

    public void add(int index, E element) {
        throw new UnsupportedOperationException();
    }

	public E remove(int index) {
        throw new UnsupportedOperationException();
    }

    public void clear() {
        removeRange(0, size());
    }

    protected void removeRange(int fromIndex, int toIndex) {
        ListIterator<E> it = listIterator(fromIndex);
        for (int i=0, n=toIndex-fromIndex; i<n; i++) {
            it.next();
            it.remove();
        }
    }

```



方法二：

在上述方法的基础上加以改进，使用以下转换方法即可:

```java
        String[] strArray = {"one", "two", "three"};
        List<String> stringArrayList = Arrays.asList(strArray);
        List<String> strList = new ArrayList<>(stringArrayList);
		strList.set(0, "1");
        System.out.println(strList.get(0));						// 输出 1
        System.out.println(strList.add("bbb"));					// 输出 true
        System.out.println(strList.get(strList.size() - 1));	// 输出 bbb
        System.out.println(strList.remove(1));					// 输出 two
        strList.clear();
        System.out.println(strList.size());						// 输出 0
```

方法三：

使用JDK1.8的stream。

对于包装类型：

```java
				Integer[] array = {1, 2, 3};
        List<Integer> list = Arrays.stream(array).collect(Collectors.toList());
```

对于基础类型(需要额外使用boxed()装箱）:

```java
				int[] array = {1, 2, 3};
        List<Integer> list = Arrays.stream(array).boxed().collect(Collectors.toList());
```



#### 集合转数组

```java
public class ListToArray {

    public static void main(String[] args) {

        List<String> list = new ArrayList<>();
        list.add("one");
        list.add("two");
        list.add("three");

        // 开始转换
        // 情况一
        // 返回的是Object[], 使用String[]接收会无法编译
        Object[] strArray1 = list.toArray();

        // 情况二
        // 创建一个相应类型的数组, 且长度小于list的size
        String[] strArray2 = new String[2];
        String[] tStrArray2 = list.toArray(strArray2);		
        System.out.println(Arrays.asList(strArray2));		//数组无法打印内容，转为list
        System.out.println(strArray2 == tStrArray2);

        // 情况三
        // 创建一个相应类型的数组, 且长度等于list的size
        String[] strArray3 = new String[3];
        String[] tStrArray3 = list.toArray(strArray3);
        System.out.println(Arrays.asList(strArray3));
        System.out.println(strArray3 == tStrArray3);

        // 情况四
        // 创建一个相应类型的数组, 且长度大于list的size
        String[] strArray4 = new String[4];
        String[] tStrArray4 = list.toArray(strArray4);
        System.out.println(Arrays.asList(strArray4));
        System.out.println(tStrArray4 == strArray4);
    }
}


// 输出结果
[null, null]
false
[one, two, three]
true
[one, two, three, null]
true
```



情况一：转换出来的数组只能是`Object[]`类型的。

情况二：这里创建了一个小于原`List`集合的`size`的对象数组，然后调用了`toArray(T[] a)`这个方法。方法内首先去判断`length`与`size`的大小关系，如果是`length < size`，则会抛弃传入的这个对象数组的引用，自己内部调用`Arrays.copyOf()`创建一个相应的对象数组，长度为`Min(elementData.length, length)`。最终返回这个新生成的对象数组的引用，所以原数组`strArray2`并没有被用到，所以输出`[null, null]`(空数组的值)，从打印结果也能看出这两个对象不是同一个对象。

情况三：原理与情况二相同，只是在进入`toArray(T[] a)`方法后,判断其长度刚好等于`size`，故直接使用传入的对象数组进行`System.arraycopy()`操作，并返回这个对象数组的引用，所以最后能看到打印结果为`true`。

情况四：与情况三类似，只是`size`比`length`小，多出来的本身数组就是填充`null`。

```java
// 相关源码
    public <T> T[] toArray(T[] a) {
        if (a.length < size)
            // Make a new array of a's runtime type, but my contents:
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
        System.arraycopy(elementData, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }

    public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) 	  {
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }

```



最后，根据验证得出，传入长度与`size`相等的，类型与`list`相同的对象数组，效率最高，空间最省。

### List与Map互转

#### List转Map

```java
List<String> list = new ArrayList<>();
list.add("a");
list.add("b");
list.add("c");
list.add("d");
Map<String, String> map = list.stream().collect(Collectors.toMap(e -> e, e -> e));
Set<Map.Enrty<String, String>> entrys = map.entrySet();
entry.forEach(e -> {
            System.out.println("key = " + e.getKey());
            System.out.println("value = " + e.getValue());
});

// 输出
key = a
value = a
key = b
value = b
key = c
value = c
key = d
value = d
```

#### Map转List

```java
Map<String, String> map = new HashMap<>();
map.put("a", "1");
map.put("b", "2");
map.put("c", "3");
// 转collections
List<String> list = new ArrayList<>(map.values());
list.forEach(e -> System.out.println(e));

// 输出
1
2
3
```



### Fail-fast

#### SubList

`SubList`是`ArrayList`的一个内部类，它是通过`ArrayList.subList(int Indexof, int toOf)`获得一个`ArrayList`视图。



看一段代码：

```java

public class SubListFailList {

    public static void main(String[] args) {

        List masterList = new ArrayList();
        masterList.add("one");
        masterList.add("two");
        masterList.add("three");
        masterList.add("four");
        masterList.add("five");

        List branchList = masterList.subList(0,3);

        // 下方三行代码，如不注释掉，则操作branchList会抛出异常
        masterList.remove(0);
        masterList.add("ten");
        masterList.clear();

        // 下方四行能正确执行
        branchList.clear();
        branchList.add("six");
        branchList.add("seven");
        branchList.remove(0);

        // 正常遍历结束， 只有一个元素: "seven"
        for (Object t : branchList) {

            System.out.println(t);
        }

        // 子列表修改导致父列表也修改，输出: [seven, four, five]
        System.out.println(masterList);
    }
}
```

在操作子视图时，父列表是不可以进行改变`modCount`的操作的，原因如下：

```java
    private class SubList extends AbstractList<E> implements RandomAccess {
        private final AbstractList<E> parent;
        private final int parentOffset;
        private final int offset;
        int size;
        ……
        public void add(int index, E e) {	
            rangeCheckForAdd(index);
            checkForComodification();
            parent.add(parentOffset + index, e);
            this.modCount = parent.modCount;
            this.size++;
        }    
```

这里拿`add()`方法来说，其它方法也一样，因为该内部类自带的方法在执行之前都会先判断modCount是否修改过了，如果修改过了则抛出异常。

再来看第二块代码，子视图的操作实际上是对父列表产生影响的，因为它的方法操作的都是父视图。上面`add()`方法中的`parent`指向的就是父列表。

另外注意，`subList`是不可以被实例化的。



#### 特殊情况

直接看代码

```java
public class ArrayListFailFast {

    public static void main(String[] args) {

        List<String> list = new ArrayList<>();
        list.add("one");
        list.add("two");
        list.add("three");


        for (String str : list) {

            if ("two".equals(str)) {

                list.remove(str);
            }
        }

        System.out.println(list);
    }
}

// 输出
[one, three]
```

很奇怪吧，为什么在遍历的时候操作了`list`却没抛出`ConcurrentModificationException`，实际上这是一个巧合。我们知道`forEach`机制是用`hasNext()`进行判断，而`hashNext()`内部实现如下：

```java
public boolean hasNext() {
            return cursor != size;
}
```

它是根据游标`cursor`与`list`的`size`进行判断的，最初`cursor`在第一个元素之前为0，越过第一个元素之后为1，越过第二个元素之后为2，此时执行`if`内的语句，此时`list`的`size`变为2:

```java
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // 此处修改了size = size - 1

        return oldValue;
    }
```

当进行下一次`hasNext()`时，得到`false`，退出遍历，而抛出异常则是在`next()`方法（获取下一个元素）中进行判断，故此段代码不抛出异常，所以在删除元素时还是要注意`fail-fast`机制。



如果想要不抛出异常的删除，可以采用`Iterator`机制

```java

        List<String> list2 = new ArrayList<>();
        list2.add("one");
        list2.add("two");
        list2.add("three");
        Iterator<String> i = list2.iterator();
        while (i.hasNext()) {

            String item = i.next();
            if ("one".equals(item)) {

                i.remove();
            }
        }
        System.out.println(list2); // 输出: [two, three]
```

```java

// Itr内部类方法
        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;						// 此处游标进1
            return (E) elementData[lastRet = i];// 此处保存上一个位置的游标
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);	// 删除上一处的元素
                cursor = lastRet;				// 游标矫正归位
                lastRet = -1;					// 临时变量重新赋值为 -1
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
```

通过一个临时变量`lastRet`找到上一个元素的位置，矫正游标，使得操作正确。



### 面试题

最终会抛出角标越界还是`ConcurrentModificationException`呢？

```java
ArrayList<String> list = new ArrayList<String>();
for (int i = 0; i < 10; i++) {
  list.add("sh" + i);
}

for (int i = 0; list.iterator().hasNext(); i++) {
  list.remove(i);             //记住add 和 remove都会使cursor越过这个元素
  System.out.println("秘密" + list.get(i)); //所以这里get的是remove的后一个元素
}
```

会抛出角标越界的错误。因为`list.iterator().hasNext()`等于每次都重新获取了一个迭代器，`modCount`与`list`一致。所以不会出现`ConcurrentModificationException`错误。而由于每次获取，导致`cursor`一直为0，上述代码每`remove`输出一个元素，一共输出五个元素，当第六次是`remove`时执行`remove(5)`，而此时数组长度为5，故越界。

ArrayList.hasNext()

```java
public boolean hasNext() {
                    return cursor != SubList.this.size;
}
```

### 保证线程安全

#### synchronizedList

为了获得线程安全的 ArrayList，可以使用 `Collections.synchronizedList();` 得到一个线程安全的 `ArrayList`。

```java
List<String> list = new ArrayList<>();
List<String> synList = Collections.synchronizedList(list);
```



#### CopyOnWriteArrayList

属于COW家族。

## Vector

### 特点

1.线程同步，所以开销比ArrayList大，访问速度慢。

2.扩容是原来的2倍，而ArrayList是1.5倍。

## LinkList

### 特点

1.底层实现为双向链表

2.集合中元素允许存null

3.查询慢，因为需要一个个next()，但增删快（先查后改）

4.有序

5.非线程安全，想要保证线程安全下使用，可以采用`List list = Collections.synchronizedList(new LinkedList(...));`

关于链表参见此文章：https://www.jianshu.com/p/73d56c3d228c



### 双向链表实现

```java
private static class Node<E> {
        E item;
        Node<E> next;      //指向下一个节点
        Node<E> prev;  	   //指向上一个节点

        Node(Node<E> prev, E element, Node<E> next) {   //实例化节点对象
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
}
```



### 继承与实现

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

1.AbstractSequentialList继承自AbstractList，这个类是队列的，后面讲

2.一样实现了List接口，标识

3.Cloneable，可克隆，标识

4.可序列化



### 成员变量

```java
transient int size = 0;       //节点个数
 
transient Node<E> first;      //链表的第一个节点
 
transient Node<E> last;       //链表的最后一个节点
```

之所以保存头尾节点是为了解决链表在查询方面效率较低，而如果在查询元素时，根据index和size/2，就可以选择从头节点还是尾节点开始，一定程度上弥补了这项缺陷。



### 构造器

- ```java
  public LinkedList() {
  }
  
  public LinkedList(Collection<? extends E> c) {
          this();
          addAll(c);
  }
  
  public boolean addAll(Collection<? extends E> c) {
          return addAll(size, c);
      }
  
  public boolean addAll(int index, Collection<? extends E> c) {
          checkPositionIndex(index);  //检查元素角标是否越界
  
          Object[] a = c.toArray();   //转成数组
          int numNew = a.length;      //获取长度
          if (numNew == 0)            //如果长度为0直接返回
              return false;
  
          Node<E> pred, succ;         //声明两个节点,succ为当前节点，pred为上一个节点
          if (index == size) {        //如果index为size，则代表在链表末端添加，succ就为null
              succ = null;
              pred = last;
          } else {                 	//否则，则是在链表中间插入
              succ = node(index);     //找到此时index位置的节点，succ指向它
              pred = succ.prev;       //pred指向index节点的prev
          }
  
          for (Object o : a) {        //遍历数组a
              @SuppressWarnings("unchecked") E e = (E) o;  //向下转型，强转成类的泛型
              Node<E> newNode = new Node<>(pred, e, null); //生成新的节点
              if (pred == null)		//只有first节点pred为null
                  first = newNode;
              else					//pred节点的next指向新节点
                  pred.next = newNode;
              pred = newNode; 		//pred重新指向新节点
          }
  
          if (succ == null) {			//如果是在末尾添加，则添加完成时，最后一个元素为pred
              last = pred;            //last指向它
          } else {					//否则要将之前的元素接在后面
              pred.next = succ;		//找到刚才的succ元素，让此时pred的next指向它
              succ.prev = pred;		//双向链表，succ的pred也要指向此时的pred
          }
  
          size += numNew;				//size = size + numNew
          modCount++;					//修改次数变化
          return true;				//返回添加成功
  }
  
  private void checkPositionIndex(int index) {
          if (!isPositionIndex(index))
              throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
  }
  
  private boolean isPositionIndex(int index) {
          return index >= 0 && index <= size;
  }
  
  Node<E> node(int index) {
          // assert isElementIndex(index);
  
          if (index < (size >> 1)) {      	//二分法寻址，如果在index < size/2，从左开始
              Node<E> x = first;
              for (int i = 0; i < index; i++) //向后寻址，直到index位置
                  x = x.next;
              return x;
          } else {                            //从右边开始
              Node<E> x = last;
              for (int i = size - 1; i > index; i--)//向前寻址，直到index位置
                  x = x.prev;
              return x;
          }
  }
  ```




总结起来就是：

1.检查index是否越界

2.获取传入数组的长度，为0直接返回false

3.为了连接之前的元素，必须有pred作为临时存储引用，考虑到可能index < size，所以还需要一个succ引用指向当前index节点的对象

4.判断是在链表末尾添加还是在链表中添加，如果在链表末端添加，则succ为null，pred为此时的last，即最后一个节点。如果是在链表中，则succ为此时index节点对象，pred为succ的pred指向的对象。

5.存储完以上对象后，succ用来添加完成的后续连接。此时开始添加传入的对象数组，使用构造器创建节点，传入pred，对象，null。null为next，如果pred为null则一定是first节点，要额外将first指向此时的node。否则pred的next就是此时的node，再将pred从新指向成nownode，以此迭代。

6.全部存储完后，要判断succ是否为null，如果为null，则last要指向pred；否则pred.next要指向succ，从新连接后续元素，同时succ.pred指向pred。

7.最后修改size 为oldSize + newNum，并且修改次数+1，返回true。

### 添加元素

#### add(E e):

在链表末尾添加一个节点

```java
public boolean add(E e) {
        linkLast(e);
        return true;
}

void linkLast(E e) {
        final Node<E> l = last;         //通过last引用找到此时最后一个节点对象 
        final Node<E> newNode = new Node<>(l, e, null); //Node结构（previous，element，next）
        last = newNode;                 //将上面生成的新节点赋给last
        if (l == null)                  //如果l=null，则代表链表为空，则该节点应为头结点first
            first = newNode;
        else                            //否则，原last节点的next指向该节点
            l.next = newNode; 
        size++;                         //链表长度+1
    modCount++;
}
```

综合起来的步骤：

1.保留现last节点

2.生成一个新节点

3.修改last节点为新节点

4.判断是否要修改first节点

5.长度自增



#### addLast(E e):

作用同add（E e)

```java
public void addLast(E e) {
        linkLast(e);
}
```



#### addFirst(E e)

在链表头添加一个节点

```java
public void addFirst(E e) {
        linkFirst(e);
}

private void linkFirst(E e) {
        final Node<E> f = first;      //根据first引用找到此时第一个节点，临时存头节点对象
        final Node<E> newNode = new Node<>(null, e, f); //新建一个节点，previous = null，对象element，next指向现在的first
        first = newNode;              //将first指向newNode
        if (f == null)                //如果链表为空，则last也为newNode
            last = newNode;
        else
            f.prev = newNode;         //否则原头节点的previous指向newNode
        size++;                       //长度自增1
        modCount++;
    }
```

综合起来：

1.找到原头节点，并使用临时变量指向这个对象

2.新建一个节点（null，e，f），f指向原头节点。

3.first指向新节点

4.判断f是否为空（链表是否为空），如果为空则last节点也要指向新节点；否则，f的pred指向新节点

5.长度+1，修改次数+1，返回true



#### add(int index, E element)

在指定index位置插入节点



```java
    public void add(int index, E element) {
        checkPositionIndex(index);       //角标检测是否越界

        if (index == size)				 //这里有两种可能，一种是在末端，一种是链表为空
            linkLast(element);
        else							 //由于上面判断了链表为空的情况，所以下面就不存在链表为空情况
            linkBefore(element, node(index));
    }

    void linkLast(E e) {
        final Node<E> l = last;			//根据last获取最后一个元素
        final Node<E> newNode = new Node<>(l, e, null);//新建节点
        last = newNode;					//last指向新节点
        if (l == null)					//如果l为空则代表原链表为空，first指向新节点
            first = newNode;
        else
            l.next = newNode;			//否则原节点.next指向新节点
        size++;
        modCount++;
    }

    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;      断言succ肯定不为空
        final Node<E> pred = succ.prev;		//final不可变，找到succ的前驱节点
        final Node<E> newNode = new Node<>(pred, e, succ); //新建节点，插入其中
        succ.prev = newNode;				//succ的前驱指向新节点
        if (pred == null)					//这里要判断succ是否为first，即pred是否为null
            first = newNode;				//为null则first指向新节点
        else
            pred.next = newNode;			//否则pred的next指向新节点
        size++;								//长度+1
        modCount++;							//修改次数+1
    }
```



总结步骤：

1.检查数组角标是否越界

2.如果链表为空或在链表尾端插入则调用linkLast，这里也确保了linkBefore只添加位置在size中的节点

3.添加元素需要三个对象，一个是原节点对象，通过node（index）获得，还有原节点.pred指向的节点，以及新节点（传入对象E用作节点item）。

4.实例化一个节点，pred指向原节点.pred，next指向succ，而后succ.pred指向新节点。

5.这里注意节点是否只有一个节点，如果是的话，succ.pred==null，则新插入的节点为first，否则pred.next为新节点。



### 删除元素



#### removeFirst()

删除头节点



```java
    public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }

	public E unlinkFirst(Node<E> f) {
        
        final E element = f.item;  //用于返回值
        final Node<E> next = f.next;
        f.item = null;
        f.next = null;
        
        first = next;
        
        if (next == null)
            last = null;
        else 
            next.prev = null;
        
        size--;
        modCount++;
        return element;
	}
```



#### removeLast()

删除末尾元素



```java
public E removeLast() {
    
    final Node<E> l = last;
    
    if (l == null)
        throw new NoSuchElementException();
    
    return unlinkLast(l);	
}

public E unlinkLast(Node<E> l) {
    
    final E element = l.item;
    final Node<E> prev = l.prev;
    
    l.item = null;
    l.prev = null;
    
    last = prev;
    
    if (prev == null) 
        first = null;
    else
        prev.next = null;
    size--;
    modCount++;
    return element;
}
```



#### remove(Object o)

删除指定元素（第一个）



```java
    public boolean remove(Object o) {
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {  //分情况，如果元素为null用==，否则用equal
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }

	E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }	
```



总结：

1.获取返回的oldValue，获取n.prev,n.next

2.如果prev为null，则为头节点，first指向next；否则prev.next指向next，x.prev = null，置空。

3.如果next为null，则为尾节点，last指向prev；否则next.prev指向prev，x.next = null，置空。

4.最后将x.item置空

5.长度-1，修改次数+1，返回oldValue



#### clear()

清空链表



```java
public void clear() {
   // 依次清除节点，帮助释放内存空间
   for (Node<E> x = first; x != null; ) {
       Node<E> next = x.next;
       x.item = null;
       x.next = null;
       x.prev = null;
       x = next;
   }
   first = last = null;
   size = 0;
   modCount++;
}
```



### 查询元素

除了头节点和尾节点，如果查询次数大于修改次数，建议使用ArrayList。

#### indexOf(Object o)

查询对象第一次出现在链表的位置，从前向后遍历

​	

```java
    public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {  //null和对象分开判断
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }
```



#### lastIndexOf(Object o)

查询对象最后一次出现在链表中的位置，从后往前查



```java
    public int lastIndexOf(Object o) {
        int index = size;
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (x.item == null)
                    return index;
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (o.equals(x.item))
                    return index;
            }
        }
        return -1;
    }
```



#### contains(Object o)

查询对象是否存在链表中



```java
public boolean contains(Object o) {
    return indexOf(o) != -1;
}
```



### 修改元素

```java
    public E set(int index, E element) {
        
        checkElementIndex(index); //检查角标
        Node<E> x = node(index);  //获取指定位置的节点
        E oldVal = x.item;        //保留oldValue，用于返回值
        x.item = element;         //重新设置newValue
        return oldVal;			  //返回oldValue
    }
```



### 队列（暂略）https://www.jianshu.com/p/2721088cd2eb