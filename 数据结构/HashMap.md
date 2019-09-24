

<!-- TOC -->

- [1. 引言](#1-引言)
- [2. 继承体系](#2-继承体系)
- [3. 存储结构](#3-存储结构)
- [4. 源码解析](#4-源码解析)
    - [4.1. HashMap属性](#41-hashmap属性)
    - [4.2. 构造方法](#42-构造方法)
    - [4.3. put( K key,V value)方法](#43-put-k-keyv-value方法)
        - [4.3.1. 存储元素的过程及其哈希冲突解决办法](#431-存储元素的过程及其哈希冲突解决办法)
    - [4.4. get(Object key)方法](#44-getobject-key方法)
- [5. 更新映射项](#5-更新映射项)
    - [5.1. 方法一：getOrDefault()方法](#51-方法一getordefault方法)
    - [5.2. 方法二：hashMap.putIfAbsent()](#52-方法二hashmapputifabsent)
- [6. 映射视图](#6-映射视图)
- [7. key的哈希值得获取方式](#7-key的哈希值得获取方式)
- [8. 总结](#8-总结)
    - [8.1. 如果HashMap的key是一个自定义的类该怎么办？](#81-如果hashmap的key是一个自定义的类该怎么办)

<!-- /TOC -->
 --------------------- 
作者：彤哥读源码 
来源：CSDN 
原文：https://blog.csdn.net/tangtong1/article/details/88934809 
版权声明：本文为博主原创文章，转载请附上博文链接！
 

# 1. 引言

jdk版本：1.8

HashMap采用key/value的存储结构，每个key可以对应一个唯一的value，查询和修改的速度很快，接近于O(1)的平均时间复杂度，它是线性不安全的，且不能保证元素的顺序性，因为底层是数组，元素呈散列分布。

# 2. 继承体系

[外链图片转存失败(img-KwLxe35G-1567756986031)(https://gitee.com/alan-tang-tt/yuan/raw/master/%E6%AD%BB%E7%A3%95%20java%E9%9B%86%E5%90%88%E7%B3%BB%E5%88%97/resource/HashMap.png)]

1.HashMap实现了Serializable接口，可以被序列化

2.HashMap实现了Cloneable接口，可以被克隆

3.HashMap继承自AbstractMap类，实现了Map接口，具有Map的所用功能

# 3. 存储结构
![Alt](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9naXRlZS5jb20vYWxhbi10YW5nLXR0L3l1YW4vcmF3L21hc3Rlci8lRTYlQUQlQkIlRTclQTMlOTUlMjBqYXZhJUU5JTlCJTg2JUU1JTkwJTg4JUU3JUIzJUJCJUU1JTg4JTk3L3Jlc291cmNlL0hhc2hNYXAtc3RydWN0dXJlLnBuZw?x-oss-process=image/format,png)

HashMap采用了(**数组 + 链表 + 红黑树**)的实现结构，数组的一个元素称作为**桶**。

**数组中的每一个元素值为链表，此链表为单向链表**

在添加元素时，会根据key的hash值计算出元素在数组中的位置，如果该位置中没有元素，则直接把元素存储在此位置，如果该位置有元素了，则把元素以链表的形式存放在链表的尾部。

当链表的元素个数达到一定的数量后(且数组的长度达到一定的长度))，则把链表转换为红黑树，从而提高效率。


# 4. 源码解析


## 4.1. HashMap属性
```java
    //默认的初始容量为16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    //最大的容量为2的30次方
    static final int MAXIMUM_CAPACITY = 1 << 30;
 
    //默认的装载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    //当桶中元素的个数大于8时进行树化
    static final int TREEIFY_THRESHOLD = 8;
    
    //当桶中的元素个数小于等于6时转换为链表
    static final int UNTREEIFY_THRESHOLD = 6;
   
    //桶的个数达到64个时进行树化
    static final int MIN_TREEIFY_CAPACITY = 64;

    //数组，元素称为桶
    transient Node<K,V>[] table;
   
    transient Set<Map.Entry<K,V>> entrySet;
 
    transient int size;


```
* **Node内部类**
**Node是一个单向链表节点**
```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;//存储key的hash值
        final K key;
        V value;
        Node<K,V> next;
 
        ......
}
```


1.容量：
容量为数组的长度，即桶的个数，默认为16个，最大为2的30次方个，当容量大于64时才进行树化

2.装载因子：
装载因子用来计算容量达到多少时进行扩容，默认为0.75
实际容量/初始容量

3.树化：
当容量达到64个且链表的长度达到8个时才进行树化，当链表小于6个时才进行反树化

## 4.2. 构造方法
1. HashMap()构造方法
空构造方法，全部使用默认值
```java
 public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```

2. HashMap(int initialCapacity)构造方法
指定初始容量
使用此构造方法时调用HashMap(int initialCapacity, float loadFactor)构造方法,传入默认的装载因子
```java
public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
```

3. HashMap(int initialCapacity, float loadFactor)构造方法
指定装载因子及其初始容量
使用构造方法时先判断初始容量和装载因子是否合法，然后计算扩容门槛
```java
public HashMap(int initialCapacity, float loadFactor) {
    //判断初始容量是否合法
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                            initialCapacity);
    //判断初始容量是否大于最大容量，如果大于则初始化为最大容量
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    //判断装载因子是否合法
    //isNan()方法判断是否为数字
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                            loadFactor);
    
    this.loadFactor = loadFactor;
    //计算扩容门槛
    this.threshold = tableSizeFor(initialCapacity);
}

//计算扩容门槛
//取初始容量往上最近的2的n次方的值
static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```



## 4.3. put( K key,V value)方法
put(k key, V value)

```java

public V put(K key, V value) {
    //1.hash(key)：计算key的hash值
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        //tab与p的关系：tab为数组，代替table，p为桶,即tab中的元素，也就是key所在的链表
        Node<K,V>[] tab; //创建新的tab，代替table
        Node<K,V> p; //创建p节点，此节点代表的是新添加的元素所在的桶
        int n, i;
        
        //2.如果桶的个数为0或者桶为null则进行桶的初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;//n为桶中的元素个数
        
        //3.如果tab中还没有这个元素，则将这个元素添加在第一个位置
        //(n-1) & hash：计算元素在那个桶中
        if ((p = tab[i = (n - 1) & hash]) == null)
            //新建节点
            tab[i] = newNode(hash, key, value, null);
        
        //如果tab中已经存在元素
        else {
            Node<K,V> e; //用于临时保存key相同的值
            K k;
            //4.如果桶中第一个元素的key值与待插入的元素的key值相同，则保存到e用于后续修改value值
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            
            //5.如果第一个元素p是树节点，则调用树节点的putTreeVal方法插入元素，此时链表为红黑树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //6.遍历整个桶对应的链表
            else {
                for (int binCount = 0; ; ++binCount) {
                    //8.如果遍历完整个链表都没有找到相同的key元素，说明key对应的元素不存在，则在链表最后插入一个新的节点
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //如果插入新节点之后链表长度大于8，则判断是否需要进行树化
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果待插入的key在链表中找到了则退出循环
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }

            //9.如果找到了对应的key的元素
            if (e != null) { // existing mapping for key
                //记录下旧值
                V oldValue = e.value;
                //判断是否需要替换旧值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }

        ++modCount;
        //10.判断是否需要扩容
        if (++size > threshold)
            //扩容函数
            resize();

        afterNodeInsertion(evict);
        //如果没有找到元素则返回null
        return null;
    }
```
### 4.3.1. 存储元素的过程及其哈希冲突解决办法

1. 计算key的hash值
2. 如果桶(数组)数量为0，则进行桶的初始化
3. 如果key所在的桶p没有元素，即下标处还未存储元素，则直接将此元素插入
4. 如果key所在的桶p中的第一个元素的key与待插入的key相同，则说明找到了了元素，此时考虑新旧值的替换
5. 如果第一个元素是树节点，此时链表为红黑树，则调用树节点的putTreeVal()方法进行插入
6. 如果不是以上三种情况，此链表已经是一个存在哈希冲突的链表，则遍历桶(单向链表)查看key是否存在与链表中
7. 如果key存在于链表中，则此时考虑key的value值得替换
8. 如果遍历链表没有找到此元素，则在链表最后插入新的节点，并判断是否需要进行树化
9. 如果插入了节点，则数量+1并判断是否需要进行扩容


* 哈希冲突解决办法为链地址法，即当key值不相同而哈希值相同的时候，使用单向链表存储key和value值


## 4.4. get(Object key)方法
查询元素
```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; 
    Node<K,V> first, e; //first存储桶的第一个节点
    int n; K k;

    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {

        //检查第一个节点是否为要找的节点，如果是则直接返回
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            //返回first节点
            return first;
        //如果第一个节点的key值不相同，则遍历节点
        if ((e = first.next) != null) {
            //如果第一个节点是树节点，则按树的方式进行查找
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //如果hash值相同并且key值相同则返回
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    //如果没有找到返回null
    return null;
}

```
查询过程：

1.计算key的hash值

2.找到key所在的桶及其第一个元素

3.如果第一个元素的key等于待查找的key，则直接返回

4.如果第一个元素为树节点，则按树的方式进行查找；否则按链表方式进行查找



# 5. 更新映射项
场景分析：当需要统计一个文章单词出现的次数时，可以使用如下方法进行统计，即每次当word出现时对它的值加1，再进行存储。
```java
HashMap hashMap = new HashMap();
hashMap.put(word,hashMap.get(word)+1);
```
Bug:但是如果在word第一次出现时hashMap.get(word)方法返回的是一个null值，此时会抛出异常。
## 5.1. 方法一：getOrDefault()方法
```java
@Override
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    //如果是第一次出现则返回defaultValue值 
    return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
}
```
更新方法:
```java
//在添加时如果word是第一次出现则返回0，然后加1
hashMap.put(word,hashMap.getOrDefault(word,0)+1);
```
## 5.2. 方法二：hashMap.putIfAbsent()
方法得含义是当键原先存在时才会放入一个值
```java

hashMap.putIfAbsent(word,1)
hashMap.put(word,hashMap.get(word)+1);
```

# 6. 映射视图
集合框架不认为映射本身是一个集合，Map也没有实现Collection接口，但是可以得到映射的视图——键集，值集，键/值对集

获取key集 Set<K> keySet();
获取value集 Collection values();
获取键/值对集 Set<Map.Entry<K,V>> entrySet()


# 7. key的哈希值得获取方式
```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);// ^ 进行异或运算
    }

    public native int hashCode();
```

由三元表达式可见如果key为null则将此键值对存储在下标为0的位置处

 
# 8. 总结
1. HashMap是一种散列表，**采用(数组+链表+红黑树)的存储结构，链表为单向链表**
2. HashMap的默认初始值为16，装载因子为0.75f，总容量为2的n次方
3. HashMap扩容时每次容量变为原来的两倍
4. 当桶的数量即数组的容量大于64且单个桶的数量即链表的长度大于8时，进行树化
5. 当单个桶中元素的数量小于6时，进行反树化
6. hashMap是非线性安全的容器
7. HashMap添加查找元素的时间复杂度为O(1)
8. HashMap支持一个key为null值，多个value为null

## 8.1. 如果HashMap的key是一个自定义的类该怎么办？

Object类的equals()方法是根据类的引用值判断两个值是否相同，在具体的使用过程中要根据实际情况进行重写,因为equals()方法与hashCode()方法是相互一致的,当equals()方法判断两个类相同为true时，则他们的散列值必定是相同的,因此要重写hashCode()方法.

**key为String的情况：**

如果key为String时,在进行存储时会根据String的hashCode()方法获取散列值和equals()方法判断key值是否唯一,可知String的equals()方法被重写了,是根据String的内容进行判断是否相同

**Key为自定义类的情况：**

如果key为自定义的类,则要重写自定义类的hashCode()方法和equals()方法,来达到key的唯一性,在HashSet中也是如此,重写类的hashCode()和equals()方法来保证HashSet类中的元素无重复.