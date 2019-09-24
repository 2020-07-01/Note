<!-- TOC -->

- [1. 简介](#1-简介)
- [2. 继承体系](#2-继承体系)
- [3. 存储结构](#3-存储结构)
- [4. 源码分析](#4-源码分析)
    - [4.1. 主要属性](#41-主要属性)
    - [4.2. 主要内部类Entry](#42-主要内部类entry)
    - [4.3. 构造方法](#43-构造方法)
    - [4.4. 添加元素](#44-添加元素)
    - [4.5. 移除失效的key](#45-移除失效的key)
- [5. 总结](#5-总结)

<!-- /TOC -->
# 1. 简介
WeakHashMap是一种弱引用的map，内部的key会被存储为弱引用。存活时间为下一次垃圾回收器回收之前。

**通常用来实现缓存。**

# 2. 继承体系
```java
public class WeakHashMap<K,V> extends AbstractMap<K,V> implements Map<K,V> 
```

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

    //存储元素的数组，长度必须总是2的幂次方，此数组中存储的是Entry类型的节点
    //此时创建了引用队列
    Entry<K,V>[] table;
    
    //元素的大小
    private int size;
    //扩容阈值
    private int threshold;
    //指定的装载因子
    private final float loadFactor;
    
    //当key失效的时候会被存储到这个队列中
    private final ReferenceQueue<Object> queue = new ReferenceQueue<>();

    //操做次数记录
    int modCount;
```

## 4.2. 主要内部类Entry

**此节点Entry实现了弱引用WeakReference,在进行创建节点的时候，key会被传递到Reference类的构造方法中，实际存储在Referent属性中。**

```java
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
    //此处没有key，它存储在Referent类中
    V value;
    final int hash;
    Entry<K,V> next;//单向链表

    //构造方法，此处创建了引用队列
    Entry(Object key, V value,
            ReferenceQueue<Object> queue,
            int hash, Entry<K,V> next) {
        super(key, queue);//此处调用WeakReference的构造方法
        this.value = value;
        this.hash  = hash;
        this.next  = next;
    }
    
    //WeakReference的构造方法
    public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);//此处调用Reference的构造方法
    }

    //Reference的构造方法
    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }
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
    Entry<K,V>[] tab = getTable();//获取table数组
    int i = indexFor(h, tab.length);//计算要存储的下标地址

    //遍历地址i处的链表，e为链表的首节点，在进行遍历之后，e为尾节点
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
    //将节点插入到链表的头部
    tab[i] = new Entry<>(k, value, queue, h, e);

    if (++size >= threshold)//如果sie大于等于扩容阈值则进行扩容为原来的两倍
        resize(tab.length * 2);//扩容为原来的两倍
    return null;
}

//这个方法在返回table表之前，会先进行删除移除失效的key
private Entry<K,V>[] getTable() {
    //移除失效的key
    expungeStaleEntries();
    return table;//存储元素的数组
}

```
1. 计算hash值和key的下标
2. 遍历链表查找是否存在相同的值，如果存在则替换value值，返回旧值
3. 如果不存在则添加在链表头部，返回null；HashMap是添加在链表尾部
4. 最后检查是否进行扩容,HashMap中大于threshold进行扩容，WeakHashMap中大于等于threshold时进行扩容

 
## 4.5. 移除失效的key

当key失效的时候，gc会自动将key对应的Entry存储到引用队列queue中，在下次进行引用WeakHashMap时会先根据queue中的Entry删除无效的key

```java
//从表中删除失效的条目
private void expungeStaleEntries() {
    //遍历引用列表queue
    for (Object x; (x = queue.poll()) != null; ) {
        synchronized (queue) {
            @SuppressWarnings("unchecked")
                Entry<K,V> e = (Entry<K,V>) x;
            int i = indexFor(e.hash, table.length);

            Entry<K,V> prev = table[i];
            Entry<K,V> p = prev;
            while (p != null) {
                Entry<K,V> next = p.next;
                if (p == e) {
                    if (prev == e)
                        table[i] = next;
                    else
                        prev.next = next;
                    // Must not null out e.next;
                    // stale entries may be in use by a HashIterator
                    e.value = null; // Help GC
                    size--;
                    break;
                }
                prev = p;
                p = next;
            }
        }
    }
}

```

# 5. 总结
1. 底层结构采用**数组+链表**的形式存储
2. 每次对map的操作都是移除失效的key ——> getTable()方法
3. WeakHashMap是在链表头部插入元素，而HashMap是在链表尾部进行插入元素
4. WeakHashMap常用来作为缓存使用。

 