

tomcat整体架构和请求处理流程





tomcat部署应用的三种方式 ：HostConfig#deployApps()

servlet默认是单例的

多例：servlet实现SingleThreadModel



![Coyote组件](images/Coyote组件.png)





<img src="images/tomcat架构.png" alt="tomcat架构" style="zoom:80%;" />

Engine

Host

Context

Wrapper







长连接的处理逻辑

Connection:keep-alive 客户端告诉服务器 响应后不要关闭tcp连接





浏览器同一时间针对同一域名下的请求有一定数量限制  6



connector.RequestFacade --> connector.Request -->  coyote.Request



**ByteChunk** 字节块





1、解析请求行  先使用ByteChunk标记inputBuffer字节数组哪些部分(start到end)表示method uri。。。。

只有调用request.getMethod才会将ByteChunk标记的字节转为字符串

connector.RequestFacade --> connector.Request -->  coyote.Request

2、解析请求头   

请求行 请求头 -> Request对象

3、处理一些特殊请求头（connection:keep-alive）

//请求体暂不做处理

4、把请求交给容器处理

servlet中可能会操作请求体

4.1servlet没有处理完请求体 

4.2servlet没有处理请求体



5、endRequest() tomcat做一些收尾工作  比如跳过未处理完的请求体   涉及到ByteChunk的start和end **InputBuffer的pos和lastValid**

InputBuffer中包含当前请求的请求体和下一个请求的请求数据（跟是否长连接？？）

先使用ByteChunk标记InputBuffer中的数据 再使用ByteChunk读取InputBuffer中的数据 

pos和lastValid指针的移动：ByteChunk从inputBuffer中获取标记数据时    endRequest()中会对pos指针做修正



request.getInputStream().read(byte[]) 返回-1表示Tomcat已经读取了contentLength指定大小的请求体







解析请求体 tomcat判断请求体是否结束

1、tomcat根据contentLength判断请求体的长度    IdentityInputFilter 处理ContentLength   

 2、分块传输  ChunkedInputFilter   

```
transfer-encoding:chunked     和 contentLength:1000 都有的情况下  以前者的方式解析请求体
```











响应

分contentLength和chunked





  CoyoteOutputStream out = request.getOutputStream()   

out.write()

out.flush()

out.close();







CoyoteOutputStream#write(byte[]) -> OutputBuffer#write(byte[], int, int)





CoyoteOutputStream#flush



响应有两级缓存 

一级缓存  ：OutputBuffer中ByteChunk中的buff属性（有数据复制）

二级缓存：InternalOutputBuffer.OutputStreamOutputBuffer中ByteChunk中的buff属性（有数据复制）

一级缓存和二级缓存都是ByteChunk类完成的，当buff属性足够装下数据时，就把数据复制到buff中，如果装不下时，就调用ByteOutputChannel.realWriteBytes()方法将数据写到下一级缓存 （一级缓存写到二级缓存，二级缓存写到操作系统缓存(socket的sednbuff)），ByteOutputChannel不同的实现类 写到不同的缓存 ，都是ByteChunk类型 不过两个ByteChunk里面的output的类型不一样  一个是OutputBuffer，一个是InternalOutputBuffer



ByteOutputChannel接口 有两个实现类 OutputBuffer 和 InternalOutputBuffer：

OutputBuffer#realWriteBytes：将数据写到InternalOutputBuffer的ByteChunk中（如果没有InternalOutputBuffer这一级缓存，则直接发送到socket的在操作系统的send buff中）

InternalOutputBuffer#realWriteBytes：将数据写到socket的在操作系统的send buff中

InternalOutputBuffer在ChunkedOutputFilter#doWrite或IdentityOutputFilter#doWrite中被调用的



响应体发送之前 必须发送响应头（此时tomcat就要确定响应体使用哪种格式contentLength或chunked）响应头直接发送到二级缓存（如果有的话）

1、CoyoteOutputStream.write()调用过程中，如果能调用到AbstractOutputBuffer#doWrite中会判断有没有发送响应头

2、调用了flush()，会判断有没有发送响应头



场景：

1、

write()

flush()

write()

结果：chunked



2、write()   上层缓冲区和底层缓冲区能装下

结果：contentLength

connector.Response#finishResponse     tomcat会调用flush

3、write()  上层缓冲区和底层缓冲区装不下

结果：chunked

connector.Response#finishResponse    tomcat会调用flush   





响应体的发送格式也有两种：

1、contentLength

2、chunked

哪些情况下使用contentLength（前提：tomcat必须能够确定响应体的长度），哪些情况下使用chunked：

1、只要调用了flush()(发送数据到socket的sendBuff)，就使用chunked（flush()后还可以调用write方法 此时flush无法确定响应体长度）

2、即使没有调用flush(),如果响应体太大，一二级缓存都装不下 使用chunked

3、没有调用flush()，响应体比较小，servlet执行完后一二级缓存还能容纳（servlet都执行完了，tomcat能确定响应体长度），使用contentLength   



write()完整流程：从servlet中调用write【flush()可调可不调】到数据发送到socket在操作系统的sendBuff

CoyoteOutputStream.write()  --> OutputBuffer#write() --> ByteChunk.append()-->OutputBuffer#realWriteBytes-->coyote.Response#doWrite-->AbstractOutputBuffer#doWrite-->ChunkedOutputFilter#doWrite-->InternalOutputBuffer.OutputStreamOutputBuffer#doWrite-->ByteChunk#append()-->InternalOutputBuffer#realWriteBytes-->

SocketOutputStream.write()





flush()完整流程

flush()之前的数据都会**马上**被发送到客户端



ByteChunk.append()：如果ByteChunk的buff装的下要发送的数据，就把要发送的的数据先复制到buff，否则调用ByteOutputChannel.realWriteBytes()写到下一级的缓冲区



首先Servlet中调用ResponseFacade.write(byte[])，



connector.Response#getOutputStream   ==> CoyoteOutputStream









```java
class ByteChunk{
  
  byte[] buff;
  //1、ByteOutputChannel接口 有两个实现类 OutputBuffer 和 InternalOutputBuffer
  //2、OutputBuffer和InternalOutputBuffer都有ByteChunk类型的成员变量，每个ByteChunk成员变量的out类型是当前类型
  ByteOutputChannel out;
 
}

class OutputBuffer{

	ByteChunk bb; //这个bb中的out属性实现类是OutputBuffer
}

class InternalOutputBuffer{
  ByteChunk socketBuffer;//这个bb中的out属性实现类是OutputBuffer
  
  //socket对应的outputStream
  outputStream = socketWrapper.getSocket().getOutputStream();

}







```





tomcat调优

1、处理请求的线程池 一个连接对应一个线程

2、Acceptor的数量  用来接受连接

3、响应的两个缓存







nio：

tomcat7（servlet3.0） nio：接受请求 -- 阻塞    读请求行、请求头是非阻塞的   读请求体-阻塞  响应-阻塞 

tomcat8(servlet3.1规范)



读请求体：

1、先尝试着读一次

2、注册一个读事件    辅助Selector   -- BlockingSelector

3、阻塞   如何实现？？CountDownLatch

4、读







类加载器

启动类加载器

扩展类加载器

应用类加载器

1、每种加载器都有自己的工作目录 加载指定文件夹下的class文件

2、双亲委派



A.class.getClassLoder()   哪个类加载器加载了A类 就返回哪个类加载



打破双亲委派机制：

自定义类加载器 重写loadClass方法



为什么要打破双亲委派机制？

- 对于各个 `webapp`中的 `class`和 `lib`，需要相互隔离，不能出现一个应用中加载的类库会影响另一个应用的情况，而对于许多应用，需要有共享的lib以便不浪费资源。
- 与 `jvm`一样的安全性问题。使用单独的 `classloader`去装载 `tomcat`自身的类库，以免其他恶意或无意的破坏；
- 热加载。相信大家一定为 `tomcat`修改文件不用重启就自动重新装载类库而惊叹吧。



 

热加载：WebappLoader

1、class文件有修改（内容修改，不包括增加class）

2、jar包有增加删除



热加载：

WEB-INF/classes目录下的文件发生了变化，WEB-INF/lib目录下的jar包添加、删除、修改都会触发热加载，重新加载class

重新创建一个WebappClassLoader





热部署：

重新部署应用（webapp下的某个应用）

重新创建Context(代表一个应用)





 commonLoader：Tomcat最基本的类加载器，加载路径中的class可以被Tomcat容器本身以及各个Webapp访问；
catalinaLoader：Tomcat容器私有的类加载器，加载路径中的class对于Webapp不可见；
sharedLoader：各个Webapp共享的类加载器，加载路径中的class对于所有Webapp可见，但是对于Tomcat容器不可见；
WebappClassLoader：各个Webapp私有的类加载器，加载路径中的class只对当前Webapp可见；





一个StandardContext对应一个应用

name

path

reloadable

webappLoader

​	管理的 目录：web-inf/classes;web-inf/lib

解析web.xml





tomcat启动流程







**任务 tomcat8   nio**

session

jsp