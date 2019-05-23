# 简介
LinkedHashMap内部维护了一个双向链表，保证元素按照插入的顺序访问，也能以访问顺序访问。
LinkedHashMap可以看出是LinedList + HashMap

# 继承体系
![Alt](https://gitee.com/alan-tang-tt/yuan/raw/master/%E6%AD%BB%E7%A3%95%20java%E9%9B%86%E5%90%88%E7%B3%BB%E5%88%97/resource/LinkedHashMap.png)

LinkedHashMap继承自HashMap，拥有HashMap的所有特性，并且额外增加了一定的顺寻访问特定。

# 源码解析
## 属性

```java
//双向链表的头节点
transient LinkedHashMap.Entry<K,V> head;
//双向链表的尾节点
transient LinkedHashMap.Entry<K,V> tail;
//是否按访问顺序排序
final boolean accessOrder;
```
accessOrder为true时按访问顺序存储元素，为false时按插入顺序存储元素

插入排序指的是put时候的顺序


## 构造方法
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


## afterNodeAccess(boolean evict)方法
在hashmap中的调用put方法插入元素时，如果插入的是新的元素，则调用此方法
```java
void afterNodeInsertion(boolean evict) { 
    LinkedHashMap.Entry<K,V> first;

    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
```

## afterNodeAccess(Node e)方法
此方法在hashMap中put方法插入新节点之后调用

```java
//此时传过来的e为要插入的新的节点
//在进行插入新节点之后进行双向链表的维护

void afterNodeAccess(Node<K,V> e) { // move node to last
    //创建最后一个节点
    LinkedHashMap.Entry<K,V> last;
    //
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

