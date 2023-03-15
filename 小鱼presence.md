```
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;


public class PresenceTest {

    class LocalCache{

        //从设备找到小鱼号
        Map<String,String> uriKeyRelationMap = new HashMap<>();
        //从小鱼号找到设备号
        Map<String, List<StringTest>> keyUrlRelationMap = new HashMap<>();

        /**
         *         ConcurrentHashMap<String,DeviceStatus> deviceStatuaMap = new ConcurrentHashMap<>();
         *
         *
         *         DeviceStatus是一个组合状态（因为设备状态可能是在线和在会：这并不矛盾）
         *         DeviceStatus{
         *             CallStatus 在会、离会
         *             PresenceStatus 在线、离线
         *         }
         *
         *redis:
         *
         * watcherList:这个设备被哪些设备关注
         * watcheeList:这个设备关注了哪些设备
         *
         *      提供rest接口供其他服务查询  jmeter 2核8G 并发1000请求 每个请求携带了1-30个不等的number
         *      （30这个数字是因为通信录每次查询一般是15-20，不会超过30）  通信录列表1分钟刷新一次
         *       使用pipeline(一次查询20个终端)  一个请求一个pipeline
         *
         *
         * LocalCache和Redis的key都是设备的uri，value类型是hash，存储DeviceStatus、watcherList、watcheeList（json格式）
         * 每个key、value几十字节
         *
         *
         * kafka:
         *  每台终端和access都有一个websocket长连接，所以access第一时间能感知到各个终端的状态（在线），发送到kafka
     *      Access会和终端保持心跳，access会将心跳发送到kafka
     *      presence会检查access的状态，acess每隔10秒向kafka写入推送消息
     *      信令知道每台终端的会议状态（入会\离会），presence是通过access和sig来获取终端的状态
         *
         *  消费者消费topic为presnece_publish的消息,先读取LocalCache比较状态是否发生变更，发送变更就直接写到LocalCache，再写到redis
         *
         *  三个topic：每个topic每500ms poll一次，每次500条消息，保持现有逻辑不变。
         *  1、presence_publish 两个presence是一个消费者组 四个分区（分区数是消费者的倍数） 手动提交
         *  每个终端的kafak消息会固定地由一台固定的presence消费 producer指定key为终端uri
         *  包括access、sig、access发送的heartbeat
         *
         *  2、access_keepalive 每个presence是一个消费者组 一个分区 每个presence都能消费到这条消息
         *  access每10秒向kafka推送消息
         *
         *  3、presence_dev_current_status 每个presence都是一个消费者组，这个topic的生产者就是其中一个presence服务
         *
         * presence:保存终端状态 多活）
         *
         * presence生产配置 4核16G包括redis和LocalCache
         *
         * 功能
         *      通过access和sig收集所有终端状态
         *      通过订阅供callcenter感知设备状态变化
         *      Presence提供一个rest api接口供其他服务查询终端状态
         *
         * 终端状态：在线、离线、入会、离会
         * 主要是通讯录会查询presence
         *
         * presence 多活
         * 两台presence上游前置一台nginx
         *
         */


        ConcurrentHashMap<String,String> deviceStatuaMap = new ConcurrentHashMap<>();


        /**
         * rest接口查询：先查LocalCache，内存里没有再查redis，并同步更新到LocalCache中，redis是通过pipeline查询
         * 最后返回结果   offline && incall => offline
         *              offline && offcall => offline
         *              online ** incall => incall
         *              其他 ==> idle
         */
    }
}
```









redis key

设备号 == 小鱼号 String

小鱼号 == 设备号 set 

设备号 == 状态  map  组合状态

access == 设备列表



kafka  topic

presence_publish

access_keepalive







