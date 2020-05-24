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

![](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590071437422.png?raw=true)

接着打开 ZookeeperRegistry.this.recover();

![1590072129407](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590072129407.png?raw=true)

查看 结构为：

destroy

doRegister 注册临时节点 doUnregister  删除节点

doSubscribe  如下：

![1590074154433](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590074154433.png?raw=true)



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

打开 dubbo-registry-redis 模块只有RedisRegistryFactory 里面依旧是RedisRegistry 打开 RedisRegistry 

其构造前面的都是一些参数设置

重点是：

![1590075886174](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590075886174.png?raw=true)

注意：服务主动下线会有 unregister 广播，但是服务宕机，那么没有unregister ，服务不会完成下线，其依赖于服务治理中心定时调度，去清理超时未延期的key，并发布 unregister 广播，从而保证一致性

![1590075895779](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590075895779.png?raw=true)

查看RedisRegistry  结构

destroy

doRegister ： hset(key, value, expire)      publish(key, register)

doUnregister :  hdel(key, value)   publish(key, unregister)

doSubscribe  如下：

![1590077149225](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590077149225.png?raw=true)

拉取为：

![1590077169416](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590077169416.png?raw=true)

关于 AbstractRegistry 、FailbackRegistry 其实就是增加了 缓存 和 重试

![1590115203670](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590115203670.png?raw=true)

**所有注册中心 都是通过对应的工厂实现，即 AbstractRegistryFactory （父类 RegistryFactory）**

在 RegistryFactory 接口 暴露出来

## 3.扩展加载机制

### 1.加载机制

dubbo spi跟java spi区别：

1. java spi 一次性实例化扩展点的全部实现很耗时，不能按需加载

2. java spi 加载失败会将异常吃掉，不会报错

3. dubbo 增加了扩展的IOC（setter）和AOP（增加包装类）

同时： 增加了缓存 class缓存（class字节码）、 实例缓存（实例）等

​           增加了普通扩展类、包装扩展类（Wrapper）、自适应扩展类（Adaptive）

核心类 就是 ExtensionLoader 

**扩展的特点：**

自动包装   Wrapper

自动加载   setter

自适应  @Adaptive

自动激活 @Activate 

### 2.扩展注解

@SPI  标记为扩展点

@Adaptive  一般写在方法上，通过动态获取实现类，现在实现类上 就是默认实现 （只能有一个，多个会出异常）

@Adaptive可以穿多个key 根据url?后面的key做匹配，没有的话默认用SPI上的key 寻找实现

@Activate  根据传入的条件 匹配激活 group value before after（前后顺序）order （排序）

### 3.ExtensionLoader 原理

1. getExtension  获取普通扩展类

    详情：

   ![1590137922811](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590137922811.png?raw=true)

   其实方法  createExtension 具体实现：

   ![1590140823943](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590140823943.png?raw=true)

   最终返回普通包装 实例

2. getAdaptiveExtension 获取自适应扩展类

   具体实现：

   ![1590142190201](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590142190201.png?raw=true)

   方法createAdaptiveExtension实现：

   ![1590142232043](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590142232043.png?raw=true)

   其中代码生成为：

   ![1590142289865](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590142289865.png?raw=true)

   

3. getActivateExtension  获取激活扩展类

   其实现大概是通过url 过滤要激活的类 

ExtensionFactory 工厂生产ExtensionLoader  

ExtensionFactory 的实现有：

![1590144711903](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590144711903.png?raw=true)

### 4.扩展点动态编译实现

​       上面我们看到其生成的class 就是字符串，生成类还有一段路，虽然我们可以通过反射生成，但是在性能上与直接编译好的class 还是有一定差距，dubbo怎么解决的呢？动态编译器，dubbo中的设计：

![1590327031956](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590327031956.png?raw=true)

AdaptiveCompiler 类中有方法setDefaultCompiler 设置默认编译实现，调用设置的编译实现编译，调用的地方是ApplicationConfig#setCompiler 也就是读取 标签中 compiler的值，在这里AdaptiveCompiler  类似于之前的工厂

AbstractCompiler 类 一些通用的功能，url拼接，Class.forName防止重复编译

**JavassistCompiler**的实现：

![1590327983604](https://github.com/RyzeUserName/dubbo/blob/master/assets/1590327983604.png?raw=true)

正如上标识一样，举个小例子：

```java
public static void main(String[] args) throws CannotCompileException, IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException {
        ClassPool pool = ClassPool.getDefault();
        CtClass helloWorld = pool.makeClass("HelloWorld");
        CtMethod make = CtNewMethod.make(" public static void hello(){\n" +
            "        System.out.println(\"hello world\");\n" +
            "    }", helloWorld);
        helloWorld.addMethod(make);
        Class aClass = helloWorld.toClass();
        Object instance = aClass.newInstance();
        Method hello = aClass.getDeclaredMethod("hello", null);
        hello.invoke(instance,null);
    }
```

控制台打出来hello world

**JdkCompiler**的实现：

jdk的动态编译需要实现：JavaFileObject 接口、ForwardingJavaFileManager接口、JavaCompiler.CompilationTask 方法

1. JavaFileObject 

   字符串会被包装成文件对象，输入输出都是ByteArray流，

2. JavaFileManager

   管理文件的读取和输出位置，jdk中不提供默认的实现

3. JavaCompiler.CompilationTask

   该方法将JavaFileObject 对象编译成具体的类


## 4.dubbo启停原理

 ### 1.dubbo配置解析

#### 1.xml

1.schema设计

详细内容 http://dubbo.apache.org/zh-cn/docs/user/references/xml/introduction.html

2.解析

逻辑入口在 DubboNamespaceHandler #init 方法中主要是讲逻辑委托给了 DubboBeanDefinitionParser类

实现了BeanDefinitionParser 接口，主要就是parse 的实现

![1590334702342](E:\study\dubbo\assets\1590334702342.png)

负责把标签解析成对应的bean注册到spring上下文,service 标签处理：

![1590334986021](E:\study\dubbo\assets\1590334986021.png)

标签属性的解析：

![1590336123959](E:\study\dubbo\assets\1590336123959.png)

具体的值解析：

![1590336171323](E:\study\dubbo\assets\1590336171323.png)

将匹配不到的值存在一个map里

![1590336204376](E:\study\dubbo\assets\1590336204376.png)

#### 2.注解

 ### 2.dubbo服务暴露原理



 ### 3.dubbo服务消费原理



 ### 4.dubbo优雅停机解析









