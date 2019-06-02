
<!-- TOC -->

1. [1. 简介](#1-%E7%AE%80%E4%BB%8B)
2. [2. 继承体系](#2-%E7%BB%A7%E6%89%BF%E4%BD%93%E7%B3%BB)
3. [3. 源码解析](#3-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)
   1. [3.1. 属性](#31-%E5%B1%9E%E6%80%A7)
   2. [3.2. 构造方法](#32-%E6%9E%84%E9%80%A0%E6%96%B9%E6%B3%95)
   3. [3.3. 内部类](#33-%E5%86%85%E9%83%A8%E7%B1%BB)
   4. [3.4. afterNodeInsertion(boolean evict)方法](#34-afternodeinsertionboolean-evict%E6%96%B9%E6%B3%95)
   5. [3.5. afterNodeAccess(Node e)方法](#35-afternodeaccessnode-e%E6%96%B9%E6%B3%95)
   6. [3.6. afterNodeRemoval(Node<K,V> e)](#36-afternoderemovalnodekv-e)
4. [4. 默认情况下是如何维护双向链表的](#4-%E9%BB%98%E8%AE%A4%E6%83%85%E5%86%B5%E4%B8%8B%E6%98%AF%E5%A6%82%E4%BD%95%E7%BB%B4%E6%8A%A4%E5%8F%8C%E5%90%91%E9%93%BE%E8%A1%A8%E7%9A%84)
5. [5. 总结](#5-%E6%80%BB%E7%BB%93)

<!-- /TOC -->
 

# 1. 简介
LinkedHashMap内部维护了一个双向链表，默认保证元素按照插入的顺序访问，也能以访问顺序访问。

插入顺序指的是put的顺序
访问顺序指的是只要访问该元素，在双向链表中就将此元素移除然后添加到链表末尾

# 2. 继承体系
![Alt](https://gitee.com/alan-tang-tt/yuan/raw/master/%E6%AD%BB%E7%A3%95%20java%E9%9B%86%E5%90%88%E7%B3%BB%E5%88%97/resource/LinkedHashMap.png)

**LinkedHashMap继承自HashMap，拥有HashMap的所有特性，并且额外增加了一定的顺寻访问特性。**

# 3. 源码解析
## 3.1. 属性

```java
//双向链表的头节点
transient LinkedHashMap.Entry<K,V> head;
//双向链表的尾节点
transient LinkedHashMap.Entry<K,V> tail;
//是否按访问顺序排序
final boolean accessOrder;
```

**accessOrder为true时按访问顺序存储元素，为false时按插入顺序存储元素**


## 3.2. 构造方法
```java
//指定初始容量和加载因子
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}

//指定初始容量
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}
   
//以默认的初始容量创建
public LinkedHashMap() {
    super();
    accessOrder = false;
}

//以Map集合创建
public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super();
    accessOrder = false;
    putMapEntries(m, false);
}


public LinkedHashMap(int initialCapacity,
                        float loadFactor,
                        boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}

```
前四个构造方法accessOrder为false，说明双向链表是以插入顺序存储元素
最后一个构造方法accessOrder从构造方法参数传入，如果为true，则实现按访问顺序存储元素

## 3.3. 内部类
```java
//位于linkedHashMap中
//此内部类用于维护双向链表
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;//前置节点和后置节点
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

//位于HashMap中
//此内部类用于在桶数组中存储元素
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
```

## 3.4. afterNodeInsertion(boolean evict)方法
在hashmap中的调用put方法插入元素时如果在桶中没有找到key元素，新插入元素时，插入完毕后调用此方法进行维护双向链表。

```java
void afterNodeInsertion(boolean evict) { 
    //创建双向链表中的首节点
    LinkedHashMap.Entry<K,V> first;

    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}

```

当删除元素时，此处并没有在双向链表中移除旧的元素，旧的原属还在双向链表的默认原始位置

## 3.5. afterNodeAccess(Node e)方法
此方法在hashMap中put方法插入新节点(桶中存在key元素)或者get方法时调用,此时表示元素被访问，因此按照访问顺序维护双向链表
具体查看HashMap源码中此方法的使用情景
```java
//此时传过来的e为要插入的新的节点，也就是最新访问的元素
//在进行插入新节点之后进行双向链表的维护
void afterNodeAccess(Node<K,V> e) {
    //创建最后一个节点
    LinkedHashMap.Entry<K,V> last;
    //如果accessorder为true(表示按访问顺序维护链表)并且访问的节点不是尾元素
    if (accessOrder && (last = tail) != e) {
        //先创建p节点并赋值e节点，并创建p节点的前置和后置节点并对其赋值
        LinkedHashMap.Entry<K,V> p = (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        //将p的后置节点赋值为null
        p.after = null;
        //如果p的前置节点b为空的化则表示p为头节点此时删除p之后，将p的后置节点a置为头节点
        if (b == null)
            head = a;
        else//否则的话置空p节点
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        //将p节点添加到双向链表的末尾
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        //双项链表的尾节点为p
        tail = p;
        ++modCount;//操作次数加1
    }
}
```
实现原理详解：
1. 如果accessOrder为true表示按访问顺序维护链表，并且要访问的元素不是尾节点
2. 上面个条件成立的情况下，则将要访问的节点从双向链表中删除，并将他添加到尾节点。
3. 尾节点尾最新的访问的元素。


## 3.6. afterNodeRemoval(Node<K,V> e)
将节点删除之后需要在双向链表中删除此节点，在remove方法中调用

```java
void afterNodeRemoval(Node<K,V> e) { 
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    //将p的前置节点和后置节点置为空
    p.before = p.after = null;
    //如果p为头节点，此时后置节点a为头节点
    if (b == null)
        head = a;
    else//b的后置节点为a
        b.after = a;
    //如果p节点为尾节点，此时将前置节点b置为尾节点
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```
实现原理详解：
1. 在删除节点之后需要调用此方法对双向链表进行维护，即删除此节点
2. 此时只需删除即可不需考虑双向链表的顺序


# 4. 默认情况下是如何维护双向链表的
上面几种方法是在accessOrder为true的情况下如何维护双向链表，使双向链表按照访问顺序进行维护。

但是在默认的情况下双向链表是按照插入顺序进行维护的，下面通过源码来分析如何进行维护

1. **在HashMap的put()方法中调用了一个重要的方法newNode()方法**

```java
  
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    /*
                    * 此处有一个重要的方法newNode
                    */
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
2. 先来查看HashMap中的newNode()方法如何实现
此方法会创建一个新的Node节点并返回
```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
    return new Node<>(hash, key, value, next);
}
```

3. 在LinkedHashMap中重写newNode()方法

重写newNode()方法并调用linkNodeLast(p)方法
```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}

``` 
4. 查看linkNodeLast()方法
```java
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    //创建节点
    LinkedHashMap.Entry<K,V> last = tail;
    //将心插入的节点赋值给尾节点
    tail = p;
    //下面代码将新节点插入到双向链表的尾部以完成对双向链表的维护
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```
原理分析：
在LinkedHashMap中重写newNode()方法在其中调用linkNodeLast()方法对双向链表进行维护，因此在插入的过程中已经进行维护。


# 5. 总结
1. LinkedHashMap继承自HashMap，处理具有HashMap的特性之外，还增加了自己的方法，主要是对双向链表维护的方法
2. LinkedHashMap内部维护了一个双向链表使元素有序
3. 当LinkedHashMap删除元素的时候，并没有在双向链表中移除元素，若需移除，需要重写LinkedHashMap
4. LinkedHashMap构造方法默认accessOrder为false，以插入的顺序维护双向链表
5. 适用于LRU缓存：最近最少未使用策略