# Map

## 结构图

![](.\pictures\Map.png)



## HashMap

### 特点

1.HashMap采用key/value来存储，底层实现是基于HashTable+链表+红黑树（自1.8之后）。因为采用了hashTable，所以在重写equal()时，务必连hashCode()方法一起重写。

2.key的元素是唯一的，可以存且只有一个null为key，hashSet的实现其实就是key的实现，而value可以存任意多个null，即元素可以重复。

3.HashMap的存储位置与hashCode有关，根据hashCode来确认将数据存储到哪个bucket。由于hashCode是根据hash算法计算的，所以可能存到相同的bucket中。

4.HashMap存储是无序的，并且可能因为扩容而导致元素位置发生改变，所以遍历顺序是不确定的。

5.HashMap是线程不安全的，要使得其安全可以使用`Collections.synchronizedMap(Map map)` 方法使 `HashMap` 具有线程安全的能力，或者使用 `ConcurrentHashMap`。



### 重要参数

```java
/*默认初始容量*/
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/*最大存储容量*/
static final int MAXIMUM_CAPACITY = 1 << 30;

/*默认加载因子*/
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/*默认树化阈值*/
static final int TREEIFY_THRESHOLD = 8;

/*默认非树化阈值*/
static final int UNTREEIFY_THRESHOLD = 6;

/*默认最小树化容量*/
static final int MIN_TREEIFY_CAPACITY = 64;

// 扩容阈值 = 容量 x 加载因子
int threshold;

//存储哈希桶的数组，哈希桶中装的是一个单链表或一颗红黑树，长度一定是 2^n
transient Node<K,V>[] table;  
  
// HashMap中存储的键值对的数量注意这里是键值对的个数而不是数组的长度
transient int size;
  
//所有键值对的Set集合 区分于 table 可以调用 entrySet(）得到该集合
transient Set<Map.Entry<K,V>> entrySet;
  
//操作数记录 为了多线程操作时 Fast-fail 机制
transient int modCount;

```



### 存储结构

1.7之前采用的是拉链法，即数组和链表。

![](.\pictures\hash1.7.jpg)

由于存在hash冲突，拉链法可以有效的解决。但我们都知道单链表寻址并不快，假如是非常大的数据，O(n)的时间也是非常久的。于是在1.8便引入了红黑树，底层还是数组不变，但当桶里的数据大于8时，就会将链表转化为红黑树，其查找时间为O(logn)，大大的提高了查找效率。

![](.\pictures\hash1.8.jpg)

### 基本存储单元

HashMap 在 JDK 1.7 中只有 `Entry` 一种存储单元，而在 JDK1.8 中由于有了红黑树的存在，就多了一种存储单元，而 `Entry` 也随之应景的改为名为 Node。我们先来看下单链表节点的表示方法 ：



```java
/**
 * 内部类 Node 实现基类的内部接口 Map.Entry<K,V>
 * 
 */
static class Node<K,V> implements Map.Entry<K,V> {
   //此值是在数组索引位置
   final int hash;
   //节点的键
   final K key;
   //节点的值
   V value;
   //单链表中下一个节点
   Node<K,V> next;
    
   Node(int hash, K key, V value, Node<K,V> next) {
       this.hash = hash;
       this.key = key;
       this.value = value;
       this.next = next;
   }

   public final K getKey()        { return key; }
   public final V getValue()      { return value; }
   public final String toString() { return key + "=" + value; }
    //节点的 hashCode 值通过 key 的哈希值和 value 的哈希值异或得到，没发现在源码中中有用到。
   public final int hashCode() {
       return Objects.hashCode(key) ^ Objects.hashCode(value);
   }

   //更新相同 key 对应的 Value 值
   public final V setValue(V newValue) {
       V oldValue = value;
       value = newValue;
       return oldValue;
   }
 //equals 方法，键值同时相同才节点才相同
   public final boolean equals(Object o) {
       if (o == this)
           return true;
       if (o instanceof Map.Entry) {
           Map.Entry<?,?> e = (Map.Entry<?,?>)o;
           if (Objects.equals(key, e.getKey()) &&
               Objects.equals(value, e.getValue()))
               return true;
       }
       return false;
   }
}
```



1.8的存储结构

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // 父节点
        TreeNode<K,V> left;    // 左子节点
        TreeNode<K,V> right;   // 右子节点
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;           // 是否红色
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }

        /**
         * Returns root of tree containing this node.
         */
        final TreeNode<K,V> root() {
            for (TreeNode<K,V> r = this, p;;) {
                if ((p = r.parent) == null)
                    return r;
                r = p;
            }
        }
    ……
}        
```



### 构造方法

HashMap一共有三个构造方法。



#### 1.传入初始容量以及自定义扩容因子

```java
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)  //传入的初始化容量不可以小于0
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY) //传入的初始化容量不可以大于Int Max，否则等于
            initialCapacity = MAXIMUM_CAPACITY; //Int Max
        if (loadFactor <= 0 || Float.isNaN(loadFactor)) //传入的加载因子大于0且不为无穷大
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;          //将loadFactor作为加载因子，初始化全局变量
        this.threshold = tableSizeFor(initialCapacity); //根据初始容量获得扩容阈值	
    }


//根据期望容量返回一个 >= cap 的扩容阈值，并且这个阈值一定是 2^n 
static final int tableSizeFor(int cap) {
   int n = cap - 1;
   n |= n >>> 1;
   n |= n >>> 2;
   n |= n >>> 4;
   n |= n >>> 8;
   n |= n >>> 16;
   //经过上述面的 或和位移 运算， n 最终各位都是1 
   //最终结果 +1 也就保证了返回的肯定是 2^n 
   return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

在以上构造函数中，并没有初始化Node<K,V>[] table，事实上table容量的加载在第一次put时。

tableSizeFor通过一系列移位或运算得到值+1一定是2^n



#### 2.只指定初始容量的构造函数

```java
public HashMap(int initialCapacity) {
   this(initialCapacity, DEFAULT_LOAD_FACTOR); //只是把上面的方法的加载因子改成默认加载因子
}
```



#### 3.无参构造函数

```java
public HashMap() {
   this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

这也是我们最常用的一个构造函数，该方法初始化了加载因子为默认值，并没有调动其他的构造方法，跟我们之前说的一样，哈希表的大小以及其他参数都会在第一调用扩容函数的初始化为默认值。

#### 4.传入一个Map集合的构造函数

```java
public HashMap(Map<? extends K, ? extends V> m) {
   this.loadFactor = DEFAULT_LOAD_FACTOR;
   putMapEntries(m, false);
}
```

比较特殊



### 确定添加元素的位置

HashMap底层是由hash表来确定位置的，然后元素的存放位置是根据hash值来确定的。但是键的 hashCode 函数返回值不一定满足，也就是说不一定在表中找得到位置。因为往往hashCode较大，所以需要一个扰动函数来处理hashCode的返回值，使得返回值满足hash表。

1.7的扰动函数

```java
//4次位运算 + 5次异或运算 
//这种算法可以防止低位不变，高位变化时，造成的 hash 冲突
static final int hash(Object k) {
   int h = 0;
   h ^= k.hashCode(); 
   h ^= (h >>> 20) ^ (h >>> 12);
   return h ^ (h >>> 7) ^ (h >>> 4);
}
```



1.8的扰动函数

JDK1.8中再次优化了这个哈希函数，把 key 的 hashCode 方法返回值右移16位，即丢弃低16位，高16位全为0 ，然后在于 hashCode 返回值做异或运算，即高 16 位与低 16 位进行异或运算，这么做可以在数组 table 的 length 比较小的时候，也能保证考虑到高低Bit都参与到 hash 的计算中，同时不会有太大的开销，扰动处理次数也从 4次位运算 + 5次异或运算 降低到 1次位运算 + 1次异或运算

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```



扰动函数只是得到了合适的hash值，并没有确定位置，也就是没得到Node[]的角标，所以需要这么一个函数

```java
static int indexFor(int h, int length) {
     return h & (length-1);  // 取模运算
}
```

jdk1.7存在这个函数，但1.8把它移动到了put方法内，效果是一样的。



我们知道HashMap底层数组的长度一定是2^n，转化为二进制即1后面全部为0。此时一个数与2^n取模等价于这个数与2^n - 1做位与运算。

![](.\pictures\hashmod.jpg)



#### 小结

通过上边的分析我们可以到如下结论：

- **在存储元素之前，HashMap 会对 key 的 hashCode 返回值做进一步扰动函数处理，1.7 中扰动函数使用了 4次位运算 + 5次异或运算，1.8 中降低到 1次位运算 + 1次异或运算**
- **扰动处理后的 hash 与 哈希表数组length -1 做位与运算得到最终元素储存的哈希桶角标位置**



### 添加元素

```java
// 可以看到具体的添加行为在 putVal 方法中进行
public V put(K key, V value) {
   return putVal(hash(key), key, value, false, true);
}
```

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
    	//如果是第一次添加，table == null，需要扩容
        if ((tab = table) == null || (n = tab.length) == 0) //注意，这里给n赋值为length
            n = (tab = resize()).length;
    	//这里就是上面所说的length与hash做与运算得到hash表的角标
    	//如果这个位置不存在其他元素，直接插入
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else { //否则执行以下逻辑
            Node<K,V> e; K k;
            //如果这个位置存在元素，且这个单链表的首元素的键与传入的键一致，则覆盖对象
            //这里没有直接覆盖，因为底下有共通逻辑，根据e是否为空判断，所以这里把e指向p
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //如果key值不想等，则判断这个位置的存储结构是不是红黑树，是的话就直接插入（即赋给e）
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //否则就从头遍历单链表
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {    //进行迭代，直到链表末尾，即next == null
                        //并且p.next指向新节点，这里e也指向新节点了，所以e != null
                        p.next = newNode(hash, key, value, null);
                        //添加完之后要判断是否达到树化阈值，是的话进行树化
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果在遍历过程中，存在元素的key与传入元素的key一致，则结束遍历
                    //此时 e = p，不为空
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e; //e 赋给 p进行迭代
                }
            }
            //如果该位置存在元素
            //且通过上述元素添加逻辑，e != null
            //则将oldValue返回
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //onlyIfAbsent一般为false,所以即将新的value覆盖旧的value
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                //这个方法在 HashMap 中是空实现，在 LinkedHashMap 中有关系
                afterNodeAccess(e);
                return oldValue;
            }
        }
    	//如果数组长度变化，修改次数+1，else逻辑不会走底下逻辑
        ++modCount;
    	//如果size+1大于了扩容阈值，则需要进行扩容
        if (++size > threshold)
            resize();
    	//HashMap中为空实现
        afterNodeInsertion(evict);
        return null;
    }
```



用图总结

![](.\pictures\hashPut.jpg)

添加元素过程：

1. 如果 `Node[] table` 表为 null ,则表示是第一次添加元素，讲构造函数也提到了，及时构造函数指定了期望初始容量，在第一次添加元素的时候也为空。这时候需要进行首次扩容过程。
2. 计算对应的键值对在 table 表中的索引位置，通过`i = (n - 1) & hash` 获得。
3. 判断索引位置是否有元素如果没有元素则直接插入到数组中。如果有元素且key 相同，则覆盖 value 值，这里判断是用的 equals 这就表示要正确的存储元素，就必须按照业务要求覆写 key 的 equals 方法，上篇文章我们也提及到了该方法重要性。
4. 如果索引位置的 key 不相同，则需要遍历单链表，如果遍历过如果有与 key 相同的节点，则保存索引，替换 Value；如果没有相同节点，则在但单链表尾部插入新节点。这里操作与1.7不同，1.7新来的节点总是在数组索引位置，而之前的元素作为下个节点拼接到新节点尾部。
5. 如果插入节点后链表的长度大于树化阈值，则需要将单链表转为红黑树。
6. 成功插入节点后，判断键值对个数是否大于扩容阈值，如果大于了则需要再次扩容。至此整个插入元素过程结束。



### 扩容过程



```java
final Node<K,V>[] resize() {
    	//oldTab指向当前table
        Node<K,V>[] oldTab = table;
    	//oldCap为当前容量，初次加载函数的话为null
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
    	//旧的扩容阈值
        int oldThr = threshold;
    	//用来存储新的容量，新的扩容阈值
        int newCap, newThr = 0;
    	//如果旧的容量不为0，则进行两倍的扩容
        if (oldCap > 0) {
            //如果旧容量 >= 最大容量
            if (oldCap >= MAXIMUM_CAPACITY) {
                //阈值直接设置为最大值并返回oldTab
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //否则进行oldCap容量扩容一倍赋值给新容量，并且得确保这个新容量的值小于最大容量
            //并且oldCap大于默认容量16
            //同时将新的阈值扩容为原来的两倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
    	//如果是使用有参的构造函数，一开始是没有oldCap的，流程走的是这里
        else if (oldThr > 0) // initial capacity was placed in threshold
            //将传入的oldThr设置为新容量
            newCap = oldThr;
    	//使用无参构造函数，新容量为默认容量，
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;  //16
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY); // 16*0.75
        }
      	//如果新的扩容阈值是0，对应的是当前 table 为空，但是有阈值的情况
        if (newThr == 0) {
            //计算新的阈值
            float ft = (float)newCap * loadFactor;
            // 如果新的容量不大于 2^30 且 ft 不大于 2^30 的时候赋值给 newThr 
       		//否则 使用 Integer.MAX_VALUE
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
    	//更新全局扩容阈值
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    	//将table设置为newTab，此时newTab为空，但已完成扩容
        table = newTab;
    	//如果oldTab为空直接返回
        if (oldTab != null) {
            //遍历老数组中每个位置的链表或者红黑树重新计算节点位置，插入新数组
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;//用来存储对应数组位置链表头节点
                //将e指向原来的元素，如果这个位置存在元素，
                if ((e = oldTab[j]) != null) {
                    //将这个位置置空，GC回收
                    oldTab[j] = null;
                    //如果数组这个位置仅有一个元素，则直接在新数组对应的位置插入这个元素
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //否则，如果e是一棵树，则进行红黑树的插入
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //否则，e是链表
                    else { // preserve order
                     	//因为扩容是容量翻倍，
                   		//原链表上的每个节点 现在可能存放在原来的下标，即low位，
                   		//或者扩容后的下标，即high位
                        
                        //低位链表的头结点、尾节点
                        Node<K,V> loHead = null, loTail = null;
                        //高位链表的头节点、尾节点
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;//用来存放原链表中的节点
                        do {
                            //e这里是[j]中的头节点，指向下一个节点
                            next = e.next;
                            //利用哈希值 & 旧的容量，可以得到哈希值取模后
                            //是大于等于 oldCap 还是小于 oldCap
                            //等于 0 代表小于 oldCap，应该存放在低位
                            //否则存放在高位（稍后有图片说明）
                            if ((e.hash & oldCap) == 0) {
                                //首次遍历，头尾节点都为e
                                if (loTail == null)
                                    //低位头节点指向e
                                    loHead = e;
                                //否则，当前尾节点的next指向e
                                else
                                    loTail.next = e;
                                //进行next
                                loTail = e;
                            }
                            //高位
                            else {
                                //首次加入高位数组，hiTail和hiHead都指向e
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null); //直到遍历完成
                        //如果低位尾节点为空
                        if (loTail != null) {
                            //尾节点的next为null
                            loTail.next = null;
                            //新数组中， 低位数组存放的位置与旧数组中的一致
                            newTab[j] = loHead;
                        }
                        //如果高位尾节点为空
                        if (hiTail != null) {
                            //尾节点的next为null
                            hiTail.next = null;
                            //新数组中， 高位数组存放的位置为旧数组+旧数组容量
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

相信大家看到扩容的整个函数后对扩容机制应该有所了解了，整体分为两部分：**1. 寻找扩容后数组的大小以及新的扩容阈值，2. 将原有哈希表拷贝到新的哈希表中**。

第一部分没的说，但是第二部分我看的有点懵逼了，但是踩在巨人的肩膀上总是比较容易的，美团的大佬们早就写过一些有关 HashMap 的源码分析文章，给了我很大的帮助。在文章的最后我会放出参考链接。下面说下我的理解：

JDK 1.8 不像 JDK1.7中会重新计算每个节点在新哈希表中的位置，而是通过 `(e.hash & oldCap) == 0`是否等于0 就可以得出原来链表中的节点在新哈希表的位置。为什么可以这样高效的得出新位置呢？

因为扩容是容量翻倍，所以原链表上的每个节点，可能存放新哈希表中在原来的下标位置， 或者扩容后的原位置偏移量为 oldCap 的位置上

![](.\pictures\hashResize.jpg)

元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

![](.\pictures\resize.jpg)

所以在 JDK1.8 中扩容后，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap。

### HashMap **查询元素**



```java
 public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        //tab用来指向table，first指向头节点，e为临时节点
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //tab指向table，且不为null，length>0
        //获取头节点，first指向头节点，且first不为null
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //如果是数组头节点，返回first
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //如果first.next不为null，e指向next，进行遍历
            if ((e = first.next) != null) {
                //如果结构是树
                if (first instanceof TreeNode)
                    //采用树的getNode
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                //否则进行单链表遍历
                do {
                    //找到这个节点则返回
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null); //否则进行next，直到next为null跳出循环
            }
        }
        return null;
    }
```



JDK 1.8新增 get 方法，在寻找 key 对应 Value 的时候如果没找大则返回指定默认值

```java
@Override
public V getOrDefault(Object key, V defaultValue) { //没找到返回默认值，替代null
   Node<K,V> e;
   return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
}
```



### HashMap 的删操作

`HashMap` 没有 `set` 方法，如果想要修改对应 key 映射的 Value ，只需要再次调用 `put` 方法就可以了。我们来看下如何移除 `HashMap` 中对应的节点的方法：

这里有两个参数需要我们提起注意：

- matchValue 如果这个值为 true 则表示只有当 Value 与第三个参数 Value 相同的时候才删除对一个的节点
- movable 这个参数在红黑树中先删除节点时候使用 true 表示删除并其他数中的节点。

```java
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }

    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        //tab为table，p为查找的数组节点，n为size
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        //tab赋值且判断是否为null，长度大于0
        //p为根据hash找到的数组内的头节点
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            //node 用来存放要移除的节点，  e 表示下个节点 k ，v 每个节点的键值
            Node<K,V> node = null, e; K k; V v;
            //同样，判断头节点是不是要找的元素
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                //是的话 node就为p
                node = p;
            //否则使用e来进行临时节点存储，并把e指向p.next
            else if ((e = p.next) != null) {
                //根据p判断是否为树
                if (p instanceof TreeNode)
                    //是得话采用树的getTreeNode
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    //否则为单链表
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            //找到该节点，并让node指向它
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null); //迭代
                }
            }
            //上述的查找都存在node
            //如果node不为空，且！matchValue || 节点值对象相同 || 节点值相同	
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                //如果是树，采用树的删除法
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                //这一个if是针对头节点的，如果是但链表node此时指向e，p.next为e
                else if (node == p)
                    //让原来头节点的next取代头节点
                    tab[index] = node.next;
                else
                    //否则p.next指向node.next(e.next)，原来p.next指向的是e
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```

### HashMap 的迭代器

我们都只我们知道 Map 和 Set 有多重迭代方式，对于 Map 遍历方式这里不展开说了，因为我们要分析迭代器的源码所以这里就给出一个使用迭代器遍历的方法：

```java
public class Test {

	
	public static void main(String[] args) {
		
		Map<String,String> map = new HashMap<>();
		map.put("a", "1");
		map.put("b", "2");
        
		//通过迭代器：先获得 key-value 对（Entry）的Iterator，再循环遍历 
		Set<Map.Entry<String, String>> s =  map.entrySet();
        //此处才获取table里的数据
		Iterator i = s.iterator();
		
		while(i.hasNext()) {
			Map.Entry<String, String> m = (Entry<String, String>) i.next();
			System.out.println(m.getKey());
			System.out.println(m.getValue());
		}
	}
}

 public Set<Map.Entry<K,V>> entrySet() {
        Set<Map.Entry<K,V>> es;
        return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
    }

    final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        //获取size
        public final int size()                 { return size; }
        //清除键值对
        public final void clear()               { HashMap.this.clear(); }
        public final Iterator<Map.Entry<K,V>> iterator() {
            return new EntryIterator();
        }
        public final boolean contains(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>) o;
            Object key = e.getKey();
            Node<K,V> candidate = getNode(hash(key), key);
            return candidate != null && candidate.equals(e);
        }
        public final boolean remove(Object o) {
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>) o;
                Object key = e.getKey();
                Object value = e.getValue();
                return removeNode(hash(key), key, value, true, true) != null;
            }
            return false;
        }
        public final Spliterator<Map.Entry<K,V>> spliterator() {
            return new EntrySpliterator<>(HashMap.this, 0, -1, 0, 0);
        }
        public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
    }

//EntryIterator 继承自 HashIterator
final class EntryIterator extends HashIterator
   implements Iterator<Map.Entry<K,V>> {
   // 这里可能是因为大家使用适配器的习惯添加了这个 next 方法
   public final Map.Entry<K,V> next() { return nextNode(); }
}

   
abstract class HashIterator {
        Node<K,V> next;        // next entry to return
        Node<K,V> current;     // current entry
        int expectedModCount;  // for fast-fail
        int index;             // current slot

        HashIterator() {
            //初始化操作数 Fast-fail 
            expectedModCount = modCount;
            // 将 Map 中的哈希表赋值给 t
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
            //从table 第一个不为空的 index 开始获取 entry
            if (t != null && size > 0) { // advance to first entry
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }
        
        public final boolean hasNext() {
            return next != null;
        }

        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            //如果当前链表节点遍历完了，则取哈希桶下一个不为null的链表头  
            //这里if条件进行遍历（进入if的条件是当前链表结束），所以如果没结束是赋值
            //然后return e
            if ((next = (current = e).next) == null && (t = table) != null) {
                //否则找下一个数组不为null的节点，next指向其
                do {} while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }
        //这里还是调用 removeNode 函数不在赘述
        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }
```

除了 `EntryIterator` 以外还有 `KeyIterator` 和 `ValueIterator` 也都继承了`HashIterator` 也代表了 HashMap 的三种不同的迭代器遍历方式。



```java
final class KeyIterator extends HashIterator
   implements Iterator<K> {
   public final K next() { return nextNode().key; }
}

final class ValueIterator extends HashIterator
   implements Iterator<V> {
   public final V next() { return nextNode().value; }
}
```

可以看出无论哪种迭代器都是通过，遍历 table 表来获取下个节点，来遍历的，遍历过程可以理解为一种深度优先遍历，即优先遍历链表节点（或者红黑树），然后在遍历其他数组位置。

### HashTable 的区别

`HashMap` 是线程不安全的，HashTable是线程安全的。

`HashMap` 允许 key 和 Vale 是 null，但是只允许一个 key 为 null,且这个元素存放在哈希表 0 角标位置。 `HashTable` 不允许key、value 是 null

`HashMap` 内部使用`hash(Object key)`扰动函数对 key 的 `hashCode` 进行扰动后作为 `hash` 值。`HashTable` 是直接使用 key 的 `hashCode()` 返回值作为 hash 值。

`HashMap`默认容量为 2^4 且容量一定是 2^n ; `HashTable` 默认容量是11,不一定是 2^n

`HashTable` 取哈希桶下标是直接用模运算,扩容时新容量是原来的2倍+1。`HashMap` 在扩容的时候是原来的两倍，且哈希桶的下标使用 &运算代替了取模。



## LinkedHashMap

### LinkedHashMap与HashMap的关系

![](.\pictures\LinkedHashMap.jpg)

从图可知，LinkedHashMap直接继承自HashMap

```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V> //直接继承，拥有hashMap的一切
    implements Map<K,V>  //标识它是map接口的子类   
```



但是，与HashMap不同的是，它除了存储结构采用数组+单链表+红黑树的结构外。每个节点（Entry）之间还采用了双向链表来维护节点顺序，解决了HashMap不能保持遍历顺序的缺陷。

同时，LinkedHashMap可以实现LRU原则。



### LinkHashMap的基础结构

#### 成员变量

```java
    private static final long serialVersionUID = 3801124242820219131L;

    //双向链表头节点
    transient LinkedHashMap.Entry<K,V> head;

    //双向链表尾节点
    transient LinkedHashMap.Entry<K,V> tail;

 	//是否保持插入排序，这里注意，因为LinkedHashMap本身在newNode已经实现了根据添加元素排序
	//所以这里的accessOrder是用于是否开启将put或get的元素放到链表末端
	//也就是说本身存在的元素，使用put进行value更新，如果accessOrder为true，则被放到末端(LRU)
	//只有一个构造器能指定accessOrder，否则默认为false
	//final确定了只能通过构造器进行赋值
    final boolean accessOrder;
```



#### 构造器

全部都调用父类HashMap的构造器，只是多了accessOrder的赋值。

```java
    public LinkedHashMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
        accessOrder = false;
    }

    public LinkedHashMap(int initialCapacity) {
        super(initialCapacity);
        accessOrder = false;
    }

    public LinkedHashMap() {
        super();
        accessOrder = false;
    }

  	public LinkedHashMap(Map<? extends K, ? extends V> m) {
        super();
        accessOrder = false;
        putMapEntries(m, false);
    }

    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {   //只有这个构造器能设置accessOrder
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;  
    }
```



### LinkedHashMap 双向链表的构建过程

![](.\pictures\LinkedHashMap2.jpg)



蓝色的线即HashMap的存储实现，而每个节点间因为有一个双向链表维护，所以多了红黄的线，表示添加顺序。

所以LinkedHashMap底层存储结构不变，只是每个节点间多了一层关系。

我们知道HashMap的Node里只有一个next来进行遍历，这在双向链表中是不够的，所以LinkedHashMap对HashMap的Node类进行了拓展。

**HashMap.Node**

```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
        ……
    }
```



**LinkedHashMap.Entry<K,V>**

```java
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```

可以看到LinkedHashMap在原基础上，添加了两个节点，分别是before,after，用于实现双向链表的前驱后继节点。

既然节点结构发生变化，则实例化应该也会改变。在HashMap添加节点时，采用的是HashMap.newNode()方法。同样的，LinkedHashMap也对这个方法进行了覆写，但本身的例如添加元素的方法是不变的。

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
    	//如果是第一次添加，table == null，需要扩容
        if ((tab = table) == null || (n = tab.length) == 0) //注意，这里给n赋值为length
            n = (tab = resize()).length;
    	//这里就是上面所说的length与hash做与运算得到hash表的角标
    	//如果这个位置不存在其他元素，直接插入
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else { //否则执行以下逻辑
            Node<K,V> e; K k;
            //如果这个位置存在元素，且这个单链表的首元素的键与传入的键一致，则覆盖对象
            //这里没有直接覆盖，因为底下有共通逻辑，根据e是否为空判断，所以这里把e指向p
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //如果key值不想等，则判断这个位置的存储结构是不是红黑树，是的话就直接插入（即赋给e）
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //否则就从头遍历单链表
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {    //进行迭代，直到链表末尾，即next == null
                        //并且p.next指向新节点，这里e也指向新节点了，所以e != null
                        p.next = newNode(hash, key, value, null);
                        //添加完之后要判断是否达到树化阈值，是的话进行树化
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果在遍历过程中，存在元素的key与传入元素的key一致，则结束遍历
                    //此时 e = p，不为空
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e; //e 赋给 p进行迭代
                }
            }
            //如果该位置存在元素
            //且通过上述元素添加逻辑，e != null
            //则将oldValue返回
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //onlyIfAbsent一般为false,所以即将新的value覆盖旧的value
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                //这个方法在 HashMap 中是空实现，在 LinkedHashMap 中有关系
                afterNodeAccess(e);
                return oldValue;
            }
        }
    	//如果数组长度变化，修改次数+1，else逻辑不会走底下逻辑
        ++modCount;
    	//如果size+1大于了扩容阈值，则需要进行扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```



**HashMap.newNode()**

```java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
        return new Node<>(hash, key, value, next);
    }
```



**LinkedHashMap.newNode()**

```java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);//调用构造器
        linkNodeLast(p);//因为构造器并没有设置before和after，所以在此设置
        return p;//返回节点
    }
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        //获取链表末端节点
        LinkedHashMap.Entry<K,V> last = tail;
        //将链表末端设置为p
        tail = p;
        //如果链表末端为null,代表链表为空链表,则将头节点也设置为p
        if (last == null)
            head = p;
        else { //否则链表不为空，则将p与上一个节点关联
            p.before = last;
            last.after = p;
        }
    }
```

![](.\pictures\LinkedHashMapAdd.jpg)`LinkedHashMap` 链表创建步骤，可用上图几个步骤来描述，蓝色部分是 `HashMap` 的方法，而橙色部分为 `LinkedHashMap` 独有的方法。

当我们创建一个新节点之后，通过`linkNodeLast`方法，将新的节点与之前双向链表的最后一个节点（tail）建立关系，在这部操作中我们仍不知道这个节点究竟储存在哈希表表的何处，但是无论他被放到什么地方，节点之间的关系都会加入双向链表。如上述图中节点 3 和节点 4 那样彼此拥有指向对方的引用，这么做就能确保了双向链表的元素之间的关系即为添加元素的顺序。



### LinkedHashMap 删除节点的操作

同样，删除节点的基本操作与HashMap一致，只是有些HashMap中是空实现的方法，在LinkedHashMap得到了实现。

```java
public V remove(Object key) {
   Node<K,V> e;
   return (e = removeNode(hash(key), key, null, false, true)) == null ?
       null : e.value;
}

// HashMap 中实现
 final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
   Node<K,V>[] tab; Node<K,V> p; int n, index;
   //判断哈希表是否为空，长度是否大于0 对应的位置上是否有元素
   if ((tab = table) != null && (n = tab.length) > 0 &&
       (p = tab[index = (n - 1) & hash]) != null) {
       
       // node 用来存放要移除的节点， e 表示下个节点 k ，v 每个节点的键值
       Node<K,V> node = null, e; K k; V v;
       //如果第一个节点就是我们要找的直接赋值给 node
       if (p.hash == hash &&
           ((k = p.key) == key || (key != null && key.equals(k))))
           node = p;
       else if ((e = p.next) != null) {
            // 遍历红黑树找到对应的节点
           if (p instanceof TreeNode)
               node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
           else {
                //遍历对应的链表找到对应的节点
               do {
                   if (e.hash == hash &&
                       ((k = e.key) == key ||
                        (key != null && key.equals(k)))) {
                       node = e;
                       break;
                   }
                   p = e;
               } while ((e = e.next) != null);
           }
       }
       // 如果找到了节点
       // !matchValue 是否不删除节点
       // (v = node.value) == value ||
                            (value != null && value.equals(v))) 节点值是否相同，
       if (node != null && (!matchValue || (v = node.value) == value ||
                            (value != null && value.equals(v)))) {
           //删除节点                 
           if (node instanceof TreeNode)
               ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
           else if (node == p)
               tab[index] = node.next;
           else
               p.next = node.next;
           ++modCount;
           --size;
           afterNodeRemoval(node);// 注意这个方法 在 Hash表的删除操作完成调用该方法
           return node;
       }
   }
   return null;
}
```

LinkedHashMap 通过调用父类的  HashMap 的 remove 方法将 Hash 表的中节点的删除操作完成即：

1. 获取对应 key 的哈希值 hash(key)，定位对应的哈希桶的位置
2. 遍历对应的哈希桶中的单链表或者红黑树找到对应 key 相同的节点，在最后删除，并返回原来的节点。

对于 `afterNodeRemoval（node）` HashMap 中是空实现，而该方法，正是 LinkedHashMap 删除对应节点在双向链表中的关系的操作：

```java
void afterNodeRemoval(Node<K,V> e) { // unlink
    //新建变量p=e，以及p的前驱变量b，后继变量a
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    //删除e本身的before和after的指向
    p.before = p.after = null;
    //如果b为空，则p为头节点，则删除p，将头节点设置为p.after，即head = a
    if (b == null)
        head = a;
    else
        //否则b的后继节点绕过p指向a
        b.after = a;
    //如果a为null，代表p为末端节点，则将tail指向b
    if (a == null)
        tail = b;
    else
        //否则a的before绕过p指向b
        a.before = b;
}
```
因此 LinkedHashMap 节点删除方式如下图步骤一样：

![](.\pictures\LinkedHashMapDel.jpg)

### LinkedHashMap 维护节点访问顺序

1. `HashMap` 的遍历结果是跟添加顺序并无关系
2. `LinkedHashMap` 的遍历结果就是添加顺序

上面有提到afterNodeAccess(E e)这个方法没说到。由于LinkedHashMap本身会根据双向链表来存储节点存入的顺序，但并不会更新这个顺序。所以它就是用来维护更新操作的节点顺序。当accessOrder 为true时，会将进行get或put操作的节点放到节点末端，以维持顺序，保证LRU原则。

看个例子：

```java
public class Test {

	
	public static void main(String[] args) {
		
		Map<String,String> map = new LinkedHashMap<>(16,0.75f,true);
		//Map<String,String> map = new HashMap<>();
		map.put("e", "5");
		map.put("b", "2");
		map.put("d", "4");
		map.put("a", "1");
		map.put("c", "3");
		Set<Map.Entry<String, String>> s =  map.entrySet();
		Iterator i = s.iterator();
		
		while(i.hasNext()) {
			Map.Entry<String, String> m = (Entry<String, String>) i.next();
			System.out.print(m.getKey());
			System.out.println(m.getValue());
		}
		
		map.get("e");
		s = map.entrySet();
		i = s.iterator();
		
		while(i.hasNext()) {
			Map.Entry<String, String> m = (Entry<String, String>) i.next();
			System.out.print(m.getKey());
			System.out.println(m.getValue());
		}
	}

//输出
e5
b2
d4
a1
c3
----------------------
b2
a1
c3
e5
d222
```

如果accessOrder为true，只要进行了put或get操作的节点，就会被放到链表末端，如果是false，只会维护第一次插入的顺序，而get和put也不会影响顺序。（注：put相同的元素跟replace方法一样，所以不列出来了）

下面看看这个方法

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
    	//当accessOrder为true且最后一个元素不为e时触发
    	//这里提一下，如果是空链表，tail为null也不等于e，与下面的a!=null不矛盾
        if (accessOrder && (last = tail) != e) {
            //设置p的初始值为e，b为p的前驱，a为p的后继
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            //直接将p.after置为空，因为p此时一定是在链表末端
            p.after = null;
            //如果b为空，代表p为头节点，则此时p移到末端，a要顶替p成为头节点
            if (b == null)
                head = a;
            //否则，p的前驱节点的after指向p的后继节点
            else
                b.after = a;
            //如果a不为空，代表p不为末端元素，将a的前驱设置为b
            if (a != null)
                a.before = b;
            //否则p为末端节点，此时只有p一个节点，把b赋给last，实际上就是把last指向null
            else
                last = b;
            //如果last = null，则head也指向null，这种情况为只有p一个元素
            if (last == null)
                head = p;
            //否则先前末端节点last的后继指向p，p的前驱指向last
            else {
                p.before = last;
                last.after = p;
            }
            //操作过后，p一定是在末端，即tail要指向p
            tail = p;
            ++modCount;
        }
    }
```

上述测试例子中是使用了 LinkedHashMap 的迭代器，由于有双向链表的存在，它相比 HashMap 遍历节点的方式更为高效，我们来对比看下两者的迭代器中的 `nextNode` 方法：

```java
// HashIterator nextNode 方法
 final Node<K,V> nextNode() {
       Node<K,V>[] t;
       Node<K,V> e = next;
       if (modCount != expectedModCount)
           throw new ConcurrentModificationException();
       if (e == null)
           throw new NoSuchElementException();
        //遍历 table 寻找下个存有元素的 hash桶   
       if ((next = (current = e).next) == null && (t = table) != null) {
           do {} while (index < t.length && (next = t[index++]) == null);
       }
       return e;
   }
   
  // LinkedHashIterator nextNode 方法
final LinkedHashMap.Entry<K,V> nextNode() {
       LinkedHashMap.Entry<K,V> e = next;
       if (modCount != expectedModCount)
           throw new ConcurrentModificationException();
       if (e == null)
           throw new NoSuchElementException();
       current = e;
       //直接指向了当前节点的 after 后驱节点
       next = e.after;
       return e;
   }
```

更为明显的我们可以查看两者的 containsValue 方法：

```java
//LinkedHashMap 中 containsValue 的实现
public boolean containsValue(Object value) {
    // 直接遍历双向链表去寻找对应的节点
   for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
       V v = e.value;
       if (v == value || (value != null && value.equals(v)))
           return true;
   }
   return false;
}
//HashMap 中 containsValue 的实现
public boolean containsValue(Object value) {
   Node<K,V>[] tab; V v;
   if ((tab = table) != null && size > 0) {
        //遍历 哈希桶索引
       for (int i = 0; i < tab.length; ++i) 
            //遍历哈希桶中链表或者红黑树
           for (Node<K,V> e = tab[i]; e != null; e = e.next) {
               if ((v = e.value) == value ||
                   (value != null && value.equals(v)))
                   return true;
           }
       }
   }
   return false;
}
```

### Java 中最简单的 LRU 构建方式

下面我们来讲解下，Java 中 LRU 算法的最简单的实现。我们还记得在每次调用 HashMap 的 putVal 方法添加完元素后还有个后置操作，`void afterNodeInsertion(boolean evict) { }` 就是这个方法。 LinkedHashMap 重写了此方法：

```java
// HashMap 中 putVal 方法实现 evict 传递的 true，表示表处于创建模式。
public V put(K key, V value) {
   return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) { .... }


//evict 由上述说明大部分情况下都传 true 表示表处于创建模式
void afterNodeInsertion(boolean evict) { // possibly remove eldest
   LinkedHashMap.Entry<K,V> first;
   //由于 evict = true 那么当链表不为空的时候 且 removeEldestEntry(first) 返回 true 的时候进入if 内部
   if (evict && (first = head) != null && removeEldestEntry(first)) {
       K key = first.key;
       removeNode(hash(key), key, null, false, true);//移除双向链表中处于 head 的节点
   }
}

 //LinkedHashMap 默认返回 false 则不删除节点。 返回 true 双向链表中处于 head 的节点
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
   return false;
}
```

由上述源码可以看出，如果如果 `removeEldestEntry(Map.Entry<K,V> eldest)` 方法返回值为 true 的时候，当我们添加一个新的元素之后，`afterNodeInsertion`这个后置操作，将会删除双向链表最初的节点，也就是 head 节点。那么我们就可以从 `removeEldestEntry` 方法入手来构建我们的 LruCache 。

```java
public class LruCache<K, V> extends LinkedHashMap<K, V> {

   private static final int MAX_NODE_NUM = 2<<4;

   private int limit;

   public LruCache() {
       this(MAX_NODE_NUM);
   }

   public LruCache(int limit) {
       super(limit, 0.75f, true);
       this.limit = limit;
   }

   public V putValue(K key, V val) {
       return put(key, val);
   }

   public V getValue(K key) {
       return get(key);
   }
   
   /**
    * 判断存储元素个数是否预定阈值
    * @return 超限返回 true，否则返回 false
    */
   @Override
   protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
       return size() > limit;
   }
}
```

我们构建了一个 `LruCache` 类， 他继承自 `LinkedHashMap` 在构建的时候，调用了 `LinkedHashMap` 的三个参数的构造方法且 `accessOrder` 传入 true，并覆写了 `removeEldestEntry` 方法，当 Map 中的节点个数超过我们预定的阈值时候在 `putValue` 将会执行 `afterNodeInsertion` 删除最近没有访问的元素。 下面我们来测试一下：

```java
//构建一个阈值为 3 的 LruCache 类
    LruCache<String,Integer> lruCache = new LruCache<>(3);
    
    
    lruCache.putValue("老大", 1);
    lruCache.putValue("老二", 2);
    lruCache.putValue("老三", 3);
    
    lruCache.getValue("老大");
    
    //超过指定 阈值 3 再次添加元素的 将会删除最近最少访问的节点
    lruCache.putValue("老四", 4);
    
    System.out.println("lruCache = " + lruCache);

```

运行结果当然是删除 key 为 "老二" 的节点：

```java
lruCache = {老三=3, 老大=1, 老四=4}
```



### 总结

本文并没有从以往的增删改查四种操作上去分析 `LinkedHashMap` 的源码，而是通过 `LinkedHashMap` 中不同于 `HashMap` 的几大特点来展开分析。

**1.LinkedHashMap 拥有与 HashMap 相同的底层哈希表结构，即数组 + 单链表 + 红黑树，也拥有相同的扩容机制。**

**2.LinkedHashMap 相比 HashMap 的拉链式存储结构，内部额外通过 Entry 维护了一个双向链表。**

**3.HashMap 元素的遍历顺序不一定与元素的插入顺序相同，而 LinkedHashMap 则通过遍历双向链表来获取元素，所以遍历顺序在一定条件下等于插入顺序。**

**4.LinkedHashMap 可以通过构造参数 accessOrder 来指定双向链表是否在元素被访问后改变其在双向链表中的位置。**