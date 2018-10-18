## MQCloud - [RocketMQ](https://github.com/apache/rocketmq)的企业级运维平台
**它具备以下特性：**

* 跨集群：可以同时管理多个集群，对使用者透明。
* 预警功能：针对生产或消费堆积，失败，异常等情况预警，处理。
* 简单明了：用户视图-对拓扑，流量，消费状况等指标进行直接展示；管理员视图-集群维护监控，流程审批等。
* 安全：用户隔离，操作审批，数据安全。
* 更多特性正在开发中，视频介绍：[传送门](https://tv.sohu.com/v/dXMvMTIyNTQ2MTgvMTA2NTI4OTI1LnNodG1s.html)。


----------

## 特性概览
* 用户topic列表-不用用户看到不同的topic，管理员可以管理所有topic

  ![用户topic列表]()

* topic详情-分三块 基本信息，今日流程，拓扑

  ![topic详情]()

* 生产详情

  ![生产详情]()

* 消费详情

  ![消费详情]()

* 消息

  ![消息]()

* 运维后台

  ![admin]()

* 创建broker

  ![addBroker]()


----------

## 模块介绍及编译
1. 模块介绍

   1. mq-client-common-open与mq-client-open为客户端模块，它封装了rocketmq-client，客户端需要依赖它才能和MQCloud进行交互。
   2. mq-cloud-common与mq-cloud为MQCloud的web端，实现管理监控等一系列功能。

2. 编译

   1. mq-client-common-open与mq-client-open最低依赖jdk1.7。
   2. mq-cloud依赖jdk1.8，其采用spring-boot实现。
   3. 编译：
      1. 在sohu-tv-mq/下，执行mvn clean install -pl "!mq-cloud"，将编译并install mq-client-common-open，mq-client-open，mq-cloud-common模块
      2. 在sohu-tv-mq/mq-cloud/下，执行mvn clean package，将编译并打包mq-cloud-online.war


----------

## 初始化配置及运行
1. 数据库配置

   数据库使用mysql，不同的环境需要修改不同的配置文件，默认为local环境，默认配置参见`application-local.yml`：

   ```
   url: jdbc:mysql://127.0.0.1:3306/mq-cloud?useUnicode=true&characterEncoding=UTF-8
   username: mqcloud
   password: mqcloud

   ```

   线上数据库配置在`application-online.yml` 中。

2. 初始化数据

   初始化sql脚本，参见`mq-cloud/sql/init.sql`

3. 运行

   1. 开发工具内运行：

      经过编译后，模块的依赖会解析成功，可以直接执行`com.sohu.tv.mq.cloud.Application` 中的main函数即可。

   2. 外部运行：

      使用下载或编译的war包，执行`java -Dspring.profiles.active=online -server -Xmx8g -Xms8g -Xss256k -XX:+UseG1GC -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m -XX:MaxGCPauseMillis=100 -DPROJECT_DIR=部署路径/logs/gc.log -XX:+PrintGCDateStamps -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=部署路径/logs/java.hprof -XX:+DisableExplicitGC -XX:+PrintCommandLineFlags-Dfile.encoding=UTF-8 -jar 部署路径/mq-cloud-online.war`

4. 访问

   直接访问[127.0.0.1:8080](http://127.0.0.1:8080)即可，默认管理员用户名：admin@admin.com 密码为：admin

----------

## 集群创建

1. 添加机器

   在`机器管理`模块，使用`+机器`功能将要部署name server和broker的linux机器都添加进来。

2. 创建集群记录

   1. 在`集群管理`模块，使用`+集群` 功能创建集群。
   2. 各项含义请参考提示，创建成功会在`cluster`表多一条记录，这里需要记住`集群id`，会在nginx配置中使用。

3. nginx配置

   RocketMQ官方推荐采用域名方式发现name server，这样可以保证无缝升级或迁移name server以及broker。这里采用nginx，配置参考如下设置：

   ```
   server {
        listen       80;
        server_name  10.10.x.x;
        location / {
             root download;
        }
        location /rocketmq/nsaddr-1 {
             default_type text/html;
             return 200 '1.1.1.1:9876;2.2.2.2:9876';
        }
   }
   ```

   释义：

   1. `location /`作用：用于提供rocketmq.zip和nmon.zip的下载路径，供MQCloud部署RocketMQ集群和监控服务器状况。具体配置：需要将rocketmq.zip和nmon.zip放置到`${nginx_install_dir}/download/software/mqcloud`下。

   2. `location /rocketmq/nsaddr-1`用于供RocketMQ client&broker 发现name server集群使用。

      其中`1`为`创建集群`中的`集群id`。多个集群可以配置多个location。

   3. `return 200 '1.1.1.1:9876;2.2.2.2:9876';`中的地址为name server实例地址。

4. 修改通用配置

   1. 在`通用配置`模块，将`softDomain`选项修改为`nginx配置`中的nginx地址(或域名)。

      MQCloud将使用`${softDomain}/download/software/mqcloud/nmon.zip`和`${softDomain}/download/software/mqcloud/rocketmq.zip`来下载。

   2. 将`nameServerDomain`选项修改为`nginx配置`中的nginx地址(或域名)+端口，一定要带端口(原因请参考admin的`快速指南->集群管理->集群发现`中的介绍)，即使端口是80。RocketMQ client和broker及MQCloud将使用`${nameServerDomain}/rocketmq/nsaddr-${cluster_id}`来发现name server集群。

5. 创建name server实例

   1. 在`集群管理`模块，使用`+NameServer` 功能创建name server实例。

6. 创建broker实例

   1. 在`集群管理`模块，使用`+Master` 功能创建broker实例。

7. 已有集群如何使用MQCloud管理？

   1. 在`机器管理`模块，使用`+机器`功能将目前部署name server和broker的linux机器都添加进来。
   2. 执行上面步骤中的`2. 创建集群记录`。
   3. 执行上面步骤中的`3. nginx配置`。
   4. 执行上面步骤中的`4. 修改通用配置`。
   5. 导入topic：使用`集群管理`模块的`topic初始化`完成。
   6. 导入consumer：使用`集群管理`模块的`consumer初始化`完成。
   7. 用户使用`mq-client-open`客户端包（也可以使用原生RocketMQ客户端），并通过`我是老用户`完成绑定。

8. 如果添加了新的集群，最好将MQCloud重启一下，这样可以将新的集群加入到监控采集任务中。

----------

## 客户端的使用

1. 【可选】客户端增加如下依赖：

   ```
   <dependency>
       <groupId>com.sohu.tv</groupId>
       <artifactId>mq-client-open</artifactId>
   </dependency>
   ```

   *当然可以使用RocketMQ原生客户端，但是生产者客户端统计，客户端版本将无法上报。*

2. 初始化后设置

   `RocketMQConsumer`和`RocketMQProducer`在调用`start()`方法前需要进行如下配置：

   ```
   setMqCloudDomain("MQCloud中通用配置模块的domain");
   setNameServerDomain("MQCloud中通用配置模块的nameServerDomain");
   ```

   如果消息希望序列化，还需要如下配置(采用protostuff)：

   ```
   setMessageSerializer(new DefaultMessageSerializer<Object>());
   ```

3. 消息序列化相关

   如果发送消息时使用参数为Message对象的方法，那么消息不会经过序列化。

   如果发送消息时使用参数为Object对象的方法，那么消息将会进行序列化，需要在MQCloud的`common_config`表中加入如下配置：

   ```
   INSERT INTO `common_config` (`key`, `value`, `comment`) VALUES ( 'messageSerializerClass', 'com.sohu.tv.mq.serializable.DefaultMessageSerializer', '消息序列化工具，如果发送和消费使用了该类，MQCloud也需要配置该类');
   ```

4. 其余的使用参照 用户端 -> 快速指南 -> 客户端接入模块。

   ​

   ​
