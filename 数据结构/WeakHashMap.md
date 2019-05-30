<!-- TOC -->

- [1. 简介](#1-简介)
- [2. 继承体系](#2-继承体系)
- [3. 存储结构](#3-存储结构)
- [4. 源码分析](#4-源码分析)
    - [4.1. 主要属性](#41-主要属性)
    - [4.2. 主要内部类](#42-主要内部类)
    - [4.3. 构造方法](#43-构造方法)
    - [4.4. 添加元素](#44-添加元素)
    - [4.5. 扩容方法](#45-扩容方法)
    - [4.6. get(Object key)方法](#46-getobject-key方法)
    - [4.7. 删除元素](#47-删除元素)
- [5. 总结](#5-总结)
- [6. 附加：引用](#6-附加引用)

<!-- /TOC -->
# 1. 简介
WeakHashMap是一种弱引用的map，内部的key会被存储为弱引用。存活时间为下一次垃圾回收之前。通常用来实现缓存。

# 2. 继承体系

# 3. 存储结构
底层结构：数组+单向链表
WeakHashMap在没次进行垃圾回收的时候，会把没有强引用的key回收掉，所以里面没有太多的元素，因此不需要转化为红黑数来处理。 

# 4. 源码分析
## 4.1. 主要属性
```java
   //默认的初始容量
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    //最大容量
    private static final int MAXIMUM_CAPACITY = 1 << 30;
    //默认装载因子
    private static final float DEFAULT_LOAD_FACTOR = 0.75f;
    //存储元素的数组
    Entry<K,V>[] table;
    //元素的大小
    private int size;
    //扩容阈值
    private int threshold;
    //指定的装载因子
    private final float loadFactor;
    //创建引用队列
    private final ReferenceQueue<Object> queue = new ReferenceQueue<>();
    //操做次数记录
    int modCount;
```

## 4.2. 主要内部类

**此节点Entry实现了弱引用WeakReference**

```java
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
    V value;
    final int hash;
    Entry<K,V> next;//单向链表

    //构造方法
    Entry(Object key, V value,
            ReferenceQueue<Object> queue,
            int hash, Entry<K,V> next) {
        super(key, queue);
        this.value = value;
        this.hash  = hash;
        this.next  = next;
    }
    ......
}
```

## 4.3. 构造方法
```java
//指定初始值和装载因子
public WeakHashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)//如果指定容量小于0则抛出异常
        throw new IllegalArgumentException("Illegal Initial Capacity: "+
                                            initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)//如果指定值大于最大值，则以最大值进行初始化
        initialCapacity = MAXIMUM_CAPACITY;
    //如果装载因子小于0或者转载因子不是float类型则抛出异常
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal Load factor: "+
                                            loadFactor);
    //容量为1
    int capacity = 1;
    //如果容量小于初始容量则进行扩容，直到大于初始容量并且最接近2的n次方
    while (capacity < initialCapacity)
        capacity <<= 1;
    //初始数组
    table = newTable(capacity);
    this.loadFactor = loadFactor;//初始转载因子
    threshold = (int)(capacity * loadFactor);//初始扩容阈值
}

//指定初始容量，默认转载因子为0.75
public WeakHashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

//创建初始值，默认为16和0.75
public WeakHashMap() {
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
}

```


## 4.4. 添加元素
```java
public V put(K key, V value) {
    Object k = maskNull(key);//判断key值是否为null,如果key为null的话，存储的是空对象
    int h = hash(k);//计算hash值
    Entry<K,V>[] tab = getTable();//获取tab数组
    int i = indexFor(h, tab.length);//计算下标


    for (Entry<K,V> e = tab[i]; e != null; e = e.next) {
        //如果hash值相同并且key值和value值也相同同，则进行替换
        if (h == e.hash && eq(k, e.get())) {
            V oldValue = e.value;
            if (value != oldValue)
                e.value = value;
            return oldValue;//返回旧值
        }
    }
    //操作次数加1
    modCount++;
    Entry<K,V> e = tab[i];//创建头节点    
    tab[i] = new Entry<>(k, value, queue, h, e);//创建新的节点添加在尾，e为头节点
    if (++size >= threshold)//如果sie大于等于扩容阈值则进行扩容为原来的两倍
        resize(tab.length * 2);//扩容为原来的两倍
    return null;
}

//获取tab数组
private Entry<K,V>[] getTable() {
    expungeStaleEntries();
    return table;//存储元素的数组
}

//根据散列码和数组的长度返回key在数组中的下标
private static int indexFor(int h, int length) {
    //hash值与数组长度-1做与运算得到key的下标
    return h & (length-1);
}

```
1. 计算hash值和key的下标
2. 遍历链表查找是否存在相同的值，如果存在则替换value值，返回旧值
3. 如果不存在则添加在链表头部，返回null；HashMap是添加在链表尾部
4. 最后检查是否进行扩容,HashMap中大于threshold进行扩容，WeakHashMap中大于等于threshold时进行扩容


## 4.5. 扩容方法
```java
void resize(int newCapacity) {
    //此时的newCapacity为size的两倍
    //getTable()获取旧数组的时候会移除null的节点

    //判断旧桶中的容量是否已经达到了最大的容量
    Entry<K,V>[] oldTable = getTable();//获取旧数组
    int oldCapacity = oldTable.length;//获取旧容量
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
    //创建新数组并进行赋值
    Entry<K,V>[] newTable = newTable(newCapacity);
    transfer(oldTable, newTable);
    table = newTable;

    //如果元素个数大于扩容门槛的一半，则使用新桶和新容量
    if (size >= threshold / 2) {
        threshold = (int)(newCapacity * loadFactor);
    } else {
        //如果元素个数不大于扩容门槛的一半
        //则将元素再转移到旧桶
        //此时元素中移除了null的节点，元素个数减少，不再需要进行扩容
        expungeStaleEntries();
        transfer(newTable, oldTable);
        table = oldTable;
    }
}
```

## 4.6. get(Object key)方法
```java
public V get(Object key) {
    Object k = maskNull(key);//检查是否为null
    int h = hash(k);//计算hash值
    Entry<K,V>[] tab = getTable();//此操作会移除失效的key
    int index = indexFor(h, tab.length);//计算key的下标
    Entry<K,V> e = tab[index];//创建首节点
    while (e != null) {
        if (e.hash == h && eq(k, e.get()))//当hash值与key相同时返回value值
            return e.value;
        e = e.next;
    }
    //如果没有找到返回null
    return null;
}

```
查找过程：
1. 首先检查是否为null值
2. 计算hash值、key的下标，然后创建头节点
3. 遍历链表，查找是否存在此元素，如果存在返回value值
4. 如果不存在返回null值

## 4.7. 删除元素
```java
public V remove(Object key) {
    Object k = maskNull(key);//检查是否为null
    int h = hash(k);
    Entry<K,V>[] tab = getTable();//此操作会移除失效的key
    int i = indexFor(h, tab.length);
    Entry<K,V> prev = tab[i];//创建链表头节点
    Entry<K,V> e = prev;//将头节点赋值给e

    //遍历链表
    while (e != null) {
        Entry<K,V> next = e.next;
        //如果hash值和key值相同则进行删除
        if (h == e.hash && eq(k, e.get())) {
            modCount++;
            size--;
            
            if (prev == e)
                tab[i] = next;
            else
                prev.next = next;//跳过e节点
            //返回节点的值
            return e.value;
        }
        //进行节点转换
        prev = e;
        e = next;
    }
    //如果没有找到节点则返回null
    return null;
}
```
删除过程：
遍历key所在的链表进行查找节点，如果找到则进行删除，然后返回值，如果没有找到则返回null

# 5. 总结
1. 底层结构采用 **数组+链表**的形式存储
2. 初始容量默认为16，装载因子为0.75,有HashMap一样
3. WeakHashMap的扩容方式为原来的两倍
4. 每次对map的操作都是移除失效的key ——> getTable()方法
5. WeakHashMap是在链表头部插入元素，而HashMap是在链表尾部进行插入元素
6. WeakHashMap常用来作为缓存使用。


# 6. 附加：引用
出自《深入理解Java虚拟机》
有这样一类对象：当内存空间还足够时，则保留在内存之中；如果内存空间在垃圾回收之后还是很紧张的话则将这类对象抛弃。很多系统的缓存功能符合这样的应用场景。

在JDK1.2之后，Java对引用进行了扩充，将引用分为强引用(Strong Reference),软引用(Soft Reference),弱引用(Weak Reference),虚引用(Plantom Reference)四种

强引用：强引用是Java中普遍存在的，类似于"Object obj = new Object()"这类的引用，只要强引用还存在，垃圾回收器不会回收掉被引用的对象。

软引用：软引用是用来描述一些有用但是非必须的对象。如果内存足够将不会回收他，如果内存不足够时将会回收，如果回收完毕后内存还是不足才会抛出内存溢出的异常。通常用来实现缓存。JDK1.2之后提供SoftReference类来实现软引用。

弱引用：弱引用也是用来描述非必需的对象的，但是他的强度比软引用更弱一些，在进行垃圾回收的时候，无论内存是否足够，都将进行回收。通常用来实现缓存。JDK1.2之后提供WeakReference类来实现弱引用。

虚引用：虚引用是最弱的一种缓存关系，他的作用是能在JVM对对象进行回收时收到一个回收通知。JDK1.2之后提供PlantomReference来实现虚引用。
