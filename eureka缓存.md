																								eureka缓存	

# Eureka server





服务端三级缓存架构

|         缓存          |           类型           |                             说明                             |
| :-------------------: | :----------------------: | :----------------------------------------------------------: |
|     **registry**      |    ConcurrentHashMap     | **实时更新**，服务注册、服务下架、服务剔除服务剔除加读锁实时更新， 客户端做服务发现时加写锁，类AbstractInstanceRegistry成员变量，UI端请求的是这里的服务注册信息 |
| **readWriteCacheMap** | Guava Cache/LoadingCache | **实时更新**，服务注册、服务下架、服务剔除实时更新，类ResponseCacheImpl成员变量，缓存时间180秒 |
| **readOnlyCacheMap**  |    ConcurrentHashMap     | **周期更新**，类ResponseCacheImpl成员变量，默认每**30s**从readWriteCacheMap更新，Eureka client默认从这里更新服务注册信息，可配置直接从readWriteCacheMap更新 |



缓存相关配置

|                        配置                        | 默认  |                             说明                             |
| :------------------------------------------------: | :---: | :----------------------------------------------------------: |
|      `eureka.server.useReadOnlyResponseCache`      | true  | Client从readOnlyCacheMap更新数据，false则跳过readOnlyCacheMap直接从**readWriteCacheMap**更新 |
|   `eureka.server.responsecCacheUpdateIntervalMs`   | 30000 |   readWriteCacheMap更新至readOnlyCacheMap周期，默认**30s**   |
|     `eureka.server.evictionIntervalTimerInMs`      | 60000 |            清理未续约节点(evict)周期，默认**60s**            |
| `eureka.instance.leaseExpirationDurationInSeconds` |  90   |             清理未续约节点超时时间，默认**90s**              |



# Eureka client

Eureka Client 启动后，作为服务提供者立即向 Server 注册，默认情况下每 30s心跳 续约(renew)；作为服务消费者立即向 Server 全量更新服务注册信息，默认情况下每 30s 增量更新服务注册信息；Ribbon 延时 1s 向 Client 获取使用的服务注册信息，默认每 30s 更新使用的服务注册信息，只保存状态为 UP 的服务。



二级缓存（eureka client + ribbon）

|        缓存         |       类型        |                             说明                             |
| :-----------------: | :---------------: | :----------------------------------------------------------: |
|   localRegionApps   |  AtomicReference  | **周期更新**，类DiscoveryClient成员变量，Eureka Client保存服务注册信息，启动后立即向Server全量更新，默认每**30s**增量更新 |
| upServerListZoneMap | ConcurrentHashMap | **周期更新**，类LoadBalancerStats成员变量，Ribbon保存使用且状态为**UP**的服务注册信息，启动后延时1s向Client更新，默认每**30s**更新 |



缓存相关配置

| `eureka.instance.leaseRenewalIntervalInSeconds` | 30    | Eureka Client 心跳续约周期，默认**30s**                      |
| ----------------------------------------------- | ----- | ------------------------------------------------------------ |
| `eureka.client.registryFetchIntervalSeconds`    | 30    | Eureka Client 增量更新周期，默认**30s**（正常情况下增量更新，超时或与Server端不一致等情况则全量更新） |
| `ribbon.ServerListRefreshInterval`              | 30000 | Ribbon 更新周期（ribbon是从eureka client的缓存更新到serverList），默认**30s** |





# 默认配置下服务消费者最长感知时间

| Eureka Client |                    时间                    |                             说明                             |
| :-----------: | :----------------------------------------: | :----------------------------------------------------------: |
|     上线      | 30(readOnly)+30(Client)+30(Ribbon)=**90s** |       readWrite -> readOnly -> Client -> Ribbon 各30s        |
|   正常下线    | 30(readonly)+30(Client)+30(Ribbon)=**90s** | 服务正常下线（kill或kill -15杀死进程）会给进程善后机会，DiscoveryClient.shutdown()将向Server更新自身状态为DOWN，然后发送DELETE请求注销自己，registry和readWriteCacheMap实时更新，故UI将不再显示该服务实例 |
|  非正常下线   |      30+60(evict)*2+30+30+30=**240s**      | 服务非正常下线（kill -9杀死进程或进程崩溃）不会触发DiscoveryClient.shutdown()方法，Eureka Server将依赖每60s清理超过90s未续约服务从registry和readWriteCacheMap中删除该服务实例 |



**考虑如下情况**

- 0s 时服务未主动通知 Eureka Client 直接下线；
- 29s 时第一次过期检查 evict 未超过 90s；
- 89s 时第二次过期检查 evict 未超过 90s；
- 149s 时第三次过期检查 evict 未续约时间超过了 90s，故将该服务实例从 registry 和 readWriteCacheMap 中删除；
- 179s 时定时任务从 readWriteCacheMap 更新至 readOnlyCacheMap;
- 209s 时 Eureka Client 从 Eureka Server 的 readOnlyCacheMap 更新；
- 239s 时 Ribbon 从 Eureka Client 更新。



![服务消费者最长感知时间](images/服务消费者最长感知时间.jpeg)







# 应对措施



## Eureka Server

1. **缩短 readOnlyCacheMap 更新周期**。缩短该定时任务周期可减少滞后时间。
2. **关闭 readOnlyCacheMap**。中小型系统可以考虑该方案，Eureka Client 直接从 readWriteCacheMap 更新服务注册信息。



## Eureka Client

1. **服务消费者使用容错机制**。如 Spring Cloud Retry 和 Hystrix，Ribbon、Feign、Zuul 都可以配置 Retry，服务消费者访问某个已下线节点时一般报 ConnectTimeout，这时可以通过 Retry 机制重试下一个节点。
2. **服务消费者缩短更新周期**。Eureka Client 和 Ribbon 二级缓存影响状态更新，缩短这两个定时任务周期可减少滞后时间，例如配置：
3. **服务提供者保证服务正常下线**。服务下线时使用 kill 或 kill -15 命令，避免使用 kill -9 命令，kill 或 kill -15 命令杀死进程时将触发 Eureka Client 的 shutdown()方法，主动删除 Server 的 registry 和 readWriteCacheMap 中的注册信息，不必依赖 Server 的 evict 清除。
4. **服务提供者延迟下线**。服务下线之前先调用接口使 Eureka Server 中保存的服务状态为 DOWN 或 OUT_OF_SERVICE 后再下线，二者时间差根据缓存机制和配置决定，比如默认情况下调用接口后延迟 90s 再下线服务即可保证服务消费者不会调用已下线服务实例。









