<!-- TOC -->

- [1. 引言](#1-%E5%BC%95%E8%A8%80)
- [2. Spring中加载bean的过程](#2-Spring%E4%B8%AD%E5%8A%A0%E8%BD%BDbean%E7%9A%84%E8%BF%87%E7%A8%8B)
- [3. AOP思想](#3-AOP%E6%80%9D%E6%83%B3)

<!-- /TOC -->

# 1. 引言

Spring的两个核心思想是IOC和AOP，IOC底层使用了反射技术，而AOP底层使用了动态代理技术
回顾：
* **反射技术**：在运行过程中，获取一个类的对象从而获取一个类的属性和方法的技术
    动态获取一个类的对象的方式有三种：
    一是通过Class的forName()方法
    二是通过类的class属性
    三是通过类的getClass()方法进行反射
* **动态代理技术**：动态代理就是在目标类的方法执行之前，先执行代理类的方法，目标类和代理类是被整合到一个切面类中执行
    动态代理有两种方式：
    一种是JDK动态代理：通过接口和实现类实现
    另一种是cglib动态代理：通过实现类实现

# 2. Spring中加载bean的过程
Spring加载xml文件并创建bean主要经历了三个过程:首先是将xml文件加载到内存之中，然后对xml文件进行解析，最后通过解析的内容创建类的对象，也就是bean
 
1. 首先加载xml配置文件
BeanFactory作为Spring容器的基础，用于存放所有已经加载的bean

加载配置文件的两个具体实现类:BeanFactory和ApplicationContext，ApplicationContext是对BeanFactory的扩展，一般使用后者。ApplicationContext包含了BeanFactory接口的所用功能，并对其功能进行了扩展，二者都可以形成IOC容器，对其中的Bean进行管理。

都是通过传入xml文件的路径，底层使用字节流的方式对xml文件进行读取到内存中。

```java
BeanFactory bf  = new XmlBeanFactory(new ClassPathResource("beanFactoryTest.xml"));

ApplicationContext bf = new ClassPathXmlApplicationContext("beanFactoryTest.xml");
```

加载配置文件的思路是传入xml文件的路径，在底层使用InputStream字节流进行文件的读取

2. 将xml文件加载到内存之后进行xml的解析

* 要对xml文件进行解析，首先要获取xml文件的验证模式
xml文件的验证模式有两种DTD和XSD,通过判断DOCTYPE来确定使用那种验证方式
DTD验证模式就是使用一些元素来定义xml文档的结构，要在xml文件中使用**DOCTYPE**进行引入这个文档类型定义文件

* 加载xml文件后得到对应的document
document代表整个xml文档，通过document对文档进行操作，根据document注册bean信息

3. 创建bean
在对xml文件进行解析和获取xml文件对应的document后来创建bean
 在Spring中创建bean使用接口**FactoryBean**来获取对象

4. 在Spring中Bean默认为单例模式创建，只创建一次，后续如果需要获取bean直接从单例缓存中获取，因此获取到的是同一个bean。
spring中创建的单例bean会被放在hashmap中，也就是单例缓存，而非单例的bean直接在spring容器中直接进行获取

# 3. AOP思想

Aop指的是面向切面编程,在实际编写代码的过程中存在在特定场景下重复使用的功能，比如说数据库事务，因此将重复使用的代码单独封装为功能模块，在需要的时候直接调用。

1. 在AOP中存在以下术语：
一是连接点(join point)指的是有可能增强的那些方法，指的是目标类中的所有方法
二是切入点(point cut)指的是正在被增强的那个目标类中的方法，
三是通知：指的是增强代码，具体是在目标类的方法执行之前或者之后要执行的代码(方法)
四是织入：指的是将增强方法引入到目标对象，创建代理对象使之结合的过程
五是切面(aspect)指的是切入点和通知方法的结合，即此时所处的环境
六是目标类指的是要被代理的类

2. 代理对象的思想就是代理目标对象的所有方法，并将切面类的通知方法增加在目标对象的指定方法之上

3. 在Spring中使用AOP框架-AspectJ进行aop的编程，有两种配置方式，一般使用注解方式进行开发
4. 关于动态代理：分为基于接口的动态代理和基于子类的动态代理
    jdk动态代理：基于接口的代理


    cglib动态代理：基于子类的代理，在运行时创建目标类的子类，从而对目标类进行增强
 
     