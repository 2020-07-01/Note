<!-- TOC -->

- [1. 简介](#1-简介)
- [2. 继承体系](#2-继承体系)
- [3. 存储结构](#3-存储结构)
- [4. 源码解析](#4-源码解析)
    - [4.1. 主要属性](#41-主要属性)
    - [4.2. 内部节点类](#42-内部节点类)
    - [4.3. 构造方法](#43-构造方法)
    - [4.4. 入队](#44-入队)
    - [4.5. 出队](#45-出队)
- [5. 总结](#5-总结)
- [6. ArrayBlockingQueue与LinkedBlockingQueue比较](#6-arrayblockingqueue与linkedblockingqueue比较)

<!-- /TOC -->

# 1. 简介
LinkedBlockingQueue是Java并发并发包下的一个阻塞队列，他是线程安全的。

# 2. 继承体系


# 3. 存储结构
1. 底层用单向链表进行存储
2. 默认的初始容量为int的最大值，可以指定初始容量
3. 无法进行扩容

# 4. 源码解析
## 4.1. 主要属性
```java
    //容量
    private final int capacity;

    //元素的数量
    private final AtomicInteger count = new AtomicInteger();

    //链表头
    transient Node<E> head;

    //链表尾部
    private transient Node<E> last;

    //takeLock重入锁
    private final ReentrantLock takeLock = new ReentrantLock();

    //当队列无元素的时候，take锁会阻塞在notEmpoty条件上，等待其他线程唤醒
    private final Condition notEmpty = takeLock.newCondition();

    //putLock重入锁
    private final ReentrantLock putLock = new ReentrantLock();
    
    //
    private final Condition notFull = putLock.newCondition();
```
入队与出队使用两个不同的锁，使用锁分离技术，提高效率


## 4.2. 内部节点类
此节点类是一个单向链表

```java
static class Node<E> {
    E item;
    
    Node<E> next;
    //构造方法
    Node(E x) { item = x; }
}
```

## 4.3. 构造方法
```java
public LinkedBlockingQueue() {
    //默认使用int最大值初始化容量
    this(Integer.MAX_VALUE);
}

public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    //初始化head和last指针为空值节点
    last = head = new Node<E>(null);
}
```

## 4.4. 入队
```java
public void put(E e) throws InterruptedException {
    //不允许为null值
    if (e == null) throw new NullPointerException();
    int c = -1;
    //新建节点
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    
    //获取锁除非当前线程中断
    putLock.lockInterruptibly();
    try {
        //如果队列满了，putLock锁会进行阻塞
        //等待被其他线程唤醒
        while (count.get() == capacity) {
            notFull.await();
        }

        //队列不满了就入队
        enqueue(node);
        //队列的长度加1
        c = count.getAndIncrement();
        //队列未满则唤醒线程
        if (c + 1 < capacity)
            notFull.signal();//唤醒一个线程
    } finally {
        putLock.unlock();
    }
    //唤醒notEmpty条件
    if (c == 0)
        signalNotEmpty();
}

//将元素添加在末尾
private void enqueue(Node<E> node) {
    last = last.next = node;
}
```
存储过程：
1. 添加元素使用putLock锁
2. 如果队列满了putLock锁就阻塞在notFull条件上
3. 如果队列未满则入队
4. 如果入队后队列中的元素小于其容量，则唤醒其他阻塞在notFull条件上的线程
5. 释放锁
6. 如果放元素之前，队列长度为0，则唤醒notEmpty条件阻塞takeLock元素

## 4.5. 出队
```java

public E take() throws InterruptedException {
    E x;
    int c = -1;
    //队列元素个数
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();

    try {
        //如果队列中元素为0，则唤醒notEmpty条件阻塞takeLock锁
        while (count.get() == 0) {
            notEmpty.await();
        }
        //如果队列中元素个数不为0，则进行出队
        x = dequeue();
        //获取出队前队列的长度
        c = count.getAndDecrement();
        //如果出队后还有元素，则唤醒阻塞队列中的takeLock元素
        if (c > 1)
            notEmpty.signal();
    } finally {
        //释放锁
        takeLock.unlock();
    }
    //如果出队之前队列的长度等于容量，则唤醒notFull条件阻塞putLock锁
    if (c == capacity)
        signalNotFull();
    return x;
}

//出队方法
//移除头节点
private E dequeue() {
    //头节点本身不存储任何值，将第二个节点的值返回，然后将其值置为null，并设置为头节点
    Node<E> h = head;//创建头节点
    Node<E> first = h.next;//创建第一个节点
    h.next = h; //架空此节点，等待回收

    head = first;//此时头节点为第二个节点
    E x = first.item;
    first.item = null;
    return x;
}
```


# 5. 总结
1. LinkedBlockQueue的底层实现为单链表
2. LinkedBlocingQueue采用锁分离技术，使入队和出队互补阻塞
3. LinkedBlockingQueue默认初始容量为int的最大值
4. LinkedBlockingQueue是有界队列，不进行扩容


# 6. ArrayBlockingQueue与LinkedBlockingQueue比较
1. 前者出队和入队采用同一把锁，会相互阻塞，效率低
2. 后者采用锁分离技术，出队和入队不会相互阻塞，效率高
3.后者使用默认初始值，如果出队速度跟不上入队速度，会导致占用大量内存