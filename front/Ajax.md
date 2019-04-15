# Ajax优点:
　　是不在加载整个网页的情况下，可以与服务器交换数据并更新部分网页内容。而实现的原理是：网页的DOM对象可以精确的对网页中的部分内容进行操作、XML作为单纯的数据存储载体使得客户端与服务器之间交换的只是网页内容的数据而没有网页样式等附属信息。

# Ajax的技术组成:
　　JavaScript、XML、XMLHTTPRequest对象、DOM和CSS

![Alt](https://images2015.cnblogs.com/blog/1018541/201612/1018541-20161202170336021-461606131.png)

# 原理：
　　Ajax的原理：通过XMLHTTPRequest对象来向服务器发送异步请求，从服务器获得数据，然后通过JavaScript来操作DOM使得更新页面。

# XMLHTTPRequest：
　　XMLHTTPRequest是Ajax的核心技术，是一种支持异步请求的技术，也就是可以及时向服务器提出请求和处理响应，而不阻塞用户，达到无刷新的效果。他是一个具有应用程序接口的JavaScript对象。


# 属性:
1. onreadystatechange 每次状态改变时所触发的事件处理器属性
2. reponseText：从服务器进程返回数据的字符串形式
reponseXML
3. status：返回服务器的HTTP状态码属性
4. readyState：对象状态值
    * 0 (未初始化)表示对象已经建立，但是尚未初始化
    * 1 (初始化)表示对象已经建立，但是尚未调用send方法
    * 2 (发送数据)表示send方法已经调用，但是当前状态以及http头未知
    * 3 (数据传送中)
    * 4 (完成)表示数据接收完毕，此时可以通过reponseXML和reponseText获取完整的数据

# 方法：
1. open()：用于设置进行异步请求目标的URL、请求方法以及其他参数信息
    method：用于指定请求的类型
    URL：用于指定请求的地址，可以使用相对地址或者绝对地址
    async：用于指定请求的方式，异步为true，同步为false
    password：用于指定请求的密码

2. send(string)：用于向服务器发送请求
    string：仅限于post请求

3. setRequestHeader():用于为请求的http头设置值

4. abort()：停止或者放弃当前异步请求


# XMLHTTPRequest的创建：
　　根据各浏览器的不同，创建XMLHTTPRequest有不同的方式：
```js

function CreateXmlHttp() {
    //非IE浏览器创建XmlHttpRequest对象
    if (window.XmlHttpRequest) {
        xmlhttp = new XmlHttpRequest();
    }

    //IE浏览器创建XmlHttpRequest对象
    if (window.ActiveXObject) {
        try {
            xmlhttp = new ActiveXObject("Microsoft.XMLHTTP");
        }
        catch (e) {
            try {
                xmlhttp = new ActiveXObject("msxml2.XMLHTTP");
            }
            catch (ex) { }
        }
    }
}
```




