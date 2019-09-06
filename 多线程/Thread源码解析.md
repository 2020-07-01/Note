<!-- TOC -->

- [1. 引言](#1-引言)
- [2. Thread源码](#2-thread源码)
    - [2.1. Thread属性](#21-thread属性)
    - [2.2. native方法](#22-native方法)
    - [2.3. sleep()方法](#23-sleep方法)
    - [2.4. Thread的初始化方法init():](#24-thread的初始化方法init)
    - [2.5. Thread构造方法](#25-thread构造方法)
    - [2.6. start()方法](#26-start方法)
    - [2.7. run()方法](#27-run方法)
    - [2.8. 中断方法](#28-中断方法)
    - [2.9. join()方法](#29-join方法)
    - [2.10. 一般方法](#210-一般方法)
    - [2.11. 其他方法](#211-其他方法)
- [3. 理解](#3-理解)
    - [3.1. Thread的初始化过程](#31-thread的初始化过程)
    - [3.2. 线程的启动过程](#32-线程的启动过程)

<!-- /TOC -->
# 1. 引言
多线程的创建有两种方法，一种是继承Thread类，一种是实现Runnable接口，两种方法的实质是Thread类，Thread类种包含了线程的初始化，线程的启动，ThreadGroup和ThreadLocal等内容，这里从源码角度对Thread进行分析，理清自己的逻辑思维。

# 2. Thread源码
jdk版本：1.8

对于在jdk已经启用的方法和关于堆栈追踪的方法这里没有分析，关于堆栈追踪方法有待研究。

## 2.1. Thread属性
```java
public class Thread implements Runnable {

    private static native void registerNatives();
    static {
        registerNatives();
    }

    private volatile String name;//线程名称
    private int            priority;//线程优先级
    private Thread         threadQ;
    private long           eetop;

    //是否单步执行此线程
    private boolean     single_step;

    //是否为守护线程
    private boolean     daemon = false;

    //jvm的状态
    private boolean     stillborn = false;

    //目标执行类
    private Runnable target;

    //线程组
    private ThreadGroup group;

    //类加载器
    private ClassLoader contextClassLoader;

    /* The inherited AccessControlContext of this thread */
    private AccessControlContext inheritedAccessControlContext;

    //对匿名线程进行编号
    private static int threadInitNumber;
    private static synchronized int nextThreadNum() {
        return threadInitNumber++;
    }

    //创建ThreadLocalMap对象，在存储线程数据时ThreadLocal会调用此对象，将它与当前线程联系起来
    ThreadLocal.ThreadLocalMap threadLocals = null;
 
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;

    //为该线程请求的堆栈大小，一般情况下忽略，默认为0
    private long stackSize;

    private long nativeParkEventPointer;

    //线程的ID
    private long tid;

    //生成线程的ID值
    private static long threadSeqNumber;

    //线程的状态，初始化时为0表示线程未启动
    private volatile int threadStatus = 0;

    //生成线程的ID值
    private static synchronized long nextThreadID() {
        return ++threadSeqNumber;
    }

    volatile Object parkBlocker;

    //待研究
    private volatile Interruptible blocker;
    private final Object blockerLock = new Object();

    //待研究
    void blockedOn(Interruptible b) {
        synchronized (blockerLock) {
            blocker = b;
        }
    }

    //设置线程的优先级，1优先级最低，10优先级最高
    public final static int MIN_PRIORITY = 1;
    public final static int NORM_PRIORITY = 5;
    public final static int MAX_PRIORITY = 10;
    
```

## 2.2. native方法
```java

    //返回当前执行线程对象的引用
    public static native Thread currentThread();

    //表示当前线程愿意进行让步，将cpu的使用权让步给就绪队列中具有同等优先级的其他线程
    public static native void yield();

    //使当前线程暂时处于休眠状态，以毫秒为单位
    public static native void sleep(long millis) throws InterruptedException;

    //开启一个线程
    private native void start0();

```

## 2.3. sleep()方法
```java
    //使当前线程处于休眠状态，以毫秒和纳秒为单位
    public static void sleep(long millis, int nanos)
    throws InterruptedException {
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }
        //时间粒度的转换
        if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
            millis++;
        }

        sleep(millis);
    }
```
**sleep()方法使调用它的线程处于休眠状态，此方法不会释放锁对象**

## 2.4. Thread的初始化方法init():
```java
    /*
    * 线程的初始化方法
    * ThreadGroup为线程组
    * Runnable target为被调用run方法的目标对象
    * String name为线程的名字
    * long stackSize为新线程请求的堆栈大小
    */
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize) {
        init(g, target, name, stackSize, null);
    }

    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
        //如果name为null则抛出异常
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }
        //初始化name
        this.name = name;
        //获取新线程的父线程，即在一个线程之下创建另一个线程，因此线程有父与子的关系
        Thread parent = currentThread();
        //获取系统的安全管理器
        SecurityManager security = System.getSecurityManager();
        //如果线程组为空
        if (g == null) {
            //如果security不为null则新线程所在group为security的group
            if (security != null) {
                g = security.getThreadGroup();
            }
            //如果新线程的group还为null，则新线程的group为父线程的group
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }
        g.checkAccess();

        
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }
        //增加线程组中未启动的线程的数量
        g.addUnstarted();
        //设置线程组
        this.group = g;
        //当前线程是否未守护线程与其父线程一致
        this.daemon = parent.isDaemon();
        //设置当前线程的优先权为父线程的优先权
        this.priority = parent.getPriority();

        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
        if (parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        //设置此线程请求的堆栈大小
        this.stackSize = stackSize;
        //设置此线程的ID
        tid = nextThreadID();
    }

    
```

## 2.5. Thread构造方法

```java
    /*
    *   线程的构造方法
    * ThreadGroup为线程组
    * Runnable target为被调用run方法的目标对象
    * String name为线程的名字
    * long stackSize为新线程请求的堆栈大小
    */
    public Thread() {
        init(null, null, "Thread-" + nextThreadNum(), 0);
    }

    public Thread(Runnable target) {
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }

    Thread(Runnable target, AccessControlContext acc) {
        init(null, target, "Thread-" + nextThreadNum(), 0, acc);
    }

    public Thread(ThreadGroup group, Runnable target) {
        init(group, target, "Thread-" + nextThreadNum(), 0);
    }

    public Thread(String name) {
        init(null, null, name, 0);
    }

    public Thread(ThreadGroup group, String name) {
        init(group, null, name, 0);
    }

    public Thread(Runnable target, String name) {
        init(null, target, name, 0);
    }
     
    public Thread(ThreadGroup group, Runnable target, String name) {
        init(group, target, name, 0);
    }
     
    public Thread(ThreadGroup group, Runnable target, String name,
                  long stackSize) {
        init(group, target, name, stackSize);
    }

```

## 2.6. start()方法
```java
    //在线程启动的时候会将此线程添加到线程组中，如果线程启动失败则会通知线程组进行回滚
    public synchronized void start() {
        //线程的状态为0则表示线程未启动，如果不为0则抛出异常
        if (threadStatus != 0)
            throw new IllegalThreadStateException();
        //将此线程添加到线程组中
        group.add(this);
        //是否已启动
        boolean started = false;
        try {
            //启动线程的方法，启动之后设置标识未已启动
            start0();
            started = true;
        } finally {
            try {
                //如果未成功启动则通知线程组线程启动失败
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                
            }
        }
    }
    
```
Thread通过start方进行线程的启动，在启动之后线程的启动标志会发生变化，然后将此线程添加到ThreadGroup中，如果线程未能正常启动则ThreadGroup会进行回滚操作，恢复之前的状态。

线程存在父子关系，一般在父线程中创建并启动一个子线程，子线程的线程组为系统安全管理器的线程组，(安全管理器如果为空则为父线程的线程组)，所以父线程与子线程共用一个线程组


## 2.7. run()方法
```java
    //实现了runnable接口的run方法
    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }

    //此方法又系统调用，即在线程退出之前进行清理
    private void exit() {
        if (group != null) {
            group.threadTerminated(this);
            group = null;
        }
        /* Aggressively null out all reference fields: see bug 4006245 */
        target = null;
        /* Speed the release of some of these resources */
        threadLocals = null;
        inheritableThreadLocals = null;
        inheritedAccessControlContext = null;
        blocker = null;
        uncaughtExceptionHandler = null;
    }
```
Thread类继承自Runnable接口，要实现他的run方法，如果目标了target不为null则直接调用target的run方法，此时一般时继承Runnable接口实现多线程；
也可以继承Thread类直接重写run方法实现多线程(本人推荐后种方法)

## 2.8. 中断方法
```java
    public void interrupt() {
        //检查是否中断当前线程
        if (this != Thread.currentThread())
            checkAccess();//检查当前线程是否有修改权限

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();//设置中断状态为true
                b.interrupt(this);
                return;
            }
        }
        interrupt0();//设置中断状态为true
    }

    //测试当前线程是否被中断，该方法会清除线程的中断状态
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);//传入参数true表示要重置中断状态
    }

    //测试调用的线程是否已经被中断，即中断状态是否为true
    public boolean isInterrupted() {
        return isInterrupted(false);//调用下面的nativce方法，false表示不重置中断状态
    }

    //测试某个线程是否已经被中断，参数ClearInterrupted表示是否进行中断状态的重置
    private native boolean isInterrupted(boolean ClearInterrupted);

```
由上面注解可以发现当线程发生中断时他的中断状态被设置为true

注意interrupted为静态方法，此方法判断线程是否发生中断并且会进行中断状态的清除

## 2.9. join()方法
```java

    //设置线程等待的时间为毫秒
    public final synchronized void join(long millis) throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;
        //不合法参数，抛出异常
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {//当线程在活动中时
                /*
                *关键之处：wait()方法暂停的是父线程，因为子线程是在父线程上调用的
                */
                wait(0);//调用Object的wait()方法，此时线程处于休眠状态
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;//计算休眠时间
                if (delay <= 0) {
                    break;
                }
                wait(delay);//使当前线程处于休眠状态
                now = System.currentTimeMillis() - base;
            }
        }
    }

    //设置线程等待的时间为毫秒和纳秒
    public final synchronized void join(long millis, int nanos)
    throws InterruptedException {

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
            millis++;
        }

        join(millis);
    }

    //设置线程等待死亡
    public final void join() throws InterruptedException {
        join(0);
    }
```
join()方法作用就是同步，将并行线程变为串行线程。

如果在线程A中调用线程B的join()方法则在线程B执行结束之后，线程A再继续执行。

join()方法的底层使用的是wait()方法，所以join()方法是同步方法


## 2.10. 一般方法
```java
    //设置该线程的优先级
    public final void setPriority(int newPriority) {
        ThreadGroup g;
        checkAccess();
        //newPriority超出优先级范围则抛出异常
        if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {
            throw new IllegalArgumentException();
        }
        //线程的优先级不能大于线程组的优先级
        if((g = getThreadGroup()) != null) {
            if (newPriority > g.getMaxPriority()) {
                newPriority = g.getMaxPriority();
            }
            setPriority0(priority = newPriority);//调用native方法设置优先级
        }
    }

    //返回线程的优先级
    public final int getPriority() {
        return priority;
    }

    //设置线程的名称
    public final synchronized void setName(String name) {
        checkAccess();
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name;//此时该线程的名字进行设置
        if (threadStatus != 0) {//在线程启动的时候会对操作系统内部的线程名字进行改变
            setNativeName(name);
        }
    }

    //返回线程名称
    public final String getName() {
        return name;
    }

    //返回当前线程属于的线程组
    public final ThreadGroup getThreadGroup() {
        return group;
    }

    //返回线程组中活动线程的估计值，调用的是ThreadGroup的activeCount()方法
    public static int activeCount() {
        return currentThread().getThreadGroup().activeCount();
    }
 
    //将当前线程组及其子组中的活动线程复制到指定的线程组中
    public static int enumerate(Thread tarray[]) {
        return currentThread().getThreadGroup().enumerate(tarray);
    }

```

## 2.11. 其他方法
```java
      
    public static void dumpStack() {
        new Exception("Stack trace").printStackTrace();
    }

    //设置当前线程是否为守护线程
    public final void setDaemon(boolean on) {
        checkAccess();
        if (isAlive()) {
            throw new IllegalThreadStateException();
        }
        daemon = on;
    }

    //测试当前线程是否为守护线程
    public final boolean isDaemon() {
        return daemon;
    }

    //检查当前运行的线程是否有访问权限
    public final void checkAccess() {
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkAccess(this);
        }
    }

    //返回线程的字符串表示形式：线程名称+线程优先级+线程组名称
    public String toString() {
        ThreadGroup group = getThreadGroup();
        if (group != null) {
            return "Thread[" + getName() + "," + getPriority() + "," +
                           group.getName() + "]";
        } else {
            return "Thread[" + getName() + "," + getPriority() + "," +
                            "" + "]";
        }
    }
 
    //返回此线程的类加载器
    @CallerSensitive
    public ClassLoader getContextClassLoader() {
        if (contextClassLoader == null)
            return null;
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            ClassLoader.checkClassLoaderPermission(contextClassLoader,
                                                   Reflection.getCallerClass());
        }
        return contextClassLoader;
    }

    //设置此线程的类加载器
    public void setContextClassLoader(ClassLoader cl) {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            sm.checkPermission(new RuntimePermission("setContextClassLoader"));
        }
        contextClassLoader = cl;
    }

    private native static StackTraceElement[][] dumpThreads(Thread[] threads);
    private native static Thread[] getThreads();

    //返回此线程的ID标识符
    public long getId() {
        return tid;
    }

    //线程的状态枚举类
    public enum State {
        //尚未启动的线程的状态
        NEW,
        //可运行的线程的状态
        RUNNABLE,
        //阻塞线程的状态
        BLOCKED,
        //等待线程的状态        
        WAITING,
        //具有指定等待时间的等待线程的状态
        TIMED_WAITING,
        //终止线程的状态
        TERMINATED;
    }

    //返回此线程的状态
    public State getState() {
        return sun.misc.VM.toThreadState(threadStatus);
    }
}

```
# 3. 理解

## 3.1. Thread的初始化过程


    1.判断线程名称是否为null
    2.获取新线程的父线程parent
    3.获取系统的安全管理器
    4.判断线程组g是否为null，如果为null则将security的ThreadGroup设置为g；如果security为null则将parent的ThreadGroup设置为g
    5.未启动的线程的数量加1
    6.设置新线程的守护状态为parent的守护状态
    7.设置新线程的优先级为parent的优先级
    8.设置目标对象
    9.设置新线程的堆栈大小
    10.设置新线程的ID

线程初始化完成之后，线程还是处于未启动状态，还未加入到ThreadGroup中

## 3.2. 线程的启动过程
Thread通过start()方法启动一个线程，底层调用的是start0()方法，线程存在父与子的关系，一般是在父线程之上启动一个新得线程，在启动线程的时候会改变线程得启动标志状态，然后将此线程添加到他的ThreadGroup中，然后调用start0()方法启动，如果启动失败则通知它的ThreadGroup进行回滚，即恢复到启动线程之前的状态。


<hr>
关于Thread中wait()方法、join()方法、sleep()方法、notify()方法、notifyAll()方法和yield()方法的区别