
# 1. Java语言基础知识
1. Java语言有哪些优点
2. Java程序的初始化顺序
    静态变量优于非静态变量、父类优于子类
    静态块在类被加载时就会调用
3. 变量的三种类型
4. 构造函数的特点
5. 接口的特点：接口的方法都是抽象的、接口中的成员变量都是public static final修饰，必须赋初值，所有的成员方法都是public abstract
6. 抽象类的特点：成员变量默认为default，不允许创建对象
7. java中的标识接口：
8. java中clone()方法的作用及其浅复制与深复制
9. 反射机制中获取class类的方式
10. package包的作用:提供多层命名空间，对类按功能进行分类
11. 面向对象的三大特征
12. 四种修饰符的权限：默认的是同包权限，Protected是在父类与子类之间进行修饰
13. 多态的两种方式
14. 如何获取父类的类名：getClass().getSupperClass().getName()
15. this与super的区别
16. break、continue、return的区别
17. final、finally、finalize的区别：finally中的代码在return之前执行
18. switch的参数类型：byte,char,int,String(java7),enum(java5)
19. volatile的作用
20. default关键字的作用：case执行失败时进行调用
21. instanceof的作用:某个对象是否为某个类的实例，返回boolean值
22. java中的8种基本数据类型，注意双精度与单精度的写法
23. 不可变类有哪些：String
24. 值传递与引用传递的区别
25. Java中强制类型转换注意事项int
26. Math类中的ceil,floor和round的区别
27. String、StringBuffer、StringBuider的区别：StringBuffer线程安全，执行速度慢
28. length属性和length()方法的区别
29. 检查型异常与非检查型异常的区别：
    * 检查型异常需要使用try、catch、finally在编译期间进行处理，否则会出现编译器报错，继承与Exception类，用来处理程序错误，请求的资源不可用。

30. ==与equals与hashCode有什么区别
    * 对于基本数值类型使用==来比较两个变量两个变量所存储的数值是否相等
    * 对于引用类型变量:比如String s = new String()此时涉及到两块内存，变量s占用一块内存，而new String存储在另一块内存中，此时变量s所存储的数值就是对象占用的那块内存的首地址；如果要比较两个变量是否指向同一个对象，则可以使用==来判断，如果要比较对象的内容是否相等，则==无法实现。
    * equals在没有重写之前，与==一样比较的是引用；在String中，equals比较的是对象的内容是否相同
    * hashCode()返回对象在内存中地址转换成的一个int值,哈希码值
31. 字符串创建与存储的机制原理




  