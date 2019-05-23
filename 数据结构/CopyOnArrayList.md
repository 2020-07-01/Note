<!-- TOC -->

1. [1. 简介](#1-%E7%AE%80%E4%BB%8B)
2. [2. 继承体系](#2-%E7%BB%A7%E6%89%BF%E4%BD%93%E7%B3%BB)
3. [3. 源码解析](#3-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)
   1. [3.1. 属性](#31-%E5%B1%9E%E6%80%A7)
   2. [3.2. 构造方法](#32-%E6%9E%84%E9%80%A0%E6%96%B9%E6%B3%95)
   3. [3.3. add()方法](#33-add%E6%96%B9%E6%B3%95)
   4. [3.4. add(int index, E element)](#34-addint-index-e-element)
   5. [3.5. get(int index)方法](#35-getint-index%E6%96%B9%E6%B3%95)
   6. [3.6. size()方法](#36-size%E6%96%B9%E6%B3%95)
4. [4. 总结](#4-%E6%80%BB%E7%BB%93)

<!-- /TOC -->

# 1. 简介
CopyOnWriteArrayList是 **ArrayList的线程安全版本，内部也是通过数组来实现的，每次对数组的操作都是通过拷贝一份新的数组来实现，修改完了再替换掉老的数组，**

# 2. 继承体系
![Alt](https://img-blog.csdnimg.cn/20190331151449459.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Rhbmd0b25nMQ==,size_16,color_FFFFFF,t_70)

1.CopyOnWriteArrayList实现了List，提供了基础的添加、删除、遍历等操作。

2.CopyOnWriteArrayList实现了RandomAccess，提供了随机访问的能力。

3.CopyOnWriteArrayList实现了Cloneable，可以被克隆。

4.CopyOnWriteArrayList实现了Serializable，可以被序列化


# 3. 源码解析
## 3.1. 属性
```java

//修改时加锁
final transient ReentrantLock lock = new ReentrantLock();

//真正存储元素的地方，只能通过getArray()和setArray()方法进行访问
private transient volatile Object[] array;

```
1.Lock：用于修改时进行加锁
2.array：真正存储元素的地方，volatile表示一个线程对这个字段的修改对另一个线程是可见的

## 3.2. 构造方法

1.CopyOnWriteArrayList()
创建空的数组

```java
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}

//赋值给array数组
final void setArray(Object[] a) {
    array = a;
}
```
2. CopyOnWriteArrayList(Collection<? extends E> c)

```java
 public CopyOnWriteArrayList(Collection<? extends E> c) {

        Object[] elements;
        //如果c是CopyOnWriteArrayList类型的，则直接将c数组拿过来使用
        if (c.getClass() == CopyOnWriteArrayList.class)
            elements = ((CopyOnWriteArrayList<?>)c).getArray();
        else {
            //否则将集合元素转换为数组
            elements = c.toArray();
           //复制新的数组
            if (elements.getClass() != Object[].class)
                //实现数组的复制，返回复制后的数组
                elements = Arrays.copyOf(elements, elements.length, Object[].class);
        }
        setArray(elements);
    }
```
3. CopyOnWriteArrayList(E[] toCopyIn)构造方法
把toCopyIn的元素拷贝给当前的list数组
```java
public CopyOnWriteArrayList(E[] toCopyIn) {
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}

```

## 3.3. add()方法
添加元素到末尾,时间复杂读O(1)
```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    //获得锁
    lock.lock();
    try {
        //获取旧的数组
        Object[] elements = getArray();
        int len = elements.length;
        
        //拷贝旧数组到新的数组中，并且大小+1
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        //将元素添加到末尾
        newElements[len] = e;
        //将新的数组返回给旧数组array
        setArray(newElements);

        return true;
    } finally {
        //释放锁
        lock.unlock();
    }
}
```
添加过程：
1.加锁
2.获取旧数组，创建新数组，大小为旧数组+1，并把旧数组的元素拷贝到新数组中
3.将新的元素添加到末尾
4.把新的数组赋值给当前的array属性，覆盖原数组
5.解锁

## 3.4. add(int index, E element)
添加元素到指定的位置，时间复杂度O(n)
```java
public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    //获取锁
    lock.lock();
    try {
        //获取旧数组
        Object[] elements = getArray();
        int len = elements.length;
        //检查下标是否合法
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+
                                                ", Size: "+len);
        //创建新数组
        Object[] newElements;
        int numMoved = len - index;
        //不需要移动元素，直接插在末尾
        if (numMoved == 0)
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            //需要移动元素
            newElements = new Object[len + 1];
            System.arraycopy(elements, 0, newElements, 0, index);//复制前index个元素
            System.arraycopy(elements, index, newElements, index + 1,//复制后面的元素
                                numMoved);
        }
        newElements[index] = element;
        setArray(newElements);
    } finally {
        lock.unlock();//释放锁
    }
}

```
添加过程：
1.加锁；
2.检查索引是否合法，如果不合法抛出IndexOutOfBoundsException异常，注意这里index等于len也是合法的；
3.如果索引等于数组长度（也就是数组最后一位再加1），那就拷贝一个len+1的数组；
4.如果索引不等于数组长度，那就新建一个len+1的数组，并按索引位置分成两部分，索引之前（不包含）的部分拷贝到新数组索引之前（不包含）的部分，索引之后（包含）的位置拷贝到新数组索引之后（不包含）的位置，索引所在位置留空；
5.把索引位置赋值为待添加的元素；
6.把新数组赋值给当前对象的array属性，覆盖原数组；
7.解锁

## 3.5. get(int index)方法
获取指定索引的元素，支持随机访问，时间复杂度为O(1)
```java
//获取元素不需要进行加锁
//直接返回index处的元素
//数组本身会做越界检查
public E get(int index) {
    return get(getArray(), index);
}

private E get(Object[] a, int index) {
    return (E) a[index];
}
```
获取过程：
1.获取元素数组
2.获取指定位置处的元素

## 3.6. size()方法
返回数组的长度
```java
//不加锁
public int size() {
    return getArray().length;
}
```



# 4. 总结
1.CopyOnWriteArrayList使用ReentrantLock重入锁，保证线程的安全
2.CopyOnWriteArrayList的写操作都要先拷贝一份新数组，在新的数组中作修改，修改结束后用新数组代替老的数组，所以空间复杂度为O(n),性能地下。
3.CopyOnWriteArrayList读操作支持随机访问，时间复杂度为O(1)
4.CopyOnWriteArrayList采用读写分离的思想，读操作不加锁，写操作加锁，写操作占用较大的内存空间，适合读多写少的场合。
5.CopyOnWriteArrayList只能保证最终的一致性，不能保证实时一致性。
6.每次进行添加元素的操作的时候，都是先copy一个arra
y.length+1的大小的新数组，刚好可以存储目标元素，因此不需要size属性。