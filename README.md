# dubbo 运行 源码分析

官方文档在http://dubbo.apache.org/zh-cn/docs/user/quick-start.html

## 1.运行

https://github.com/apache/dubbo  项目源码

download 下来 idea 导入 mvn clean install  -Dmaven.test.skip=true

下载 zookeeper 地址 https://zookeeper.apache.org/ 

解压运行bin目录下的 zkServer.cmd 或者 zkServer.sh

打开项目的 dubbo-demo（例子） 模块可以看到

dubbo-demo-annotation    注解模式实现的例子

dubbo-demo-api        api模式实现的例子

dubbo-demo-xml      xml模式实现的例子

dubbo-demo-interface     服务接口

每个模块都分provider 和 consumer 子模块，分别启动其对应的 Application 可看到对应的服务调用

## 2.注册中心

dubbo-registry 模块

服务注册，服务发现，动态调整，统一配置

打开这个模块看到常见的注册中心 zookeepe、redis、nacos、eureka、consul、etcd3、sofa还有default（内存），api（包含注册中心的所有api和抽象实现类），multicast（广播），multiple（多注册表）

### 1.zookeeper

![](http://dubbo.apache.org/docs/zh-cn/user/sources/images/zookeeper.jpg)

- 服务提供者启动时: 向 `/dubbo/com.foo.BarService/providers` 目录下写入自己的 URL 地址
- 服务消费者启动时: 订阅 `/dubbo/com.foo.BarService/providers` 目录下的提供者 URL 地址。并向 `/dubbo/com.foo.BarService/consumers` 目录下写入自己的 URL 地址
- 监控中心启动时: 订阅 `/dubbo/com.foo.BarService` 目录下的所有提供者和消费者 URL 地址。

**注意:在2.7.x的版本中已经移除了zkclient的实现（目前curator ）,如果要使用zkclient客户端,需要自行拓展**

**订阅与发布：**

有pull和push 两种方式，dubbo是启动主动去pull，之后接收事件通知之后去pull，zk的实现就是注册监听

打开dubbo-registry-zookeeper 看到 ZookeeperRegistryFactory打开 注释 zk的注册工厂

方法 createRegistry 注册 打开是 new ZookeeperRegistry(url, zookeeperTransporter);

点开 ZookeeperRegistry

![1590071437422](E:\study\dubbo\assets\1590071437422.png)

### 2.redis

![](http://dubbo.apache.org/docs/zh-cn/user/sources/images/dubbo-redis-registry.jpg)

使用 Redis 的 Key/Map 结构存储数据结构：

- 主 Key 为服务名和类型
- Map 中的 Key 为 URL 地址
- Map 中的 Value 为过期时间，用于判断脏数据，脏数据由监控中心删除 [[3\]](http://dubbo.apache.org/zh-cn/docs/user/references/registry/redis.html#fn3)

使用 Redis 的 Publish/Subscribe 事件通知数据变更：

- 通过事件的值区分事件类型：`register`, `unregister`, `subscribe`, `unsubscribe`
- 普通消费者直接订阅指定服务提供者的 Key，只会收到指定服务的 `register`, `unregister` 事件
- 监控中心通过 `psubscribe` 功能订阅 `/dubbo/*`，会收到所有服务的所有变更事件

调用过程：

1. 服务提供方启动时，向 `Key:/dubbo/com.foo.BarService/providers` 下，添加当前提供者的地址

2. 并向 `Channel:/dubbo/com.foo.BarService/providers` 发送 `register` 事件

3. 服务消费方启动时，从 `Channel:/dubbo/com.foo.BarService/providers` 订阅 `register` 和 `unregister`事件

4. 并向 `Key:/dubbo/com.foo.BarService/consumers` 下，添加当前消费者的地址

5. 服务消费方收到 `register` 和 `unregister` 事件后，从 `Key:/dubbo/com.foo.BarService/providers` 下获取提供者地址列表

6. 服务监控中心启动时，从 `Channel:/dubbo/*` 订阅 `register` 和 `unregister`，以及 `subscribe` 和`unsubsribe`事件

7. 服务监控中心收到 `register` 和 `unregister` 事件后，从 `Key:/dubbo/com.foo.BarService/providers` 下获取提供者地址列表

8. 服务监控中心收到 `subscribe` 和 `unsubsribe` 事件后，从 `Key:/dubbo/com.foo.BarService/consumers`下获取消费者地址列表

   **注意：默认实现是jedis**