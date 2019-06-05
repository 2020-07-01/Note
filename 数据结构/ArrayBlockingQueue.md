
<!-- TOC -->

- [1. 简介](#1-简介)
- [2. 继承体系](#2-继承体系)
- [3. 存储结构](#3-存储结构)
- [4. 源码解析](#4-源码解析)
    - [4.1. 主要属性](#41-主要属性)
    - [4.2. 构造方法](#42-构造方法)
    - [4.3. 入队](#43-入队)
    - [4.4. 出队](#44-出队)
- [5. 总结](#5-总结)

<!-- /TOC -->
# 1. 简介
ArrayBlockingQueue是Java并发包下的一个以数组实现的有界阻塞队列，他是线程安全的，不需要进行扩容。


# 2. 继承体系
![Alt](https://raw.githubusercontent.com/hello-qiang/Note/master/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/%E5%9B%BE%E7%89%87/ArrayBlockingQueue.png?token=AKE36TE3BQP2YWZ5EPMCIYK447UKI)


# 3. 存储结构
ArrayBlockingQueue的底层实现是确定的容量的数组，采用入队指针与出队指针循环利用数组。

# 4. 源码解析
## 4.1. 主要属性
```java
    
    //存储元素的数组
    final Object[] items;
    //存元素的指针
    int takeIndex;
    //放元素的指针
    int putIndex;
    //元素的容量
    int count;
    //重入锁
    final ReentrantLock lock;
    //非空条件
    private final Condition notEmpty;
    //非满条件
    private final Condition notFull;
    
    transient Itrs itrs = null;
```

## 4.2. 构造方法

```java
//以指定的容量进行初始
//默认为false，
public ArrayBlockingQueue(int capacity) {
    this(capacity, false);//
}


public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    //创建数组
    this.items = new Object[capacity];
    //创建锁
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```
1.ArrayBlockingQueue初始化时必须指定容量
2.在初始化时可以指定公平锁还是非公平锁

## 4.3. 入队
入队有四种方法分别为:
add(E e)：添加成功返回true，队列抛出异常
offer(E e)：添加成功返回true，否则返回false
put(E e)：不进行返回，如果队列满了则进行阻塞等待
offer(E e,long timeout,TimeUnit unit)：可以设置等待时间，在数组满了进行等待，时间到了队列依然满则返回false

```java

//如果队列已满抛出异常，否则返回true
public boolean add(E e) {
    //调用父类的add方法
    return super.add(e);
}

//super.add()
public boolean add(E e) {
    //使用了接口的offer()方法，此方法在此类中进行实现
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}

//实现queue接口的offer方法
//如果队列已满返回false，否则返回true
public boolean offer(E e) {
    //检查添加的元素不能为null
    checkNotNull(e);
    
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    try {
        //如果数组满了返回false
        if (count == items.length)
            return false;
        //如果数组没满添加元素返回true
        else {
            enqueue(e);
            return true;
        }
    } finally {//释放锁
        lock.unlock();
    }
}

//往数组中添加元素
private void enqueue(E x) {
    //创建添加元素的数组
    final Object[] items = this.items;
    //把元素直接放在指针的位置上
    items[putIndex] = x;
    //如果指针到尽头了，则指针返回头部
    if (++putIndex == items.length)
        putIndex = 0;
    //元素数量加1
    count++;
    //此时元素不为空
    notEmpty.signal();
}

//使用put方法添加元素
public void put(E e) throws InterruptedException {
    //检查元素是否为null
    checkNotNull(e);
    //加锁
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    //如果数组满了进行线程等待
    try {//使用while是因为会有其他线程也在等待，如果释放一个元素后被其他线程抢先，则此线程继续进行等待
        while (count == items.length)
            notFull.await();
        //添加元素
        enqueue(e);
    } finally {//释放锁
        lock.unlock();
    }
}


public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    checkNotNull(e);
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();

    try {
        //如果数组满了，阻塞nanos纳秒
        //如果唤醒这个线程时依然没有空间，则返回false
        while (count == items.length) {
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        //添加元素
        enqueue(e);
        return true;
    } finally {//释放元素
        lock.unlock();
    }
}
```
利用存指针循环从队列中存放元素

## 4.4. 出队
remove()：如果队列为空则抛出异常
poll():如果队列为空则返回null，否返回弹出的元素
```java
//删除队列头的元素
public E remove() {
    E x = poll();//调用poll()方法进行删除
    //如果有元素出队就返回这个元素，否则抛出异常
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}

//删除队首元素，并返回出队的元素
//如果队列为空，则返回null
public E poll() {
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    try {
        //如果队列为空返回null,否则进行删除
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();//释放锁
    }
}


//删除方法的底层实现
private E dequeue() {
 
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    //将takeIndex下标下的元素置为null
    items[takeIndex] = null;
    //取指针前移，如果数组到头了就返回数组前端循环利用
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;//元素个数减1
    if (itrs != null)
        itrs.elementDequeued();
    //唤醒非满
    notFull.signal();
    return x;
}
```

# 5. 总结
1. ArrayBlockingQueue不需要进行扩容，因为是初始化时进行指定容量，并循环利用数组。
2. 入队和出队定义有多种不同的方法
3. 利用冲入锁保证并发安全
4. 公平锁与非公平锁：公平锁每次都是从队首进行取值(FIFO)，非公平锁可以有机会直接获取到锁。