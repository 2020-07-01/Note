 
<!-- TOC -->

1. [1. 简介](#1-%E7%AE%80%E4%BB%8B)
2. [2. 继承体系](#2-%E7%BB%A7%E6%89%BF%E4%BD%93%E7%B3%BB)
3. [3. 存储结构](#3-%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84)
4. [4. 源码解析](#4-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)
   1. [4.1. HashMap属性](#41-hashmap%E5%B1%9E%E6%80%A7)
   2. [4.2. Node内部类](#42-node%E5%86%85%E9%83%A8%E7%B1%BB)
   3. [4.3. 构造方法](#43-%E6%9E%84%E9%80%A0%E6%96%B9%E6%B3%95)
   4. [4.4. put( K key,V value)方法](#44-put-k-keyv-value%E6%96%B9%E6%B3%95)
   5. [4.5. get(Object key)方法](#45-getobject-key%E6%96%B9%E6%B3%95)
5. [5. 总结：](#5-%E6%80%BB%E7%BB%93)

<!-- /TOC -->
# 1. 简介
HashMap采用key/value的存储结构，每个可以对应一个唯一的value，查询和修改的速度很快，接近于O(1)的平均时间复杂度。**他是线性安全的，但是不能保证元素的顺序性。**

# 2. 继承体系

![Alt](https://gitee.com/alan-tang-tt/yuan/raw/master/%E6%AD%BB%E7%A3%95%20java%E9%9B%86%E5%90%88%E7%B3%BB%E5%88%97/resource/HashMap.png)

1. HashMap实现了Serializable接口，可以被序列化
2. HashMap实现了Cloneable接口，可以被克隆
3. HashMap继承自AbstractMap类，实现了Map接口，具有Map的所用功能
4. HashMap的顶层接口为Map

# 3. 存储结构
![Alt](https://gitee.com/alan-tang-tt/yuan/raw/master/%E6%AD%BB%E7%A3%95%20java%E9%9B%86%E5%90%88%E7%B3%BB%E5%88%97/resource/HashMap-structure.png)

* HashMap采用了(**数组 + 链表 + 红黑树**)的实现结构，数组中的一个元素称作为桶。

* 在添加元素时，会根据key的hash值计算出元素在数组中的位置，如果该位置中没有元素，则直接把元素存储在此位置，如果该位置有元素了，**则把元素以链表的形式存放在链表的尾部**。

* 当链表的元素个数达到一定的数量后(且数组的长度达到一定的长度))，**则把链表转换为红黑树，从而提高效率**。


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
## 4.2. Node内部类
Node是一个单向链表节点
```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;//存储key的hash值
        final K key;
        V value;
        Node<K,V> next;
        ......
}
```


1. 容量
**容量为数组的长度，即桶的个数，默认为16个，最大为2的30次方个，当容量大于64时才进行树化**
2. 装载因子
装载因子用来计算容量达到多少时进行扩容，**默认为0.75**
实际容量/初始容量
3. 树化
树化指的是将单向链表节点以红黑树的方式进行存储
**当容量达到64个且链表的长度达到8个时才进行树化，当链表小于6个时才进行反树化**

## 4.3. 构造方法
1. HashMap()构造方法
空构造方法，全部使用默认值，默认容量为16，装载因子为0.75，扩容阈值为12
```java
 public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;  
    }
```


2. HashMap(int initialCapacity)构造方法
指定初始容量initialCapacity
使用此构造方法时调用HashMap(int initialCapacity, float loadFactor)构造方法,传入默认的装载因子0.75
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

## 4.4. put( K key,V value)方法
put(k key, V value)
如果key相同的时候，此方法会替换旧值并返回旧值
```java

public V put(K key, V value) {
    //1.hash(key)：计算key的hash值
    return putVal(hash(key), key, value, false, true);
}

//evict指的是是否进行
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
                e = p;//将桶中第一个元素的值赋值给临时变量e
            
            //5.如果第一个元素p是树节点，则调用树节点的putTreeVal方法插入元素，此时链表为红黑树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //6.遍历整个桶p对应的链表查找是否有与key相同的元素
            
            else {
                for (int binCount = 0; ; ++binCount) {
                    //8.如果遍历完整个链表都没有找到相同的key元素，说明key对应的元素不存在，则在链表最后插入一个新的节点
                    //将桶p的下一个节点赋值给临时变量e,如果下一个节点为null的话则进行创建新的节点插入
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

            //9.如果找到了对应的key的元素，则e存储的是与key相同的节点
            if (e != null) { 
                //记录下旧值
                V oldValue = e.value;

                //判断是否需要替换旧值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;//此时e为新节点，替换旧值

                //在节点插入之后，会进行一些操作在LinkedHashMap中用得到
                //此时传入的e为插入的新节点
                afterNodeAccess(e);

                return oldValue;
            }
        }

        ++modCount;
        //10.判断是否需要扩容
        if (++size > threshold)
            //扩容函数
            resize();

        //如果没有找到元素，在linkedHashMap中进行此操作
        afterNodeInsertion(evict);
        //如果没有找到元素则返回null
        return null;
    }
```
存储过程:
1. 计算key的hash值
2. 如果桶(数组)数量为0，则进行桶的初始化
3. 如果key所在的桶p没有元素，则直接将此元素插入
4. 如果key所在的桶p中的第一个元素的key与待插入的key相同，则说明找到了了元素，进行9步骤
5. 如果第一个元素是树节点，则调用树节点的putTreeVal()方法进行插入
6. 如果不是以上三种情况，则遍历桶查看key是否存在与链表中
7. 如果存在于链表中，则进行9步骤
8. 如果遍历链表没有找到此元素，则在链表最后插入新的节点，并判断是否需要进行树化
9. 判断是否进行新旧值的替换
10. 如果插入了节点，则数量+1并判断是否需要进行扩容


## 4.5. get(Object key)方法
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
查询过程:
1. 计算key的hash值
2. 找到key所在的桶及其第一个元素
3. 如果第一个元素的key等于待查找的key，则直接返回
4. 如果第一个元素为树节点，则按树的方式进行查找；否则按链表方式进行查找

# 5. 总结：
1. HashMap是一种散列表，采用(**数组+链表+红黑树**)的存储结构
2. HashMap的默认初始值为16，装载因子为0.75f，总容量为2的n次方
3. hashCode最大容量为2的30次方
4. HashMap扩容时每次容量变为原来的两倍,即2的n次方方式进行扩容
5. 当桶的数量大于64且单个桶的数量大于8时，进行树化
6. 当单个桶中元素的数量小于6时，进行反树化
7. hashMap是非线性安全的容器
8. HashMap添加查找元素的时间复杂度为O(1)
9. HashMap支持一个key为null值，多个value为null
10. HashMap与Object的hashCode方法关系：(**面试题**)
    * 每个类都默认继承Object类，Object类为每个类的超类
    * 因此hashMap重写了Object类的hashCode方法