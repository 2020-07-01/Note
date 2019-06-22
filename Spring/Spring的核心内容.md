<!-- TOC -->

- [1. 引言](#1-引言)
- [2. Spring中加载bean的过程](#2-spring中加载bean的过程)
    - [2.1. 如何加载配置文件](#21-如何加载配置文件)
    - [2.2. 读取XML配置文件和实例化bean](#22-读取xml配置文件和实例化bean)
- [3. AOP思想](#3-aop思想)
    - [AOP术语](#aop术语)

<!-- /TOC -->

# 1. 引言

Spring的两个核心思想是IOC和AOP，IOC底层使用了反射技术，而AOP底层使用了动态代理技术

# 2. Spring中加载bean的过程

1. 加载配置文件xml  
2. 读取xml配置文件中的配置信息
3. 根据配置信息进行类的实例化

## 2.1. 如何加载配置文件

BeanFactory作为Spring容器的基础，用于存放所有已经加载的bean
加载配置文件有两种方式:BeanFactory和ApplicationContext，ApplicationContext是对BeanFactory的扩展，一般使用后者。

```java
BeanFactory bf  = new XmlBeanFactory(new ClassPathResource("beanFactoryTest.xml"));

ApplicationContext bf = new ClassPathXmlApplicationContext("beanFactoryTest.xml");
```

加载配置文件的思路是传入xml文件的路径，在底层使用InputStream字节流进行文件的读取

## 2.2. 读取XML配置文件和实例化bean

1. 获取xml文件的验证模式
xml文件的验证模式有两种DTD和XSD,通过判断DOCTYPE来确定使用那种验证方式

2. 加载xml文件得到对应的document
 document代表整个xml文档，通过document对文档进行操作，根据document注册bean信息
  
3. 在对bean进行注册之前首先要对xml文件中的标签进行解析

4. 实例化bean
一般情况下Spring通过bean的class属性来指定实现类来实例化bean ,但是在bean比较复杂的情况下，需要在<bean>标签中提供大量的配置信息，因此定义统一接口**FactoryBean**来实例化bean

* Bean为单例模式创建
bean在Spring容器中之创建一次，后续如果需要获取bean直接从单例缓存中获取，因此获取到的是同一个bean。
spring中创建的单例bean会被放在hashmap中，而非单例的bean直接在spring容器中直接进行获取

# 3. AOP思想

Aop指的是面向切面编程,在实际编写代码的过程中存在在特定场景下重复使用的功能，比如说数据库事务，因此将重复使用的代码单独封装为功能模块，在需要的时候直接调用。

1. 在AOP中存在以下术语：
一是连接点(join point)指的是有可能增强的那些方法
二是切入点(point cut)指的是正在被增强的那个方法
三是通知指的是增强代码，具体是在目标类的方法执行之前或者之后要执行的代码(方法)
四是织入指的是将增强方法引入到目标对象创建对象代理的过程
五是切面(aspect)指的是切入点和通知方法的结合，即此时所处的环境
六是目标类指的是要被代理的类

2. 代理对象的思想就是代理目标对象的所有方法，并将切面类的通知方法增加在目标对象的指定方法之上



动态代理：基于接口的动态代理和基于子类的动态代理


aop底层使用代理的方式实现
oop面向对象编程


接口+实现类：Spring采用jdk的动态代理
实现类：Spring采用cglib字节玛增强




