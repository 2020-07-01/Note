<!-- TOC -->

- [1. 简介](#1-简介)
- [2. 存储结构](#2-存储结构)
- [3. 继承体系](#3-继承体系)
- [4. 源码解析](#4-源码解析)
    - [4.1. 主要属性](#41-主要属性)
    - [4.2. 添加元素put(K key,V value)](#42-添加元素putk-keyv-value)
    - [4.3. 初始化桶数组](#43-初始化桶数组)
    - [4.4. 删除元素remove()](#44-删除元素remove)
- [5. 总结](#5-总结)

<!-- /TOC -->

# 1. 简介
ConcurrentHashMap是HashMap的线程安全版本

# 2. 存储结构
ConcurrentHashMap与HashMap一样，内部也是采用(数组+链表+红黑树)的结构来存储元素。
ConcurrentHashMap内部采用的是分段锁，每次只锁住一个桶，而HashTable每次锁住的是整个table数组，因此相对于HashTable，ConcurrentHashMap的性能具有很大的提高。

# 3. 继承体系
![Alt](https://raw.githubusercontent.com/hello-qiang/Note/master/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/%E5%9B%BE%E7%89%87/ConcurrentHashMap.png?token=AKE36TBMJGOBVAQMHZ5EKO246JL6M)

# 4. 源码解析

## 4.1. 主要属性
```java
    private transient volatile int sizeCtl;
```
关于sizeCtl:
sizeCtl存储的是扩容阈值
-1:表示有线程正在进行初始化工作
&gt;0：表示初始化和扩容完成后下一次的扩容阈值
0：默认值，后续在真正进行初始化的时候使用默认容量16
-(1+nThreads):表示有n个线程正在一起扩容

## 4.2. 添加元素put(K key,V value)
先找到元素所在的桶，然后采用分段锁的思想锁住整个桶，在进行操作。

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    //key和value均不能为null
    if (key == null || value == null) throw new NullPointerException();
    //计算hash值，key的hash值
    int hash = spread(key.hashCode());
    
    int binCount = 0;
    
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f;//f为key元素的所在的桶的首节点
        int n, i, fh;
        //如果桶tab未进行初始化或者元素为0，则进行初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();

        //如果要插入的元素所在的桶中还没有元素，则把这个元素插入到桶中
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,new Node<K,V>(hash, key, value, null)))
                break;                     
        }

        //如果桶中第一个元素f的hash值为MOVED，则当前线程帮忙一起迁移元素
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);

        //如果这个桶不为空，并且不存在迁移元素，则锁住这个桶
        else {
            V oldVal = null;//创建旧节点
            synchronized (f) {//锁住这个桶
                //如果要插入的元素在这个桶中
                if (tabAt(tab, i) == f) { 
                    if (fh >= 0) {
                        //桶中元素的个数赋值为1然后进行遍历桶，每次遍历后binCount的值加1
                        binCount = 1;
                        //e为首节点
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //如果节点的hash值相同，并且key值相同
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;//替换旧值
                                break;//退出循环
                            }
                            Node<K,V> pred = e;//上一个节点
                            //如果遍历链表到最后一个节点还没有找到元素则把新节点插入到链表的尾部，并退出循环
                            if ((e = e.next) == null) {//如果为最后一个节点
                                pred.next = new Node<K,V>(hash, key,value, null);
                                break;
                            }
                        }
                    }
                    //如果第一个节点是树节点，则调用红黑树的方法进行插入
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                        value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            //表示成功插入的了元素或者找到了元素
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                
                //如果当前元素存在则返回旧值
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```
 
## 4.3. 初始化桶数组
```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; 
    int sc;
    //将table赋值给tab，如果桶为空或者table数组的长度为0的情况下
    while ((tab = table) == null || tab.length == 0) {
        //sizeCtl<0说明正在进行初始化或者进行扩容，则让出cpu 
        if ((sc = sizeCtl) < 0)//-1表示有其他线程正在进行初始化工作
            Thread.yield(); //让步当前执行的线程给其他具有相同优先级的线程
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {//Unsafe是一个比较底层的类，在于提高Java的运行性能
     
            try {
                //如果tab还未进行初始化
                if ((tab = table) == null || tab.length == 0) {
                    //n存储的是-1或者初始化容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];//以n为初始化容量创建数组
                    table = tab = nt;//进行赋值操作
                    //设置sc为数组长度的0.75倍
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;//sizeCtl存储的是扩容阈值
            }

            break;
        }
    }
    return tab;
}
```
初始化过程：
1.sizeCtl在初始化后存储的是扩容阈值
2.扩容阈值sizeCtl是数组大小的0.75倍


## 4.4. 删除元素remove()
先找到元素所在的桶，然后采用分段锁的思想，将整个桶锁住再进行操作
```java
public V remove(Object key) {
    return replaceNode(key, null, null);
}

final V replaceNode(Object key, V value, Object cv) {
    int hash = spread(key.hashCode());//计算hash值
    //创建tab数组进行死循环
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; 
        int n, i, fh;
        //如果桶数组为null或者元素数量为0或者key所在的桶为空，则退出循环
        if (tab == null || (n = tab.length) == 0 ||
            (f = tabAt(tab, i = (n - 1) & hash)) == null)
            break;
        //此时f为key所在桶的首节点
        //如果f的hash值为MOVED，表示此时正在进行扩容中，则协助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);

        else {
            V oldVal = null;//创建旧值
            boolean validated = false;//标记是否处理过
            //锁住整个桶
            synchronized (f) {//f为桶的首节点
        
                if (tabAt(tab, i) == f) {
                    //fh存储的是桶中首节点的hash值
                    if (fh >= 0) {
                        validated = true;//表示进行处理过 
                        //遍历链表寻找目标节点
                        for (Node<K,V> e = f, pred = null;;) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {
                                V ev = e.val;
                                if (cv == null || cv == ev ||
                                    (ev != null && cv.equals(ev))) {
                                    oldVal = ev;
                                    if (value != null)
                                        e.val = value;
                                    else if (pred != null)
                                        pred.next = e.next;
                                    else
                                        setTabAt(tab, i, e.next);
                                }
                                break;
                            }
                            pred = e;
                            if ((e = e.next) == null)
                                break;
                        }
                    }
                    //如果首节点f为树节点则采用树节点的方法进行删除元素
                    else if (f instanceof TreeBin) {
                        validated = true;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val;
                            if (cv == null || cv == pv ||
                                (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                else if (t.removeTreeNode(p))
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                }
            }
            //如果进行处理过
            if (validated) {
                if (oldVal != null) {
                    if (value == null)
                        addCount(-1L, -1);
                    return oldVal;//如果找到则返回旧值
                }
                break;
            }
        }
    }
    return null;//返回空值
}

```
 

# 5. 总结
1. 扩容后数组大小为旧桶数组大小的两倍
2. ConcurrentHashMap是HashMap的线程安全版本
3. 底层与HashMap一样采用数组+链表+红黑树的方式进行存储
4. ConcurrentHashMap采用分段锁的思想，使用Java关键字Synchronized来锁住每个桶，然后对桶进行操作
