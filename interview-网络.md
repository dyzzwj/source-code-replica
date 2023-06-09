

 

## 七层网络模型（物链网输会示用）



应用层

表示层

会话层

传输层

网络

数据链路层

物理层







## 四层网络模型

应用层：http、ftp、ssh、smtp

传输层：tcp、udp

网络层：ip

数据链路层：



## tcp、udp

1. 传输控制协议-TCP：提供面向连接的，可靠的数据传输服务。
2. 用户数据协议-UDP：提供无连接的，尽最大努力的数据传输服务（不保证数据传输的可靠性）







## 三次握手：

- 客户端给服务端发送 SYN=1 seq=k 进入SYN_SENT状态

- 服务端收到后，发送SYN=1 ACK=1 acknowledge number=k+1 seq = j 进入SYN_RECV状态

- 客户端收到后，发送ACK= 1 acknowledge number= j+1 进入ESTABLISHED状态，服务器检查ACK为1和acknowledge number为序列号+1之后，也进入ESTABLISHED状态；



第三次握手中，如果客户端的ACK未送达服务器，会怎样？

`Server端：由于Server没有收到ACK确认，因此会每隔 3秒 重发之前的SYN+ACK（默认重发五次，之后自动关闭连接进入CLOSED状态），Client收到后会重新传ACK给Server`



三次握手过程中可以携带数据吗？

**第三次握手的时候，是可以携带数据的**。但是，**第一次、第二次握手绝对不可以携带数据**





三次握手的本质是确认通信双方收发数据的能力

首先，我让信使运输一份信件给对方，**对方收到了，那么他就知道了我的发件能力和他的收件能力是可以的**。

于是他给我回信，**我若收到了，我便知我的发件能力和他的收件能力是可以的，并且他的发件能力和我的收件能力是可以**。

此时他还不知道他的发件能力和我的收件能力到底可不可以，于是我最后回馈一次，**他若收到了，他便清楚了他的发件能力和我的收件能力是可以的**。

四次挥手的目的是关闭一个连接



## 四次挥手

第一次挥手：Client发送FIN = 1,seq = k给server,进入`FIN_WAIT_1`状态；

第二次挥手：Server收到FIN之后，发送ACK=1，acknowledge number=k+1；进入`CLOSE_WAIT`状态。此时客户端已经没有要发送的数据了，但仍可以接受服务器发来的数据。客户端收到服务端的确认后，进入`FIN_WAIT_2`（终止等待 2）状态，等待服务端发出的连接释放报文段

第三次挥手：Server发送FIN=1，seq = j给Client；进入`LAST_ACK`状态；

第四次挥手：Client收到服务器的FIN后，进入TIME_WAIT状态；发送ACK=1，acknowledge number= j + 1给服务器；服务器收到后，确认acknowledge number后，变为`CLOSED`状态，不再向客户端发送数据。客户端等待2*MSL（**一个报文的来回时间**）时间后，也进入`CLOSED`状态。完成四次挥手。







## 从输入 URL 到页面展示到底发生了什么

1、浏览器查找域名的ip地址；

浏览器DNS缓存 -> 操作系统DNS缓存 -> hosts文件 -> 本地DNS服务器 ->根域DNS服务器 

2、浏览器获得域名对应的IP地址以后，浏览器会以一个随机端口（1024<端口<65535）向服务器的WEB程序80端口发起TCP的连接请求，tcp连接的三次握手；

3、TCP/IP连接建立起来后，浏览器向服务器发送HTTP请求；

4、服务器接收到这个请求，根据http请求协议的格式解析请求报文（请求首行、请求头、空行、请求体），并根据路径参数映射到特定的请求处理器进行处理，并将处理结果及相应的视图返回给浏览器；

5、浏览器根据http响应协议的格式解析并渲染视图，若遇到对js文件、css文件及图片等静态资源的引用，则重复上述步骤并向服务器请求这些资源；

6、HTML解析成DOM树，解析css，执行js代码

7、keep-alive **HTTP1.1** 默认开启keep-alive机制（即在请求头）

8、断开连接 四次挥手



## http

无状态。默认端口号80 https端口443

### http请求协议格式

请求首行（请求类型、请求资源、协议版本）、请求头、空行、请求体

```
 POST / HTTP/1.1
 Host: www.example.com
 User-Agent: Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.7.6)
 Gecko/20050225 Firefox/1.0.1
 Content-Type: application/x-www-form-urlencoded
 Content-Length: 40
 Connection: Keep-Alive

 sex=man&name=Professional  
```



### http响应协议格式

响应首行（协议版本、状态码、状态消息）、响应头、空行、响应体

```
HTTP/1.1 200 OK

Server:Apache Tomcat/5.0.12
Date:Mon,6Oct2003 13:23:42 GMT
Content-Length:112

<html>
```









## get与post的区别

①点击刷新按钮，post会被重复提交

②get编码类型是application/x-www-form-urlencoded，post编码类型是 application/x-www-form-urlencoded、multipart/form-data、application/json、text/html

③get数据长度有限制，post数据长度无限制

④get安全性较差

⑤get只允许 ASCII字符，post没有限制









## Keep-alive

http默认（在非keep-alive模式时）是无状态的，每个请求/应答客户和服务器都要新建一个连接，完成之后立即断开连接（HTTP 协议为无连接的协议）

当使用keep-alive模式，Keep-Alive 功能使客户端到服务器端的连接持续有效，当出现对服务器的后继请求时，**Keep-Alive 机制能保持当前的TCP连接，避免了建立或者重新建立连接**

在 HTTP 1.0 中默认是关闭的，如果浏览器要开启 Keep-Alive，它必须在请求头中添加：`Connection: Keep-Alive`

如果要关闭 Keep-Alive，需要在 HTTP 请求的包头里添加：`Connection:close`

HTTP 长连接不可能一直保持，例如 `Keep-Alive: timeout=5, max=100`，表示这个TCP通道可以保持5秒，max=100，表示这个长连接最多接收100次请求就断开







## 对称加密和非对称加密

- 对称加密的加密和解密使用的同一个密钥；非对称加密，一般使用公钥进行加密，使用私钥进行解密。
- 对称加密速度快，效率高，非对称加密速度慢，效率低



## https

 HTTPS = HTTP + SSL / TLS。

对称加密+非对称加密



![https原理](\images\https原理.png)





1.客户端请求 HTTPS 网址，然后连接到 server 的 443 端口 (HTTPS 默认端口，类似于 HTTP 的80端口)。

2.采用 HTTPS 协议的服务器必须要有一套数字 CA (Certification Authority)证书，证书是需要申请的，并由专门的数字证书认证机构(CA)通过非常严格的审核之后颁发的电子证书 (当然了是要钱的，安全级别越高价格越贵)。颁发证书的同时会产生一个私钥和公钥。私钥由服务端自己保存，不可泄漏。公钥则是附带在证书的信息中，可以公开的。证书本身也附带一个证书电子签名，这个签名用来验证证书的完整性和真实性，可以防止证书被篡改。

3.服务器响应客户端请求，将证书传递给客户端，证书包含公钥和大量其他信息，比如证书颁发机构信息，公司信息和证书有效期等。Chrome 浏览器点击地址栏的锁标志再点击证书就可以看到证书详细信息。

![CA证书](images\CA证书.png)

4.客户端解析证书并对其进行验证。如果证书不是可信机构颁布，或者证书中的域名与实际域名不一致，或者证书已经过期，就会向访问者显示一个警告，由其选择是否还要继续通信。就像下面这样：

![不合法的CA证书](images\不合法的CA证书.png)





如果证书没有问题，客户端就会从服务器证书中取出服务器的公钥A。然后客户端还会生成一个随机码 KEY，并使用公钥A将其加密。

5.客户端把加密后的随机码 KEY 发送给服务器，作为后面对称加密的密钥。

6.服务器在收到随机码 KEY 之后会使用私钥B将其解密。经过以上这些步骤，客户端和服务器终于建立了安全连接，完美解决了对称加密的密钥泄露问题，接下来就可以用对称加密愉快地进行通信了。

7.服务器使用密钥 (随机码 KEY)对数据进行对称加密并发送给客户端，客户端使用相同的密钥 (随机码 KEY)解密数据。

8.双方使用对称加密愉快地传输所有数据。









## 防止表单重复提交



1、通过JavaScript屏蔽提交按钮（不推荐）

2、给数据库增加唯一键约束（简单粗暴）

3、利用Session防止表单重复提交（推荐）

服务器返回表单页面时，会先生成一个subToken保存于session，并把该subToen传给表单页面。当表单提交时会带上subToken，服务器拦截器Interceptor会拦截该请求，拦截器判断session保存的subToken和表单提交subToken是否一致。若不一致或session的subToken为空或表单未携带subToken则不通过。

首次提交表单时session的subToken与表单携带的subToken一致走正常流程，然后拦截器内会删除session保存的subToken。当再次提交表单时由于session的subToken为空则不通过。从而实现了防止表单重复提交。

4、使用AOP自定义切入实现 （业务参数存redis自动过期去重) 





## 处理重复请求

1、利用唯一请求编号去重



2、**业务参数去重**

**key:  用户ID:接口名:请求参数** + 自动过期





3、 **计算请求参数的摘要作为参数标识**

**用户ID:接口名:请求参数摘要(md5) **+ 自动过期

 

4、**剔除部分时间因子**

问题：某些请求用户短时间内重复的点击了（例如1000毫秒发送了三次请求），但绕过了上面的去重判断（不同的KEY值）。

原因是这些请求参数的字段里面，**是带时间字段的**，这个字段标记用户请求的时间，服务端可以借此丢弃掉一些老的请求（例如5秒前）。如下面的例子，请求的其他参数是一样的，除了请求时间相差了一秒：

```java
    //两个请求一样，但是请求时间差一秒
    String req = "{\n" +
            "\"requestTime\" :\"20190101120001\",\n" +
            "\"requestValue\" :\"1000\",\n" +
            "\"requestKey\" :\"key\"\n" +
            "}";

    String req2 = "{\n" +
            "\"requestTime\" :\"20190101120002\",\n" +
            "\"requestValue\" :\"1000\",\n" +
            "\"requestKey\" :\"key\"\n" +
            "}";
```

这种请求，我们也很可能需要挡住后面的重复请求。所以求业务参数摘要之前，需要剔除这类时间字段。还有类似的字段可能是**GPS的经纬度字段**（重复请求间可能有极小的差别）。







