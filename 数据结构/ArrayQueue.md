# 简介

双端队列是一种两端都可以进行操作元素的队列
ArrayDeque是一种以数组方式实现的双端队列，他是非线性安全的。

每次扩容都是2的n次方
不能包含null指针


# 源码解析

## 属性
```java
   //存储元素的数组
    transient Object[] elements;  
    //队列的头节点
    transient int head;
    //队列的尾节点
    transient int tail;
    //最小初始容量
    private static final int MIN_INITIAL_CAPACITY = 8;
```

ArrayDeque使用数组存储元素，并使用头节点和尾节点表示队列的头和尾，最小容量是8



## 构造方法

```java
//默认构造方法，初始容量为16
public ArrayDeque() {
    elements = new Object[16];
}
```






#　总结
ArrayDeque是采用数组方式实现的双端队列
ArrayDeque的出队和入队是通过头尾指针循环利用数组来实现的
