# mq消息幂等性

## 出现非幂等的情况

1、生产者已把消息发送到mq，在mq给生产者返回ack的时候网络中断，故生产者未收到确定信息，生产者认为消息未发送成功，但实际情况是，mq已成功接收到了消息，在网络重连后，生产者会重新发送刚才的消息，造成mq接收了重复的消息

2、消费者在消费mq中的消息时，mq已把消息发送给消费者，消费者在给mq返回ack时网络中断，故mq未收到确认信息，该条消息会重新发给其他的消费者，或者在网络重连后再次发送给该消费者，但实际上该消费者已成功消费了该条消息，造成消费者消费了重复的消息



## RabbitMQ自动重试机制





## 解决方案

1. 使用全局messageId判断消息是否已经消费过

2. 使用业务逻辑保证消息唯一（比如订单号）

   生产者代码：

   在消息中添加messageId

   ```java
   @Component
   public class FanoutProducer {
   	@Autowired
   	private AmqpTemplate amqpTemplate;
   
   	public void send(String queueName) {
   		String msg = "my_fanout_msg:" + System.currentTimeMillis();
   		//请求头设置消息id（messageId）
   		Message message = MessageBuilder.withBody(msg.getBytes()).setContentType(MessageProperties.CONTENT_TYPE_JSON)
   				.setContentEncoding("utf-8").setMessageId(UUID.randomUUID() + "").build();
   		System.out.println(msg + ":" + msg);
   		amqpTemplate.convertAndSend(queueName, message);
   	}
   }
   ```

   消费者代码：

   获取messageId，通过redis判断是否消费过此消息

   ```java
   @RabbitListener(queues = "fanout_email_queue")
   	public void process(Message message) throws Exception {
   		// 获取消息Id
   		String messageId = message.getMessageProperties().getMessageId();
   		String msg = new String(message.getBody(), "UTF-8");
   		//② 判断唯一Id是否被消费，消息消费成功后将id和状态保存在日志表中，我们从（①步骤）表中获取并判断messageId的状态即可
   		//从redis中获取messageId的value
   		String value = redisUtils.get(messageId)+"";
   		if(value.equals("1") ){ //表示已经消费
   			return; //结束
   		}
   		System.out.println("邮件消费者获取生产者消息" + "messageId:" + messageId + ",消息内容:" + msg);
   		JSONObject jsonObject = JSONObject.parseObject(msg);
   		// 获取email参数
   		String email = jsonObject.getString("email");
   		// 请求地址
   		String emailUrl = "http://127.0.0.1:8083/sendEmail?email=" + email;
   		JSONObject result = HttpClientUtils.httpGet(emailUrl);
   		if (result == null) {
   			// 因为网络原因,造成无法访问,继续重试
   			throw new Exception("调用接口失败!");
   		}
   		System.out.println("执行结束....");
   		//① 执行到这里已经消费成功，我们可以修改messageId的状态，并存入日志表(可以存到redis中，key为消息Id、value为状态)
   	}
   ```




#  Exchange(交换器) 属性

* Type：direct、topic、 fanout、headers

* Durability：是否需要持久化、true为持久化

* Auto Delete：当最后一个绑定到Exchange上的队列删除后，自动删除该exchange

* Internal：当前exchange 是否用于RabbitMQ内部使用，默认是False

* Arguments：扩展参数，用于扩展AMQP协议自制化使用



## 4种类型

### direct

* 所有发送到Direct exchange 的消息会被转发到RouteKey中指定的Queue
* 注意：Direct模式可以使用RabbitMQ自带的Excahnge：default
  exchange，所以不需要将Exchange进行任何绑定操作，消息传递时，RouteKey必须完全匹配才会被队列接收，否则该消息会被抛弃。

### topic 主题交换机

* 所有发送到Topic Exchange 的消息被转发到所有关心RouteKey中指定Topic的Queue上 Exchange

* 将RouteKey和某Topic 进行模糊匹配，此时队列需要绑定一个Topic

* 注意：可以使用通配符进行模糊匹配 符号“#” 匹配一个或多个词 符号“*”匹配一个词

```java
 *(星号) 用来表示一个单词 (必须出现的)
 #(井号) 用来表示任意数量（零个或多个）单词
 通配的绑定键是跟队列进行绑定的，举个小例子
 队列Q1 绑定键为 *.TT.*   队列Q2绑定键为 TT.#
 如果一条消息携带的路由键为 A.TT.B，那么队列Q1将会收到；
 如果一条消息携带的路由键为TT.AA.BB，那么队列Q2将会收到；
```

### fanout

- 不处理路由键，只需要简单的将队列绑定到交换机上 发送到交换机的消息都会被转发到与该交换机绑定的所有的队列上
- Fanout 交换机转发消息是最快的

### headers

不处理路由键。而是根据发送的消息内容中的headers属性进行匹配。在绑定Queue与Exchange时指定一组键值对；当消息发送到RabbitMQ时会取到该消息的headers与Exchange绑定时指定的键值对进行匹配；如果完全匹配则消息会路由到该队列，否则不会路由到该队列。headers属性是一个键值对，可以是Hashtable，键值对的值可以是任何类型。而fanout，direct，topic 的路由键都需要要字符串形式的。

匹配规则x-match有下列两种类型：

x-match = all ：表示所有的键值对都匹配才能接受到消息

x-match = any ：表示只要有键值对匹配就能接受到消息