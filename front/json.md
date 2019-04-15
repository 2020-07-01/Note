
JSON:JavaScript的对象表示法
JSON是交换和存储文本信息的语法，类似于XML
JSON比XML更小更快更易解析
JSON是轻量级的文本数据交换格式
JOSN文本格式在语法上与创建JavaScript对象的代码相同
由于这种相似性，无需解析器，JavaScript程序能够使用内建的eval()函数，用JSON数据来生成原生的JavaScript对象。


# JavaScript对象语法与JSON语法的比较
　　JavaScript语法格式为键值对：name:"value"
```js
//这是一个对象
var person = {
    firstName: "Jhon",
    lastNmae："Deo",
    age:"50",
    eyeColor:"blue"
    //定义对象方法
    fullName: function() {
        return this.firstName+ " "+this.lastName;
    } 
};
```

# JSON语法规则
　　因为json使用Javascript语法，无需额外的软件就可以处理JavaScript中的JSON
JSON语法是JavaScript语法的子集
JSON数据的书写格式为：名称/值对
```json
"name" : "菜鸟教程"
//等价于JavaScript语句
//name="菜鸟教程"
```

## JSON数字
JSON数字可以是整型或者浮点型
```json
{"age":30}
```
## JSON对象
　　JSON对象保存在大括号{}中，对象可以包含多个key/value(键值对)
　　key必须是字符串，value是合法的JSON数据类型(字符串、数字、对象、数组、布尔值、null)
```json
{
    "name":"菜鸟教程",
    "URL":"www.runoob.com"
}
```
## JSON数组
JSON数组在中括号中书写，数组可以包含多个对象
```json
{
    "name":"网站",
    "sites":[
        {
            "name":"菜鸟教程","url":"www.runoob.com"
        }
        {
            "name":"goole",
            "url":"www.goole.com"
        }
        {
            "name":"微博",
            "url":"www.weibo.com"
        }
    ]

}
```
## JSON布尔值
JSON布尔值可以为true或者false
```json
{
    "flag":true
}
```
## JSON null
```json
{
    "runoob":null
}
```

# JSON.parse()：字符串 -> JavaScript
　　JSON通常与服务端进行交换数据，在接受服务端数据时一般为字符串，可以使用JSON.parse()将数据转换为JavaScript对象
语法:
JSON.parse(text[,reviver])
text:必需，一个有效的JSON字符串
reviver:可选，一个转换函数，将为对象的每个成员调用此函数

# JSON.stringify():JavaScript -> 字符串
　　在向服务器发送数据时一般为字符串，可以使用JSON.stringify将JavaScript对象转换为字符串。

语法：
JSON.stringify(value,replacer,space)
value:必需，需要转换的JavaScript值(通常为数组或者对象)

space：可选，文本添加缩进、空格和换行符
　　如果space是一个数字，则返回值文本在每个级别缩进指定数目的空格，如果space大于10，则文本缩进10个空格

# JavaScript.eval()：JSON文本->JavaScript对象

　　JSON最常见的用法之一，是从web服务器上读取JSON数据(作为文本或者作为HttpRequest)，将JSON数据转换为JavaScript对象，然后在网页上使用该数据。
　　由于JSON语法是JavaScript语法的子集，JavaScript函数的eval()可用于将JSON文本转换为JavaScript对象。




