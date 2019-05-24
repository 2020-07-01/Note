

# 源码解析

## 主要属性
```java
    
    private final transient ReentrantLock lock = new ReentrantLock();

    //底层用PriorityQueue存储元素    
    private final PriorityQueue<E> q = new PriorityQueue<E>();

```


##　构造方法
```java

    //无参构造方法
    public DelayQueue() {}

    //
    public DelayQueue(Collection<? extends E> c) {
        this.addAll(c);
    }
```


## 添加元素

```java
//返回boolean值
public boolean add(E e) {
    return offer(e);//调用offer方法添加
}

//返回boolean值
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    //获取锁
    lock.lock();
    try {
        //底层使用priorityQueue存储元素
        q.offer(e);
        if (q.peek() == e) {
            leader = null;
            available.signal();
        }
        return true;
    } finally {
        lock.unlock();
    }
}



```