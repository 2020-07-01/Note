<!-- TOC -->

- [1. 引言](#1-%e5%bc%95%e8%a8%80)
- [2. 源码解读](#2-%e6%ba%90%e7%a0%81%e8%a7%a3%e8%af%bb)
- [3. 注意](#3-%e6%b3%a8%e6%84%8f)

<!-- /TOC -->
# 1. 引言

Object类是所有对象及其数组的超类，任何对象都可以使用Object的方法。
# 2. 源码解读

jdk版本：1.8

```java
 
public class Object {

    private static native void registerNatives();
    static {
        registerNatives();
    }

    //返回运行时对象得类，一般在反射中常用到
    public final native Class<?> getClass();

    //返回对象得哈希码值
    public native int hashCode();

    //判断其他对象是否与此对象相等，此处是根据对象的引用地址进行判断
    public boolean equals(Object obj) {
        return (this == obj);
    }

    //进行对象的克隆，此处分为浅克隆和深克隆两种
    protected native Object clone() throws CloneNotSupportedException;

    //对象的字符产表示形式
    public String toString() {
        return getClass().getName() + "@" + Integer.toHexString(hashCode());
    }

    //唤醒阻塞队列中的某个线程     
    public final native void notify();
    
    //唤醒阻塞队列中的所有线程
    public final native void notifyAll();

    //使线程进入暂停状态
    public final native void wait(long timeout) throws InterruptedException;

    public final void wait(long timeout, int nanos) throws InterruptedException {
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos > 0) {
            timeout++;
        }

        wait(timeout);
    }

     
    public final void wait() throws InterruptedException {
        wait(0);
    }
    //垃圾收集器调用此方法进行垃圾的收集
    protected void finalize() throws Throwable { }
}

```

# 3. 注意
Object中的wait()方法和notify(),notifyAll()方法**必须在同步代码块或者同步方法中调用**

1. wait()方法的官网解释：

    导致当前线程等待，直到另一个线程为该对象调用{@link java.lang.Object#notify()}方法或{@link java.lang.Object#notifyAll()}方法，或者指定的时间已经过去。


    一个对象的非同步方法可以被任何线程调用，一个对象的同步方法必须在线程获取对象的独占锁之后才能对其进行访问。如果在线程调用对象的同步方法时，该对象的独占锁在被其他线程占用，那么此线程将被阻塞，并添加到阻塞队列中。

    wait()方法的作用是强制当前线程释放锁，并且释放对CPU的控制权。所以在线程释放某个对象的锁之前，先必须获取对象的锁，而只有在线程访问同步方法时才会获取对象的锁，所以wait()方法必须在同步代码块或者同步方法中调用。

    notify()方法和notifyAll()方法是唤醒阻塞队列中的线程。

