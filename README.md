## RabbitMQ使用说明及范例：

本文档着重描述如何使用RabbitMQ进行MQ消息发送和消费功能

#### 生产者发送消息

------

###### 后端范例

* 在子系统[比如：cjkj-message-service]pom.xml中添加公共模块的MQ组件pom依赖

  ```java
     <dependency>
        <groupId>com.cjkj</groupId>
        <artifactId>cjkj-mq-spring-boot-starter</artifactId>
        <version>1.5.0</version>
     </dependency>
  ```
  
* 在子系统[比如：cjkj-message-service]配置文件中配置需要用到的所有MQ信息[队列，交换机，路由key]，支持配置多个
  * 配置的时候注意{}对象中的key必须写正确，交换机对应exchange, 路由键对应routingKey, 队列对应queueName
  * 这里子系统[比如：cjkj-message-service]为例, 在子系统的bootstrap.yml文件中配置:
    
  ```java
     #业务代码, 交换机, 路由键, 消息队列
     queues:
       messageQueues: [{bizCode: 101, exchange: modules_message_service, routingKey: message.push, queueName: modules.message.push}]
  ```

* 在子系统[比如：cjkj-message-service]定义自己业务用到的MQ信息[队列，交换机，路由key]常量
  * 定义交换机和对应路由key封装的常量比如：PUSH_MESSAGE("modules_message_service", "message.push");  
  * 定义业务对应的队列，这里以激光推送队列为例：在内部类MessageQueueConstants中添加public static final String QUEUE_MESSAGE_PUSH = "modules.message.push";
  * 常量格式定义您可以定义成您自己喜欢的格式，不一定非要按照我下面的格式
  
  ```java
     @Slf4j
     public enum EnumMessageTopics {  
         /**
          * 交换机和对应路由key封装的常量
          */
         PUSH_MESSAGE("modules_message_service", "message.push");  
         /**
          * 交换机
          */
         private final String systemCode;  
         /**
          * 主题
          */
         private final String topic;
     
         EnumMessageTopics(final String systemCode, final String topic) {
             this.systemCode = systemCode;
             this.topic = topic;
         }  
         public String getFullTopicName() {
             return systemCode + ":" + topic;
         }
     
         public String getSystemCode() {
             return systemCode;
         }  
         public String getTopic() {
             return topic;
         }  
         /**
          * 平台 Queues
          */
         public static class MessageQueueConstants {
             /**
              * 激光推送信息队列
              */
             public static final String QUEUE_MESSAGE_PUSH = "modules.message.push";
         }  
     }
  
  ```
* 调用MQ组件的工具类MessageSender的sendRabbit方法发送消息，这里以激光推送代码为例:

  ```java
     @Autowired
     private MessageSender messageSender;
  
     @Override
     public void pushMessage(PushMessageReq pushMessageReq) {
          final List<PushMessageEntity> pushMessageEntityList = getPushMessage(pushMessageReq);
          pushMessageEntityList.forEach(pushMessageEntity -> {
              pushMessageEntity.setId(SnowflakeIdWorker.nextId());
              pushMessageDao.insertApikeyMessageInfo(pushMessageEntity);
              messageSender.sendRabbit(pushMessageEntity, EnumMessageTopics.PUSH_MESSAGE.getSystemCode(), EnumMessageTopics.PUSH_MESSAGE.getTopic());
          });
      }
  ```

#### 消费者消费消息

------

###### 后端范例

* 定义一个接受MQ消息的回调方法用于接收生产者发送的消息，这里以激光推送为例:
  * 回调方法上添加@RabbitHandler和@RabbitListener注解
  * 在@RabbitListener注解中指定队列
  
  ```java
     @RabbitHandler
     @RabbitListener(queues = EnumMessageTopics.MessageQueueConstants.QUEUE_MESSAGE_PUSH)
     private void pushMessageFunc(Message message, Channel channel) {
       TEASPOON.execute(() -> {
         final DateTimeFormatter dtf2 = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
         final String topic = message.getMessageProperties().getReceivedRoutingKey();
         String jsonMessage = new String(message.getBody(), StandardCharsets.UTF_8);
         log.info(dtf2.format(LocalDateTime.now()) + "收到 Topic: [{}]，消息 [{}]", topic, jsonMessage);
         final PushMessageEntity pushMessageEntity;
         try {
            pushMessageEntity = JsonUtils.fromJson(jsonMessage, PushMessageEntity.class);
            // 手动ack应答
            // 告诉服务器收到这条消息 已经被我消费了 可以在队列删掉 这样以后就不会再发了
            // 否则消息服务器以为这条消息没处理掉 后续还会在发，true确认所有消费者获得的消息
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
            log.info("消息消费成功：id：{}", message.getMessageProperties().getDeliveryTag());
            if (pushMessageEntity != null) {
                this.sendMessage(pushMessageEntity);
            }
         } catch (Exception e) {
           log.error("消息处理失败：id:{}", message.getMessageProperties().getDeliveryTag());  
         }  
       });
     }
  ```
 
