title: Pulsar使用初探
date: 2019-12-06 10:00:00
tags: [技术,消息队列]
category: 技术
---

Pulsar 是一个多租户，服务器到服务器消息的高性能解决方案。Pulsar最初由Yahoo开发，它目前由 Apache软件基金会管理，类似kafka和rabbitMQ的结合体，能够支持多租户、流/队列的配置(可以完美支持顺序消费)、持久化(bookkeeper)等.

<!-- toc -->


## 特性

* Pulsar实例原生支持多集群，能够无缝的基于地理位置 进行跨集群的备份.
* 非常低的消息发布和端到端的延迟.
* 无缝的扩展到超过一百万个topic.
* 一套简单的客户端API ，支持 Java, Python, 和 C++.
* Topic支持多种 订阅模式: 独占（exclusive）, 共享（shared、shared_key）, and 备援（failover）
* 通过Apache BookKeeper提供的 持久化消息存储机制 保证消息的送达.
* 一个serverless的轻量级计算框架 Pulsar Functions 提供了原生的流数据处理
* 一个serverless的连接器框架Pulsar IO，构建于 Pulsar Functions之上，能够轻松的将数据从Apache Pulsar中移入和移出
* 当数据老化时，分层式存储 将数据从热存储卸载到冷存储中(比如S3、GCS等)
* 支持sql查询（2.0版本之后）

## 安装

* 从github上直接下载源码或者压缩包（https://pulsar.apache.org/zh-CN/download/）
* 单机模式，可以直接在bin下启动即可（http://pulsar.apache.org/docs/zh-CN/standalone/）
* 集群模式，安装部署详情见链接（https://pulsar.apache.org/docs/zh-CN/deploy-bare-metal/）

## 架构图

![订阅模型图片](/images/pulsar-system-architecture.png)

## 基本使用


### 命令行操作

#### 1. pulsar-admin tools

https://pulsar.apache.org/docs/zh-CN/pulsar-admin/

```
展示topics
pulsar-admin topics list public/default

创建
pulsar-admin topics create persistent://public/default/topic9

删除
pulsar-admin persistent delete  persistent://public/default/topic2

```

#### 2. client tools
https://pulsar.apache.org/docs/zh-CN/reference-cli-tools/

### 订阅模型

Pulsar有三种订阅模式：exclusive，shared(shared_key)，failover

* exclusive: 独享性，只存在一个Consumer可以消费，可以保证完全的有序处理.
* failover: 可以认为是高可用的exclusive，1主N从的方案，主Consumer挂掉，其他Consumer变成主，继续处理消息.
* shared: 共享性，多个consumer订阅，那么会随机处理不同的message.
* shared_key: shared型的扩展，可以将相同的key值的消息发送到同一个consumer，为批量、分布式处理提供便利.

![订阅模型图片](/images/pulsar-subscription-modes.png)

### Client操作

#### 1. client

client实例创建，默认链接的端口为6650，内部实现使用了netty实现维持长连接

```
PulsarClient client = PulsarClient.builder().serviceUrl("pulsar://localhost:6650").build();

client.close();
```

#### 2. producer

* 给指定的topic发送数据，producer发送数据，如果检测到没有该topic，会创建这个topic。
* topic默认的完整描述如下：

```
persistent://public/default/topic1
{是否持久化: persistent/non-persistent}://{tenant名称: 默认为 public}/{namespace名称: 默认为default}/{topic名称}
```

* producer创建代码

发送String类型消息
```
Producer<String> producer = client.newProducer(Schema.STRING)
            .topic("topic1")
            .enableBatching(false) // 注意该配置，默认批量发送为true
            .blockIfQueueFull(true)
            .create();

producer.newMessage().key("key-1").value("message-1-1").send();
producer.newMessage().key("key-1").value("message-1-2").send();
producer.newMessage().key("key-1").value("message-1-3").send();
producer.newMessage().key("key-2").value("message-2-1").send();
producer.newMessage().key("key-2").value("message-2-2").send();
producer.newMessage().key("key-2").value("message-2-3").send();
producer.newMessage().key("key-3").value("message-3-1").send();
producer.newMessage().key("key-3").value("message-3-2").send();
producer.newMessage().key("key-4").value("message-4-1").send();
producer.newMessage().key("key-4").value("message-4-2").send();
producer.newMessage().key("key-5").value("message-4-1").send();
producer.newMessage().key("key-5").value("message-4-2").send();
producer.newMessage().key("key-6").value("message-4-1").send();
producer.newMessage().key("key-6").value("message-4-2").send();

producer.close();
```

发送对象类型消息
```
Producer<DataMessage> producer = client.newProducer(JSONSchema.of(DataMessage.class))
            .topic("topic1")
            .batchingMaxPublishDelay(10, TimeUnit.MILLISECONDS) // 支持批量发送
            .blockIfQueueFull(true)
            .create();

DataMessage msg = new DataMessage(1,1);
producer.newMessage().send(msg);

producer.close();
```

#### 3. consumer

创建consumer
```
Consumer<DataMessage> consumer = client.newConsumer(JSONSchema.of(DataMessage.class)).topic("topic1")
		.ackTimeout(10, TimeUnit.SECONDS)
        .subscriptionName("topic1-subscription")
        .subscriptionType(SubscriptionType.Key_Shared)
        .subscribe();
```

消息处理
```
while (true) {
    Message<DataMessage> msg = consumer.receive();
    try {
        DataMessage dm = msg.getValue();
        // op dataMessage
        System.out.println("dm:" + dm.toString());
        consumer.acknowledge(msg);
    }catch(Exception ex) {
        CommonLogger.error("message send error ,ex:", ex);
        consumer.negativeAcknowledge(msg);
    }
}

```

#### 4. reader

如果想直接从某个位置开始读取队列中的消息，那么可以使用reader根据起始的MessageId进行读取。

```
// MessageIdImpl的参数: long ledgerId, long entryId, int partitionIndex
MessageId id = new MessageIdImpl(2312, 0, 0);

Reader<String> reader = client.newReader(Schema.STRING)
        .topic("ttt")
        .startMessageId(id)
        .create();

while (true) {
    Message<String> message = reader.readNext();
    System.out.println(message.getKey() + ":" + message.getValue());

    // 如果想指定读取到那个Message，可以判断MessageId，然后break
}

```

从一个队列中最早/最新的消息开始读取
```
// Create a reader on a topic and for a specific message (and onward)
Reader<byte[]> reader = pulsarClient.newReader()
    .topic("reader-api-test")
    .startMessageId(MessageId.earliest) // .startMessageId(MessageId.latest)
    .create();

while (true) {
    Message message = reader.readNext();

    // Process the message
}

```

#### 5. 相关问题

* 问题1：新建producer发送到了topic信息，但是consumer创建时间晚于producer，如果想读取到这部分信息，需要通过Reader来获取数据.
* 问题2：保持Message处理的完全有序，可以使用exclusive，如果需要保持高可用，那么使用failover.
* 问题3：Schema会定义producer和consumer处理消息的方式，两端一定要统一，如果不配置，那么默认为Byte[]类型.
* 问题4：在shared_key类型的使用上需要注意，producer不能使用batching的方式提交，否则Shared_key将失效，Shared类型无影响，创建producer的时候注意使用enableBatching(false).


### SQL操作

#### 1. 启动SQL Worker

```
	pulsar sql-worker run
```

#### 2. 基本SQL命令

命令行打开sql模式
```
pular sql
```

SQL命令
```
查询租户列表
show catalogs;

查询空间（tenant/namespce）
show schemas in pulsar;

查询topic
show tables in pulsar."public/default";

查询某个topic下数据
select * from pulsar."public/default".ttt limit 50;

```

#### 3. 相关问题

* 问题1：如果topic名称中带有中划线 “-”，那么查询不到该topic，暂时不太确定怎样转译
* 问题2：遇到在sql中查询缺失信息的情况，但是通过reader来读取能够读到


## 相关文档

https://github.com/apache/pulsar
https://blog.csdn.net/zpf_940810653842/article/details/95320943

官方文档
https://pulsar.apache.org/docs/zh-CN/standalone
pulsar sql 启动
http://pulsar.apache.org/docs/zh-CN/sql-getting-started/


