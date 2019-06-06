<!-- TOC -->

- [1. 简介](#1-简介)
    - [1.1. 附加](#11-附加)
- [继承体系](#继承体系)
- [2. Lock接口](#2-lock接口)
- [3. 源码解析](#3-源码解析)
    - [3.1. ReentrantLock构造方法](#31-reentrantlock构造方法)
    - [3.2. 主要内部类](#32-主要内部类)
    - [3.3. 获取公平锁](#33-获取公平锁)
    - [3.4. 获取非公平锁](#34-获取非公平锁)
    - [3.5. 获取可中断的锁lockInterruptibly()](#35-获取可中断的锁lockinterruptibly)
    - [3.6. 尝试获取锁tryLock()](#36-尝试获取锁trylock)
    - [3.7. 释放锁](#37-释放锁)
- [4. 创建条件锁](#4-创建条件锁)
    - [4.1. Condition接口](#41-condition接口)
    - [4.2. 关于AQS的内部类ConditionObject](#42-关于aqs的内部类conditionobject)
- [5. 总结](#5-总结)

<!-- /TOC -->
# 1. 简介
* ReentrantLock是一个重入锁
* 重入锁：指的是当一个线程在获取锁之后，再尝试获取锁时会自动获取锁。
* ReentrantLock继承了Lock接口，定义了获取锁和释放锁的方法，可以获取公平锁，非公平锁，条件锁，中断锁和获取一次锁尝试

## 1.1. 附加
1. 公平锁：当多个线程要获取锁时，按照线程的请求顺序来获取锁。AQS中维护基于FIFO的链表来实现，准取得说应该是当线程第一个获取锁失败的时候会进去到链表之中。
2. 非公平锁：在获取锁的时候直接获取不会进行排队
3. 条件锁：条件所是基于公平锁(非公平锁)获取的，在未满足条件时调用await()方法等待，在满足条件时调用singal()方法唤醒继续执行。
4. 中断锁：线程在获取锁的时候如果线程被设置为中断则放弃获取锁。

# 继承体系
![Alt](https://raw.githubusercontent.com/hello-qiang/Note/master/%E5%9B%BE%E7%89%87/ReentrantLock.png)

# 2. Lock接口

```java
public interface Lock {
    //获取锁
    void lock();
    //获取锁(可中断)
    void lockInterruptibly() throws InterruptedException;
    //尝试获取锁如果成功则返回true，否则返回false
    boolean tryLock();
    //尝试获取锁可以设置等待时间
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    //释放锁
    void unlock();
    //条件锁
    Condition newCondition();
}
```

# 3. 源码解析
## 3.1. ReentrantLock构造方法
* ReentrantLock默认创建非公平锁

```java
//默认创建非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}

//boolean为true时可以创建公平锁   
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

## 3.2. 主要内部类
主要内部类有三个：
* Sync:继承自AQS类，用来实现锁
* FairSync：继承自Sync，用来实现公平锁
* NonFairSync：继承自Sync，用来实现非公平锁

```java
//Sync继承自AQS类
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    //此处省略Sync类的属性和方法
    ......
}

//此类为公平锁类
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
    //调用ASQ中的方法获取一个锁
    final void lock() {
        acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}

//此类为非公平锁
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

```

## 3.3. 获取公平锁
调用ReentrantLock的构造方法创建一个重入锁实例，在创建实例的同时会创建FairSync实例
```java   
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

* 创建公平锁的流程以及所使用的方法
```java
//ReentrantLock.lock()
//目的：创建公平锁上层方法
public void lock() {
    sync.lock();
}

//FairSync.lock()方法
//目的：创建公平锁的上层方法
final void lock() {
    //AQS.acquire(1)方法
    acquire(1);
}

//AQS.acquire()
//此时如果获取失败就进行排队
//目的：获取锁
public final void acquire(int arg) {
    //调用FairSync.tryAcquire()
    //尝试获取锁
    //如果获取锁失败，则tryAcquire()为false，则&&前面为true，将会执行后面的addWaiter()方法
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}


//FairSync.tryAcquire()方法
//此时传入的值acquires为1
//目的：尝试获取锁，如果获取成功返回true，否则返回false
protected final boolean tryAcquire(int acquires) {
        //获取当前线程
        final Thread current = Thread.currentThread();
        //获取当前线程状态变量的值
        int c = getState();
        //如果当前状态变量为0则说明没有线程占有锁
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                //如果获取锁成功返回true
                return true;
            }
        }
        //如果当前线程本身占有着锁
        else if (current == getExclusiveOwnerThread()) {
            //状态变量的值加1
            int nextc = c + acquires;
            
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            //设置状态值
            setState(nextc);
            //获取锁成功返回true
            return true;
        }
        //如果未获取成功放回false
        return false;
    }


//ASQ.addWaiter()方法
//目的：将此节点添加到链表的尾部
//调用此方法说明前面获取锁失败
//此Nodo为互斥的
private Node addWaiter(Node mode) {
    //创建节点，此节点中存在当前获取锁失败的那个线程
    Node node = new Node(Thread.currentThread(), mode);
    //添加新的节点到链表的尾部使变为新的尾节点
    //如果添加成功则返回尾节点node
    Node pred = tail;
    
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //如果添加新的节点没有成功则调用此方法
    enq(node);
    //最后返回新节点
    return node;
}

//如果添加新的节点失败则调用ASQ.enq()方法
//此node为创建的新节点
private Node enq(final Node node) {
    for (;;) {//死循环
        Node t = tail;//创建尾节点
        //如果队列为空则说明还未进行初始化
        if (t == null) {  
            if (compareAndSetHead(new Node()))
                //初始化头节点和尾节点
                tail = head;
        } else {
            //如果队列已经初始化
            //此处使用了CAS操作
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}


//上面讲节点插入到队列中
//目的：尝试让当前节点获取锁
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;//失败标志
    try {
        boolean interrupted = false;//中断标志，表示未中断
        //此时使用自旋锁
        for (;;) {
            //获取前一个节点
            final Node p = node.predecessor();
            //此双向队列在尾部进行添加在头部进行出队，因此当节点在头部的时候进行获取锁
            //如果前一个节点为头节点，则尝试获取锁
            //头节点在创建时为null
            if (p == head && tryAcquire(arg)) {
                //如果获取锁成功
                //设置当前节点为node
                setHead(node);
                //并将当前节点node的上一个节点删除
                p.next = null; 
                failed = false;
                return interrupted;
            }
            //如果此节点的前一个节点不为头节点则进行阻塞，直到此节点到达头部后的第一个节点
            //判断是否需要进行阻塞，如果需要进行阻塞
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                //获取锁成功后将中断标志设置为true
                //中断设置为true
                interrupted = true;
        }
    } finally {
        //如果获取锁失败了则取消获取锁
        if (failed)
            cancelAcquire(node);
    }
}

//判断是否需要进行阻塞
//pred为此节点的前一个节点，node为当前节点
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //获取前一个节点的等待状态
    int ws = pred.waitStatus;
    //如果前一个节点为等待唤醒状态-1则返回true表示需要进行阻塞
    if (ws == Node.SIGNAL)
        return true;
    //如果前一个节点的状态大于0表示线程已取消，即已经获取到锁，则需要删除前面的节点
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        //如果前一个节点的状态为0，则设置为等待唤醒状态
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

//真正进行阻塞的方法
//如果前面的节点处于等待-1则进行阻塞
private final boolean parkAndCheckInterrupt() {
    //此处底层调用的时Unsafe.park()：用来阻塞线程
    //Unsafe.unPark()：表示唤醒线程
    LockSupport.park(this);
    return Thread.interrupted();
}

```
获取锁的过程：
1. 如果获取锁成功则直接返回true表示成功，此时用tryAcquire(int acquires)方法获取锁
2. 如果获取锁失败，则调用addWaiter(Node.EXCLUSIVE)方法创建节点添加在ASQ中队列的尾部，等待获取锁
3. 然后再此调用 acquireQueued()方法内部使用自旋锁尝试获取锁，如果成功了则直接返回true
4. 如果再次失败调用方法将节点的等待状态设置为等待唤醒(SIGNAL)
5. 调用parkAndCheckInterupt()方法阻塞当前线程(节点)
6. 如果被唤醒则继续进行获取锁的尝试，如果成功则返回true
7. 如果失败则继续阻塞在继续尝试


## 3.4. 获取非公平锁
创建非公平锁调用的是 NonfairSync.lock()方法
```java
public void lock() {
    sync.lock();
}

//NonfairSync.lock()
final void lock() {
    //直接使用原子操作设置状态值
    if (compareAndSetState(0, 1))
    //获取到锁后将当前线程设置为独占线程
        setExclusiveOwnerThread(Thread.currentThread());
    //如果没有获取到锁调用acquire(1)方法
    else
        acquire(1);
}

public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
}

//直接进行锁的获取
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) 
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

## 3.5. 获取可中断的锁lockInterruptibly()
* 此方法与lock()方法不同的是线程在获取锁的时候如果线程被标记为中断则放弃获取锁，但是lock方法不会放弃获取锁，他会在获取锁之后将线程标记为中断true
* 中断的思想就是为一个设置标记，然后根据设置的标记执行操作，比如结束线程.
* 获取中断锁时根据构造方法创建的是公平锁还是非公平锁进行获取

```java
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}

public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    //此处调用tryAcquire()方法进行获取
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}

```

## 3.6. 尝试获取锁tryLock()
* 尝试获取一次锁，成功返回true，失败返回false
* trylock()方法获取到的锁为非公平锁

```java
public boolean tryLock() {
    //此处使用非公锁的方法来获取锁
    return sync.nonfairTryAcquire(1);
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();//获取状态变量
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

## 3.7. 释放锁
* 在Node中，如果线程占有锁，则state为1，释放锁完毕后state为0，表示无锁占用
```java
//释放锁调用release(1)方法
public void unlock() {
    sync.release(1);//设置状态变量为1
}

//ASQ.realse(1)
public final boolean release(int arg) {
    //释放锁成功
    if (tryRelease(arg)) {
        //获取头节点
        Node h = head;
        //如果头节点为null并且状态不为0，表示有锁
        if (h != null && h.waitStatus != 0)
            //
            unparkSuccessor(h);
        return true;
    }
    return false;
}

//ReentrantLock.Sync.tryRelease()释放锁
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;//将状态值减1，预期结果为0
    //如果当前线程不占有锁则抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();

    boolean free = false;
    //如果状态变量变为0则说明释放了锁
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

# 4. 创建条件锁
1. 如何创建:首先要获取公平锁(非公平锁)，然后根据获取到的锁再创建条件锁
2. 思想:在不满足条件时调用await()方法使线程进入等待状态，在条件满足时激活线程
3. 条件锁比较经典的使用场景是当阻塞队列为空的时候阻塞在条件上
## 4.1. Condition接口
```java
public interface Condition {
    //使当前线程进入等待状态，直到某个条件的出现
    void await() throws InterruptedException;
    
    long awaitNanos(long nanosTimeout) throws InterruptedException;
        
    boolean await(long time, TimeUnit unit) throws InterruptedException;

    boolean awaitUntil(Date deadline) throws InterruptedException;
    //通知线程条件已经出现
    void signal();
    //唤醒所有等待的线程
    void signalAll();
}

```

## 4.2. 关于AQS的内部类ConditionObject
1. ConditionObject和Node类同为AQS中的内部类
2. 此处也维护了一个单项链表
```java
public class ConditionObject implements Condition, java.io.Serializable {
    private static final long serialVersionUID = 1173984872572414699L;
    //队列的头节点
    private transient Node firstWaiter;
    //队列的尾节点
    private transient Node lastWaiter;
    //构造方法
    public ConditionObject() { }
    
    //此处省略部分源码
    ......
}
```
* 获取条件锁的过程
```java
//ReentrantLock.newCondition
public Condition newCondition() {
    return sync.newCondition();
}

//ReentrantLock.Sync.newCondition()
final ConditionObject newCondition() {
    return new ConditionObject();
}

//ASQ.ConditionObject()
public ConditionObject() { }
```
可知，创建条件锁最后创建的是一个ConditionObject，ConditionObject是AQS中的内部类，继承于Condition接口

# 5. 总结
1. ReentrantLock中的重入锁是通过不断叠加state的值来实现的
2. 释放锁要跟获取锁匹配，即获取几次就要释放几次