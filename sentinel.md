



插槽链表顺序：

- `NodeSelectorSlot` 负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级；
- `ClusterBuilderSlot` 则用于存储资源的统计信息以及调用者信息，例如该资源的 RT, QPS, thread count 等等，这些信息将用作为多维度限流，降级的依据；
- `StatistcSlot` 则用于记录，统计不同维度的 runtime 信息；
- `FlowSlot` 则用于根据预设的限流规则，以及前面 slot 统计的状态，来进行限流；
- `AuthorizationSlot` 则根据黑白名单，来做黑白名单控制；
- `DegradeSlot` 则通过统计信息，以及预设的规则，来做熔断降级；
- `SystemSlot` 则通过系统的状态，例如 load1 等，来控制总的入口流量；





DefaultNode：保存着某个resource在某个context中的实时指标，每个DefaultNode都指向一个ClusterNode

ClusterNode：保存着某个resource在所有的context中实时指标的总和，同样的resource会共享同一个ClusterNode，不管他在哪个context中





```java
ArrayMetric -> LeapArray -> WindowWrap -> MetricBucket
```

![sentinel滑动窗口关系图](images/sentinel滑动窗口关系图.png)





![sentinel滑动窗口关系图2](images/sentinel滑动窗口关系图2.png)



















