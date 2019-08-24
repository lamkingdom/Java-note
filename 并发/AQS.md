# Abstract Queued Synchronizer

## 基本概念

`AQS(Abstract Queueed Synchronizer)`即队列同步器，它是Java并发用来构建锁和其他同步组件的基础框架。

> AQS的实现依赖内部的同步队列（FIFO双向队列），如果当前线程获取同步状态失败，AQS会将该线程以及等待状态等信息构造成一个Node，将其加入同步队列的尾部，同时阻塞当前线程，当同步状态释放时，唤醒队列的头节点。

AQS主要的三个变量:

```java
// 双向队列的头节点（当前持有锁的线程的节点）
private transient volatile Node head;
// 双向队列的尾节点
private transient volatile Node tail;
// 当前锁的状态(0为可被获取，大于0代表已被占用)
private volatile int state;
```

## 源码分析

### 获取锁（以ReentrantLock为例）

查看`ReentrantLock`类的源码：

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer {
       // ...
}
```

可以看到`ReentrantLock`内部的同步是依赖于`Sync`这个内部抽象类。

而`Sync`的实现类有两种，`FairSync`和`NonfairSync`，即公平锁与非公平锁。

在对代码加锁的使用的是`lock()`方法，其中`lock()`也是`Sync`的抽象方法，具体实现分别在`Sync`子类中。

这里以NonfairSync类为例（默认new出来时Sync的实现类)，看下它的Lock()的实现：

```java
  final void lock() {
    if (compareAndSetState(0, 1))	// 1
      setExclusiveOwnerThread(Thread.currentThread());	// 2
    else
      acquire(1);	// 3
  }
```

第一处：

```java
protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

运用`unsafe`的CAS将锁的状态值由0改为1，如果这个操作成功的话证明当前锁是无人持有的，可以抢占，返回true，反之返回false进入3。

第二处：

上面说到成功占有锁之后，要使用`setExclusiveOwnerThread()`方法将`exclusiveOwnerThread`的属性值改为当前线程，表示当前锁的持有者。

```java
protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
}
```

第三处：

如果第一步抢占失败的话，证明当前的锁已经被其他线程占有，需要进行下一步操作。

```java
public final void acquire(int arg) {	// 此时传入的是1
        if (!tryAcquire(arg) &&	// 4
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))	// 5
            selfInterrupt();	// 6
}
```

第四处：

```java
protected final boolean tryAcquire(int acquires) {
  			return nonfairTryAcquire(acquires);
}

final boolean nonfairTryAcquire(int acquires) {
  	// 获取当前线程
    final Thread current = Thread.currentThread();
  	// 获取锁当前的状态
    int c = getState();
  	// 如果锁的状态为0，代表可抢占（因为此时可能锁已释放，类似自旋锁）
    if (c == 0) {
      // 接下来的部分是重复上面第一处的
      if (compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
      }
    }
  	// 如果锁的状态还是不为0（大多数情况），判断是否是当前线程（可重入锁）
    else if (current == getExclusiveOwnerThread()) {
      // 累加state + acquires
      int nextc = c + acquires;
      // 判断是否越界
      if (nextc < 0) // overflow
        throw new Error("Maximum lock count exceeded");
      // 设置新的nextc为新的state
      setState(nextc);
      return true;
    }
    return false;
}
```

`tryAcquire()`是再次尝试获取锁，避免因为阻塞锁而降低效率。如果成功获取了返回true，第四步的if就不进入，反之。

第五处：

这部分应该是最复杂的，先看括号里的表达式`addWaiter(Node.EXCLUSIVE)`:

```java
// Node.EXCLUSIVE是两种Node之一，代表排他锁
// 另一种则是Node.SHARED，共享锁
private Node addWaiter(Node mode) {
  	// 这里传入的是排他锁模式，构建一个新的节点，参数为当前线程以及模式
    Node node = new Node(Thread.currentThread(), mode);
    // tail为末尾节点，因为队列要新增一个节点在末尾，取出原末尾节点存放在临时变量pred
    Node pred = tail;
  	// 如果pred为null代表head = tail = null，需要走enq(node)部分
    // 或者在CAS替换tail节点失败时，走enq()插入到队列
  	// 总之，这一步就是要确保将节点加入到队列
    if (pred != null) {
      // 将当前new出来的节点的前驱设置为pred(原tail)
      node.prev = pred;
      // 尝试使用CAS设置tail节点
      if (compareAndSetTail(pred, node)) {
        // 如果替换成功则将pred的next设置为当前节点
        pred.next = node;
        // 返回当前节点（代表成功）
        return node;
      }
    }
    enq(node);
    return node;
}

private final boolean compareAndSetTail(Node expect, Node update) {
		return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}

// 初始化队列
private Node enq(final Node node) {
  	// 可以看到这里是一个死循环，本身没有锁，可以多个线程并发访问
    for (;;) {
      Node t = tail;
      // 这里代表一个节点都还没创建，第一个线程刚刚进入
      if (t == null) { // Must initialize
        // CAS设置头节点为Node
        if (compareAndSetHead(new Node()))
          // 同时设置tail
          tail = head;
      } else {	
        // 当其他线程进入时，设置失败了，则直接走以下流程，由于是死循环，除第一个线程，其他线程只可能走此流程
        // 这个流程就是enq上面的流程
        node.prev = t;
        if (compareAndSetTail(t, node)) {
          t.next = node;
          return t;
        }
      }
    }
}
```

可以看到这个方法很巧妙，既能保证后续线程进入的高效，也能保证最初使用`enq()`方法创建队列时的并发情况，为的是确保节点添加到队列中。

下面继续分析`acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`

在`addWaiter()`部分我们将当前节点加入到了队列，成功之后返回当前节点。

```java
// 该方法主要是循环线程直到获取临界资源（可能在等待中被中断）
final boolean acquireQueued(final Node node, int arg) {	// arg本例为1
  	// 是否成功获取到临界资源
    boolean failed = true;
    try {
      // 最后返回的是该属性，是否在等待过程中被中断
      boolean interrupted = false;
      // 这里同样用了个死循环
      for (;;) {
        // predecessor为获取上一个节点，即当前节点的前驱节点，赋给p
        final Node p = node.predecessor();
        // 如果p为头节点，则代表当前节点为第二个节点，这点很重要
        // 因为除第二个节点外此时其他节点都不能获取锁
        // tryAcquire(1)再次自旋，查看是否释放锁
        if (p == head && tryAcquire(arg)) {
          // 如果已经释放则设置头节点为当前节点
          setHead(node);
          // 帮助GC收集
          p.next = null; // help GC
          // 执行成功，不需要走finally的if逻辑
          failed = false;
          // 不需要挂起线程
          return interrupted;
        }
        // 如果当前节点位置不在第二，则执行以下逻辑
        // 如果位置在第二，第一次循环没有获取到锁则将前驱节点（也就是当前持有锁的节点）状态改为-1
        // 第二次再循环就直接挂起了
        if (shouldParkAfterFailedAcquire(p, node) &&
            parkAndCheckInterrupt())
          // 设置interrupted为true
          interrupted = true;
      }
    } finally {
      if (failed)
        // 清除关联	
        cancelAcquire(node);
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
      // 前继节点还在等待触发,还没当前节点的什么事儿,所以当前节点可以被park
      return true;
    if (ws > 0) {
      // 前继节点是CANCELLED ,则需要充同步队列中删除,并检测新接上的前继节点的状态
      // 若还是为CANCELLED ,还需要重复上述步骤
      // 直到找到状态不为CANCELLED的节点，并将这个节点与当前节点node关联
      do {
        node.prev = pred = pred.prev;
      } while (pred.waitStatus > 0);
      pred.next = node;
    } else {
      // 到这一步,waitstatus只有可能有2种状态,一个是0,一个是PROPAGATE
      // 无论是哪个都需要把前继节点的状态设置为SIGNAL
      compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

// 调用LockSupport阻塞线程
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

第六处：

如果线程在等待获取中被阻塞或中断则会执行此逻辑。

### 释放锁（以ReentrantLock为例）

```java
public void unlock() {
  	sync.release(1);
}

public final boolean release(int arg) {
    if (tryRelease(arg)) {	// 1
      // 设置锁状态成功则进入释放锁阶段
      Node h = head;
      if (h != null && h.waitStatus != 0)
        unparkSuccessor(h);	// 2
      return true;
    }
    return false;
}
```

第一处：

```java
protected final boolean tryRelease(int releases) {
  	// 计算新的state值
    int c = getState() - releases;
  	// 如果当前线程不是排他锁的持有者则抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
      throw new IllegalMonitorStateException();
  	// state值为0时释放成功，否则失败
    boolean free = false;
    if (c == 0) {
      // state为0时可以释放锁
      free = true;
      // 将锁的持有者设置为null
      setExclusiveOwnerThread(null);
    }
  	// 重新设置Lock的state
    setState(c);
  	// 返回Lock类本身释放前的处理结果是否成功
    return free;
}
```

第二处：

这边才是真正的调用系统释放锁

```java
private void unparkSuccessor(Node node) {
    // 获取head节点的状态    
    int ws = node.waitStatus;
  	// 如果小于0代表正常，替换回0
    if (ws < 0)
      compareAndSetWaitStatus(node, ws, 0);
    // 获取head节点的下一个节点
    Node s = node.next;
  	// 如果为空或者waitStatus > 0，则代表该节点不满足要求，需要从末尾向前排查至最近的一个正常节点或排查失败
  	// 失败则代表队列不存在需要唤醒的线程了
    if (s == null || s.waitStatus > 0) {
      s = null;
      for (Node t = tail; t != null && t != node; t = t.prev)
        if (t.waitStatus <= 0)
          s = t;
    }
  	// 如果不为null则唤醒head的后继节点线程
    if (s != null)
      LockSupport.unpark(s.thread);
}
```

## 放弃竞争

`parkAndCheckInterrupt()`中有`cancelAcquire()`方法会将`waitStatus`设置为`CANCELLED`，然而实际上这只是一个兜底的操作，除了异常的情况，是不会走到这里的。

如果需要主动放弃竞争锁，**不采用`lock.lock()`方法,而是`lock.lockInterruptibly()`**。

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        // 有线程中断，直接抛出中断异常
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}

private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    // 尾插节点成功
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 重点在这里，一旦出现了异常，则立马抛出该异常情况
                // 此时failed 字段肯定是true了
                // 需要进入到下面的cancelAcquire 方法中去取消竞争了
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            // 开始取消竞争
            cancelAcquire(node);
    }
}
```

看一下`cancelAcquire()`方法：

```java
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

    node.thread = null;
    // 节点线程数据设置为null

    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    // 从当前节点的前置节点开始扫描去掉废弃的节点，这样操作不会出现并发问题

    Node predNext = pred.next;

    // 设置为取消状态！
    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) {
        // node是尾部节点，而且通过CAS自旋成功，设置pred为tail
        // pred的next为nul，可以彻底和node 节点切割，node节点等待GC回收
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        if (pred != head &&
            // 前节点不是头部节点，而且包含的线程不为null（上面就有操作设置线程为null的情况）
            // 或者节点状态可以设置为-1
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            // 这样说么下个节点是可以串在一起的！
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
                // CAS 操作，设置后置节点
             // 这个操作就相当去双写链表去掉node节点   
        } else {
            unparkSuccessor(node);
            // 唤醒node的后置节点
            // 注意！此时node节点还在链表中
            // 他的真正移除是依靠node后置节点的shouldParkAfterFailedAcquire 去移除节点操作完成的
        }
        node.next = node; // help GC
    }
}
```

总结一下

- 如果尾部节点或者中间节点就依靠从链表中脱离操作实现放弃竞争
- 如果是头部的下一个节点，是依靠唤醒下一个节点后进行的链表清楚操作完成

## 参考资料

https://segmentfault.com/a/1190000008471362#articleHeader5

https://www.cnblogs.com/waterystone/p/4920797.html

https://www.jianshu.com/p/282bdb57e343