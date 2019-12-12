# activemq
* **基础知识**

主要有4个角色：生产者、消息、mq服务器、消费者。
消息传递有2种模式：
* 点对点。发送目的地是`Queue`。一个生产者生成的消息只能被一个消费者去消费，消费完了，消息就没有了。
特点：
   * 队列可以持久化保存消息，直到被消费，消费者不需要担心消息丢失而时刻保持和队列的连接。可以通过`producer.setDeliveryMode(DeliveryMode.NON_PERSISTENT);`设置是否要持久化。
   * 如果同时开启多个消费者，再运行生产者，那么每个消费者都会接收到消息，默认采用轮询机制，平均分配消息。
   * 如果`Session`关闭时由部分消息已经收到，但是未签收`acknowledge`，当消费者在连接这个队列时，消息还会被再次接收。会造成重复消费。
   
* 发布/订阅。发送目的地是`Topic`。类似关注微信公众号，一个生产者产生消息，可以由多个消费者接收。特点：
   * 如果没有消费者就生产了消息，那么这些消息就是废消息。在启动消费者是消费不了刚才生产的消息的。只能消费订阅之后的消息。
   * 只要订阅了，不管消费者在不在线，等它上线(启动后)，会收到消息。类始于，手机开机了，微信公众号会把从你关机后的这几天的消息，都发给你。

`Message`分三个部分：消息头、消息体、消息属性。
1. 常用的部分消息头属性：
* `JMSDestination`:消息目的地。`Queue`和`Topic`。
* `JMSDeliveryMode`:是否持久化。
* `JMSPriority`:优先级。0-9一共十个级别。0-4是普通优先级。5-9是高优先级。
* `JMSMessageID`:消息唯一ID。
* `JMSExpiration `:过期时间。

默认值：
```
public interface Message {

    static final int DEFAULT_DELIVERY_MODE = DeliveryMode.PERSISTENT;

    static final int DEFAULT_PRIORITY = 4;

    static final long DEFAULT_TIME_TO_LIVE = 0;
}
```
2. 常用的部分消息体有五种格式：`TextMessage`、`MapMessage`、`BytesMessage`、`StreamMessage`、`ObjectMessage`

3. 消息属性：`textMessage.setStringProperty();`

activemq支持的协议有TCP、NIO、UDP、SSL、Http(s)、VM。

消息可靠性：
* 持久性 `producer.setDeliveryMode(DeliveryMode.NON_PERSISTENT);`
* 事务(生产者和消费者都有事务) `Session session = connection.createSession(true, Session.AUTO_ACKNOWLEDGE);`如果设置为`true`那么在`Session`关闭前必须要`session.commit();`;
* 签收(偏向于消费者) 如果是开启事务的方式，那么就是自动签收。如果没有开启事务，或者没有commit，签收改成`Session.CLIENT_ACKNOWLEDGE`手动签收，就必须要`message.acknowledge();`
小结：
* 在事务会话中，当一个事务被成功提交则消息自动签收。
* 如果事务回滚，则消息会被再次发送，会造成重复消费。但是mq认为事务没有提交，所以认为该消息并没有被消费。
* 非事务会话中，消息何时被确认签收，取决于创建会话时的签收模式。如果是手动签收，必须要`message.acknowledge();`,如果没有签收，再次启动时会再次收到信息，会造成重复消费


activemq持久化方式：AMQ、kahaDB、jdbc、levelDB。执行逻辑都是一样的，发送至将消息方式出去后，会把消息存储到本地数据文件、内存数据库、或者远程数据等，在试图把消息发送给接收者，如果成功则将消息从存储中删除，失败则继续尝试发送。
消息中心启动后先检查指定的存储位置是否有消息。如果有未成功发送的消息，会把这些消息发送出去。常用的是`kahaDB`、`jdbc`
* AMQ是一种文件存储形式(基本不怎么用了)。消息存储在一个个文件中，文件的默认大小为32M，如果一条消息的大小超过了 32M，那么这个值必须设置大一点。当一个存储文件中的消息已经全部被消费，那么这个文件将被标识为可删除，在下一个清除阶段，这个文件被删除。AMQ适用于ActiveMQ5.3之前的版本。
* kahaDB 是5.4版本后默认的持久化方式。所有消息顺序添加到一个日志文件中，同时另外有一个索引文件记录指向这些日志的存储地址，还有一个事务日志用于消息回复操作。
* jdbc可以将消息存储到数据库中。
* levelDB这种文件系统是从ActiveMQ5.8之后引进的，它和KahaDB非常相似，也是基于文件的本地数据库储存形式，但是它提供比KahaDB更快的持久性。与KahaDB不同的是，它不是使用传统的B-树来实现对日志数据的提前写，而是使用基于索引的LevelDB。
* JDBC Message Store with ActiveMQ Journal(jdbc加强版)，普通jdbc模式每次消息过来都要写库和读库，这个加强版使用了高速缓存，提高了性能。当消费者速度能跟得上生产者的速度时候，加强版有个joimal缓存文件，这个文件能大大减少写库的数据。例如：生产者生产了1000条消息，这1000条消息会保存到joumal文件，如果消费的速度很快，在joumal文件还没有同步到DB之前就已经消费了800条，那么只会把没消费的200条信息保存到DB。
点对点模式如果设置成`DeliveryMode.NON_PERSISTENT`非持久化时，消息被保持在内存中。设为：`DeliveryMode.PERSISTENT`时，消息保存在broker想对应的文件或者数据库中。在点对点模式中，消息一旦被消费就从broker中删除掉该记录。
