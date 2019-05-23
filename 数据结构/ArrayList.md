
<!-- TOC -->

1. [1. 简介](#1-%E7%AE%80%E4%BB%8B)
2. [2. 继承体系](#2-%E7%BB%A7%E6%89%BF%E4%BD%93%E7%B3%BB)
3. [3. 源码解析](#3-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)
   1. [3.1. 属性](#31-%E5%B1%9E%E6%80%A7)
   2. [3.2. 构造方法](#32-%E6%9E%84%E9%80%A0%E6%96%B9%E6%B3%95)
   3. [3.3. add(E e)方法](#33-adde-e%E6%96%B9%E6%B3%95)
   4. [3.4. remove(int index)](#34-removeint-index)
   5. [3.5. remove(Object o)](#35-removeobject-o)

<!-- /TOC -->

# 1. 简介

ArrayList是一种以数组实现的list，与数组相比，他具有 **动态扩展**的能力，因此称为动态数组。

# 2. 继承体系
![Alt](https://img-blog.csdnimg.cn/20190329160747473.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Rhbmd0b25nMQ==,size_16,color_FFFFFF,t_70)

1.ArrayList实现了List接口，提供了基础的添加，删除，遍历等操作
2.ArrayList实现了RandomAccess，提供了随机访问的能力
3.ArrayList实现了Cloneable接口，可以被克隆
4.ArrayList实现了Serializable，可以被序列化


# 3. 源码解析
## 3.1. 属性

```java
    //默认容量为10
    private static final int DEFAULT_CAPACITY = 10;
    
    //空数组，如果传入的容量为0时使用
    private static final Object[] EMPTY_ELEMENTDATA = {};
 
    //空数组，传入容量时使用，添加第一个元素的时候会重新初始为默认容量的大小
    private static final Object[] 
    DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    
    //存储元素的数组
    transient Object[] elementData; 

    //集合中的元素的个数  
    private int size;

```
1.DEFAULT_CAPACITY

默认容量为10，也就是通过new ArrayList()创建时的默认容量。

2.EMPTY_ELEMENTDATA

空的数组，这种是通过new ArrayList(0)创建时用的是这个空数组。

3.DEFAULTCAPACITY_EMPTY_ELEMENTDATA

也是空数组，这种是通过new ArrayList()创建时用的是这个空数组，与EMPTY_ELEMENTDATA的区别是在添加第一个元素时使用这个空数组的会初始化为DEFAULT_CAPACITY（10）个元素。

4.elementData

真正存放元素的地方，使用transient是为了不序列化这个字段

5.size
真正存储元素的个数，而不是elementData数组的长度

## 3.2. 构造方法

1. ArrayList()构造方法
不传入初始容量，初始化为空数组，会在传入第一个元素的时候扩容为默认的大小
```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```


2. ArrayList(Collection c)构造方法
传入集合并初始化为elementData，会使用拷贝将传入的集合的元素拷贝到elementData数组中，如果元素个数为0，则初始化为空数组
```java
 public ArrayList(Collection<? extends E> c) {
        //将集合转换为数组
        elementData = c.toArray();

        if ((size = elementData.length) != 0) {
            //拷贝集合
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {

            // 如果c是空集合转换为空数组
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

```
3. ArrayList(int initialCapacity)构造方法
传入初始容量，如果大于0就初始化为elementData对应的大小，如果等于0则使用空数组，如果小于0则抛出异常
```java
public ArrayList(int initialCapacity) {
    //大于0则创建新的数组
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        //等于0则创建空的数组
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        //小于则抛出异常
        throw new IllegalArgumentException("Illegal Capacity: "+
                                            initialCapacity);
    }
}
```


## 3.3. add(E e)方法
```java
public boolean add(E e) {
    //检查是否需要进行扩容
    ensureCapacityInternal(size + 1);  
    //添加元素到elementData中
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}


private void ensureExplicitCapacity(int minCapacity) {
   
    modCount++;
    //进行扩容mincapacity=size+1是实际容量，length属性指的是初始的容量
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}


//扩容函数
private void grow(int minCapacity) {
        //旧值为初始值
        int oldCapacity = elementData.length;
        //新值为旧值的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //如果新的扩容值小于实际值，则将之际值赋值给新值，以实际值为准
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //如果新的扩容之大于最大容量，则以最大容量为准
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        //以新的容量拷贝出一个新的数组
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

```
添加过程：
1.检查是否需要扩容
2.添加元素到末尾
3.新容量为旧容量的1.5倍，如果扩容后发现新容量还是比实际容量小则以实际容量为准
4.将旧容量的数组拷贝到新容量的数组中去

2. add(int index, E element)
添加元素到指定的位置,时间复杂度为n
```java
public void add(int index, E element) {
    //判断下标是否越界
    rangeCheckForAdd(index);
    //判断是否需要进行扩容
    ensureCapacityInternal(size + 1);
    //采用复制的方式将数组后移一位
    System.arraycopy(elementData, index, elementData, index + 1,
                        size - index);
    //添加新的元素在指定下表处         
    elementData[index] = element;
    size++;
}
//判断下标是否越界
private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

```
添加过程：
1.检查下标是否越界
2.判断是否扩容
3.将插入下标后的元素往后移动一位
4.在指定下标处插入元素
5.size大小+1


## 3.4. remove(int index)
删除指定下标处的元素,时间复杂度为O(n)
```java
public E remove(int index) {
    //检查下表是否越界
    rangeCheck(index);

    modCount++;
    //获取index处的元素值
    E oldValue = elementData(index);
    //计算需要移动的元素的长度
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                            numMoved);
    //删除最后一个元素
    elementData[--size] = null; 

    return oldValue;
}
```
删除过程：
1.检查下标是否越界
2.将元素从index处向前移动一位进行覆盖
3.size值-1



## 3.5. remove(Object o)
删除指定元素值的元素,时间复杂度为O(n)
```java
public boolean remove(Object o) {
    //遍历数组，找到元素第一次出现时位置，然后快速删除
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                            numMoved);
    elementData[--size] = null; // clear to let GC do its work
}

```
删除过程:
1.遍历数组，找到元素的下标
2.进行快速删除


总结：
1. 添加元素到末尾，时间复杂度为O(1)
2. 添加元素到指定的位置，中间时比较慢O(n)，尾部时比较快O(1)
3. 删除元素，中间时比较慢O(n)，尾部时比较快O(1)
4. ArrayList内部使用数组进行存储，支持数组的扩容，每次扩容为原来得1.5倍，不支持缩容
5. 支持随机访问，时间复杂度为O(1)
6. ArrayList支持求并集，addAll(Collection c)方法
7. ArrayList支持求交集，retain(Collection c)方法
8. ArrayList支持求差集，removeAll(Collection c)方法
