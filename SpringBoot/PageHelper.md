<!-- TOC -->

- [1. SpringBoot后端开发如何进行分页](#1-springboot%e5%90%8e%e7%ab%af%e5%bc%80%e5%8f%91%e5%a6%82%e4%bd%95%e8%bf%9b%e8%a1%8c%e5%88%86%e9%a1%b5)
- [2. 项目中的具体实例：](#2-%e9%a1%b9%e7%9b%ae%e4%b8%ad%e7%9a%84%e5%85%b7%e4%bd%93%e5%ae%9e%e4%be%8b)

<!-- /TOC -->
在之前学习公司demo的时候使用PageHepler插件进行分页，**但是此插件网上有两种不同的版本**，而且在编码的过程中出现无法进行分页和分页后前端无法进行显示问题，这里进行记录，避免再次踩坑。


PageHelper插件官方文档： https://pagehelper.github.io/docs/howtouse/


# 1. SpringBoot后端开发如何进行分页

* 添加PageHelper依赖

在pom.xml文件中添加依赖
```xml

        <!--引入pageHelper插件-->
       <!--springboot集成的PageHelper-->
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
            <version>1.2.5</version>
        </dependency>

    <!--pagehelper原生版本-->
    <dependency>
        <groupId>com.github.pagehelper</groupId>
        <artifactId>pagehelper</artifactId>
        <version>5.1.10</version>
    </dependency>
        
```
自己在项目中个使用的是PageHelper的原生版本，但是两个版本在配置上有差异


* PageHelper

```java
//PageHelper的继承体系
public class PageHelper extends PageMethod implements Dialect 
```

以下内容来自官方文档

* dialect：默认情况会使用PageHelper方式进行分页，如果想要实现自己的分页逻辑，可以实现dialect接口进行配置



PageInfo的分页方式有很多，此处使用PageHelper的startPage(pageNum, pageSize)方法进行分页，**此方法默认将紧跟在后面的查询语句进行分页**

分页之后将结果集封装为PageInfo，然后传到前端页面

* 关于PageInfo

PageInfo中常用的重要属性
```java
//PageInfo源码
public class PageInfo<T> extends PageSerializable<T> {
    //  当前页码数
    private int pageNum;
    //每页的数量
    private int pageSize;
    // 当前页的数量
    private int size;
    
    private int startRow;
    
    private int endRow;
    //总页数
    private int pages;
    // 前一页
    private int prePage;
    // 后一页
    private int nextPage;
    // 是否是第一页
    private boolean isFirstPage;
    // 是否为最后一页
    private boolean isLastPage;
    // 是否有前一页
    private boolean hasPreviousPage;
    // 是否有下一页
    private boolean hasNextPage;
    
    private int navigatePages;
    private int[] navigatepageNums;
    private int navigateFirstPage;
    private int navigateLastPage;
}
```


# 2. 项目中的具体实例：

* 后端如何进行分页
```java
    /**
     * 默认进入后台主页
     *
     * @return
     */
    @RequestMapping(value = "/index")
    public String admin(Model model, @RequestParam(defaultValue = "1", value = "pageNum") Integer pageNum
            , @RequestParam(defaultValue = "9", value = "pageSize") Integer pageSize) {

        PageHelper.startPage(pageNum, pageSize);
        try {
            List<Article> articleList = articleService.selectAll();
            PageInfo<Article> pageInfo = new PageInfo<>(articleList);

            model.addAttribute("pageInfo", pageInfo);
        } catch (Exception e) {
            System.out.println(e);
        }

        log.info("分页成功");
        return "admin/index";
    }

```

* 前端如何将分页信息传回到后端
  
在前端传回页码信息到后端，后端接受后进行分页
```html
<ul class="pagination" style="float:right">
                <li><a th:href="@{/admin/index}">首页</a></li>
                <!--
                    此处使用三元表达式
                    如果存在上一页则将上一页的页码传回到后端，否则传回页码1
                    如果存在下一页则将下一页的页码传回到后端，否则传回总页数
                -->
                <li><a th:href="@{/admin/index/(pageNum=${pageInfo.hasPreviousPage}?${pageInfo.prePage}:1)}">上一页</a>
                </li>
                <li>
                    <a th:href="@{/admin/index/(pageNum=${pageInfo.hasNextPage}?${pageInfo.nextPage}:${pageInfo.pages})}">下一页</a>
                </li>
                <li><a th:href="@{/admin/index/getAllPerson(pageNum=${pageInfo.pages})}">尾页</a></li>
</ul>      
```



未完。。。