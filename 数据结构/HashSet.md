#引入
1. 集合(Collection)和集合(Set)有什么区别？
2. hashSet如何实现元素的不重复？
3. hashSet是否允许null元素？
4. HashSet是有序的吗?
5. HashSet是同步的吗?
6. 什么是fail-fast?


# 简介
广义上讲，集合包含Java.util包下的容器类，包括Collection及Map相关的类。

中义上讲，一般指Java集合中的Collection相关的类，不包含map相关的类，Collection接口下有三个接口：list，set和queue

侠义上讲，数学上的集合指的是不包含重复元素的容器，即集合中不存在两个相同的元素，在Java中对应的set

HashSet是Set的一种实现方式，底层主要是使用HashMap来确保元素的不重复。

#　源码分析
## 属性

```java
//内部使用的是hashmap
private transient HashMap<E,Object> map;
//虚拟对象，用来作为value存储在map中
private static final Object PRESENT = new Object();
```

## 构造方法
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

## 添加元素
添加元素直接调用hashmap的put方法，把元素本身作为key，把present作为value，也就是这个map中的所有value都是一样的。

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

## 删除元素
map的remove方法返回的是删除元素的value值，而set返回的是boolean类型，如果没有该元素，则map返回的是null，此时与PRESENT相比较之后返回false，否则map返回的是PRESENT，相比较之后返回的是true

```java
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```

##　查询元素
Set没有get方法，只有contains()方法，直接调用map的containsKey()方法

```java
public boolean contains(Object o) {
    return map.containsKey(o);
}
```


# 总结
1. HashSet内部使用的是HashMap的key存储元素，保证元素的不重复
2. HashSet是无序的，因为HashMap的key元素是无序的
3. HashSet中允许使用一个null元素，因为HashMap中允许key为null
4. HashSet是非线程安全的
5. HashSet没有get方法，使用contains方法，HashMap中有get方法

# 快速迭代机制
fail-fast是Java集合中的一种错误机制。
当使用迭代器迭代时，如果发现集合有修改，则快速失败做出响应，抛出异常。
这种修改有可能是其他线程的修改，也有可能是当前线程自己修改导致的，比如迭代的过程中直接调用remove方法删除元素等。

并不是Java中的所有集合都有快速迭代机制。

fail-fast机制是如何实现的？
在ArrayList、HashMap中都有一个属性叫做modCount，每次对集合的修改这个值都会加1，在遍历前记录这个值到expectedModCount，在遍历中会检查两者是否一至，如果出现不一致则说明有修改，抛出异常。