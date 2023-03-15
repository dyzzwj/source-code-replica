​																							nacos 配置中心服务端																					



本章分析Nacos配置中心1.4.1版本服务端的几个核心功能，**服务端使用MySQL数据源（ -DembeddedStorage=false），以集群方式启动（-Dnacos.standalone=false）。**

- **GET /v1/cs/configs**服务端查询配置逻辑
- **POST /v1/cs/configs/listener**接口负责配置监
- **POST /v1/cs/configs**负责发布配置，基于非集群derby启动的流程



# 一、配置存储

![三层存储.png](../images/336698327c3046bc962cd5cc7fff92c9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0-20230306215921487.awebp)





从整体上Nacos服务端的配置存储分为三层：

- 内存：Nacos每个节点都在内存里缓存了配置，但是**只包含配置的md5**（缓存配置文件太多了），所以**内存级别的配置只能用于比较配置是否发生了变更，只用于客户端长轮询配置等场景**。
- 文件系统：文件系统配置来源于数据库写入的配置。如果是集群启动或mysql单机启动，服务端会以本地文件系统的配置响应客户端查询。
- 数据库：所有写数据都会先写入数据库。只有当以derby数据源（-DembeddedStorage=true）单机（-Dnacos.standalone=true）启动时，客户端的查询配置请求会实时查询derby数据库。



## 1、内存

**对于写请求，Nacos会将数据先更新到数据库，之后异步写入所有节点的文件系统并更新内存**。

**CacheItem**在Nacos服务端对应一个配置文件，缓存了配置的md5，持有一把读写锁控制访问冲突。

```java
public class CacheItem {
    // groupKey = namespace + group + dataId
    final String groupKey;
    // 配置md5
    public volatile String md5 = Constants.NULL;
    // 更新时间
    public volatile long lastModifiedTs;
    // nacos自己实现的简版读写锁
    public SimpleReadWriteLock rwLock = new SimpleReadWriteLock();
    // 配置文件类型：text/properties/yaml
    public String type;
    // ... 省略其他betaIp和tag相关属性
}
```



**ConfigCacheService**一个重要的服务，负责管理所有内存配置CacheItem。

```java
public class ConfigCacheService {
		// 持久层服务（Derby或MySQL）
    private static PersistService persistService;
    // groupKey -> cacheItem.
    private static final ConcurrentHashMap<String, CacheItem> CACHE = new ConcurrentHashMap<String, CacheItem>();
  
    // 更新配置的md5
    public static void updateMd5(String groupKey, String md5, long lastModifiedTs) {
        CacheItem cache = makeSure(groupKey);
        if (cache.md5 == null || !cache.md5.equals(md5)) {
            cache.md5 = md5;
            cache.lastModifiedTs = lastModifiedTs;
            NotifyCenter.publishEvent(new LocalDataChangeEvent(groupKey));
        }
    }
    // 比较入参md5与缓存md5是否一致
    public static boolean isUptodate(String groupKey, String md5, String ip, String tag) {
        String serverMd5 = ConfigCacheService.getContentMd5(groupKey, ip, tag);
        return StringUtils.equals(md5, serverMd5);
    }
    // 获取缓存配置
    public static CacheItem getContentCache(String groupKey) {
        return CACHE.get(groupKey);
    }
    // 创建配置
    static CacheItem makeSure(final String groupKey) {
        CacheItem item = CACHE.get(groupKey);
        if (null != item) {
            return item;
        }
        CacheItem tmp = new CacheItem(groupKey);
        item = CACHE.putIfAbsent(groupKey, tmp);
        return (null == item) ? tmp : item;
    }
    // 获取配置读锁
    public static int tryReadLock(String groupKey) {
        CacheItem groupItem = CACHE.get(groupKey);
        int result = (null == groupItem) ? 0 : (groupItem.rwLock.tryReadLock() ? 1 : -1);
        return result;
    }
    // 释放配置读锁
    public static void releaseReadLock(String groupKey) {
        CacheItem item = CACHE.get(groupKey);
        if (null != item) {
            item.rwLock.releaseReadLock();
        }
    }
}
```



## 2、文件系统

**Nacos刚启动时，内存中与文件系统中未必存在所有配置，所以DumpService会全量dump数据库的配置到文件系统与内存中。** 另外当数据库配置发生变化时，也会dump到本地文件系统。

```java
// 启动时DumpService全量dump
protected void dumpOperate(DumpProcessor processor, DumpAllProcessor dumpAllProcessor,
        DumpAllBetaProcessor dumpAllBetaProcessor, DumpAllTagProcessor dumpAllTagProcessor) throws NacosException {
    // 构造各类runnable任务...

    // 首次启动，dump数据库中所有配置到文件系统和内存中
    dumpConfigInfo(dumpAllProcessor);
    // 非单机部署，提交dump任务
    if (!EnvUtil.getStandaloneMode()) {
      ConfigExecutor.scheduleConfigTask(heartbeat, 0, 10, TimeUnit.SECONDS);
      // dump all config
      ConfigExecutor.scheduleConfigTask(dumpAll, initialDelay, DUMP_ALL_INTERVAL_IN_MINUTE, TimeUnit.MINUTES);
      // dump beta config
      ConfigExecutor
        .scheduleConfigTask(dumpAllBeta, initialDelay, DUMP_ALL_INTERVAL_IN_MINUTE, TimeUnit.MINUTES);
      // dump tag config
      ConfigExecutor
        .scheduleConfigTask(dumpAllTag, initialDelay, DUMP_ALL_INTERVAL_IN_MINUTE, TimeUnit.MINUTES);
    }
    ConfigExecutor.scheduleConfigTask(clearConfigHistory, 10, 10, TimeUnit.MINUTES);
}
```



## 3、数据库

配置文件主要存储在config_info表中。Nacos有三个配置项与数据源的选择有关：

- application.properties中的spring.datasource.platform配置项，默认为空，可以配置为mysql
- -Dnacos.standalone，true代表单机启动，false代表集群启动，默认false
- -DembeddedStorage，true代表使用嵌入式存储derby数据源，false代表不使用derby数据源，默认false

这块感觉比较乱，通过伪代码的方式理一下。主要是spring.datasource.platform在默认为空的场景下，**满足条件集群启动且-DembeddedStorage=false（默认false），还是会选择mysql数据源**。也就是说，**集群启动，如果没特殊配置，Nacos会使用MySQL数据源**。

```java
// 指定数据源为mysql，直接返回mysql
if (mysql) {
	return mysql;
}
// 如果单机部署，没指定数据源，使用derby
if (standalone) {
	return derby;
} else {
    // 如果集群部署且指定使用嵌入式存储，使用derby
	if (embeddedStorage) {
		return derby;
	} else {
	    // 集群部署，默认使用mysql存储
		return mysql;
	}
}
```



# 二、配置查询

**GET /v1/cs/configs**接口负责配置查询。这部分代码太长了，没有方法拆分，从整体梳理一下逻辑。

ConfigController#getConfig

```java
    /**
     *
     *  配置查询
     *  1、需要先获取配置项的读锁，根据获取成功与否返回不同状态码
     *  2.获取读锁成功后，查询配置。
     *      2.1如果单机部署且使用默认derby数据源，将实时查询配置返回；
     *      2.2如果集群部署或使用mysql数据源，将查询本地文件系统中的缓存配置返回。
     *
     * @throws ServletException ServletException.
     * @throws IOException      IOException.
     * @throws NacosException   NacosException.
     */
    @GetMapping
    @Secured(action = ActionTypes.READ, parser = ConfigResourceParser.class)
    public void getConfig(HttpServletRequest request, HttpServletResponse response,
            @RequestParam("dataId") String dataId, @RequestParam("group") String group,
            @RequestParam(value = "tenant", required = false, defaultValue = StringUtils.EMPTY) String tenant,
            @RequestParam(value = "tag", required = false) String tag)
            throws IOException, ServletException, NacosException {
        // check tenant
        ParamUtils.checkTenant(tenant);
        tenant = NamespaceUtil.processNamespaceParameter(tenant);
        // check params
        ParamUtils.checkParam(dataId, group, "datumId", "content");
        ParamUtils.checkParam(tag);

        final String clientIp = RequestUtil.getRemoteIp(request);
        //com.alibaba.nacos.config.server.controller.ConfigServletInner#doGetConfig
        inner.doGetConfig(request, response, dataId, group, tenant, tag, clientIp);
    }
```

ConfigServletInner#doGetConfig

```java
public String doGetConfig(HttpServletRequest request, HttpServletResponse response, String dataId, String group,
        String tenant, String tag, String clientIp) throws IOException, ServletException {
    final String groupKey = GroupKey2.getKey(dataId, group, tenant);
    // 尝试获取groupKey对应配置读锁
    int lockResult = tryConfigReadLock(groupKey);
    boolean isBeta = false;
    // 获取锁成功
    if (lockResult > 0) {
        FileInputStream fis = null;
        try {
            String md5 = Constants.NULL;
            long lastModified = 0L;
            // 从内存中获取配置信息，用于确定content_type
            CacheItem cacheItem = ConfigCacheService.getContentCache(groupKey);
            if (cacheItem != null) {
                // ... 根据配置文件类型，决定返回的报文的content_type
                response.setHeader(HttpHeaderConsts.CONTENT_TYPE, contentTypeHeader);
            }
            File file = null;
            ConfigInfoBase configInfoBase = null;
            PrintWriter out = null;
            if (isBeta) {
               // ...
            } else {
                if (StringUtils.isBlank(tag)) {
                    if (isUseTag(cacheItem, autoTag)) {
                        // ...
                    } else {
                        md5 = cacheItem.getMd5();
                        lastModified = cacheItem.getLastModifiedTs();
                        // 如果单机部署且使用derby数据源，查询实时配置
                        if (PropertyUtil.isDirectRead()) {
                            configInfoBase = persistService.findConfigInfo(dataId, group, tenant);
                        } else {
                            // 如果集群部署 或 使用mysql，读取本地文件系统中的配置
                            file = DiskUtil.targetFile(dataId, group, tenant);
                        }
                        if (configInfoBase == null && fileNotExist(file)) {
                            response.setStatus(HttpServletResponse.SC_NOT_FOUND);
                            response.getWriter().println("config data not exist");
                            return HttpServletResponse.SC_NOT_FOUND + "";
                        }
                    }
                } else {
                    // ... 请求包含tag的处理
                }
            }
            response.setHeader(Constants.CONTENT_MD5, md5);

            // Disable cache.
            // ...
            if (PropertyUtil.isDirectRead()) {
                out = response.getWriter();
                out.print(configInfoBase.getContent());
                out.flush();
                out.close();
            } else {
                fis.getChannel()
                        .transferTo(0L, fis.getChannel().size(), Channels.newChannel(response.getOutputStream()));
            }

        } finally {
            releaseConfigReadLock(groupKey);
            IoUtils.closeQuietly(fis);
        }
    } else if (lockResult == 0) {
        // 获取读锁返回0，表示配置不存在，返回404
        response.setStatus(HttpServletResponse.SC_NOT_FOUND);
        response.getWriter().println("config data not exist");
        return HttpServletResponse.SC_NOT_FOUND + "";
    } else {
        // 获取读锁返回-1，表示没有成功获取读锁，可能正有写操作发生，返回409
        response.setStatus(HttpServletResponse.SC_CONFLICT);
        response.getWriter().println("requested file is being modified, please try later.");
        return HttpServletResponse.SC_CONFLICT + "";
    }
    return HttpServletResponse.SC_OK + "";
}
```





首先，服务端会根据groupKey获取配置项的读锁，**tryConfigReadLock**会返回一个int，不同返回值代表含义和处理方式不同。

| 获取锁结果 | 含义                               | 返回HTTP状态码 |
| ---------- | ---------------------------------- | -------------- |
| 1          | 获取成功                           | 200            |
| 0          | 配置项不存在                       | 404            |
| -1         | 获取读锁失败，表示有写操作正在发生 | 409            |

获取锁成功后，会去查询配置，这里分为两种逻辑。如果是单机部署且使用derby数据源，则会立即去查询实时数据；否则，如果是集群部署或指定使用mysql数据源，会读取文件系统上的缓存配置。

```java
if (PropertyUtil.isDirectRead()) {
  // 如果单机部署，默认使用derby数据源，查询实时配置
	configInfoBase = persistService.findConfigInfo(dataId, group, tenant);
} else {
	// 如果集群部署或指定使用mysql数据源，读取本地文件系统中的配置
	file = DiskUtil.targetFile(dataId, group, tenant);
}
```

最后，根据上面的结果，分成两种方式写入客户端。

```java
if (PropertyUtil.isDirectRead()) {
    out = response.getWriter();
    out.print(configInfoBase.getContent());
    out.flush();
    out.close();
} else {
    fis.getChannel().transferTo(0L, fis.getChannel().size(),Channels.newChannel(response.getOutputStream()));
}
```

# 三、配置监听

**POST /v1/cs/configs/listener**接口负责配置监听，是长轮询服务端的逻辑。

ConfigController#listener

```java
/**
 * The client listens for configuration changes.
 *  配置监听，是长轮询服务端的逻辑   ClientWorker.LongPollingRunnable#run()中请求本接口
 *  1、先校验客户端配置项md5与服务端内存中缓存的md5是否一致，不一致直接拼接结果返回，这里返回的是发生变化的groupKey，由客户端发起二次请求/v1/cs/configs查询最新配置。
 *  2、如果客户端配置项md5已经最新，请求头中包含Long-Pulling-Timeout-No-Hangup=true，则立即返回200。
 *  3、如果不满足上述两点，开启AsyncContext，并提交一个ClientLongPolling任务等待配置发生变更后，响应客户端。
 */
@PostMapping("/listener")
@Secured(action = ActionTypes.READ, parser = ConfigResourceParser.class)
public void listener(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {
    // 设置request支持AsyncContext
    request.setAttribute("org.apache.catalina.ASYNC_SUPPORTED", true);
    String probeModify = request.getParameter("Listening-Configs");
    if (StringUtils.isBlank(probeModify)) {
        throw new IllegalArgumentException("invalid probeModify");
    }

    probeModify = URLDecoder.decode(probeModify, Constants.ENCODE);

    Map<String, String> clientMd5Map;
    try {
        // 解析客户端业务报文为 groupKey - 客户端当前配置md5
        clientMd5Map = MD5Util.getClientMd5Map(probeModify);
    } catch (Throwable e) {
        throw new IllegalArgumentException("invalid probeModify");
    }

    // do long-polling
    inner.doPollingConfig(request, response, clientMd5Map, probeModify.length());
}
```

ConfigServletInner#doPollingConfig

```java
/**
 * 轮询接口.
 */
public String doPollingConfig(HttpServletRequest request, HttpServletResponse response,
        Map<String, String> clientMd5Map, int probeRequestSize) throws IOException {

    // 判断请求头中是否包含Long-Pulling-Timeout，如果是，则执行长轮询
    if (LongPollingService.isSupportLongPolling(request)) {
        /**
         * 为长轮训添加配置监听
         */
        longPollingService.addLongPollingClient(request, response, clientMd5Map, probeRequestSize);
        return HttpServletResponse.SC_OK + "";
    }

    // Compatible with short polling logic.
    List<String> changedGroups = MD5Util.compareMd5(request, response, clientMd5Map);

    // Compatible with short polling result.
    String oldResult = MD5Util.compareMd5OldResult(changedGroups);
    String newResult = MD5Util.compareMd5ResultString(changedGroups);

    String version = request.getHeader(Constants.CLIENT_VERSION_HEADER);
    if (version == null) {
        version = "2.0.0";
    }
    int versionNum = Protocol.getVersionNumber(version);

    // Befor 2.0.4 version, return value is put into header.
    if (versionNum < START_LONG_POLLING_VERSION_NUM) {
        response.addHeader(Constants.PROBE_MODIFY_RESPONSE, oldResult);
        response.addHeader(Constants.PROBE_MODIFY_RESPONSE_NEW, newResult);
    } else {
        request.setAttribute("content", newResult);
    }

    Loggers.AUTH.info("new content:" + newResult);

    // Disable cache.
    response.setHeader("Pragma", "no-cache");
    response.setDateHeader("Expires", 0);
    response.setHeader("Cache-Control", "no-cache,no-store");
    response.setStatus(HttpServletResponse.SC_OK);
    return HttpServletResponse.SC_OK + "";
}
```



LongPollingService#addLongPollingClient

LongPollingService的addLongPollingClient方法，处理了实际/v1/cs/configs/listener接口的逻辑，分为这几步：

- **确定长轮询实际超时时间**：如果isFixedPolling=true（默认false），则设置为10s，无视客户端设置的超时时间；否则使用客户端设置的超时时间，默认30s，在这个基础上再减去500ms，防止客户端提前超时。
- **比较服务端内存中的配置md5与客户端本次请求的配置md5是否一致**：如果有配置项发生变更，则立即拼装请求返回（报文结构见第三部分）。**这一步保证了客户端长轮询后，查询配置时发生409错误，可以依靠下一次长轮询自动恢复。因为客户端会将配置的当前版本（md5）传过来，由服务端进行比较，如果客户端不传配置md5就做不到了**。
- 如果当时配置项没有发生变化，且请求头中包含Long-Pulling-Timeout-No-Hangup=true（本次长轮询包含首次监听的配置项），则立即返回。这一步说明本次请求的配置项，包含刚注册监听的配置项。
- 如果当时配置项没有发生变化，且不需要立即返回，则开启AsyncContext，将长轮询任务提交到其他线程池，等待其他线程设置AsyncContext.complete()再响应客户端。



```java
public void addLongPollingClient(HttpServletRequest req, HttpServletResponse rsp, Map<String, String> clientMd5Map, int probeRequestSize) {

    String str = req.getHeader(LongPollingService.LONG_POLLING_HEADER);
    String noHangUpFlag = req.getHeader(LongPollingService.LONG_POLLING_NO_HANG_UP_HEADER);
    String appName = req.getHeader(RequestUtil.CLIENT_APPNAME_HEADER);
    String tag = req.getHeader("Vipserver-Tag");
    int delayTime = SwitchService.getSwitchInteger(SwitchService.FIXED_DELAY_TIME, 500);

    // Add delay time for LoadBalance, and one response is returned 500 ms in advance to avoid client timeout.
    long timeout = Math.max(10000, Long.parseLong(str) - delayTime);
    /**
     * 确定长轮训实际超时时间
     * 如果isFixedPolling=true（默认false），则设置为10s，无视客户端设置的超时时间；
     * 否则使用客户端设置的超时时间，默认30s，在这个基础上再减去500ms，防止客户端提前超时。
     */
    if (isFixedPolling()) { //固定轮训 无视客户端设置的超时时间
        timeout = Math.max(10000, getFixedPollingInterval());
        // Do nothing but set fix polling timeout.
    } else {
        long start = System.currentTimeMillis();
        // 用内存缓存的md5比较，是否有配置项发生变更，如果有的话立即返回
        List<String> changedGroups = MD5Util.compareMd5(req, rsp, clientMd5Map);
        if (changedGroups.size() > 0) {
            /**
             * 如果有配置项发生变更，则立即拼装请求返回（报文结构见第三部分）。
             * 这一步保证了客户端长轮询后，查询配置时发生409错误(见ConfigServletInner#doGetConfig())，可以依靠下一次长轮询自动恢复。
             * 因为客户端会将配置的当前版本（md5）传过来，由服务端进行比较，如果客户端不传配置md5就做不到了。
             */
            generateResponse(req, rsp, changedGroups);
            LogUtil.CLIENT_LOG.info("{}|{}|{}|{}|{}|{}|{}", System.currentTimeMillis() - start, "instant",
                    RequestUtil.getRemoteIp(req), "polling", clientMd5Map.size(), probeRequestSize,
                    changedGroups.size());
            return;
        } else if (noHangUpFlag != null && noHangUpFlag.equalsIgnoreCase(TRUE_STR)) {
            // 如果客户端的本次长轮询请求，请求头包含Long-Pulling-Timeout-No-Hangup，则立即返回200
            //如果当时配置项没有发生变化，且请求头中包含Long-Pulling-Timeout-No-Hangup=true，则立即返回。这一步说明本次请求的配置项，包含刚注册监听的配置项。
            LogUtil.CLIENT_LOG.info("{}|{}|{}|{}|{}|{}|{}", System.currentTimeMillis() - start, "nohangup",
                    RequestUtil.getRemoteIp(req), "polling", clientMd5Map.size(), probeRequestSize,
                    changedGroups.size());
            return;
        }
    }

    /**
     * 如果当时配置项没有发生变化，且不需要立即返回，则开启AsyncContext，将长轮询任务提交到其他线程池，等待其他线程设置AsyncContext.complete()再响应客户端。
     */
    String ip = RequestUtil.getRemoteIp(req);

    // Must be called by http thread, or send response.
    // 开启AsyncContext
    final AsyncContext asyncContext = req.startAsync();

    // AsyncContext.setTimeout() is incorrect, Control by oneself
    asyncContext.setTimeout(0L);
    // 提交长轮询任务到其他线程  timeout时间之后执行
    ConfigExecutor.executeLongPolling(
            new ClientLongPolling(asyncContext, clientMd5Map, ip, probeRequestSize, timeout, appName, tag));
}
```



# 四、配置发布

**POST /v1/cs/configs**负责发布配置，是服务端的核心接口。**基于MySQL数据源**来看，这个接口总共做了几个事情：（单机Derby也是这个流程，集群Derby较为特殊，后续再说）

- 更新数据库
- 发布ConfigDataChangeEvent事件

​     	AsyncNotifyService监听ConfigDataChangeEvent事件，遍历所有nacos server服务端（包括自己），调用数据同步接口/v1/cs/communication/dataChange，nacos server接受到请求后，查询数据库的最新配置，写入本地文件系统，更新内存配置的md5，发布LocalDataChangeEvent事件，DataChangeTask监听该事件，会响应客户端长轮训请求。

![发布配置主流程](../images/发布配置主流程.jpeg)



## 1、数据库配置更新

ConfigController#publishConfig

**这里只更新了数据库，并没有更新本地文件系统和内存中的md5，此时客户端不会感知到配置变更，查询 的配置依然是旧的配置**

```java
// ConfigController
@PostMapping
public Boolean publishConfig(...) throws NacosException {
    if (StringUtils.isBlank(betaIps)) {
        if (StringUtils.isBlank(tag)) {
            // 更新数据库配置
          	//mysql - > ExternalStoragePersistServiceImpl.insertOrUpdate()
            persistService.insertOrUpdate(srcIp, srcUser, configInfo, time, configAdvanceInfo, true);
            // 发布ConfigDataChangeEvent事件
            ConfigChangePublisher
                    .notifyConfigChange(new ConfigDataChangeEvent(false, dataId, group, tenant, time.getTime()));
        } else {
            persistService.insertOrUpdateTag(configInfo, tag, srcIp, srcUser, time, true);
            ConfigChangePublisher.notifyConfigChange(
                    new ConfigDataChangeEvent(false, dataId, group, tenant, tag, time.getTime()));
        }
    } else {
        persistService.insertOrUpdateBeta(configInfo, betaIps, srcIp, srcUser, time, true);
        ConfigChangePublisher
                .notifyConfigChange(new ConfigDataChangeEvent(true, dataId, group, tenant, time.getTime()));
    }
    return true;
}
```





**ExternalStoragePersistServiceImpl（MySQL）插入或更新的逻辑如下，通过数据库唯一约束控制插入或更新**。

```java
@Override
public void insertOrUpdate(String srcIp, String srcUser, ConfigInfo configInfo, Timestamp time,
        Map<String, Object> configAdvanceInfo, boolean notify) {
    try {
        // 1. 尝试插入
        addConfigInfo(srcIp, srcUser, configInfo, time, configAdvanceInfo, notify);
    } catch (DataIntegrityViolationException ive) { // Unique constraint conflict
        // 2. 唯一约束冲突，更新
        updateConfigInfo(configInfo, srcIp, srcUser, time, configAdvanceInfo, notify);
    }
}
```

addConfigInfo尝试插入在一个事务里保存了config_info、config_tags_relation、his_config_info。

```java
@Override
public void addConfigInfo(final String srcIp, final String srcUser, final ConfigInfo configInfo,
        final Timestamp time, final Map<String, Object> configAdvanceInfo, final boolean notify) {
    boolean result = tjt.execute(status -> {
        try {
            // 1. 保存config_info
            long configId = addConfigInfoAtomic(-1, srcIp, srcUser, configInfo, time, configAdvanceInfo);
            // 2. 保存config_tags_relation
            String configTags = configAdvanceInfo == null ? null : (String) configAdvanceInfo.get("config_tags");
            addConfigTagsRelation(configId, configTags, configInfo.getDataId(), configInfo.getGroup(),
                    configInfo.getTenant());
            // 3. 记录日志his_config_info
            insertConfigHistoryAtomic(0, configInfo, srcIp, srcUser, time, "I");
        } catch (CannotGetJdbcConnectionException e) {
            LogUtil.FATAL_LOG.error("[db-error] " + e.toString(), e);
            throw e;
        }
        return Boolean.TRUE;
    });
}
```

当上述插入发生唯一约束冲突（**config_info中data_id,group_id,tenant_id组成唯一索引**）时，updateConfigInfo更新配置信息。

```java
    public void updateConfigInfo(final ConfigInfo configInfo, final String srcIp, final String srcUser,
            final Timestamp time, final Map<String, Object> configAdvanceInfo, final boolean notify) {
        try {
            // 1. SELECT ID,data_id,group_id,tenant_id,app_name,content,md5,type FROM config_info
            //  WHERE data_id=? AND group_id=? AND tenant_id=?
            ConfigInfo oldConfigInfo = findConfigInfo(configInfo.getDataId(), configInfo.getGroup(),
                    configInfo.getTenant());

            final String tenantTmp =
                    StringUtils.isBlank(configInfo.getTenant()) ? StringUtils.EMPTY : configInfo.getTenant();

            oldConfigInfo.setTenant(tenantTmp);

            String appNameTmp = oldConfigInfo.getAppName();
            // If the appName passed by the user is not empty, the appName of the user is persisted;
            // otherwise, the appName of db is used. Empty string is required to clear appName
            if (configInfo.getAppName() == null) {
                configInfo.setAppName(appNameTmp);
            }

            // 2. UPDATE config_info SET content=?, md5 = ?, src_ip=?,src_user=?,gmt_modified=?,app_name=?, c_desc=?,c_use=?,effect=?,type=?,c_schema=?
            //  WHERE data_id=? AND group_id=? AND tenant_id=?
            //将sql信息放到ThreadLocal
            updateConfigInfoAtomic(configInfo, srcIp, srcUser, time, configAdvanceInfo);

            String configTags = configAdvanceInfo == null ? null : (String) configAdvanceInfo.get("config_tags");
            if (configTags != null) {
                // Delete all tags and recreate them
                removeTagByIdAtomic(oldConfigInfo.getId());
                addConfigTagsRelation(oldConfigInfo.getId(), configTags, configInfo.getDataId(), configInfo.getGroup(),
                        configInfo.getTenant());
            }
            // 3. INSERT INTO his_config_info (id,data_id,group_id,tenant_id,app_name,content,md5,src_ip,src_user,gmt_modified,op_type) VALUES(?,?,?,?,?,?,?,?,?,?,?)
            insertConfigHistoryAtomic(oldConfigInfo.getId(), oldConfigInfo, srcIp, srcUser, time, "U");
            // 4. 将ConfigDumpEvent作为extendInfo放入ThreadLocal
            EmbeddedStorageContextUtils.onModifyConfigInfo(configInfo, srcIp, time);
            //真正更新
            databaseOperate.blockUpdate();
        } finally {
            EmbeddedStorageContextUtils.cleanAllContext();
        }
    }
```



数据库更新配置完成后，ConfigChangePublisher发布ConfigDataChangeEvent事件，如果使用derby数据源且使用集群模式启动将直接返回。言外之意，**使用derby数据源的集群模式下，需要通过其他方式将配置同步到所有节点（PersistService实现不一样）**。

ConfigChangePublisher#notifyConfigChange

```java
public static void notifyConfigChange(ConfigDataChangeEvent event) {
    // 集群启动（-Dnacos.standalone=false）并且嵌入式数据源（ -DembeddedStorage=true），不处理
    if (PropertyUtil.isEmbeddedStorage() && !EnvUtil.getStandaloneMode()) {
        return;
    }
    NotifyCenter.publishEvent(event);
}
```

之前客户端查询配置，对于derby数据源单机部署，会查询derby数据源中的配置；对于mysql数据源或集群部署，会查询本地文件系统中的配置。

## 2、事件发布

先看一下Nacos对于事件发布的设计。

**NotifyCenter通知中心，主要负责订阅事件与事件发布**。

```java
public class NotifyCenter {
    // NotifyCenter单例
    private static final NotifyCenter INSTANCE = new NotifyCenter();
  
		/** 普通事件 **/
    // 用于创建普通发布者的工厂
    private static BiFunction<Class<? extends Event>, Integer, EventPublisher> publisherFactory = null;
    // 普通事件发布者实现类，默认DefaultPublisher，可以通过JDKSPI调整
    private static Class<? extends EventPublisher> clazz = null;
    // 事件全类名 - 普通事件发布者
    private final Map<String, EventPublisher> publisherMap = new ConcurrentHashMap<String, EventPublisher>(16);
    // 普通事件发布者缓存事件的最大数量
    public static int ringBufferSize = 16384;
		
    /** Slow事件 **/
    // Slow事件发布者
    private DefaultSharePublisher sharePublisher;
    // slow事件发布者缓存事件的最大数量
    public static int shareBufferSize = 1024;
}
```



**NotifyCenter内部有两类Publisher发布者**：

- DefaultSharePublisher：用于处理SlowEvent类型事件。
- DefaultPublisher：用于处理其他类型事件。NotifyCenter中，每个事件（Class区分）对应一个DefaultPublisher。

**DefaultPublisher**内部维护一个阻塞队列，长度默认16384，客户端提交的事件都会放入这个队列。自身继承自Thread，负责向所有订阅者发布事件。

```java
public class DefaultPublisher extends Thread implements EventPublisher {
    // 负责处理的事件Class
    private Class<? extends Event> eventType;
    // 订阅这个事件的订阅者
    protected final ConcurrentHashSet<Subscriber> subscribers = new ConcurrentHashSet<Subscriber>();
    // 事件队列
    private BlockingQueue<Event> queue;
}
```



**Subscriber**订阅者，subscribeType返回订阅的事件Class，通过onEvent方法处理事件。

```java
public abstract class Subscriber<T extends Event> {
    // 事件处理
    public abstract void onEvent(T event);
    // 关注的事件Class
    public abstract Class<? extends Event> subscribeType();
    // 可以自定义一个执行器，不在Publisher线程执行
    public Executor executor() {
        return null;
    }
    // 是否忽略过期的事件
    public boolean ignoreExpireEvent() {
        return false;
    }
}
```

Subscriber和Publisher向NotifyCenter注册，NotifyCenter负责管理他们。另外因为Subscriber告诉外部自己所关注的事件，NotifyCenter还可以通过反射工厂创建事件所对应的Publisher。

```java
// NotifyCenter
// NotifyCenter static代码块中，创建反射Publisher工厂，这里clazz=DefaultPublisher
publisherFactory = new BiFunction<Class<? extends Event>, Integer, EventPublisher>() {
  @Override
  public EventPublisher apply(Class<? extends Event> cls, Integer buffer) {
    try {
      EventPublisher publisher = clazz.newInstance();
      publisher.init(cls, buffer);
      return publisher;
    } catch (Throwable ex) {
      LOGGER.error("Service class newInstance has error : {}", ex);
      throw new NacosRuntimeException(SERVER_ERROR, ex);
    }
  }
};

// 注册Publisher逻辑
public static EventPublisher registerToPublisher(final Class<? extends Event> eventType, final int queueMaxSize) {
  if (ClassUtils.isAssignableFrom(SlowEvent.class, eventType)) {
    return INSTANCE.sharePublisher;
  }
  final String topic = ClassUtils.getCanonicalName(eventType);
  synchronized (NotifyCenter.class) {
      // 通过publisherFactory反射创建Publisher
      // 放入publisherMap，key=topic，value=DefaultPublisher实例
    	MapUtils.computeIfAbsent(INSTANCE.publisherMap, topic, publisherFactory, eventType, queueMaxSize);
  }
  return INSTANCE.publisherMap.get(topic);
}

// 注册Subscriber逻辑
public static <T> void registerSubscriber(final Subscriber consumer) {
    // 1. 处理SmartSubscriber...

    // 2. 处理关注SlowEvent的Subscriber
    final Class<? extends Event> subscribeType = consumer.subscribeType();
    if (ClassUtils.isAssignableFrom(SlowEvent.class, subscribeType)) {
        INSTANCE.sharePublisher.addSubscriber(consumer, subscribeType);
        return;
    }
		// 3. 普通注册
    addSubscriber(consumer, subscribeType);
}
private static void addSubscriber(final Subscriber consumer, Class<? extends Event> subscribeType) {
  // 关注事件Class转换为String
  final String topic = ClassUtils.getCanonicalName(subscribeType);
  synchronized (NotifyCenter.class) {
    // 通过publisherFactory反射创建Publisher
    // 放入publisherMap，key=topic，value=DefaultPublisher实例
    MapUtils.computeIfAbsent(INSTANCE.publisherMap, topic, publisherFactory, subscribeType, ringBufferSize);
  }
  // 调用addSubscriber
  EventPublisher publisher = INSTANCE.publisherMap.get(topic);
  publisher.addSubscriber(consumer);
}

// DefaultPublisher
@Override
public void addSubscriber(Subscriber subscriber) {
  subscribers.add(subscriber);
}
```

Nacos中所有通过NotifyCenter发布的事件，都是放入Publisher里的阻塞队列，由Publisher线程异步执行。

```java
// NotifyCenter
public static boolean publishEvent(final Event event) {
    try {
        return publishEvent(event.getClass(), event);
    } catch (Throwable ex) {
        LOGGER.error("There was an exception to the message publishing : {}", ex);
        return false;
    }
}
private static boolean publishEvent(final Class<? extends Event> eventType, final Event event) {
    // 慢速事件，共享一个事件队列一个线程处理
    if (ClassUtils.isAssignableFrom(SlowEvent.class, eventType)) {
        return INSTANCE.sharePublisher.publish(event);
    }
    final String topic = ClassUtils.getCanonicalName(eventType);
    // 非慢速事件，一类事件一个事件队列一个线程处理
    EventPublisher publisher = INSTANCE.publisherMap.get(topic);
    if (publisher != null) {
        return publisher.publish(event);
    }
    return false;
}
```

对于**DefaultPublisher来说，优先放入阻塞队列，如果队列满了，会由当前线程处理事件**。

```java
// DefaultPublisher
public boolean publish(Event event) {
    // 1. 优先放入阻塞队列
    boolean success = this.queue.offer(event);
    // 2. 放入失败，则由当前线程执行
    if (!success) {
        receiveEvent(event);
        return true;
    }
    return true;
}
```



DefaultPublisher继承Thread，持续从阻塞队列中获取Event处理。

```java
@Override
public void run() {
    openEventHandler();
}

void openEventHandler() {
    try {
        // ...
        for (; ; ) {
            if (shutdown) {
                break;
            }
            // 阻塞等待事件发生
            final Event event = queue.take();
            // 处理事件
            receiveEvent(event);
            UPDATER.compareAndSet(this, lastEventSequence, Math.max(lastEventSequence, event.sequence()));
        }
    } catch (Throwable ex) {
        LOGGER.error("Event listener exception : {}", ex);
    }
}
```



DefaultPublisher的receiveEvent方法，调用Subscriber的onEvent方法处理事件。

```java
// DefaultPublisher
void receiveEvent(Event event) {
    final long currentEventSequence = event.sequence();
    for (Subscriber subscriber : subscribers) {
        // 如果订阅者忽略过期消息，直接跳过
        if (subscriber.ignoreExpireEvent() && lastEventSequence > currentEventSequence) {
            continue;
        }
				// 通知订阅者
        notifySubscriber(subscriber, event);
    }
}

// 根据Subscriber是否定义了自己的执行器，同步或异步执行onEvent方法
@Override
public void notifySubscriber(final Subscriber subscriber, final Event event) {
    final Runnable job = new Runnable() {
        @Override
        public void run() {
            subscriber.onEvent(event);
        }
    };
    final Executor executor = subscriber.executor();
    if (executor != null) {
        executor.execute(job);
    } else {
        job.run();
    }
}
```



回到**POST /v1/cs/configs方法，当更新完数据库配置后，会发布ConfigDataChangeEvent事件**，发布完成后因为事件是异步处理，这里就直接返回客户端了。

```java
// ConfigController
@PostMapping
public Boolean publishConfig(...) throws NacosException {
    if (StringUtils.isBlank(betaIps)) {
        if (StringUtils.isBlank(tag)) {
            // 更新数据库配置
            persistService.insertOrUpdate(srcIp, srcUser, configInfo, time, configAdvanceInfo, true);
            // 发布ConfigDataChangeEvent事件
            ConfigChangePublisher
                    .notifyConfigChange(new ConfigDataChangeEvent(false, dataId, group, tenant, time.getTime()));
        }
    }
}
```



## 3、本地配置更新

**ConfigDataChangeEvent的订阅者是AsyncNotifyService**，AsyncNotifyService收到事件以后，会查询nacos集群里所有的ip（包括当前实例），组装成AsyncTask，提交到另一个线程池处理。

```java
@Service
public class AsyncNotifyService {
    
    @Autowired
    public AsyncNotifyService(ServerMemberManager memberManager) {
        this.memberManager = memberManager;
        NotifyCenter.registerToPublisher(ConfigDataChangeEvent.class, NotifyCenter.ringBufferSize);
        NotifyCenter.registerSubscriber(new Subscriber() {
            @Override
            public void onEvent(Event event) {
                if (event instanceof ConfigDataChangeEvent) {
                    ConfigDataChangeEvent evt = (ConfigDataChangeEvent) event;
                    long dumpTs = evt.lastModifiedTs;
                    String dataId = evt.dataId;
                    String group = evt.group;
                    String tenant = evt.tenant;
                    String tag = evt.tag;
                    // 获取nacos集群中所有节点(包括自己)
                    Collection<Member> ipList = memberManager.allMembers();
                    // 每个节点组成一个Task
                    Queue<NotifySingleTask> queue = new LinkedList<NotifySingleTask>();
                    for (Member member : ipList) {
                        queue.add(new NotifySingleTask(dataId, group, tenant, tag, dumpTs, member.getAddress(),
                                evt.isBeta));
                    }
                    // 提交AsyncTask到其他线程执行
                    ConfigExecutor.executeAsyncNotify(new AsyncTask(nacosAsyncRestTemplate, queue));
                }
            }
            
            @Override
            public Class<? extends Event> subscribeType() {
                return ConfigDataChangeEvent.class;
            }
        });
    }
}
```



publisher线程只是将Event转换为AsyncTask，实际后续处理在一个async_notify线程中执行。这里调用了**所有nacos节点的/v1/cs/communication/dataChange**接口。

```java
class AsyncTask implements Runnable {
		// 一个任务对应一个nacos集群中的节点
    private Queue<NotifySingleTask> queue;
    private NacosAsyncRestTemplate restTemplate;
    @Override
    public void run() {
        executeAsyncInvoke();
    }
    private void executeAsyncInvoke() {
        // 循环所有nacos集群中的节点
        while (!queue.isEmpty()) {
            NotifySingleTask task = queue.poll();
            String targetIp = task.getTargetIP();
            if (memberManager.hasMember(targetIp)) {
                boolean unHealthNeedDelay = memberManager.isUnHealth(targetIp);
                if (unHealthNeedDelay) {
                    // 如果目标nacos实例非健康状态，提交一个延迟任务发起请求
                    asyncTaskExecute(task);
                } else {
                    // 请求/v1/cs/communication/dataChange?dataId=cfg0&group=DEFAULT_GROUP
                    Header header = Header.newInstance();
                    header.addParam(NotifyService.NOTIFY_HEADER_LAST_MODIFIED, String.valueOf(task.getLastModified()));
                    header.addParam(NotifyService.NOTIFY_HEADER_OP_HANDLE_IP, InetUtils.getSelfIP());
                    if (task.isBeta) {
                        header.addParam("isBeta", "true");
                    }
                    AuthHeaderUtil.addIdentityToHeader(header);
                    restTemplate.get(task.url, header, Query.EMPTY, String.class, new AsyncNotifyCallBack(task));
                }
            }
        }
    }
}
```

CommunicationController#notifyConfigInfo

/v1/cs/communication/dataChange为了快速响应发起数据更新的nacos节点，将本次数据更新通知封装为一个DumpTask，放入了NacosDelayTaskExecuteEngine的一个Map中，key是groupKey，value是DumpTask。



```java
// CommunicationController
@GetMapping("/dataChange")
public Boolean notifyConfigInfo(...) {
    // ...
    dumpService.dump(dataId, group, tenant, tag, lastModifiedTs, handleIp);
    return true;
}
// DumpService
public void dump(String dataId, String group, String tenant, String tag, long lastModified, String handleIp,
            boolean isBeta) {
  	String groupKey = GroupKey2.getKey(dataId, group, tenant);
  	dumpTaskMgr.addTask(groupKey, new DumpTask(groupKey, tag, lastModified, handleIp, isBeta));
}

// NacosDelayTaskExecuteEngine（TaskManager父类）
protected final ConcurrentHashMap<Object, AbstractDelayTask> tasks;
public void addTask(Object key, AbstractDelayTask newTask) {
   // ...
   // key = groupKey, Task = DumpTask
   tasks.put(key, newTask);
}
```

NacosDelayTaskExecuteEngine内部通过一个线程，执行ProcessRunnable，处理所有提交的Task，其中就包括了DumpTask。

```java
public NacosDelayTaskExecuteEngine(String name, int initCapacity, Logger logger, long processInterval) {
    processingExecutor = ExecutorFactory.newSingleScheduledExecutorService(new NameThreadFactory(name));
    processingExecutor
            .scheduleWithFixedDelay(new ProcessRunnable(), processInterval, processInterval, TimeUnit.MILLISECONDS);
}

private class ProcessRunnable implements Runnable {

  @Override
  public void run() {
    try {
      // ProcessRunnable处理所有任务
      processTasks();
    } catch (Throwable e) {
      getEngineLog().error(e.toString(), e);
    }
  }
}

// ProcessRunnable处理所有任务
protected void processTasks() {
  Collection<Object> keys = getAllTaskKeys();
  for (Object taskKey : keys) {
    // 获取groupKey对应的DumpTask
    AbstractDelayTask task = removeTask(taskKey);
    if (null == task) {
      continue;
    }
    // 通过groupKey找到对应的task处理器（map管理）
    NacosTaskProcessor processor = getProcessor(taskKey);
    if (null == processor) {
      continue;
    }
    try {
      // 处理器处理Task
      if (!processor.process(task)) {
        //如果处理失败（比如为了更新本地文件系统，获取写锁失败），把任务重新添加到队列中（重试）
        retryFailedTask(taskKey, task);
      }
    } catch (Throwable e) {
      retryFailedTask(taskKey, task);
    }
  }
}
```



**DumpProcessor处理DumpTask任务，这里查询了数据库里最新的配置**，构建了ConfigDumpEvent。

```java
public class DumpProcessor implements NacosTaskProcessor {
    final DumpService dumpService;
    @Override
    public boolean process(NacosTask task) {
        final PersistService persistService = dumpService.getPersistService();
        DumpTask dumpTask = (DumpTask) task;
        // ...
        ConfigDumpEvent.ConfigDumpEventBuilder build = ConfigDumpEvent.builder().namespaceId(tenant).dataId(dataId);
        if (isBeta) {
            // ...
        } else {
            if (StringUtils.isBlank(tag)) {
                // 1. 查询当前配置
                ConfigInfo cf = persistService.findConfigInfo(dataId, group, tenant);
                // 2. 设置ConfigDumpEvent的content为最新的content
                build.remove(Objects.isNull(cf));
                build.content(Objects.isNull(cf) ? null : cf.getContent());
                build.type(Objects.isNull(cf) ? null : cf.getType());
                // 3. 执行ConfigDumpEvent处理
                return DumpConfigHandler.configDump(build.build());
            } else {
              //...
            }
        }
    }
}
```



DumpConfigHandler#configDump

```java
public static boolean configDump(ConfigDumpEvent event) {
   result = ConfigCacheService.dump(dataId, group, namespaceId, content, lastModified, type);
}
```

继续深入，进入到**ConfigCacheService，这里将新配置写入文件系统，并更新了内存中的配置md5，发布LocalDataChangeEvent事件**。至此，nacos集群中所有节点的本地文件和内存配置都应该和之前写入的时候一致了。此时最新数据才被客户端能看见。**通过写锁可以阻塞配置查询**（配置监听发生变更，客户端也会反查），这一步保证客户端在/v1/cs/configs接口读配置和感知到配置更新是顺序发生的。

ConfigCacheService#dump

```java
public static boolean dump(String dataId, String group, String tenant, String content, long lastModifiedTs,
        String type) {

    /**
     * 将新配置写入文件系统，并更新了内存中的配置md5，发布LocalDataChangeEvent事件
     */

    String groupKey = GroupKey2.getKey(dataId, group, tenant);
    CacheItem ci = makeSure(groupKey);
    ci.setType(type);
    // 获取groupKey对应写锁
    final int lockResult = tryWriteLock(groupKey);
    assert (lockResult != 0);
    
		//如果获取写锁失败，会把当前任务再次添加到任务队列中（重试）
    if (lockResult < 0) {
        DUMP_LOG.warn("[dump-error] write lock failed. {}", groupKey);
        return false;
    }

    try {
        // 计算新的配置的md5
        final String md5 = MD5Utils.md5Hex(content, Constants.ENCODE);
        // 比较新md5与内存中配置的md5是否一致
        if (md5.equals(ConfigCacheService.getContentMd5(groupKey))) {
            DUMP_LOG.warn("[dump-ignore] ignore to save cache file. groupKey={}, md5={}, lastModifiedOld={}, "
                            + "lastModifiedNew={}", groupKey, md5, ConfigCacheService.getLastModifiedTs(groupKey),
                    lastModifiedTs);

        } else if (!PropertyUtil.isDirectRead()) {//不是单机或不是嵌入式数据源
            // 如果使用mysql数据库，这里会保存配置文件到磁盘上，供之后读取
            DiskUtil.saveToDisk(dataId, group, tenant, content);
        }
        /**
         *   更新内存中配置的md5，发布LocalDataChangeEvent
         *   LongPollingService监听LocalDataChangeEvent事件
         */
        updateMd5(groupKey, md5, lastModifiedTs);
        return true;
    } catch (IOException ioe) {
        DUMP_LOG.error("[dump-exception] save disk error. " + groupKey + ", " + ioe.toString(), ioe);
        if (ioe.getMessage() != null) {
            String errMsg = ioe.getMessage();
            if (NO_SPACE_CN.equals(errMsg) || NO_SPACE_EN.equals(errMsg) || errMsg.contains(DISK_QUATA_CN) || errMsg
                    .contains(DISK_QUATA_EN)) {
                // Protect from disk full.
                FATAL_LOG.error("磁盘满自杀退出", ioe);
                System.exit(0);
            }
        }
        return false;
    } finally {
        releaseWriteLock(groupKey);
    }
}
public static void updateMd5(String groupKey, String md5, long lastModifiedTs) {
    CacheItem cache = makeSure(groupKey);
    if (cache.md5 == null || !cache.md5.equals(md5)) {
      	cache.md5 = md5;
      	cache.lastModifiedTs = lastModifiedTs;
      	NotifyCenter.publishEvent(new LocalDataChangeEvent(groupKey));
    }
}
```

## 4、响应长轮询

最后处理LocalDataChangeEvent，响应正在对当前nacos节点发起长轮询的客户端，推送发生配置变更的groupKey。

**服务端LongPollingService构造时，注册了LocalDataChangeEvent的发布和订阅**。当LocalDataChangeEvent发生时，且非固定长轮询，订阅者会提交一个**DataChangeTask**任务到另一个线程池中。

```java
public LongPollingService() {
    allSubs = new ConcurrentLinkedQueue<ClientLongPolling>();

    ConfigExecutor.scheduleLongPolling(new StatTask(), 0L, 10L, TimeUnit.SECONDS);

    // 注册LocalDataChangeEvent发布者
    // Register LocalDataChangeEvent to NotifyCenter.
    NotifyCenter.registerToPublisher(LocalDataChangeEvent.class, NotifyCenter.ringBufferSize);

    // Register A Subscriber to subscribe LocalDataChangeEvent.
    // 注册LocalDataChangeEvent订阅者
    NotifyCenter.registerSubscriber(new Subscriber() {

        @Override
        public void onEvent(Event event) {
            if (isFixedPolling()) {
                // Ignore.
            } else {
                if (event instanceof LocalDataChangeEvent) {
                    LocalDataChangeEvent evt = (LocalDataChangeEvent) event;
                    ConfigExecutor.executeLongPolling(new DataChangeTask(evt.groupKey, evt.isBeta, evt.betaIps));
                }
            }
        }

        @Override
        public Class<? extends Event> subscribeType() {
            return LocalDataChangeEvent.class;
        }
    });

}
```

DataChangeTask**比较内存中的配置md5和LocalDataChangeEvent中配置的md5**，响应所有订阅这个groupKey配置的客户端。基于单个配置的事件传播，最终响应客户端时，只会告诉客户端一个groupKey的变更。

```java

    /**
     * ClientLongPolling subscibers.
     * // 向当前nacos节点发起长轮询请求的客户端
     */
    final Queue<ClientLongPolling> allSubs;

    class DataChangeTask implements Runnable {

      /**
         * 循环所有对当前nacos服务端发起长轮询请求的客户端,比较内存中的配置md5和LocalDataChangeEvent中配置的md5，响应所有订阅这个groupKey配置的客户端。
         * 基于单个配置的事件传播，最终响应客户端时，只会告诉客户端一个groupKey的变更
         */
      @Override
      public void run() {
        try {
          // 1. 获取缓存中groupKey对应的配置MD5
          ConfigCacheService.getContentBetaMd5(groupKey);
          // 2. 循环所有对当前nacos服务端发起长轮询请求的客户端
          for (Iterator<ClientLongPolling> iter = allSubs.iterator(); iter.hasNext(); ) {
            ClientLongPolling clientSub = iter.next();
            // 3. 只处理对当前groupKey有订阅的客户端
            if (clientSub.clientMd5Map.containsKey(groupKey)) {

              // ...betaIps和tag的过滤逻辑
              
              if (isBeta && !CollectionUtils.contains(betaIps, clientSub.ip)) {
                continue;
              }

              // If published tag is not in the tag list, then it skipped.
              if (StringUtils.isNotBlank(tag) && !tag.equals(clientSub.tag)) {
                continue;
              }

              getRetainIps().put(clientSub.ip, System.currentTimeMillis());
              // 4. 移除当前ClientLongPolling
              iter.remove(); 
              LogUtil.CLIENT_LOG
                .info("{}|{}|{}|{}|{}|{}|{}", (System.currentTimeMillis() - changeTime), "in-advance",
                      RequestUtil
                      .getRemoteIp((HttpServletRequest) clientSub.asyncContext.getRequest()),
                      "polling", clientSub.clientMd5Map.size(), clientSub.probeRequestSize, groupKey);

              // 5. 响应客户端
              clientSub.sendResponse(Arrays.asList(groupKey));
            }
          }
        } catch (Throwable t) {
          LogUtil.DEFAULT_LOG.error("data change error: {}", ExceptionUtil.getStackTrace(t));
        }
    }
}
```



## 5、普通长轮询与固定长轮询

在/v1/cs/configs/listener最后一步，提交了ClientLongPolling到长轮询线程池执行。

```java
public void addLongPollingClient(HttpServletRequest req, HttpServletResponse rsp, Map<String, String> clientMd5Map,
        int probeRequestSize) {
    // ...
    // 开启AsyncContext
    final AsyncContext asyncContext = req.startAsync();
    // 提交长轮询任务到其他线程
    ConfigExecutor.executeLongPolling(
            new ClientLongPolling(asyncContext, clientMd5Map, ip, probeRequestSize, timeout, appName, tag));
}
```



这个ClientLongPolling的run方法只做了两个事情：

- 提交一个超时检测任务
- 将自己加入订阅队列

```java
class ClientLongPolling implements Runnable {
    @Override
    public void run() {
        // 1. 提交超时处理任务
        asyncTimeoutFuture = ConfigExecutor.scheduleLongPolling(new Runnable() {
            @Override
            public void run() {
								// ...
            }

        }, timeoutTime, TimeUnit.MILLISECONDS);
        // 2. 将自己加入Queue<ClientLongPolling> allSubs
        allSubs.add(this);
    }
}
```



看看这个超时检测任务做了什么。

```java
public void run() {
  //1、提交一个超时检测任务  timeoutTime时间后执行 对普通长轮询做超时处理，默认30s后如果无配置变更，响应客户端空数据；
  asyncTimeoutFuture = ConfigExecutor.scheduleLongPolling(new Runnable() {
    @Override
    public void run() {
      try {
        
        // 将自己移出订阅队列
        allSubs.remove(ClientLongPolling.this);
        // 1. 固定长轮询处理，比较内存中md5与客户端发起请求的md5是否一致，并响应客户端
        if (isFixedPolling()) {
          /**
           * 服务端开启了isFixedPolling=true，默认10s执行一次这个任务，比较内存中配置md5与客户端请求配置md5是否一致，
           * 再响应客户端，而不是通过DataChangeTask等待服务端配置发生变化后主动通知客户端。
           */
         
          List<String> changedGroups = MD5Util
            .compareMd5((HttpServletRequest) asyncContext.getRequest(),
                        (HttpServletResponse) asyncContext.getResponse(), clientMd5Map);
          if (changedGroups.size() > 0) {
            sendResponse(changedGroups);
          } else {

            sendResponse(null);
          }
        } else {
          // 2. 普通长轮询（默认），代表超时没有配置变更，响应客户端空数据
          
          sendResponse(null);
        }
      } catch (Throwable t) {
        LogUtil.DEFAULT_LOG.error("long polling error:" + t.getMessage(), t.getCause());
      }

    }

  }, timeoutTime, TimeUnit.MILLISECONDS);

  // 2. 将自己加入Queue<ClientLongPolling> allSubs
  allSubs.add(this);
}
```

超时检测任务：

- 一方面，对**普通长轮询做超时处理**，默认30s后如果无配置变更，响应客户端空数据；
- 另一方面，这里体现了**固定长轮询**的作用，**当服务端开启了isFixedPolling=true，默认10s执行一次这个任务，比较内存中配置md5与客户端请求配置md5是否一致，再响应客户端，而不是通过DataChangeTask等待服务端配置发生变化后主动通知客户端**。



# 总结

本章分析了Nacos配置中心服务端的几个核心功能，服务端使用MySQL数据源（ -DembeddedStorage=false），以**集群**方式启动（-Dnacos.standalone=false）。

- GET /v1/cs/configs

  服务端查询配置逻辑

  - 需要先获取配置项的读锁，根据获取成功与否返回不同状态码
  - 获取读锁成功后，查询配置。如果单机部署且使用默认derby数据源，将实时查询配置返回；如果集群部署或使用mysql数据源，将查询本地文件系统中的缓存配置返回。

- POST /v1/cs/configs/listener

  接口负责配置监听

  - 先校验客户端配置项md5与服务端内存中缓存的md5是否一致，不一致直接拼接结果返回，这里返回的是发生变化的groupKey，由客户端发起二次请求/v1/cs/configs查询最新配置。
  - 如果客户端配置项md5已经最新，请求头中包含Long-Pulling-Timeout-No-Hangup=true，则立即返回200。
  - 如果不满足上述两点，开启AsyncContext，并提交一个ClientLongPolling任务等待配置发生变更后，响应客户端。

- POST /v1/cs/configs

  负责发布配置，基于非集群derby启动的流程如下

  - 更新数据库
  - 集群中所有服务端异步更新本地配置，包括文件系统中存储的配置文件和内存中的md5，此时，本次更新才对客户端可见。**需要先获取配置项的写锁**，保证客户端在/v1/cs/configs接口读配置和感知到配置更新是顺序发生的
  - 响应客户端长轮询















