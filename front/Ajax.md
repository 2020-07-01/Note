# 1. Ajax优点:
　　是不在加载整个网页的情况下，可以与服务器交换数据并更新部分网页内容。而实现的原理是：网页的DOM对象可以精确的对网页中的部分内容进行操作、XML作为单纯的数据存储载体使得客户端与服务器之间交换的只是网页内容的数据而没有网页样式等附属信息。

# 2. Ajax的技术组成:
　　JavaScript、XML、XMLHTTPRequest对象、DOM和CSS

![Alt](https://images2015.cnblogs.com/blog/1018541/201612/1018541-20161202170336021-461606131.png)

# 3. 原理：
　　Ajax的原理：通过XMLHTTPRequest对象来向服务器发送异步请求，从服务器获得数据，然后通过JavaScript来操作DOM使得更新页面。

# 4. XMLHTTPRequest：
　　XMLHTTPRequest是Ajax的核心技术，是一种支持异步请求的技术，也就是可以及时向服务器提出请求和处理响应，而不阻塞用户，达到无刷新的效果。他是一个具有应用程序接口的JavaScript对象。


# 5. 属性:
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

# 6. 方法：
1. open()：用于设置进行异步请求目标的URL、请求方法以及其他参数信息
    method：用于指定请求的类型
    URL：用于指定请求的地址，可以使用相对地址或者绝对地址
    async：用于指定请求的方式，异步为true，同步为false
    password：用于指定请求的密码

2. send(string)：用于向服务器发送请求
    string：仅限于post请求

3. setRequestHeader():用于为请求的http头设置值

4. abort()：停止或者放弃当前异步请求


# 7. XMLHTTPRequest的创建：
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


实现Ajax的基本步骤：
    (1)创建XMLHTTPRequest对象，也就是创建一个异步调用对象
    (2)创建一个新的HTTP请求，并指定该HTTP请求的方法、URL、以及验证信息
    (3)设置HTTP请求状态变换的函数
    (4)发送HTTP请求
    (5)获取异步调用返回的数据
    (6)使用JavaScript和DOM实现局部刷新

实例:
```js
 /定义一个变量用于存放XMLHttpRequest对象
var xmlHttpRequest;  
//定义一个用于创建XMLHttpRequest对象的函数
function createXMLHttpRequest()
{ 
    if (window.xmlHttpRequest) {
        //非IE浏览器的创建方式
        xmlHttpRequest = new XMLHttpRequest();
    } else if(window.ActiveXObject){
        try {
        //IE浏览器的创建方式
        xmlHttpRequest = new ActiveXObject("Microsoft.XMLHTTP");
        } catch (e) {
            //IE浏览器的创建方式
            xmlHttpRequest = new ActiveXObject("Msxml2.XMLHTTP");
        }
    }
}
//响应HTTP请求状态变化的函数
function httpStateChange()
{
     //判断异步调用是否完成
    if(xmlHttpRequest.readyState == 4)
   {
           //判断异步调用是否成功,如果成功开始局部更新数据
           if(xmlHttpRequest.status == 200||xmlHttpRequest.status == 0)
           {
                  //查找节点更新数据
                  document.getElementById("demo").innerHTML =  xmlHttpRequest .responseText;
           }
           else
            {
                //如果异步调用未成功,弹出警告框,并显示出错信息
                alert("异步调用出错/n返回的HTTP状态码为:"+xmlHttpRequest.status + "/n返回的HTTP状态信息为:" + xmlHttpRequest.statusText);
            }
    }
}
//异步调用服务器段数据
function getData(name,value)
{                   
    //创建XMLHttpRequest对象
    createXMLHttpRequest();
    if(xmlHttpRequest!=null)
    {
        //创建HTTP请求
         xmlHttpRequest.open("get","ajax.text",true)
        //设置HTTP请求状态变化的函数
        xmlHttpRequest.onreadystatechange = httpStateChange;
        //发送请求
        xmlHttpRequest.send(null);
   }
}
```


 

