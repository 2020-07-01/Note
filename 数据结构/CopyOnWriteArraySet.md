<!-- TOC -->

- [1. 简介](#1-简介)
- [2. 源码分析](#2-源码分析)
- [3. 总结](#3-总结)

<!-- /TOC -->
# 1. 简介
CopyOnWriteArraySet的底层实现是CopyOnWriteArrayList，采用了读写分离的思想，他是线程安全的。

# 2. 源码分析
```java

public class CopyOnWriteArraySet<E> extends AbstractSet<E>
        implements java.io.Serializable {
    private static final long serialVersionUID = 5457747651344034263L;

    //
    private final CopyOnWriteArrayList<E> al;
    
    //构造方法：底层使用CopyOnWriteArrayList来存储元素
    public CopyOnWriteArraySet() {
        al = new CopyOnWriteArrayList<E>();
    }


    public CopyOnWriteArraySet(Collection<? extends E> c) {
        if (c.getClass() == CopyOnWriteArraySet.class) {
            @SuppressWarnings("unchecked") CopyOnWriteArraySet<E> cc =
                (CopyOnWriteArraySet<E>)c;
            al = new CopyOnWriteArrayList<E>(cc.al);
        }
        else {
            al = new CopyOnWriteArrayList<E>();
            al.addAllAbsent(c);
        }
    }

    //获取元素数量
    public int size() {
        return al.size();
    }
    
    //查看集合是否为空
    public boolean isEmpty() {
        return al.isEmpty();
    }

    //查看元素是否存在集合中
    public boolean contains(Object o) {
        return al.contains(o);
    }
    
    //集合抓换为数组
    public Object[] toArray() {
        return al.toArray();
    }
 
    
    public <T> T[] toArray(T[] a) {
        return al.toArray(a);
    }

    //清空所有的元素
    public void clear() {
        al.clear();
    }
 
    //删除元素
    public boolean remove(Object o) {
        return al.remove(o);
    }

    //添加元素
    //此处会检测是否存在元素，当元素不存在的时候进行添加
    public boolean add(E e) {
        return al.addIfAbsent(e);
    }
 
    //查看集合中是否包含集合c中的所有元素
    public boolean containsAll(Collection<?> c) {
        return al.containsAll(c);
    }

    //并集
    public boolean addAll(Collection<? extends E> c) {
        return al.addAllAbsent(c) > 0;
    }

    public boolean removeAll(Collection<?> c) {
        return al.removeAll(c);
    }

 
    public boolean retainAll(Collection<?> c) {
        return al.retainAll(c);
    }

    //返回一个迭代器
    public Iterator<E> iterator() {
        return al.iterator();
    }

    /*
    * 此处下次再看
    */
    public boolean equals(Object o) {
        //如果两个对象的引用值相等-是同一个对象，返回true
        if (o == this)
            return true;
        //如果o不是set集合返回false
        if (!(o instanceof Set))
            return false;
        Set<?> set = (Set<?>)(o);
        //创建迭代器
        Iterator<?> it = set.iterator();
        //创建元素数组
        Object[] elements = al.getArray();
        int len = elements.length;

        boolean[] matched = new boolean[len];
        int k = 0;
        outer: while (it.hasNext()) {
            if (++k > len)
                return false;
            Object x = it.next();
            for (int i = 0; i < len; ++i) {
                if (!matched[i] && eq(x, elements[i])) {
                    matched[i] = true;
                    continue outer;
                }
            }
            return false;
        }
        return k == len;
    }

    public boolean removeIf(Predicate<? super E> filter) {
        return al.removeIf(filter);
    }

    public void forEach(Consumer<? super E> action) {
        al.forEach(action);
    }
 
    //分割迭代器
    public Spliterator<E> spliterator() {
        return Spliterators.spliterator
            (al.getArray(), Spliterator.IMMUTABLE | Spliterator.DISTINCT);
    }

    //比较两个元素是否相等
    private static boolean eq(Object o1, Object o2) {
        return (o1 == null) ? o2 == null : o1.equals(o2);
    }
}

```

# 3. 总结
1. CopyOnWriteArraySet的底层实现是数组
2. CopyOnWriteArrayList中的元素是可以重复的，但是CopyOnArraySet中的元素是不重复的，调用的方法不同。
3. CopyOnWriteArraySet是线程安全的，并且元素是有序的。