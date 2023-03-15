



RPC

模拟dubbo







负载均衡

### Failfast Cluster

快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录。

### Failsafe Cluster

失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作。

### Failback Cluster

失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。

### Forking Cluster

并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 `forks="2"` 来设置最大并行数。

### Broadcast Cluster

广播调用所有提供者，逐个调用，任意一台报错则报错。通常用于通知所有提供者更新缓存或日志等本地资源信息。







集群容错 

服务降级



mock



stub





泛化引用



泛化实现





# SPI机制

1、**dubbo的SPI机制增加了对IOC、AOP的支持，一个扩展点可以直接通过setter注入到其他扩展点。**

2、把配置文件中扩展实现的格式修改，例如META-INF/dubbo/com.xxx.Protocol里的com.foo.XxxProtocol格式改为了xxx = com.foo.XxxProtocol这种以键值对的形式，这样做的目的是为了让我们更容易的定位到问题，



## 相关注解

### 注解@SPI

表明该接口为可扩展接口

该注解只有一个属性value，代表默认扩展点





### 注解@Adaptive

该注解为了保证dubbo在内部调用具体实现的时候不是硬编码来指定引用哪个实现

**在实现类上面加上@Adaptive注解，表明该实现类是该接口的适配器**，有该注解的方法的参数必须包含URL（所有扩展点都通过传递URL携带配置信息）





在接口方法上加@Adaptive注解，dubbo会动态生成适配器类

```java
@SPI("netty")
public interface Transporter {

    /**
     * Bind a server.
     */
    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    Server bind(URL url, ChannelHandler handler) throws RemotingException;

    /**
     * Connect to a server.
     */
    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;

}
```

我们可以看到在这个接口的bind和connect方法上都有@Adaptive注解，有该注解的方法的参数必须包含URL，ExtensionLoader会通过createAdaptiveExtensionClassCode方法动态生成一个Transporter$Adaptive类，生成的代码如下：

```java
package com.alibaba.dubbo.remoting;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class Transporter$Adaptive implements com.alibaba.dubbo.remoting.Transporter{
    
    public com.alibaba.dubbo.remoting.Client connect(com.alibaba.dubbo.common.URL arg0, com.alibaba.dubbo.remoting.ChannelHandler arg1) throws com.alibaba.dubbo.remoting.RemotingException {
        //URL参数为空则抛出异常。
        if (arg0 == null) 
            throw new IllegalArgumentException("url == null");
        
        com.alibaba.dubbo.common.URL url = arg0;
        /**@Adaptive注解中有一些key值，比如connect方法的注解中有两个key，分别为“client”和“transporter”，URL会首先去取client对应的value来作为我上述（一）注解@SPI中写到的key值，如果为空，则去取transporter对应的value，如果还是为空，则会根据SPI默认的key，也就是netty去调用扩展的实现类，如果@SPI没有设定默认值，则会抛出IllegalStateException异常
        */
        //getParameter(String key, String defaultValue)
        String extName = url.getParameter("client", url.getParameter("transporter", "netty"));
        if(extName == null) 
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" + url.toString() + ") use keys([client, transporter])");
        //这里我在后面会有详细介绍
        com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter)ExtensionLoader.getExtensionLoader
        
        (com.alibaba.dubbo.remoting.Transporter.class).getExtension(extName);
        return extension.connect(arg0, arg1);
    }
    public com.alibaba.dubbo.remoting.Server bind(com.alibaba.dubbo.common.URL arg0, com.alibaba.dubbo.remoting.ChannelHandler arg1) throws com.alibaba.dubbo.remoting.RemotingException {
        if (arg0 == null) 
            throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg0;
        String extName = url.getParameter("server", url.getParameter("transporter", "netty"));
        if(extName == null) 
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" + url.toString() + ") use keys([server, transporter])");
        com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter)ExtensionLoader.getExtensionLoader
        (com.alibaba.dubbo.remoting.Transporter.class).getExtension(extName);
        
        return extension.bind(arg0, arg1);
    }
}
```



### 注解@Activate

扩展点自动激活加载的注解，就是用条件来控制该扩展点实现是否被自动激活加载，在扩展实现类上面使用











# spring与dubbo整合



@EnableDubbo













