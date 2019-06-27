# activemq

主要有4个角色：生产者、消息、mq服务器、消费者。
消息传递有2种模式：
* 点对点。一个生产者生成的消息只能被一个消费者去消费，消费完了，消息就没有了。
特点：
   * 如果生产者产生了消息，消费者还没有消费，mq服务器挂了，在启动后，消息默认是持久化的，没有消费的消息还在。可以通过`producer.setDeliveryMode(DeliveryMode.NON_PERSISTENT);`设置是否要持久化。
   * 如果同时开启多个消费者，再运行生产者，那么每个消费者都会接收到消息，默认采用轮询机制，平均分配消息。
* 发布/订阅。类似关注微信公众号，一个生产者产生消息，可以由多个消费者接收。特点：
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


消息可靠性：
* 持久化 `producer.setDeliveryMode(DeliveryMode.NON_PERSISTENT);`


