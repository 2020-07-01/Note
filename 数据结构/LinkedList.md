<!-- TOC -->

- [1. 简介](#1-简介)
- [2. 继承体系](#2-继承体系)
- [3. 源码分析](#3-源码分析)
    - [3.1. 主要属性](#31-主要属性)
    - [3.2. 主要内部类Node](#32-主要内部类node)
    - [3.3. 构造方法](#33-构造方法)
    - [3.4. 添加元素](#34-添加元素)
    - [3.5. 删除元素](#35-删除元素)
    - [3.6. 栈](#36-栈)
- [4. 总结](#4-总结)

<!-- /TOC -->
# 1. 简介
LinkedList是一个以 **双向链表**实现的list，他除了可以作为List来使用，还可以作为队列或者栈来是使用。

# 2. 继承体系
![Alt](https://gitee.com/alan-tang-tt/yuan/raw/master/%E6%AD%BB%E7%A3%95%20java%E9%9B%86%E5%90%88%E7%B3%BB%E5%88%97/resource/LinkedList.png)
1. LinkedList不仅实现了List接口，还实现了Queue和Deque接口，既可以作为List使用，还可以作为双端队列和栈来使用
2. LinkedList实现了Serializable接口，可以序列化
3. LinkedList实现了Cloneable接口，可以复制

# 3. 源码分析
## 3.1. 主要属性

```java
    //元素的个数
    transient int size = 0;
    
    //第一个节点
    transient Node<E> first;
    
    //最后一个节点
    transient Node<E> last;
 

```
## 3.2. 主要内部类Node
双链表结构
```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
    
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

## 3.3. 构造方法

```java
//创建一个空的列表
public LinkedList() {
}

//创建包含指定集合元素的列表
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
 
```


## 3.4. 添加元素
作为一个双端队列，添加元素主要有两种方式，一种是在队尾添加元素，一种是在队首添加元素，作为list可以在中间进行添加元素

1. addFirst(E e)和addLast(E e)
从队首和队尾添加元素，时间复杂度为O(1)

```java
//从队首添加元素
public void addFirst(E e) {
    linkFirst(e);
}

private void linkFirst(E e) {
    //首节点
    final Node<E> f = first;
    //创建新的节点，新节点的next是首节点
    final Node<E> newNode = new Node<>(null, e, f);
    //让新的节点作为新的首节点
    first = newNode;
    //如果f为空，则是第一个添加的元素，则把last也置为newNode
    if (f == null)
        last = newNode;
    else
    //将原节点的向前指针置为新节点
        f.prev = newNode;
    size++;//元素的个数加1
    //修改的次数加1，这是一个fail-fast集合
    modCount++;
}

//从队尾添加元素
public void addLast(E e) {
    linkLast(e);
}

void linkLast(E e) {
    //创建队尾节点
    final Node<E> l = last;
    //创建新的节点，新节点的上一个节点为last
    final Node<E> newNode = new Node<>(l, e, null);
    //将新的节点置尾队尾节点
    last = newNode;
    //判断是不是首次添加元素
    //如果队尾节点为null则将新节点置为首节点
    if (l == null)
        first = newNode;
    else
        //否则队尾节点l的下一个节点则是新节点  
        l.next = newNode;
    size++;//节点数加1
    modCount++;
}
```

2. add(E e)
此方法将元素添加到队列的队尾，时间复杂度为O(1)

```java
//将元素添加到此队列的队尾
public boolean add(E e) {
    linkLast(e);//具体解析看上面
    return true;
}
```

3. add(int index,E element)
此方法在指定位置处添加元素
```java
//将元素添加到队列的指定位置
public void add(int index, E element) {
    //检查下标是否合法
    checkPositionIndex(index);
    //此时将在队尾进行添加
    if (index == size)
        linkLast(element);
    else
        //此时将在链表内部进行添加
        linkBefore(element, node(index));
}

Node<E> node(int index) {
    //如果index小于二分之一size，则从前进行遍历
    if (index < (size >> 1)) {
        //创建首节点
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        //返回的是index处的节点
        return x;
    } else {
        //创建尾节点
        Node<E> x = last;
        //从后往前进行遍历，返回的是index处的节点
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

void linkBefore(E e, Node<E> succ) {
    //此时succ节点为index处的旧节点
    //创建前置节点
    final Node<E> pred = succ.prev;
    //创建新的节点，后置的后置节点为index处的旧节点，前置节点为pred
    final Node<E> newNode = new Node<>(pred, e, succ);
    //指定succ的前置节点
    succ.prev = newNode;
    //如果前置节点为null则新节点为首节点
    if (pred == null)
        first = newNode;
    else
    //否则pred的后置节点为新节点
        pred.next = newNode;
    size++;//节点数加1
    modCount++;
}

```

4. offerFirst(E  e)和offerLast(E e)方法
在队列的首尾添加元素
```java
//将指定的元素添加到队列的前面
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}

//将元素添加到队列的后面
public boolean offerLast(E e) {
    addLast(e);
    return true;
}
```

## 3.5. 删除元素
作为双端队列，删除元素也有两种方式，一种是队列首删除元素，一种是队列尾删除元素。作为List，又要支持中间删除元素.

1. remove(Object o)

删除任意元素，遍历整个链表，时间复杂度为O(n)

```java
public boolean remove(Object o) {
    //从前往后遍历链表，如果节点值为o则调用unlink()方法删除节点
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
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

    final E element = x.item;//节点元素值
    final Node<E> next = x.next;//节点的后置节点
    final Node<E> prev = x.prev;//节点的前置节点

    //如果前置节点为空，即x为首节点，则首节点为x的后置节点
    if (prev == null) {
        first = next;
    } else {
        //如果前置节点不为空，即x不为首节点，则将x的前置节点的后置节点设置为x的后置节点，即跳过x节点
        prev.next = next;
        x.prev = null;
    }

    //如果后置节点为空
    //说明是尾节点，让last指向x的前置节点
    //否则修改后置的prev为x的前置节点
    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    //清空x的元素值
    x.item = null;
    size--;//节点数减1
    modCount++;
    return element;
}

```

1. removeFirst()   
删除首节点，时间复杂度为O(1)
```java
public E removeFirst() {
    final Node<E> f = first;
    //如果首节点为null则抛出异常，否则调用方法删除
    if (f == null)
        throw new NoSuchElementException();
    //如果首节点不为空则调用方法进行删除
    return unlinkFirst(f);
}

private E unlinkFirst(Node<E> f) {
    
    final E element = f.item;//节点的值
    final Node<E> next = f.next;//f节点的后置节点next
    f.item = null;//令f节点的值为null
    f.next = null; //令f的后置指针为null
    first = next;//令首节点为f的后置节点next为首节点

    //如果next为null的话则此链表只有一个节点
    //此时链表为null，令尾节点last为null
    //否则的话令next的前置指针为null，因为此时next已经为首节点
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```
## 3.6. 栈
LinkedList实现了双端队列，所以可以作为栈来使用，栈的算法是先进先出(FIFO)，因此操作都是在栈顶，即队列的首节点。

时间复杂度为O(1)

1. 添加元素到栈顶push(E e)
```java
public void push(E e) {
    addFirst(e);
}
```
2. 移出并删除栈顶的元素pop()
```java
public E pop() {
    return removeFirst();
}
```

# 4. 总结
1. LinkedList的下标是从0开始的
2. LinkedList是一个双链实现的list
3. LinkedList是一个双端队列，具有队列，双端队列和栈的特性
4. LinkedList在队列的首尾进行操作，效率非常高效，时间复杂度为O(1)
5. LinkedList不支持随机访问，查询和修改元素效率比ArrayList慢，时间复杂度为O(n)
6. LinkedList添加、删除元素时只需要遍历链表找到要操作的位置进行添加或者删除操作，而不需要进行节点的移动。ArrayList在进行添加、删除操作时要进行元素的移动，因此效率没有LinkedList高
7. LinkedList在中间进行删除和添加元素时时间复杂度为O(n)