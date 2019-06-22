Spring实例bean的过程：
1. 读取配置文件xml  ,       xml文件的读取是通过ClassPathResource进行读取，在底层使用的是inputStream
2.根据配置文件的配置找到对应的类，并进行实例化
3.调用实例化后的实例


如何进行实例：

XmlBeanFactory对DefaultListableBeanFactory进行了扩展，主要用于从XML文档中读取BeanDefinition，对于注册及获取bean都是从父类继承的方法进行实现.

Spring的重要功能就是xml配置文件的读取。

对xml文件的读取：

ClassPathResource读取xml文件的思路就是利用构造函数传入xml文件的路径，然后底层使用InputStream字节流进行文件的读取

在底层使用Resource接口对资源进行读取，Resource对Spring底层的所有资源提供了访问的接口
而Resource继承了InputStreamSource，提供了一个InputStream输入字节流

获取xml文件的验证模式
加载xml文件，得到对应的document
根据document注册bean信息


而Document是整个xml文档，通过document可对xml文档进行操作

XML文档的两种验证模式：DTD，XSD，一般在Spring中常用的是XSD
如何判度使用那种验证方式：
在DTD验证方式中使用DOCTYPE关键词，通过判断是否包含DOCTYPE来确定验证方式


获取Document是通过DocumentLoader接口来实现



在加载xml文件并获取document后对bean进行注册


在对bean进行注册之前需要先对标签进行解析


一般情况下Spring通过bean的class属性来指定实现类来实例化bean ,但是在bean比较复杂的情况下，需要在<bean>标签中提供大量的配置信息，因此定义统一接口FactoryBean来实例化bean

bean在Spring容器中之创建一次，后续如果需要获取bean直接从单例缓存中获取，因此获取到的是同一个bean。
spring中创建的单例bean会被放在hashmap中，而非单例的bean直接在spring容器中直接进行获取











```java

public interface InputStreamSource {
    InputStream getInputStream() throws IOException;
}

public interface Resource extends InputStreamSource {
    boolean exists();

    boolean isReadable();

    boolean isOpen();

    URL getURL() throws IOException;

    URI getURI() throws IOException;

    File getFile() throws IOException;

    long contentLength() throws IOException;

    long lastModified() throws IOException;

    Resource createRelative(String var1) throws IOException;

    String getFilename();

    String getDescription();
}
```



