
<!-- TOC -->

- [1. 简介](#1-简介)
- [2. 存储结构](#2-存储结构)
    - [2.1. 常问问题：<font color = "red">如何保证元素的不重复</font>](#21-常问问题font-color--red如何保证元素的不重复font)
- [3. 源码分析](#3-源码分析)
    - [3.1. 属性](#31-属性)
    - [3.2. 构造方法](#32-构造方法)
    - [3.3. 添加元素](#33-添加元素)
    - [3.4. 删除元素](#34-删除元素)
    - [3.5. 查询元素](#35-查询元素)
- [4. 总结](#4-总结)

<!-- /TOC -->
# 1. 简介
广义上讲，集合包含Java.util包下的容器类，包括Collection及Map相关的类。

中义上讲，一般指Java集合中的Collection相关的类，不包含map相关的类，Collection接口下有三个接口：list，set和queue

侠义上讲，数学上的集合指的是不包含重复元素的容器，即集合中不存在两个相同的元素，在Java中对应的set

**HashSet是Set的一种实现方式，底层主要是使用HashMap来确保元素的不重复，因此HashSet元素与HashMap元素一样也是无序的。**

# 2. 存储结构
HashSet底层使用的是HashMap对象，因此底层的存储结构与HashMap一样，使用的是(数组+链表+红黑树)的方式
## 2.1. 常问问题：<font color = "red">如何保证元素的不重复</font>
首先说明在Collection集合中只有List集合的元素是可以重复的，Set集合和Map集合中的元素是不可以进行重复的。

1. 关于HashMap中的散列冲突问题：
在HashMap中重写了Object的hashCode方法，用来获取元素key的hash值，如果key的hash值不相同，则表示元素key不相同；如果key的hash值相同，则要进一步判断key元素的值是否相同，此处使用Object的equals方法(此方法用来判断对象的引用值是否相同)来判断 **引用值**，如果不相同则出现散列冲突-即不同对象具有同一个散列值；如果相同则判断value是否相同，然后进行新旧值的替换


2. 关于HashSet中如何保证元素不重复：
Object中的equals方法比较的对象的引用值
在hashSet中底层实现是HashMap对象，而保存的是key元素，value元素全部为Object，因此如果只通过hashcode和equals方法来判断key元素是否相同不能保证元素的不重复，比如实现同一个类的具有相同的内容的不同对象，此时两个对象是相同的。

解决方法：重写key元素的equals方法，不仅判断对象的hash值，还要判断对象的内容是否相同

3. 为什么要重写equals方法？
因为Object的equals方法比较的是两个元素的引用值，而在key元素中需要判断两个元素的内容是否相同。

# 3. 源码分析
## 3.1. 属性

```java
//内部使用的是hashmap
private transient HashMap<E,Object> map;
//虚拟对象，用来作为value存储在map中
private static final Object PRESENT = new Object();
```

## 3.2. 构造方法
```java

//底层是初始容量为16，加载因子为0.75的hashMap
public HashSet() {
    map = new HashMap<>();
}

//当c.size()大于16时，初始容量为c的大小，反之初始容量为16，加载因子默认为hashmap的0.75
public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}

//指定初始容量和加载因子
public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}

//指定初始容量
public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}

//只能被同一个包进行调用，是LinkedHashSet的专属方法
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}

```

## 3.3. 添加元素
添加元素直接调用hashmap的put方法，把元素本身作为key，把present作为value，也就是这个map中的所有value都是一样的。

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

## 3.4. 删除元素
map的remove方法返回的是删除元素的value值，而set返回的是boolean类型，如果没有该元素，则map返回的是null，此时与PRESENT相比较之后返回false，否则map返回的是PRESENT，相比较之后返回的是true

```java
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```

## 3.5. 查询元素
Set没有get方法，只有contains()方法，直接调用map的containsKey()方法

```java
public boolean contains(Object o) {
    return map.containsKey(o);
}
```


# 4. 总结
1. HashSet内部使用的是HashMap的key存储元素，保证元素的不重复
2. HashSet是无序的，因为HashMap的key元素是无序的
3. HashSet中允许使用一个null元素，因为HashMap中允许key为null
4. HashSet是非线程安全的
5. HashSet没有get方法，使用contains方法，HashMap中有get方法

 