​																								spring事件监听器

Spring事件机制涉及的重要的类主要有以下四个：

- ApplicationEvent：事件，该抽象类是所有Spring事件的父类，可携带数据比如事件发生时间timestamp。

- ApplicationListener：事件监听器，该接口被所有的事件监听器实现，基于标准的java的EventListener接口实现观察者模式。

- ApplicationEventMulticaster：事件管理者，管理监听器和发布事件，ApplicationContext通过委托ApplicationEventMulticaster来 发布事件

- ApplicationEventPublisher：事件发布者，该接口封装了事件有关的公共方法，作为ApplicationContext的超级概括，也是委托 ApplicationEventMulticaster完成事件发布。