# 1. cookie在项目中的使用
会话是由一组请求与响应组成

在请求与响应之间需要传递验证身份的数据，也就是会话的跟踪

但是http协议是一种无状态的协议，多个会话之间无数据的传递，运行于tcp协议之上，在服务层，因此需要cookie技术进行会话的跟踪

cookie是由服务器产生的，保存在客户端的一种信息载体。
此cookie保存了用户的状态信息，cookie可以保存在客户端浏览器的缓存之后,也可以写入到客户端的硬盘之中，在服务端进行cookie的设置

在用户第一次提交之后，服务器会生产cookie，并将其封装到响应头中然后发送到客户端，当客户端再次发送同类请求时，会携带客户端的cookie信息发送到客户端，然后客户端进行会话的跟踪，也就是会话身份的验证。

cookie由键值对组成，均为字符串。

实例：
```java
/**
* 登陆验证模块，验证成功后跳转至后台主页,否则跳转到登陆界面
*
* @param userMessage
* @return
*/
@ResponseBody
@RequestMapping(value = "/dologin", method = RequestMethod.POST)
public ErrorCode doLogin(@RequestBody String userMessage, HttpServletResponse response, HttpServletRequest request) {

    log.info("接受用户输入的用户名和密码：" + userMessage);

    //将string转换为静态的JSONObject
    JSONObject object = JSONObject.fromObject(userMessage);
    //获取与键关联的值
    String userName = object.getString("userName");
    String password = object.getString("password");

    User user = new User(userName, password);

    //如果可以获取到用户名和密码则成功否则失败
    if (userService.getUser(user.getUserName(), user.getUserPassword())) {
        log.info("用户验证成功");
        if (user.getUserName().equals("yq")) {
            cookie = new Cookie(user.getUserName(), user.getUserPassword());
        } else {
            //创建cookie
            cookie = new Cookie("General", "user");
        }
        //在所有路径上都绑定cookie
        cookie.setPath("/");

        //设置cookie的有效时间
        //该值大与0表示将cookie存放到客户端的硬盘
        //该值小于0与不设置相同表示将cookie存放到客户端浏览器的缓存中
        //该值等于0表示cookie一生成马上失效
        cookie.setMaxAge(-1);//浏览器关闭时清除cookie
        //加入cookie
        response.addCookie(cookie);

        return ErrorCodes.CODE_000;
    } else {
        log.info("用户验证失败");
        return ErrorCodes.CODE_011;
    }
}
```

在用户第一次进行登录的时候服务端创建cookie并实例化，然后设置cookie的相关属性，最后将通过HttpServletResponse将cookie数据传递给客户端，存储在客户端的缓存中(或者直接写入到硬盘中)
# 2. cookie中文乱码问题
如果用户使用的是中文，则cookie存储的是中文,此时会出现编码问题，根据网上了解是因为传输过程中使用的编码方式不一样导致的，因此需要在存储cookie和取出cookie的时候指定编码方式.
 
存储时指定编码方式：cookie = new Cookie(URLEncoder.encode(user.getUserName(), "UTF-8"), user.getUserPassword());
取出时指定编码方式：URLDecoder.decode(cookie.getName(), "UTF-8")



