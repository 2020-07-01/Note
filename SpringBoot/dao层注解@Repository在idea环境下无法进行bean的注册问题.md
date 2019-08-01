
问题：**在idea环境下，注解@Repository注册dao曾bean的时候显示无法注入**
```java
@Repository
public interface BloggerDao {

	//查询博主信息
	Blogger getBloggerData();

	//通过用户名查询博主信息
	Blogger getBloggerDataByName(String name);

}


@Service
public class BloggerServiceImpl implements BloggerService {

	@Autowired
	BloggerDao bloggerDao;

```
问题:在dao层接口上配置注解@Repository用来注册bean，但是在service层显示依赖错误，无法注册bloggerDao

但是在eclipse下可以进行注册，**最后在注解@Autowired中添加属性(required = false)错误集解决**，并且可以成功访问数据库，如下：

```java

@Service
public class BloggerServiceImpl implements BloggerService {

	@Autowired(required = false)
	BloggerDao bloggerDao;

```

在上面显示无法进行bean的注册，但是在spring与mybatis整合的配置文件中有如下配置：

```xml
 <!-- 配置扫描dao接口，实现动态注入到容器 -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        
        <!--此配置的作用时扫描dao层的包进行bean的注册-->
        <property name="basePackage" value="blog.dao"/>

        <property name="SqlSessionFactoryBeanName"
                  value="sqlSessionFactory"/>

        <!-- 指定标注才扫描成Mapper 此时需要在dao接口上加注解@Repository -->
        <property name="annotationClass"
                  value="org.springframework.stereotype.Repository"/>
    </bean>

```


* **关于@Autowired：**
将类在Spring中进行注册之后，可以使用此注解根据类型进行获取注册的bean
在默认的情况下如果注册失败则寻找也会失败，此时会报错，但是可以通过此注解的配置属性(required = false)来解决他








