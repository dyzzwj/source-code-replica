##目录说明

bin：存放tomcat的启动、停止等批处理脚本

conf：tomcat相关的配置文件

- ​	catalina.policy：Tomcat 运行的安全策略配置
- ​	catalina.properties：Tomcat 的环境变量配置 
- ​	**context.xml**：用于定义所有web应用均需加载的Context配置，如果web应用指定了自己的context.xml，该文件将被覆盖 
- ​	logging.properties：Tomcat 的日志配置文件， 可以通过该文件修改Tomcat 的日志级别及日志路径等 
- ​	**server.xml**：Tomcat 服务器的核心配置文件 
- ​	tomcat-users.xml：定义Tomcat默认的用户及角色映射信息配置 
- ​	**web.xml**：Tomcat 中所有应用默认的部署描述文件， 主要定义了基础Servlet和MIME映射。 

lib：Tomcat 服务器的依赖包 

webapps：Tomcat 默认的Web应用部署目录 

work：Web 应用JSP代码生成和编译的临时目录 



##整体架构

1、Http服务器：**处理socket连接**，负责网络字节流Request和Response对象的转化

2、Servlet容器：**加载和管理Servlet**，以及具体处理request请求

流程：当客户请求某个资源时，HTTP服务器会用一个ServletRequest对象把客户的请求信息封 装起来，然后调用Servlet容器的service方法，Servlet容器拿到请求后，根据请求的URL 和Servlet的映射关系，找到相应的Servlet，如果Servlet还没有被加载，就用反射机制创建这个Servlet，并调用Servlet的init方法来完成初始化，接着调用Servlet的service方法 来处理请求，把ServletResponse对象返回给HTTP服务器，HTTP服务器会把响应发送给 客户端。

##Coyote连接器

coyote是tomcat的Connector框架的名字，简单说就是coyote来处理底层的socket，并将http请求、响应等字节流层面的东西，包装成Request和Response两个类（这两个类是tomcat定义的，而非servlet中的ServletRequest和ServletResponse），供容器使用；同时，为了能让我们编写的servlet能够得到ServletRequest，tomcat使用了facade模式，将比较底层、低级的Request包装成为ServletRequest（这一过程通常发生在Wrapper容器一级

###io模型协议

bio

nio

nio2.0

arp

###应用层协议

http/1.1

http/2

ajp

###连接器组件

![Coyote组件](\images\Coyote组件.png)

####EndPoint

1、Coyote 通信端点，即通信监听的接口，是具体Socket接收和发送处理器，是对传输层的抽象，因此EndPoint用来实现TCP/IP协议的2、Tomcat 并没有EndPoint 接口，而是提供了一个抽象类AbstractEndpoint ， 里面定 义了两个内部类：Acceptor和SocketProcessor。Acceptor用于监听Socket连接请求。 SocketProcessor用于处理接收到的Socket请求，它实现Runnable接口，在Run方法里 调用协议处理组件Processor进行处理。为了提高处理能力，SocketProcessor被提交到 线程池来执行。而这个线程池叫作执行器（Executor)



####Processor

Coyote 协议处理接口 ，如果说EndPoint是用来实现TCP/IP协议的，那么Processor用来实现HTTP协议，Processor接收来自EndPoint的Socket，读取字节流解析成Tomcat Request和Response对象，并通过Adapter将其提交到容器处理，Processor是对应用层协议的抽象。

####ProtocalHandler

Coyote 协议接口， 通过Endpoint 和 Processor ， 实现针对具体协议的处理能力。Tomcat 按照协议和I/O 提供了6个实现类 ： AjpNioProtocol ， AjpAprProtocol， AjpNio2Protocol ， Http11NioProtocol ，Http11Nio2Protocol ， Http11AprProtocol。我们在配置tomcat/conf/server.xml 时 ， 至少要指定具体的 ProtocolHandler , 当然也可以指定协议名称 ， 如 ： HTTP/1.1 ，如果安装了APR，那么 将使用Http11AprProtocol ， 否则使用 Http11NioProtocol 。

#### Adaptor 适配器

由于协议不同，客户端发过来的请求信息也不尽相同，Tomcat定义了自己的Request类 来“存放”这些请求信息。ProtocolHandler接口负责解析请求并生成Tomcat Request类。 但是这个Request对象不是标准的ServletRequest，也就意味着，不能用Tomcat Request作为参数来调用容器。Tomcat设计者的解决方案是引入CoyoteAdapter，这是适配器模式的经典运用，连接器调用CoyoteAdapter的Sevice方法，传入的是Tomcat Request对象，CoyoteAdapter负责将Tomcat Request转成ServletRequest，再调用容器的Service方法。 



## Catalina容器

Catalina 是Servlet 容器实现，包含了之前讲到的所有的容器组件，以及后续章节涉及到的安全、会话、集群、管理等Servlet 容器架构的各个方面。它通过松耦合的方式集成 Coyote，以完成按照请求协议进行数据读写。同时，它还包括我们的启动入口、Shell程序等。

![tomcat模块分层](\images\tomcat模块分层.png)



![Catalina结构](\images\Catalina结构.png)

Catalina负责管理Server，而Server表示着整个服务器。Server下面有多个 服务Service，**每个Service都包含着一个或多个连接器组件Connector（Coyote 实现）和一个容器组件Container（一个Engine）**。在Tomcat 启动的时候， 会初始化一个Catalina的实例。

组件说明：

Catalina：负责解析Tomcat的配置文件 , 以此来创建服务器Server组件，并根据命令来对其进行管理 

Server：服务器表示整个Catalina Servlet容器以及其它组件，负责组装并启动Servlet引擎,Tomcat连接器。Server通过实现Lifecycle接口，提供了一种优雅的启动和关闭整个系统的方式 

Service：一个Server包含多个Service。**每个Service都包含着一个或多个连接器组件Connector（Coyote 实现）和一个容器组件Container（一个Engine）**

Connector：连接器，处理与客户端的通信，它负责接收客户请求，然后转给相关的容器处理，最后向客户返回响应结果 

Container（狭义上的catalina）：容器，负责处理用户的servlet请求，并返回对象给web用户的模块 





####Container结构

![tomcat结构](\images\tomcat结构.png)

Engine：表示整个Catalina的Servlet引擎，用来管理多个虚拟站点，**一个Service最多只能有一个Engine**，但是一个引擎可包含多个Host 

Host：代表一个虚拟主机，每个虚拟主机和某个网络域名相匹配。每个虚拟主机下都可以部署一个或者多个WebApp，每个WebApp对应于一个Context，有一个ContextPath。当Host获得一个请求时，将把该请求匹配到某个Context上，然后把该请求交给该Context来处理。匹配的方法是“最长匹配”，所以一个path==“”的Context将成为该Host的默认Context，所有无法和其他Context的路径名匹配的请求都将最终和该默认Context匹配。

Context：一个Context对应于一个Web Application（Web应用），一个Web应用有一个或多个Wrapper组成，Context在创建的时候将根据配置文件$CATALINA_HOME/conf/web.xml和$WEBAPP_HOME/WEB-INF/web.xml载入Servlet类。如果找到，则执行该类，获得请求的回应，并返回。

Wrapper：表示一个Servlet，Wrapper 作为容器中的最底层，不能包含子容器



##tomcat启动流程



##请求处理流程

![tomcat请求处理流程](\images\tomcat请求处理流程.png)



1）Connector组件Endpoint中的Acceptor监听客户端套接字连接并接收Socket。 

2) 将连接交给线程池Executor处理，开始执行请求响应任务。 

3) Processor组件读取消息报文，解析请求行、请求体、请求头，封装成Request对象。 

4) Mapper组件根据请求行的URL值和请求头的Host值匹配由哪个Host容器、Context容器、Wrapper容器处理请求。

5) CoyoteAdaptor组件负责将Connector组件和Engine容器关联起来，把生成的Request对象和响应对象Response传递到Engine容器中，调用 Pipeline。 

6) Engine容器的管道开始处理，管道中包含若干个Valve、每个Valve负责部分处理逻辑。执行完Valve后会执行基础的 Valve--StandardEngineValve，负责调用Host容器的 Pipeline。 

7) Host容器的管道开始处理，流程类似，最后执行 Context容器的Pipeline。 

8) Context容器的管道开始处理，流程类似，最后执行 Wrapper容器的Pipeline。 

9) Wrapper容器的管道开始处理，流程类似，最后执行 Wrapper容器对应的Servlet对象的处理方法。