# RocketMQ
[TOC]
# 1. 快速开始
这个快速启动指南是用来指导你在本地机器安装RocketMQ消息传递系统的详细说明。

## 前置条件
### 确定你已经安装了以下软件:
64bit OS, Linux/Unix/Mac is recommended;
64bit JDK 1.8+;
Maven 3.2.x
Git
## 下载和构建（Release版本）
点击[这里](https://www.apache.org/dyn/closer.cgi?path=rocketmq/4.3.0/rocketmq-all-4.3.0-source-release.zip)下载4.3.0源码。
执行下面的命令解压4.3.0源码，构建二进制文件
> unzip rocketmq-all-4.3.0-source-release.zip
> cd rocketmq-all-4.3.0/
> mvn -Prelease-all -DskipTests clean install -U
> cd distribution/target/apache-rocketmq

## 启动 Name Server
> nohup sh bin/mqnamesrv &
> tail -f ~/logs/rocketmqlogs/namesrv.log
> The Name Server boot success...

## 启动 Broker
> nohup sh bin/mqbroker -n localhost:9876 &
> tail -f ~/logs/rocketmqlogs/broker.log
> The broker[%s, 172.30.30.233:10911] boot success...

## 发送、接收消息
在发送或接受消息之前，我们需要告诉客户端name servers 的地址，RocketMQ提供了多种方法实现此功能。为了简单起见，我们使用环境变量**NAMESRV_ADDR**的方法
> export NAMESRV_ADDR=localhost:9876
> sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
> SendResult [sendStatus=SEND_OK, msgId= ...

>sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
>ConsumeMessageThread_%d Receive New Messages: [MessageExt...

## 停止服务
> sh bin/mqshutdown broker
The mqbroker(36695) is running...
Send shutdown request to mqbroker(36695) OK

> sh bin/mqshutdown namesrv
The mqnamesrv(36664) is running...
Send shutdown request to mqnamesrv(36664) OK

## 1.1 概念
![Alt text](./OPPO截图20181027161102.png)
根据上面的模型，我们可以深入探讨一些关于消息系统设计的话题:
- 消费者并发
- 消费热点问题
- 消费者的负载均衡
- 消息路由
- 连接多路复用
- 金丝雀部署

### 1.1.1 Producer(生产者)
生产者向代理(broker)发送业务应用程序系统生成的消息。RocketMQ提供了多种发送范例:同步、异步和单向。
#### 1.1.2 Producer Group
角色相同的生产者被分组在一起。代理(broker)可能会联系同一生产者组的不同生产者实例，以提交或回滚事务，以防原始生产者在事务后崩溃。
Warning: 考虑到所提供的生产者在发送消息方面足够强大，每个生成器组只允许一个实例，以避免不必要的生产者实例初始化。
### 1.1.3 Consumer(消费者)
消费者从代理(broker)中提取消息并将其提供给应用程序。从用户应用的角度来看，提供了两种类型的消费者:
#### 1.1.3.1 PullConsumer
PullConsumer主动从代理(broker)那里获取信息。一旦提取了一批消息，用户应用程序就会启动消费过程。
#### 1.1.3.2 PushConsumer
封装消息拉取、消费过程和维护内部的其他工作，将回调接口留给最终用户来实现，该回调接口将在消息到达时执行。
#### 1.1.3.3 Consumer Group
类似于之前提到的生产者组，角色完全相同的消费者被分组在一起，并命名为Consumer Group。
Consumer Group是一个伟大的概念，通过它在信息消费方面实现负载平衡和容错非常容易。
Warning: 消费者组的消费者实例必须具有完全相同的订阅主题。
### 1.1.4 Topic(主题)
Topic是生产者传递消息和消费者拉消息的类别。主题与生产者和消费者的关系非常松散。
具体来说，一个主题可能有0个、一个或多个发送消息给它的生产者，相反，生产者可以发送不同主题的消息。
从消费者的角度来看，一个主题可以被零、一个或多个消费者组订阅。同样，消费者组也可以订阅一个或多个主题，只要该组的实例保持订阅一致。
### 1.1.5 Message(消息)
消息是要传递的信息。消息必须有一个主题(topic)，可以将其理解为要发送的邮件的地址。消息还可以有一个可选的标记(tag)和额外的键值对。例如，您可以为消息设置业务键，并在代理服务器上查找消息，以判断开发过程中的问题。
### 1.1.6 Message Queue(消息队列)
主题被划分为一个或多个子主题“消息队列”。
### 1.1.7 Tag(标签)
标签为用户提供了额外的灵活性。同一业务模块中具有不同用途的消息可能具有相同的主题和不同的标签。标签将有助于保持代码的整洁和连贯，标记还对RocketMQ提供的查询系统有用。
### 1.1.8 Broker(代理)
代理是RocketMQ系统的主要组件。它接收来自生产者的消息，存储它们并准备处理来自消费者的拉取请求。它还存储与消息相关的元数据，包括消费者组、消费进度偏移量和主题/队列信息。
### 1.1.9 Name Server(命名服务)
名称服务器充当路由信息提供者。生产者/消费者客户端通过查找主题来找到相应的代理列表。
### 1.1.10 Message Model
- Clustering (?)
- Broadcasting (广播)
### 1.1.11 Message Order
当使用DefaultMQPushConsumer时，您需要决定有序地或并发地消费消息。
- Orderly
有序地消费消息意味着消费消息的顺序与生产者为每个消息队列发送消息的顺序相同。如果你需要强制性的全局有序，确保你使用的主题只有一个消息队列。
**Warn:** 如果指定了有序的消息使用，则消息使用的最大并发性是使用者组订阅的消息队列的数量。
- Concurrently
当并发地使用消息时，消息使用的最大并发性仅受每个使用者客户机指定的线程池的限制。
**Warn:** 在此模式下不再保证消息顺序。

# 2. 简单消息示例
- 使用三种方法通过RocketMQ发送消息：可靠同步、可靠异步、单向传输
- 使用RocketMQ消费消息
## 2.1 添加依赖
maven:
```python
    <dependency>
        <groupId>org.apache.rocketmq</groupId>
        <artifactId>rocketmq-client</artifactId>
        <version>4.3.0</version>
    </dependency>
```
## 2.2 同步地发送消息
可靠的同步传输（Reliable synchronous transmission）在广泛的场景中使用，如重要的通知消息、短信通知、短信营销系统等。
```python
public class SyncProducer {
    public static void main(String[] args) throws Exception {
        //通过生产者组名进行实例
        DefaultMQProducer producer = new
            DefaultMQProducer("please_rename_unique_group_name");
        // 指定 name server 地址
        producer.setNamesrvAddr("localhost:9876");
        //运行实例
        producer.start();
        for (int i = 0; i < 100; i++) {
            //创建消息实例，指定 topic, tag 和 message body
            Message msg = new Message("TopicTest" /* Topic */,
                "TagA" /* Tag */,
                ("Hello RocketMQ " +
                    i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
            );
            //调用send方法将message投递到某一个broker
            SendResult sendResult = producer.send(msg);
            System.out.printf("%s%n", sendResult);
        }
        //生产者实例不再调用时，及时调用shudown方法
        producer.shutdown();
    }
}
```
## 2.3 异步发送消息
异步传输通常用于响应时间敏感的业务场景
```python
public class AsyncProducer {
    public static void main(String[] args) throws Exception {
        //通过生产者组名进行实例
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        // 指定 name server 地址
        producer.setNamesrvAddr("localhost:9876");
        //运行实例
        producer.start();
        producer.setRetryTimesWhenSendAsyncFailed(0);
        for (int i = 0; i < 100; i++) {
                final int index = i;
                //创建消息实例，指定 topic, tag 和 message body
                Message msg = new Message("TopicTest",
                    "TagA",
                    "OrderID188",
                    "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
                producer.send(msg, new SendCallback() {
                    @Override
                    public void onSuccess(SendResult sendResult) {
                        System.out.printf("%-10d OK %s %n", index,
                            sendResult.getMsgId());
                    }
                    @Override
                    public void onException(Throwable e) {
                        System.out.printf("%-10d Exception %s %n", index, e);
                        e.printStackTrace();
                    }
                });
        }
        //生产者实例不再调用时，及时调用shudown方法
        producer.shutdown();
    }
}
```
## 2.4 以单向模式发送消息
单向传输用于要求中等可靠性的情况，如日志收集。
```python
public class OnewayProducer {
    public static void main(String[] args) throws Exception{
        //Instantiate with a producer group name.
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        // Specify name server addresses.
        producer.setNamesrvAddr("localhost:9876");
        //Launch the instance.
        producer.start();
        for (int i = 0; i < 100; i++) {
            //Create a message instance, specifying topic, tag and message body.
            Message msg = new Message("TopicTest" /* Topic */,
                "TagA" /* Tag */,
                ("Hello RocketMQ " +
                    i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
            );
            //Call send message to deliver message to one of brokers.
            producer.sendOneway(msg);

        }
        //Shut down once the producer instance is not longer in use.
        producer.shutdown();
    }
}
```
## 2.5 消费消息
```python
	public class Consumer {

    public static void main(String[] args) throws InterruptedException, MQClientException {

        // 用指定的使用者组名进行实例化
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name");
         
        // 指定name server addresses.
        consumer.setNamesrvAddr("localhost:9876");
        
        // 订阅主题
        consumer.subscribe("TopicTest", "*");
        // 注册回调，以便从代理获取的消息到达时执行
        consumer.registerMessageListener(new MessageListenerConcurrently() {

            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                ConsumeConcurrentlyContext context) {
                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        //Launch the consumer instance.
        consumer.start();

        System.out.printf("Consumer Started.%n");
    }
}
```
## 2.6 更多
你可以从这里获得更多的示例：https://github.com/apache/rocketmq/tree/master/example

# 3. 有序消息
RocketMQ使用FIFO顺序提供有序消息

下面的示例演示如何发送/接收全局和分区有序的消息

## 3.1 发送消息示例代码
```python
public class OrderedProducer {
    public static void main(String[] args) throws Exception {
        //Instantiate with a producer group name.
        MQProducer producer = new DefaultMQProducer("example_group_name");
        //Launch the instance.
        producer.start();
        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 100; i++) {
            int orderId = i % 10;
            //Create a message instance, specifying topic, tag and message body.
            Message msg = new Message("TopicTestjjj", tags[i % tags.length], "KEY" + i,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
            @Override
            public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                Integer id = (Integer) arg;
                int index = id % mqs.size();
                return mqs.get(index);
            }
            }, orderId);

            System.out.printf("%s%n", sendResult);
        }
        //server shutdown
        producer.shutdown();
    }
}
```

## 3.2 订阅消息示例代码
```python
public class OrderedConsumer {
    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("example_group_name");

        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        consumer.subscribe("TopicTest", "TagA || TagC || TagD");

        consumer.registerMessageListener(new MessageListenerOrderly() {

            AtomicLong consumeTimes = new AtomicLong(0);
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs,
                                                       ConsumeOrderlyContext context) {
                context.setAutoCommit(false);
                System.out.printf(Thread.currentThread().getName() + " Receive New Messages: " + msgs + "%n");
                this.consumeTimes.incrementAndGet();
                if ((this.consumeTimes.get() % 2) == 0) {
                    return ConsumeOrderlyStatus.SUCCESS;
                } else if ((this.consumeTimes.get() % 3) == 0) {
                    return ConsumeOrderlyStatus.ROLLBACK;
                } else if ((this.consumeTimes.get() % 4) == 0) {
                    return ConsumeOrderlyStatus.COMMIT;
                } else if ((this.consumeTimes.get() % 5) == 0) {
                    context.setSuspendCurrentQueueTimeMillis(3000);
                    return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                }
                return ConsumeOrderlyStatus.SUCCESS;

            }
        });

        consumer.start();

        System.out.printf("Consumer Started.%n");
    }
}
```
# 4. 广播(Broadcasting)
## 4.1 什么是广播

广播(Broadcasting)是向一个主题(topic)的所有订阅者发送一条消息。如果你想让所有的订阅用户都收到关于某个主题(topic)的消息，广播(Broadcasting)是一个不错的选择。

### 4.1.1 生产者(Producer)例子
```python
public class BroadcastProducer {
    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
        producer.start();

        for (int i = 0; i < 100; i++){
            Message msg = new Message("TopicTest",
                "TagA",
                "OrderID188",
                "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult sendResult = producer.send(msg);
            System.out.printf("%s%n", sendResult);
        }
        producer.shutdown();
    }
}
```
### 4.1.2 消费者(Consumer)例子
```python
public class BroadcastConsumer {
    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("example_group_name");

        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        //set to broadcast mode
        consumer.setMessageModel(MessageModel.BROADCASTING);

        consumer.subscribe("TopicTest", "TagA || TagC || TagD");

        consumer.registerMessageListener(new MessageListenerConcurrently() {

            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                ConsumeConcurrentlyContext context) {
                System.out.printf(Thread.currentThread().getName() + " Receive New Messages: " + msgs + "%n");
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        consumer.start();
        System.out.printf("Broadcast Consumer Started.%n");
    }
}
```

# 5. 计划例子
## 什么是计划(scheduled)消息
计划消息与普通消息的不同之处在于，它们在指定的时间之后才会被投递。
### 5.1 启动消费者等待收到订阅的消息
```python
	import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
 import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
 import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
 import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
 import org.apache.rocketmq.common.message.MessageExt;
 import java.util.List;
    
 public class ScheduledMessageConsumer {
    
     public static void main(String[] args) throws Exception {
         // Instantiate message consumer
         DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("ExampleConsumer");
         // Subscribe topics
         consumer.subscribe("TestTopic", "*");
         // Register message listener
         consumer.registerMessageListener(new MessageListenerConcurrently() {
             @Override
             public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> messages, ConsumeConcurrentlyContext context) {
                 for (MessageExt message : messages) {
                     // 粗略地打印时间延迟
                     System.out.println("Receive message[msgId=" + message.getMsgId() + "] "
                             + (System.currentTimeMillis() - message.getStoreTimestamp()) + "ms later");
                 }
                 return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
             }
         });
         // Launch consumer
         consumer.start();
     }
 }
```
### 5.2 发送计划消息
```python
	import org.apache.rocketmq.client.producer.DefaultMQProducer;
	import org.apache.rocketmq.common.message.Message;
    
 public class ScheduledMessageProducer {
    
     public static void main(String[] args) throws Exception {
         // Instantiate a producer to send scheduled messages
         DefaultMQProducer producer = new DefaultMQProducer("ExampleProducerGroup");
         // Launch producer
         producer.start();
         int totalMessagesToSend = 100;
         for (int i = 0; i < totalMessagesToSend; i++) {
             Message message = new Message("TestTopic", ("Hello scheduled message " + i).getBytes());
             // 这条消息会在10秒之后送达消费者
             message.setDelayTimeLevel(3);
             // Send the message
             producer.send(message);
         }
    
         // Shutdown producer after use.
         producer.shutdown();
     }
        
 }
```
### 5.3 验证
你应该会看到消息的使用时间比存储时间晚了大约10秒。

# 6. 批处理例子
## 6.1 为什么使用批处理
批量发送消息提高了发送小消息的性能。
## 6.2 使用约束
相同批处理的消息应该具有相同的主题、相同的waitStoreMsgOK并且不支持计划类型。
另外，一个批处理中的消息的总大小不应该超过1M。
## 6.3 如何使用批处理
如果您一次发送的消息不超过1M，那么批处理很容易使用:
```python
String topic = "BatchTest";
List<Message> messages = new ArrayList<>();
messages.add(new Message(topic, "TagA", "OrderID001", "Hello world 0".getBytes()));
messages.add(new Message(topic, "TagA", "OrderID002", "Hello world 1".getBytes()));
messages.add(new Message(topic, "TagA", "OrderID003", "Hello world 2".getBytes()));
try {
    producer.send(messages);
} catch (Exception e) {
    e.printStackTrace();
    //handle the error
}
```
## 6.4 分割成列表
当您发送大量的批消息时，复杂性只会增加，并且您可能不确定它是否超过了大小限制(1MiB)，此时，最好切割列表。
```python
   public class ListSplitter implements Iterator<List<Message>> {
    private final int SIZE_LIMIT = 1000 * 1000;
    private final List<Message> messages;
    private int currIndex;
    public ListSplitter(List<Message> messages) {
            this.messages = messages;
    }
    @Override public boolean hasNext() {
        return currIndex < messages.size();
    }
    @Override public List<Message> next() {
        int nextIndex = currIndex;
        int totalSize = 0;
        for (; nextIndex < messages.size(); nextIndex++) {
            Message message = messages.get(nextIndex);
            int tmpSize = message.getTopic().length() + message.getBody().length;
            Map<String, String> properties = message.getProperties();
            for (Map.Entry<String, String> entry : properties.entrySet()) {
                tmpSize += entry.getKey().length() + entry.getValue().length();
            }
            tmpSize = tmpSize + 20; //for log overhead
            if (tmpSize > SIZE_LIMIT) {
                //it is unexpected that single message exceeds the SIZE_LIMIT
                //here just let it go, otherwise it will block the splitting process
                if (nextIndex - currIndex == 0) {
                   //if the next sublist has no element, add this one and then break, otherwise just break
                   nextIndex++;  
                }
                break;
            }
            if (tmpSize + totalSize > SIZE_LIMIT) {
                break;
            } else {
                totalSize += tmpSize;
            }
    
        }
        List<Message> subList = messages.subList(currIndex, nextIndex);
        currIndex = nextIndex;
        return subList;
    }
}
//then you could split the large list into small ones:
ListSplitter splitter = new ListSplitter(messages);
while (splitter.hasNext()) {
   try {
       List<Message>  listItem = splitter.next();
       producer.send(listItem);
   } catch (Exception e) {
       e.printStackTrace();
       //handle the error
   }
}
```
# 7. 过滤器示例
在大多数情况下，可以通过标签这个简单而有用的设计来选择您想要的消息。
比如：
```python
	DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("CID_EXAMPLE");
    consumer.subscribe("TOPIC", "TAGA || TAGB || TAGC");
```
消费者将收到包含TAGA或TAGB或TAGC的消息。但限制是一个消息只能有一个标签，并且这可能不适用于复杂的场景。在这种情况下，可以使用SQL表达式过滤掉消息。
## 7.1 原理
SQL可以通过发送消息时输入的属性进行一些计算。在RocketMQ定义的语法下，您可以实现一些有趣的逻辑。下面是一个例子:
|message|
| :----: |
|a=10|     
|b='abc'|
|c=true|

a > 5 AND b = 'abc'  匹配
a > 15 AND b = 'abc' 不匹配

## 7.2 语法
RocketMQ只定义了一些基本的语法来支持这个特性。您也可以轻松地扩展它。
1. 数字比较，如 `>`,`>=`,`<`,`<=`,`BETWEEN`,`=`;
2. 字符比较，如 `=`,`<>`,`IN`;
3. `IS NULL`或`IS NOT NULL`;
4. 逻辑操作 `AND`, `OR`, `NOT`;
### 7.2.1 常量类型：
1. Numeric, 如 123, 3.1415;
2. Character, 如‘abc’，必须使用单引号;
3. NULL
4. Boolean, TRUE FALSE;

## 7.3 使用约束
只有push consumer可以通过SQL92选择消息。接口：
> public void subscribe(final String topic, final MessageSelector messageSelector)

## 7.4 生产者示例
在发送时，可以通过方法putUserProperty将属性放入消息中。
```python
	DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
    producer.start();

    Message msg = new Message("TopicTest",
        tag,
        ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET)
    );
    // Set some properties.
    msg.putUserProperty("a", String.valueOf(i));

    SendResult sendResult = producer.send(msg);
   
    producer.shutdown();
```
## 7.5 消费者示例
消费时，依照SQL92标准，通过MessageSelector.bySql方法选择消息。
```python
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_4");

// 只订阅有属性 a, 并且 a >=0  a <= 3 的消息
consumer.subscribe("TopicTest", MessageSelector.bySql("a between 0 and 3");

consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
consumer.start();
```

# 8. Logappender
**...**

# 9. OpenMessaging
[OpenMessaging](http://openmessaging.cloud/)，包括建立行业指南和消息、流技术规范，为金融、电子商务、物联网和大数据领域提供通用框架。设计原则是面向云、简单、灵活和独立于语言的分布式异构环境。符合这些规范使得跨平台和操作系统开发消息传递应用程序成为可能。

> 消息传递和流产品已广泛应用于现代体系结构和数据处理中，用于解耦、排队、缓冲、排序、复制等。但是，当跨不同的消息传递和流平台进行数据传输时，就会出现兼容性问题，这通常意味着需要进行大量的额外工作。虽然JMS在过去的十年中是一个很好的解决方案，但是它在java环境中受到限制，缺乏对负载平衡/容错、管理、安全性和流特性的特定指导，这使得它不能很好地满足现代面向云的消息传递和流应用

RocketMQ提供了OpenMessaging 0.1.0-alpha的部分实现，下面的示例演示了如何基于OpenMessaging访问RocketMQ。
## 9.1 OMSProducer
下面的示例展示了如何在同步、异步或单向传输中向RocketMQ代理(broker)送消息。
```python
	public class OMSProducer {
    public static void main(String[] args) {
        final MessagingAccessPoint messagingAccessPoint = MessagingAccessPointFactory
            .getMessagingAccessPoint("openmessaging:rocketmq://IP1:9876,IP2:9876/namespace");

        final Producer producer = messagingAccessPoint.createProducer();

        messagingAccessPoint.startup();
        System.out.printf("MessagingAccessPoint startup OK%n");

        producer.startup();
        System.out.printf("Producer startup OK%n");

        {
            Message message = producer.createBytesMessageToTopic("OMS_HELLO_TOPIC", "OMS_HELLO_BODY".getBytes(Charset.forName("UTF-8")));
            SendResult sendResult = producer.send(message);
            System.out.printf("Send sync message OK, msgId: %s%n", sendResult.messageId());
        }

        {
            final Promise<SendResult> result = producer.sendAsync(producer.createBytesMessageToTopic("OMS_HELLO_TOPIC", "OMS_HELLO_BODY".getBytes(Charset.forName("UTF-8"))));
            result.addListener(new PromiseListener<SendResult>() {
                @Override
                public void operationCompleted(Promise<SendResult> promise) {
                    System.out.printf("Send async message OK, msgId: %s%n", promise.get().messageId());
                }

                @Override
                public void operationFailed(Promise<SendResult> promise) {
                    System.out.printf("Send async message Failed, error: %s%n", promise.getThrowable().getMessage());
                }
            });
        }

        {
            producer.sendOneway(producer.createBytesMessageToTopic("OMS_HELLO_TOPIC", "OMS_HELLO_BODY".getBytes(Charset.forName("UTF-8"))));
            System.out.printf("Send oneway message OK%n");
        }

        producer.shutdown();
        messagingAccessPoint.shutdown();
    }
}
```
## 9.2 OMSPullConsumer
使用OMS PullConsumer从指定的队列轮询消息。
```python
public class OMSPullConsumer {
    public static void main(String[] args) {
        final MessagingAccessPoint messagingAccessPoint = MessagingAccessPointFactory
            .getMessagingAccessPoint("openmessaging:rocketmq://IP1:9876,IP2:9876/namespace");

        final PullConsumer consumer = messagingAccessPoint.createPullConsumer("OMS_HELLO_TOPIC",
            OMS.newKeyValue().put(NonStandardKeys.CONSUMER_GROUP, "OMS_CONSUMER"));

        messagingAccessPoint.startup();
        System.out.printf("MessagingAccessPoint startup OK%n");
        
        consumer.startup();
        System.out.printf("Consumer startup OK%n");

        Message message = consumer.poll();
        if (message != null) {
            String msgId = message.headers().getString(MessageHeader.MESSAGE_ID);
            System.out.printf("Received one message: %s%n", msgId);
            consumer.ack(msgId);
        }

        consumer.shutdown();
        messagingAccessPoint.shutdown();
    }
}
```
## 9.3 OMSPushConsumer
将OMS PushConsumer附加到指定的队列并通过MessageListener消费消息
```python
public class OMSPushConsumer {
    public static void main(String[] args) {
        final MessagingAccessPoint messagingAccessPoint = MessagingAccessPointFactory
            .getMessagingAccessPoint("openmessaging:rocketmq://IP1:9876,IP2:9876/namespace");

        final PushConsumer consumer = messagingAccessPoint.
            createPushConsumer(OMS.newKeyValue().put(NonStandardKeys.CONSUMER_GROUP, "OMS_CONSUMER"));

        messagingAccessPoint.startup();
        System.out.printf("MessagingAccessPoint startup OK%n");

        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            @Override
            public void run() {
                consumer.shutdown();
                messagingAccessPoint.shutdown();
            }
        }));
        
        consumer.attachQueue("OMS_HELLO_TOPIC", new MessageListener() {
            @Override
            public void onMessage(final Message message, final ReceivedMessageContext context) {
                System.out.printf("Received one message: %s%n", message.headers().getString(MessageHeader.MESSAGE_ID));
                context.ack();
            }
        });
        
    }
}
```
# 10. 事务
## 10.1 什么是事务消息？
可以认为是一个两阶段提交消息的实现，以确保分布式系统中的最终一致性。事务消息确保本地事务的执行和消息的发送可以原子性地执行。
## 10.2 使用约束
1. 事务型消息没有计划和批处理的支持。
2. 为了避免单个消息被检查太多次，并导致半队列消息积累，默认情况下，我们将单个消息的检查次数限制为15次，但用户可以改变这个限制，改变broker 的“transactionCheckMax”参数的配置，如果一条消息的被检查次数超过了"transactionCheckMax"的限制，默认情况下broker会丢弃这条消息并且同时打印一条错误日志。用户可以通过重写“AbstractTransactionCheckListener”类来更改此行为。
3. 事务消息会在broker “transactionMsgTimeout”参数设定的时间后被检查，用户可以通过设置用户属性“CHECK_IMMUNITY_TIME_IN_SECONDS”来更改此限制，当发送事务消息时，该参数优先于“transactionMsgTimeout”参数。
4. 事务消息可能被检查或使用多次。
5. 提交的消息重新放入用户的目标主题(topic)能会失败。目前，它取决于日志记录。RocketMQ自身的高可用性机制确保了高可用性。如果您希望确保事务消息不会丢失，事务完整性得到保证，建议使用同步双写机制。
6. 事务消息的生产者id不能与其他类型消息的生产者id共享。与其他类型的消息不同，事务性消息允许反向查询。MQ服务器通过其生产者id查询客户机。

## 10.3 应用
### 10.3.1 事务状态

事务消息有三种状态:
1. TransactionStatus.CommitTransaction: 事务已提交，这意味着允许消费者使用该消息。
2. TransactionStatus.RollbackTransaction:  回滚事务,这意味着消息将被删除和不允许消费。
3. TransactionStatus.Unknown: 中间状态,这意味着MQ需要核对,以确定状态。

### 10.3.2 发送事务消息
创建事务性生产者
使用TransactionMQProducer类来创建producer客户机，并指定一个惟一的producerGroup，您可以设置一个自定义线程池来处理检查请求。执行本地事务后,你需要根据执行结果回复MQ，应答状态就是上面的几种。
```python
import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.common.message.MessageExt;
import java.util.List;

public class TransactionProducer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        TransactionListener transactionListener = new TransactionListenerImpl();
        TransactionMQProducer producer = new TransactionMQProducer("please_rename_unique_group_name");
        ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setName("client-transaction-msg-check-thread");
                return thread;
            }
        });

        producer.setExecutorService(executorService);
        producer.setTransactionListener(transactionListener);
        producer.start();

        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 10; i++) {
            try {
                Message msg =
                    new Message("TopicTest1234", tags[i % tags.length], "KEY" + i,
                        ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                SendResult sendResult = producer.sendMessageInTransaction(msg, null);
                System.out.printf("%s%n", sendResult);

                Thread.sleep(10);
            } catch (MQClientException | UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        }

        for (int i = 0; i < 100000; i++) {
            Thread.sleep(1000);
        }
        producer.shutdown();
    }
}
```
### 10.3.3 实现TransactionListener 接口
“executeLocalTransaction”方法用于在发送半(half)消息成功时执行本地事务。它返回前一节中提到的三种事务状态之一。
“checkLocalTransaction”方法用于检查本地事务状态并响应MQ检查请求。它也返回前一节中提到的三种事务状态之一。
```python
import ...
   
   public class TransactionListenerImpl implements TransactionListener {
       private AtomicInteger transactionIndex = new AtomicInteger(0);
   
       private ConcurrentHashMap<String, Integer> localTrans = new ConcurrentHashMap<>();
   
       @Override
       public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
           int value = transactionIndex.getAndIncrement();
           int status = value % 3;
           localTrans.put(msg.getTransactionId(), status);
           return LocalTransactionState.UNKNOW;
       }
   
       @Override
       public LocalTransactionState checkLocalTransaction(MessageExt msg) {
           Integer status = localTrans.get(msg.getTransactionId());
           if (null != status) {
               switch (status) {
                   case 0:
                       return LocalTransactionState.UNKNOW;
                   case 1:
                       return LocalTransactionState.COMMIT_MESSAGE;
                   case 2:
                       return LocalTransactionState.ROLLBACK_MESSAGE;
               }
           }
           return LocalTransactionState.COMMIT_MESSAGE;
       }
   }
```

# 11. 最佳实践

## 11.1 broker最佳实践
### 11.1.1 Broker Role(角色)
broker role 有三种：SLAVE，SYNC_MASTER， SLAVE
如果您不能容忍消息丢失，我们建议您部署SYNC_MASTER和一个SLAVE。
如果您可以接受消息丢失，但希望Broker(代理)始终可用，那么可以将ASYNC_MASTER与SLAVE一起部署。
如果您只是想简单部署，那么您可能只需要一个没有SLAVE的ASYNC_MASTER。
### 11.1.2 FlushDiskType(刷盘类型)
推荐使用ASYNC_FLUSH，因为SYNC_FLUSH太昂贵，会造成太多的性能损失。如果您想要可靠性，我们建议您使用SYNC_MASTER和SLAVE。
### 11.1.3 ReentrantLock vs CAS
待完成
### 11.1.4 os.sh
待完成

## 11.2 Producer最佳实践
### SendStatus(发送状态)
发送消息时，您将得到SendResult，它包含SendStatus。首先，我们假设Message的isWaitStoreMsgOK=true(默认为true)。否则，如果没有抛出异常，我们总是会得到SEND_OK。以下是关于每个状态的描述列表:
## 11.2 Producer最佳实践 
### 11.2.1 SendStatus(发送状态) 
发送消息时，您将得到SendResult，它包含SendStatus。首先，我们假设Message的isWaitStoreMsgOK=true(默认为true)。否则，如果没有抛出异常，我们总是会得到SEND_OK。以下是关于每个状态的描述列表: 
#### 11.2.1.1 FLUSH_DISK_TIMEOUT 
如果broker设置MessageStoreConfig的FlushDiskType=SYNC_FLUSH(默认为ASYNC_FLUSH)，并且没有在MessageStoreConfig的syncFlushTimeout(默认为5秒)中完成刷新磁盘，您将获得此状态。

#### 11.2.1.2 FLUSH_SLAVE_TIMEOUT 
如果代理的角色是SYNC_MASTER(默认是ASYNC_MASTER)，并且SLAVE没有在MessageStoreConfig的syncFlushTimeout中完成与master的同步(默认是5秒)，您将获得此状态。 
#### 11.2.1.3 SLAVE_NOT_AVAILABLE 
如果代理的角色是SYNC_MASTER(默认是ASYNC_MASTER)，但是没有配置从代理，您将获得此状态。 
#### 11.2.1.4 SEND_OK SEND_OK
并不意味着它是可靠的。为了确保不会丢失任何消息，还应该启用SYNC_MASTER或SYNC_FLUSH。 
#### 11.2.1.5 Duplication or Missing
