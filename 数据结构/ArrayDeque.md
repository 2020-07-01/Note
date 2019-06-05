<!-- TOC -->

- [1. 简介](#1-简介)
- [2. 存储结构](#2-存储结构)
- [3. 源码解析](#3-源码解析)
    - [3.1. 属性](#31-属性)
    - [3.2. 构造方法](#32-构造方法)
    - [3.3. 入队](#33-入队)
    - [3.4. 出队](#34-出队)
    - [3.5. 扩容机制](#35-扩容机制)
- [4. 总结](#4-总结)

<!-- /TOC -->
# 1. 简介

1. 双端队列是一种两端都可以进行操作元素的队列
2. ArrayDeque是一种以数组方式实现的双端队列，它是非线性安全的。

# 2. 存储结构
* ArrayDeque的底层是数组实现的。

# 3. 源码解析

## 3.1. 属性
```java
   //存储元素的数组
    transient Object[] elements;  
    //队列的头节点
    transient int head;
    //队列的尾节点
    transient int tail;
    //最小初始容量
    private static final int MIN_INITIAL_CAPACITY = 8;
```

* ArrayDeque使用数组存储元素，并使用头节点和尾节点表示队列的头和尾，最小容量是8，初始时头节点和尾节点都为0，


## 3.2. 构造方法

```java
    //默认初始容量为16
    public ArrayDeque() {
        elements = new Object[16];
    }
 
    //指定初始容量
    public ArrayDeque(int numElements) {
        allocateElements(numElements);
    }

    //将集合c中的元素初始化到数组中
    public ArrayDeque(Collection<? extends E> c) {
        allocateElements(c.size());
        addAll(c);
    }
```

## 3.3. 入队
* 在队列尾部添加元素，尾指针向右进行移动
* 在队列头部添加元素，头指针向左进行移动

```java
//从队列头部添加元素
public void addFirst(E e) {
    //如果元素为空则抛出异常
    if (e == null)
        throw new NullPointerException();
    //在头部插入数据，头指针左移
    //当head为0时左移head变为-1，为了避免这种情况，使用下面这种写法进行与操作
    //当head为0时，左移后变为head变为elements.length - 1循环使用数组
    elements[head = (head - 1) & (elements.length - 1)] = e;
    //如果队列头尾相遇则进行扩容
    if (head == tail)
        doubleCapacity();
}

//从队列尾部添加元素
public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    //在尾部插入数据，尾指针右移
    //当tail为elements.length - 1时右移元素后会越界，使用下面这种写法进行与操作
    //当tail为elements.length - 1，右移之后变为0
    elements[tail] = e;
    //判断是否要进行扩容操作
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
}

```


## 3.4. 出队
* 将队头移出队列，头指针向右进行移动
* 将队尾移出对列，尾指针向左进行移动
```java
//将队列头元素移除队列
public E pollFirst() {
    int h = head;
    @SuppressWarnings("unchecked")
    E result = (E) elements[h];
    
    if (result == null)
        return null;
    elements[h] = null;  
    //头指针向右进行移动   
    head = (h + 1) & (elements.length - 1);
    return result;
}
//将队列尾元素移除队列
public E pollLast() {
    int t = (tail - 1) & (elements.length - 1);
    @SuppressWarnings("unchecked")
    E result = (E) elements[t];
    if (result == null)
        return null;
    elements[t] = null;
    //尾指针向左进行移动
    tail = t;
    return result;
}

```

## 3.5. 扩容机制
* 当头指针与尾指针相遇时进行扩容，每次扩容为原来的一倍
```java
private void doubleCapacity() {
    //assert表示断言
    //assert<boolean表达式> 如果Boolean为true则程序继续执行，否则失败抛出异常
    assert head == tail;
    int p = head;
    int n = elements.length;
    int r = n - p; 
    int newCapacity = n << 1;
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
    Object[] a = new Object[newCapacity];
    System.arraycopy(elements, p, a, 0, r);
    System.arraycopy(elements, 0, a, r, p);
    elements = a;
    head = 0;
    tail = n;
}
```


# 4. 总结
1. ArrayDeque是采用数组方式实现的双端队列
2. ArrayDeque的出队和入队是通过头尾指针循环利用数组来实现的
3. ArrayDeque默认的初始容量为16，每次扩容为原来的一倍
4. 循环数组的思想：在底层真正存储元素的是数组，在数组的基础之上构建构建逻辑队列，设置队列头和尾指针，指示队列中元素的移动操作，指针的实质是数组的下标。