<!-- TOC -->

- [1. 简介](#1-简介)
- [2. 存储结构](#2-存储结构)
- [3. 源码解析](#3-源码解析)
- [4. 总结](#4-总结)

<!-- /TOC -->
# 1. 简介
HashSet的底层是HashMap实现的，HashMap元素是无序的，因此使用LinkedHashMap维护一个双向对列来保证HashMap元素有序。
HashSet也是如此，使用LinkedHashSet来保证元素的有序性。

# 2. 存储结构
LinkedHashSet底层实现是LinkedHashMap,而LinkedHashMap底层实现是HashMap并且维护了一个双向链表来排序，因此底层实现还是(数组+链表+红黑树)。

# 3. 源码解析

```java
 
package java.util;

public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {

    private static final long serialVersionUID = -2851667679971038690L;

    //super()方法指的是HashSet中的一个没有修饰符的构造方法
    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }
 
    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }
 
    public LinkedHashSet() {
        super(16, .75f, true);
    }
  
    public LinkedHashSet(Collection<? extends E> c) {
        super(Math.max(2*c.size(), 11), .75f, true);
        addAll(c);
    }

    //可分割得迭代器，用于多线程并行处理
    @Override
    public Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, Spliterator.DISTINCT | Spliterator.ORDERED);
    }
}
```

* HashSet中的特殊构造方法
```java
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```
此方法调用的是LinkedHashMap中的方法：
**此方法accessOrder默认为false，因此LinkedHashSet只能按照插入顺序进行排序**
```java
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}
```

# 4. 总结
1. LinkedHashSet继承自HashSet，他的添加、删除等操作方法使用的是HashSet中的方法
2. LinkedHashSet使用LinkedHashMap来存储元素，因此元素是有序的。
3. LinkedHashSet只能按插入顺序进行排序，因此accessOrder默认为false



