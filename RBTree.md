

# 红黑树

## 定义

红黑树是一种近乎平衡的二叉搜索树，**它能够确保任何一个节点的左右子树的高度差不会超过二者中较低那个的一倍**，解决了二叉搜索树不平衡的问题。

集合框架中TreeMap就是基于红黑树实现的，TreeSet是基于TreeMap的。

JDK1.8之后，HashMap底层存储结构也由1.7的数组+单链表变成了数组+单链表+红黑树。



**红黑树是满足如下条件的二叉搜索树：**

1.每个节点都具有颜色，颜色只能是黑色或红色。

2.根节点为黑色。

3.每个叶节点（NIL节点，空节点）是黑色的。

4.不能有连续的两个红色节点。

5.从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点。



## 预备知识

红黑树是一种数据存储结构，所以会有增加元素及删除元素的操作，而这两种操作在绝大多数情况下会破坏红黑树的平衡，即破坏上述的条件。所以在操作完成后，要对红黑树进行修复，主要操作有以下两种：

**1.颜色变换**

**2.旋转**



### 节点结构



```java
    
private static final boolean RED   = false;
private static final boolean BLACK = true;

static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;       			//键，根据key来进行二分搜索
        V value;				//值，节点存储（指向)的对象
        Entry<K,V> left;		//左孩子
        Entry<K,V> right;		//右孩子
        Entry<K,V> parent;		//双亲节点
        boolean color = BLACK;	//颜色，是否为黑色

        /**
         * Make a new cell with given key, value, and parent, and with
         * {@code null} child links, and BLACK color.
         */
    	//构造器
        Entry(K key, V value, Entry<K,V> parent) {
            this.key = key;
            this.value = value;
            this.parent = parent;
        }

        /**
         * Returns the key.
         *
         * @return the key
         */
        public K getKey() {
            return key;
        }

        /**
         * Returns the value associated with the key.
         *
         * @return the value associated with the key
         */
        public V getValue() {
            return value;
        }

        /**
         * Replaces the value currently associated with the key with the given
         * value.
         *
         * @return the value associated with the key before this method was
         *         called
         */
        public V setValue(V value) {
            V oldValue = this.value;
            this.value = value;
            return oldValue;
        }

        public boolean equals(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;

            return valEquals(key,e.getKey()) && valEquals(value,e.getValue());
        }
		
    	//因为TreeMap也是散列表
        public int hashCode() {
            int keyHash = (key==null ? 0 : key.hashCode());
            int valueHash = (value==null ? 0 : value.hashCode());
            return keyHash ^ valueHash;
        }

        public String toString() {
            return key + "=" + value;
        }
    }
```



### 变色

这个比较好理解，就是把节点的颜色直接变成我们需要的颜色，例如将一个节点从黑色变成红色



### 旋转

往往直接变色并不能使红黑树平衡，所以还要通过旋转红黑树才能使其平衡。旋转分为两种：



#### 左旋

将当前节点x的右子树y绕x逆时针旋转至当前节点x的位置，将右子树y的左子树B赋给x（此时x的右子树指向从y变成了B，B的双亲节点也从y变为x），再将x的双亲节点赋给y，同时x的双亲的左或右子树指向y。最后将y的左子树指向x，x的双亲节点指向y。如图：

![](.\pictures\TreeMap_rotateLeft.png)



代码实现：

```java
private void rotateLeft(Entry<K,V> p) {
    
    if (p != null) {
        //这里不判断非空的原因是因为左旋右旋在插入、删除时，如果右子树为空，则直接插入或删除即可
        //并不会走左旋右旋逻辑，即隐含了在外部已经做了 right != null的判断
        Entry<K,V> r = p.right; 
        p.right = r.left;		//p的右子树指向变为right的左子树
        //r的左子树可能为空，此时p的右子树就为空，则不需要更改节点关系
        //但p.right一定是会有个指向的，即便为null，所以不包含在这个逻辑内
        if (r.left != null) {   
            
            r.left.parent = p;	//r的左子树的双亲节点由r变为p
        }
        
        //r的双亲指向为p的双亲，这里也是一定会执行的，如果p.parent为null，则代表p为根节点
        r.parent = p.parent;
        if (p.parent == null) {
            
            root = r;		//如果p为根节点，则左旋后将r设为根节点
        } else if (p.parent.left == p) { //否则p不是跟节点，则有双亲，判断p是双亲的左还是右
            
            p.parent.left = r;
        } else {
            
            p.parent.right = r;
        }
        
        //最后改变p与r的关系
        r.left = p;
        p.parent = r;
    }
}
```



#### 右旋

将当前节点x的左子树y绕x顺时针旋转至当前节点x的位置，将左子树y的右子树B赋给x（此时x的左子树指向从y变成了B，B的双亲节点也从y变为x），再将x的双亲节点赋给y，同时x的双亲的左或右子树指向y。最后将y的右子树指向x，x的双亲节点指向y。如图：



![](.\pictures\TreeMap_rotateRight.png)



代码实现：

```java
private void rotateRight(Entry<K,V> p) {
	
    if (p != null) {
        
        Entry<K,V> l = p.left;
        p.left = l.right;
        if (l.right != null) {
            l.right.parent = p;
        }
        
        l.parent = p.parent;
        if (p.parent == null) {
            root = l;
        } else if (p.parent.left == p) {
            p.parent.left = l;
        } else {
            p.parent.right = l;
        }
        
        l.right = p;
        p.parent = l;
    }
}
```

### 寻找节点后继

用于红黑树的删除。对于一棵二叉查找树，其后继（树中大于t的最小元素）可以通过如下方式找到：

1.如果t的右子树不为空，则t的后继是其右子树的最小元素。（也就是t的最小左子树的左孩子）

2.如果t的右子树为空，则t的后继是第一个向左走的节点的祖先。（也就是第一个是左子树节点的祖先）



![](.\pictures\TreeMap_successor.png)



代码如下：

```java
// 寻找节点后继函数successor()
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    if (t == null)
        return null;
    else if (t.right != null) {// 1. t的右子树不空，则t的后继是其右子树中最小的那个元素
        Entry<K,V> p = t.right;
        while (p.left != null)
            p = p.left;
        return p;
    } else {// 2. t的右孩子为空，则t的后继是其第一个向左走的祖先
        Entry<K,V> p = t.parent;
        Entry<K,V> ch = t;
        while (p != null && ch == p.right) {
            ch = p;
            p = p.parent;
        }
        return p;
    }
}
```



## 方法剖析

### get()

`get(Object key)`方法根据指定的`key`值返回对应的`value`，该方法调用了`getEntry(Object key)`得到相应的`entry`，然后返回`entry.value`。因此`getEntry()`是算法的核心。算法思想是根据`key`的自然顺序（或者比较器顺序）对二叉查找树进行查找，直到找到满足`k.compareTo(p.key) == 0`的`entry`。

![](.\pictures\TreeMap_getEntry.png)



代码如下：

```java
//getEntry()方法
final Entry<K,V> getEntry(Object key) {
    ......
    if (key == null)//不允许key值为null
        throw new NullPointerException();
    Comparable<? super K> k = (Comparable<? super K>) key;//使用元素的自然顺序
    Entry<K,V> p = root;
    while (p != null) {  		//到叶子节点时，下一次就为null
        int cmp = k.compareTo(p.key);
        if (cmp < 0)//向左找
            p = p.left;
        else if (cmp > 0)//向右找
            p = p.right;
        else			//cmp == 0	
            return p;
    }
    return null;
}
```



### put()

添加或更新元素。

`put(K key, V value)`方法是将指定的`key`, `value`对添加到`map`里。该方法首先会对`map`做一次查找，看是否包含该元组，如果已经包含则直接返回，查找过程类似于`getEntry()`方法；如果没有找到则会在红黑树中插入新的`entry`，如果插入之后破坏了红黑树的约束条件，还需要进行调整（旋转，改变某些节点的颜色）。



代码如下：

```java
public V put(K key, V value) {
    
    Entry<K,V> t = root;
    if (t == null) {		// 根节点为空，则为空树
         compare(key, key); // type (and possibly null) check

         root = new Entry<>(key, value, null);
         size = 1;
         modCount++;
         return null;
    }

    int cmp;
    Entry<K,V> parent;
    if (key == null)
        throw new NullPointerException();
    Comparable<? super K> k = (Comparable<? super K>) key;//使用元素的自然顺序
    do {
        parent = t;				//parent一开始指向root，随后指向t
        cmp = k.compareTo(t.key);
        if (cmp < 0) t = t.left;//向左找，t指向t的左子树
        else if (cmp > 0) t = t.right;//向右找，t指向t的右子树
        else return t.setValue(value);//替换原来的value
    } while (t != null);		//直到t为叶子节点的左或右子树
    Entry<K,V> e = new Entry<>(key, value, parent);//创建并插入新的entry
    if (cmp < 0) parent.left = e;
    else parent.right = e;
    fixAfterInsertion(e);//调整
    size++;
    return null;
}
```

上述代码的插入部分并不难理解：首先在红黑树上找到合适的位置，然后创建新的`entry`并插入（当然，新插入的节点一定是树的叶子）。难点是调整函数`fixAfterInsertion()`，前面已经说过，调整往往需要1.改变某些节点的颜色，2.对某些节点进行旋转。



#### 情况说明

**思想：将红色的不平衡点移到根节点进行换色**

先说几种比较简单的：P为parent节点，U为uncle节点，G为grandfather节点，N为当前节点

*因为插入的是红色节点，如果P为黑色，则直接插入不影响。

情况一：P和U都为红色（此时G一定是黑色才能证明原来的平衡）

​	思路：N为红色，P和N此时都为红色，违反规则4。则把P涂黑（不把N涂黑是因为要把不平衡节点往上移动，而且把N涂黑此时这条路线的黑色节点数多了1,并且不能进行处理），此时G->P的黑色节点数增加了1,为了使规则5平衡，把G变为红色。此时整棵树除了U这条都平衡了（U的树黑色节点数少了G这个点，所以少了1），所以把U涂黑，G设为当前节点N。进入情况2或情况3

​	操作：将P和U设置为黑色，G设置为红色，G设置为当前节点。

![](.\pictures\rbtree-add-case1.png)

如果N（之前的G）的P为黑色，则整棵树其实是平衡的，这种情况就不用往下操作了。否则，当P为红色时，又违反了规则4。如果P和U依然都是红，则走情况一。否则：U为黑色，且N为P的右子树进入情况2。

情况二：P为红色，U为黑色，且N为P的右子树。

​	思路：此时N是红色（情况1的结果），与P都是红色。由于原先的子树都是完好的，所以秉持把多出的节点往上推的思想，把N往上，此时N是P的右子树，所以以P为节点，进行左旋，并把P作为当前节点N。(之所以不以N进行右旋，为了保持一个三级结构）此时规则5是没有违背的，因为G->P的路径上的黑色节点数与G->N的路径上黑色节点数一致。所以左旋完后原P（先N）的红黑树结构完整。

​	操作：将当前节点设为P，以P中心进行左旋，左旋完不是情况一就是进入情况三。

![](.\pictures\rbtree-add-case2.png)

情况三：N为红色（原P左旋后的位置），P为红色，U为黑色，且N为P的左子树。

​	思路：此时P和N依旧违背规则4，但整棵树都还满足规则5。为了解决规则4，我们必须把红色节点放到根节点，并涂黑。由于G的右子树（这里讨论的都是G的左侧情况）的黑色节点数是平衡的，所以如果将G涂成红色，并不影响。然后根据我们的思想，将P（问题节点）放到G的位置，此时需要以G为节点进行右旋。由于P为红色，所以他的孩子离开他（到一个红色的节点）依然满足规则5，进行右旋。此时P在根节点的位置，颜色是红色，G在原U的位置，颜色黑色。但是这样违背了规则2，所以将根节点涂黑，但此时又违背了规则5。右侧路径黑色节点数比左侧多1个。所以将G涂成黑色，结束。

![](.\pictures\rbtree-add-case3.png)



![](.\pictures\TreeMap_put.png)



代码如下：

```java
private void fixAfterInsertion(Entry<K,V> x) {
    
    x.color = RED; //将传入的节点设置为红色，这样只违背规则4
    
    while(x != null && x != root && x.parent.color == RED) {//不为空，不为根，且父节点是红色
        
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) { //父节点是祖父节点的左子树
           
            Entry<K,V> y = rightOf(parentOf(parentOf(x))); 	//叔叔节点
            if (colorOf(y) == RED) {						//情况一
                
                setColor(parentOf(x), BLACK);				//父亲节点设置为红色
                setColor(y, BLACK);							//叔叔节点设置为红色
                setColor(parentOf(parentOf(x)), RED);		//祖父节点设置为黑色
                x = parentOf(parentOf(x));					//当前节点设置为祖父节点
            } else {
                
                if (x == rightOf(parentOf(x))) {			//情况二
               
                    x = parentOf(x);						//当前节点设置为父亲节点
                    rotateLeft(x);							//父亲节点左旋
                }
                setColor(parentOf(x), BLACK);				//情况三，父亲节点设置为黑色
                setColor(parentOf(parentOf(x)), RED);		//情况三，祖父节点设置为红色
                rotateRight(parentOf(parentOf(x)));			//情况三，右旋祖父节点
            }
        } else {											//父节点是祖父节点的左子树
            
            Entry<K,V> y = leftOf(parentOf(parentOf(x))); 	//叔叔节点
            if (colorOf(y) == RED) {						//情况四
                
                setColor(parentOf(x), BLACK);				//父亲节点设置为红色
                setColor(y, BLACK);							//叔叔节点设置为红色
                setColor(parentOf(parentOf(x)), RED);		//祖父节点设置为黑色
                x = parentOf(parentOf(x));					//当前节点设置为祖父节点
            } else {
                
                if (x == leftOf(parentOf(x))) {				//情况五
                	
                    x = parentOf(x);
                    rotateRight(x);							//右旋当前节点
                }
                              
                setColor(parentOf(x), BLACK);				//情况六，父亲节点设置为黑色
                setColor(parentOf(parentOf(x)), RED);		//情况六，祖父节点设置为红色
                rotateLeft(parentOf(parentOf(x)));			//情况六，左旋祖父节点
            }
        }
    }
}
```



### remove()

`remove(Object key)`的作用是删除`key`值对应的`entry`，该方法首先通过上文中提到的`getEntry(Object key)`方法找到`key`值对应的`entry`，然后调用`deleteEntry(Entry<K,V> entry)`删除对应的`entry`。由于删除操作会改变红黑树的结构，有可能破坏红黑树的约束条件，因此有可能要进行调整。

`getEntry()`函数前面已经讲解过，这里重点放`deleteEntry()`上，该函数删除指定的`entry`并在红黑树的约束被破坏时进行调用`fixAfterDeletion(Entry<K,V> x)`进行调整。

**由于红黑树是一棵增强版的二叉查找树，红黑树的删除操作跟普通二叉查找树的删除操作也就非常相似，唯一的区别是红黑树在节点删除之后可能需要进行调整**。现在考虑一棵普通二叉查找树的删除过程，可以简单分为两种情况：

1. 删除点p的左右子树都为空，或者只有一棵子树非空。
2. 删除点p的左右子树都非空。

对于上述情况1，处理起来比较简单，直接将p删除（左右子树都为空时），或者用非空子树替代p（只有一棵子树非空时）；对于情况2，可以用p的后继s（树中大于x的最小的那个元素）代替p，然后使用情况1删除s（此时s一定满足情况1.可以画画看）。

基于以上逻辑，红黑树的节点删除函数`deleteEntry()`代码如下：

```java
// 红黑树entry删除函数deleteEntry()
private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;
    if (p.left != null && p.right != null) {// 2. 删除点p的左右子树都非空。
        Entry<K,V> s = successor(p);// 寻找p的后继来替代p被删除
        p.key = s.key;				// p的key修改为后继节点s的key
        p.value = s.value;			// p的value修改为后继节点s的value
        p = s;						// 此时p已经修改完毕，变成s。但我们要把原来的s删除掉，p->s
    }
    Entry<K,V> replacement = (p.left != null ? p.left : p.right); //获取替代父节点的子节点
    if (replacement != null) {			// 1. 删除点p只有一棵子树非空。
        replacement.parent = p.parent;	// 将p的父节点的p唯一的子树关联
        if (p.parent == null)			// 如果p.parent为null，代表p为根节点
            root = replacement;			// 将p的子树设为根节点
        else if (p == p.parent.left)	// 否则判断p是其父节点的左子树还是右子树
            p.parent.left  = replacement;	//并把p父节点的相应子树设置为p的子树
        else
            p.parent.right = replacement;
        p.left = p.right = p.parent = null;	// 将p置空（如果是两子树都不空则是s置空）
        if (p.color == BLACK)				// 删除黑色节点才会影响结构	
            fixAfterDeletion(replacement);	// 调整
    } else if (p.parent == null) {			// 或者p本身是根节点	
        root = null;						// 则将根节点置为空
    } else { 								// 删除点p的左右子树都为空
        if (p.color == BLACK)				// 如果删除的节点是黑色
            fixAfterDeletion(p);			// 调整
        if (p.parent != null) {				// 如果p父节点不为空
            if (p == p.parent.left)			// 将p父节点相应的位置置空
                p.parent.left = null;
            else if (p == p.parent.right)
                p.parent.right = null;
            p.parent = null;				//将p的parent属性置空
        }
    }
}
```

上述代码中占据大量代码行的，是用来修改父子节点间引用关系的代码，其逻辑并不难理解。下面着重讲解删除后调整函数`fixAfterDeletion()`。首先请思考一下，删除了哪些点才会导致调整？**只有删除点是BLACK的时候，才会触发调整函数**，因为删除RED节点不会破坏红黑树的任何约束，而删除BLACK节点会破坏规则4。

跟上文中讲过的`fixAfterInsertion()`函数一样，这里也要分成若干种情况。记住，**无论有多少情况，具体的调整操作只有两种：1.改变某些节点的颜色，2.对某些节点进行旋转。**

![](.\pictures\TreeMap_fixAfterDeletion.png)

上述图解的总体思想是：将情况1首先转换成情况2，或者转换成情况3和情况4。当然，该图解并不意味着调整过程一定是从情况1开始。通过后续代码我们还会发现几个有趣的规则：a).如果是由情况1之后紧接着进入的情况2，那么情况2之后一定会退出循环（因为x为红色）；b).一旦进入情况3和情况4，一定会退出循环（因为x为root）。

删除后调整函数`fixAfterDeletion()`的具体代码如下，其中用到了上文中提到的`rotateLeft()`和`rotateRight()`函数。通过代码我们能够看到，情况3其实是落在情况4内的。情况5～情况8跟前四种情况是对称的，因此图解中并没有画出后四种情况，读者可以参考代码自行理解。



#### 情况说明：

情况一：x是黑色节点，其兄弟节点是红色。（则其父亲节点为黑色，兄弟节点的两个孩子也为黑色）

​	思路：这步只是为了进入其他case

​	操作：将P节点设置为红色，B（兄弟节点）设置为黑色，P进行左旋（x是左子树情况下），左旋后重置x的兄弟节点。

![](.\pictures\rbtree-del-case1.png)



情况二：x是黑色节点，其兄弟节点是黑色，兄弟节点的两个子节点也是黑色。

​	思想：此时x所在的路径黑色节点树依旧少1，若此时把P换为黑色则数量回归正常，但我们刚刚进行了左旋，所以B也在P的子树中，如果P变为黑色，则依旧违反规则5,此时只需要把B改为红色即可。然后把x变为P，此时X还没换色，因为X为红色已经不需要改变。但此时左边所有黑色节点数少1，所以把X改为黑色退出即可。

​	操作：将B改为红色，把P作为当前节点。最后把当前节点改为黑色。



![](.\pictures\rbtree-del-case2.png)

情况三：x是黑色节点，其兄弟节点是黑色，兄弟节点的左子树为红色，右子树为黑色。

​	思想：进入情况4，要使得B的右孩子为红色，则需要进行右旋，使得B的左孩子上位（因为他的颜色是红色）	，旋转对他的子树不会产生影响，也说明他的子树和B的右子树（黑色）节点一致，此时左孩子到了B的位置，颜色为红色。为了保证不违反规则4，则将P涂为黑色。此时右子树节点的黑色点就多了一，又要确保右孩子为红色，所以将B的右孩子（原先的B变为红色），进入情况四。

​	操作：将B的左孩子涂为黑色，将B改为红色，对B进行右旋，将P的右孩子（此时为B的左子树）设置为B。



![](.\pictures\rbtree-add-case3.png)

情况四：x是黑色节点，其兄弟节点是黑色，兄弟节点的左子树颜色任意，右子树为红色。

​	思想：此时p的左孩子黑色数目依旧少1，所以将P涂成黑色，为了保护规则5，将P的颜色赋给B，此时为了保证平衡，将P进行左旋，左旋完成后原B的右子树由于B变为黑色（它是红色），数目少1，为了保证规则5，将B的右子树涂黑，完成。

​	操作：将父节点的颜色赋给B，将父节点设置为黑色，对父节点进行左旋，将B的右子树变为黑色。



![](.\pictures\rbtree-del-case1.png)



代码如下：

```java
private void fixAfterDeletion(Entry<K,V> x) {
    while (x != root && colorOf(x) == BLACK) {
        if (x == leftOf(parentOf(x))) {					//如果当前节点是父节点的左孩子
            Entry<K,V> sib = rightOf(parentOf(x));		//获得兄弟节点
            if (colorOf(sib) == RED) {					//如果兄弟节点是红色，情况一
                setColor(sib, BLACK);                   // B设为黑色
                setColor(parentOf(x), RED);             // P设为红色
                rotateLeft(parentOf(x));                // 以P为中心进行左旋
                sib = rightOf(parentOf(x));             // 从新设置B为P当前的右孩子
            }
            if (colorOf(leftOf(sib))  == BLACK &&		// 如果B的左右孩子都为黑色
                colorOf(rightOf(sib)) == BLACK) {		// 情况二
                setColor(sib, RED);                     // 设置B为红色
                x = parentOf(x);                   // 当前节点设置为父节点，此时x为红色，跳出
            } else {
                if (colorOf(rightOf(sib)) == BLACK) {	// 如果B的左孩子为红，右孩子为黑
                    setColor(leftOf(sib), BLACK);       // 情况三，将左孩子涂黑
                    setColor(sib, RED);                 // B设置为红色
                    rotateRight(sib);                   // 以B为中心进行右旋
                    sib = rightOf(parentOf(x));         // 设置B为B的右孩子
                }										// 情况四，左孩子任意，右孩子为红色
                setColor(sib, colorOf(parentOf(x)));    // 将P的颜色赋给B
                setColor(parentOf(x), BLACK);           // 将P涂成黑色
                setColor(rightOf(sib), BLACK);          // 将B的右孩子从红色涂成黑色
                rotateLeft(parentOf(x));                // 以P为中心进行左旋
                x = root;                               // 将x设置为root，跳出循环
            }
        } else { // 跟前四种情况对称
            Entry<K,V> sib = leftOf(parentOf(x));
            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);                   // 情况5
                setColor(parentOf(x), RED);             // 情况5
                rotateRight(parentOf(x));               // 情况5
                sib = leftOf(parentOf(x));              // 情况5
            }
            if (colorOf(rightOf(sib)) == BLACK &&
                colorOf(leftOf(sib)) == BLACK) {
                setColor(sib, RED);                     // 情况6
                x = parentOf(x);                        // 情况6
            } else {
                if (colorOf(leftOf(sib)) == BLACK) {
                    setColor(rightOf(sib), BLACK);      // 情况7
                    setColor(sib, RED);                 // 情况7
                    rotateLeft(sib);                    // 情况7
                    sib = leftOf(parentOf(x));          // 情况7
                }
                setColor(sib, colorOf(parentOf(x)));    // 情况8
                setColor(parentOf(x), BLACK);           // 情况8
                setColor(leftOf(sib), BLACK);           // 情况8
                rotateRight(parentOf(x));               // 情况8
                x = root;                               // 情况8
            }
        }
    }
    setColor(x, BLACK);									//此时x是根节点，一定要确保是黑色
}
```

