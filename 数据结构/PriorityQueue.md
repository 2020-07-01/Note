<!-- TOC -->

- [1. 简介](#1-简介)
- [存储结构](#存储结构)
- [2. 源码分析](#2-源码分析)
    - [2.1. 入队](#21-入队)
    - [2.2. 出队](#22-出队)
    - [2.3. 扩容](#23-扩容)
- [3. 总结](#3-总结)

<!-- /TOC -->
# 1. 简介
PriorityQueue优先级队列是0个或多个元素的集合，每个集合中的每个元素都有一个权重值，出队都是弹出优先级最大或者是最小的元素.

# 存储结构
PriorityQueue底层实现是数组，数组采用最小堆结构进行存储，具体实现是最小堆

队列的特性是先进先出(FIFO)，只在队首进行删除操作，在队尾进行添加操作
 
# 2. 源码分析

## 2.1. 入队
入队就是添加元素在队尾进行操作

```java
public boolean add(E e) {
        return offer(e);//调用offer方法在队尾添加元素
    }

public boolean offer(E e) {
    //如果e为null，抛出异常
    if (e == null)
        throw new NullPointerException();
    modCount++;//操作加1
    int i = size;//i为容量
    //如果i为大于等于队列长度，则进行扩容
    if (i >= queue.length)
        grow(i + 1);
    //size的值加1
    size = i + 1;
    //如果队列为空，则将值添加在第一位
    if (i == 0)
        queue[0] = e;
    else
    //添加新的值的最后一位然后进行上浮
        siftUp(i, e);
    return true;
}

```
上浮操作：小的元素往上浮
```java
//选择比较器
private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x);
    else
        siftUpComparable(k, x);
}

@SuppressWarnings("unchecked")
private void siftUpComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    //k为新元素的下标
    while (k > 0) {
        int parent = (k - 1) >>> 1;//计算父节点的下标
        Object e = queue[parent];
        //如果插入的元素比父元素的值大就跳出循环，否则交换他们的值
        if (key.compareTo((E) e) >= 0)
            break;
        queue[k] = e;
        k = parent;
    }
    //将新的元素进行插入
    queue[k] = key;
}
```
实现过程：
1. 入队不允许元素为null
2. 如果容量不够先进性扩容
3. 如果队列为空则插入在第一个元素
4. 如果队列不为空，则插入在队尾最后一个元素的后面，实际并没有进行插入
5. 自下而上进行堆化，一直与父节点进行比较，
6. 如果插入的元素的值比父元素的小则进行交换位置，否则退出循环，进行插入
7. 最后PriorityQueue变为一个小顶堆

## 2.2. 出队
出队就是删除元素，在队首进行操作
```java
//删除队首的元素
public E remove() {
    E x = poll();//调用poll方法进行删除
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}

//删除并返回队尾元素
@SuppressWarnings("unchecked")
public E poll() {
    //如果队列为空返回null
    if (size == 0)
        return null;
    //s存储的是元素的个数
    int s = --size;
    modCount++;//操作加1
    //result为队首元素
    E result = (E) queue[0];
    //x为队尾元素
    E x = (E) queue[s];
    //令队尾元素为null
    queue[s] = null;
    //如果弹出元素后队列不为0

    if (s != 0)
        //将队尾的元素移到队首进行堆化
        siftDown(0, x);
    //删除后返回队首元素
    return result;
}
```
下沉操作：就是将不小于左右子节点的元素进行下沉
```java
private void siftDown(int k, E x) {
    //选择比较器
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}

@SuppressWarnings("unchecked")
private void siftDownComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>)x;
    int half = size >>> 1;        // loop while a non-leaf
    while (k < half) {

        int child = (k << 1) + 1; // assume left child is least
        Object c = queue[child];
        int right = child + 1;
        if (right < size &&
            ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            c = queue[child = right];
        if (key.compareTo((E) c) <= 0)
            break;
        queue[k] = c;
        k = child;
    }
    
    queue[k] = key;
}

```
实现过程：
1. 将队首元素进行弹出
2. 将队尾元素移到队首，队尾节点置null
3. 队首元素与自己的左右子节点元素进行比较，自上而下进行堆化
4. 如果父节点小于子节点则退出循环，堆化结束；否则进行位置的交换
5. 最后堆化为小顶堆

## 2.3. 扩容
在添加元素之前判断是否需要扩容
```java
private void grow(int minCapacity) {
    //旧容量
    int oldCapacity = queue.length;
    //如果旧容量小于64则容量翻倍
    //如果旧容量大于64时，容量增加一半
    int newCapacity = oldCapacity + ((oldCapacity < 64) ? (oldCapacity + 2) : (oldCapacity >> 1));
    
    //最大容量为Integer的最大容量-8
    //如果新的容量超过最大的容量，则以Integer的最大容量为新的容量
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    //然后进行数组的复制
    queue = Arrays.copyOf(queue, newCapacity);
}
```
扩容过程：
1. 如果旧的容量小于64，则扩容翻倍
2. 如果旧的容量大于64，则扩容为原来的1.5倍
3. 如果扩容后，新容量大于Integer的最大值，则Integer的最大值作为新的容量

# 3. 总结
1. PriorityQueue是一个小顶堆
2. PriorityQueue是非线程安全的，源码中没有对其进行加锁
3. PriorityQueue不是有序的，只是堆顶存储着最小的元素，每次将最小的元素进行移除
4. 入队就是小堆的插入操作的实现，插入在堆的尾部然后进行上浮操作
5. 出队就是小堆的删除操作的实现，每次删除堆的根节点然后将尾节点置于根节点最后进行下沉操作
6. PriorityQueue的默认是容量为11
7. 扩容操作：当容量大于64的时候，扩容为原来的1.5倍；当容量小于64的时候，扩容为原来的两倍
8. PriorityQueue的底层实现时数组