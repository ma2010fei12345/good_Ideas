# RabbitMQ实现延迟消息

## 一：背景说明
因业务需求，trans需要实现延迟消费。当时，个人想到的是利用RabbitMQ的死信队列来实现，同事提醒可以用插件来实现更加简单。一番研究后，发现利用插件实现延时队列确实更简洁方便，在此做个记录。

## 二：实现步骤
### 1.插件安装
* 去RabbitMQ的官网下载插件，插件地址：https://www.rabbitmq.com/community-plugins.html
* 直接搜索rabbitmq_delayed_message_exchange即可找到我们需要下载的插件，下载和RabbitMQ配套的版本；
* 将插件文件复制到RabbitMQ安装目录的plugins目录下；
* 进入RabbitMQ安装目录的sbin目录下，使用如下命令启用延迟插件；
* rabbitmq-plugins enable rabbitmq_delayed_message_exchange
* 启用插件成功后就可以看到如下信息，之后重新启动RabbitMQ服务即可。  

一切完成之后，可以去rabbitMQ控制台尝试新增一个交换机，可以看到交换机类型多了一个x-delayed-message，这是实现延迟消费的关键。





备注：目前测试环境和生产环境世伟哥已经帮我们安装好了插件，以后开发如有用到，此步骤可以省略了。



### 2.添加依赖和配置
在pom.xml文件中添加AMQP相关依赖，在application.yml添加RabbitMQ的相关配置；

引入方式很简单，这部分不再赘述。


### 3.创建RabbitMQ的Java配置
```java
@Configuration
public class RabbitConfig {
 
    @Value("${mq-brokers.trans_delay_exchange}")
    private String exchangeDelayName;
 
    @Value("${mq-brokers.trans_delay_queue}")
    private String queueDelayName;
 
    public static final String QUEUEDELAYROUTEKEY="trans_delay";
 
 
    /**
     * 延迟消息队列所绑定的交换机
     */
    @Bean
    CustomExchange orderPluginDirect() {
        //创建一个自定义交换机，可以发送延迟消息
        Map<String, Object> args = new HashMap<>();
        args.put("x-delayed-type", "direct");
        return new CustomExchange(exchangeDelayName, "x-delayed-message",true, false,args);
    }
 
    /**
     * 延迟队列
     */
    @Bean
    public Queue orderPluginQueue() {
        return new Queue(queueDelayName);
    }
 
    /**
     * 将延迟队列绑定到交换机
     */
    @Bean
    public Binding orderPluginBinding(CustomExchange orderPluginDirect, Queue orderPluginQueue) {
        return BindingBuilder
                .bind(orderPluginQueue)
                .to(orderPluginDirect)
                .with(QUEUEDELAYROUTEKEY)
                .noargs();
    }
 
}
```

备注说明：这里的关键是创建的交换机类型是x-delayed-message，可以发送延迟消息。

## 4.生产消息
```java
rabbitTemplate.convertAndSend(exchangeDelayName
                           , RabbitConfig.QUEUEDELAYROUTEKEY
                           , platAccountRecharge.getEnterpriseId().toString()
                           , new MessagePostProcessor() {
                               @Override
                               public Message postProcessMessage(Message message) throws AmqpException {
                                   //给消息设置延迟毫秒值
                                   message.getMessageProperties().setHeader("x-delay",10000);
                                   return message;
                               }
                           });

jafa
发送消息时，指定延迟时间即可。

## 5.消费消息
```java
@Slf4j
@Component
public class TransDelayConsumer {
 
    @Reference
    private LevyAccountRechargeService levyAccountRechargeService;
 
    @RabbitHandler
    @RabbitListener(queues = "${mq-brokers.trans_delay_queue}")
    public void getMessage(String message) {
        log.info("开始消费延迟队列trans_delay_queue消息{}",message);
        if (StringUtils.isBlank(message)){
            return;
        }
        //log.info("消费延迟队列trans_delay_queue消费中，消息{}",message);
 
        levyAccountRechargeService.checkInitAndPendingRecharge(Long.valueOf(message));
        log.info("消费延迟队列trans_delay_queue结束，消息{}",message);
    }
 
 
}
```
消费消息跟之前并无不同。

## 三：疑难解答
至此，已经利用插件实现了延迟消费。有同事提出一个致命性问题，“如果发送了第一个消息延迟一小时，第二个消息延迟一分钟，到底哪个消息会先消费？”。如果是基于死信队列实现延迟消费，那么第二个消息会在一个小时后才消息，因为这些消息都是存在队列里的，队列先进先出。那么急于插件实现的延迟消费，会不会有同样的问题呢，并不清楚，只好测试一把。

生产消息测试代码
```java
public void send() {
        log.info("充值成功触发:{}，发送延迟消息trans_delay_queue", 60);
        rabbitTemplate.convertAndSend(exchangeDelayName
                , RabbitConfig.QUEUEDELAYROUTEKEY
                , "60"
                , new MessagePostProcessor() {
                    @Override
                    public Message postProcessMessage(Message message) throws AmqpException {
                        //给消息设置延迟毫秒值
 
                        message.getMessageProperties().setHeader("x-delay", 60000);
                        return message;
                    }
                });
        log.info("充值成功触发:{}，发送延迟消息trans_delay_queue完成", 60);
 
 
        log.info("充值成功触发:{}，发送延迟消息trans_delay_queue", 30);
        rabbitTemplate.convertAndSend(exchangeDelayName
                , RabbitConfig.QUEUEDELAYROUTEKEY
                , "30"
                , new MessagePostProcessor() {
                    @Override
                    public Message postProcessMessage(Message message) throws AmqpException {
                        //给消息设置延迟毫秒值
                        message.getMessageProperties().setHeader("x-delay", 30000);
                        return message;
                    }
                });
        log.info("充值成功触发:{}，发送延迟消息trans_delay_queue完成", 30);
 
    }

```
消费消息测试代码
```java
public class TransDelayConsumer {
 
 
    @Reference
    private LevyAccountRechargeService levyAccountRechargeService;
 
 
 
    @RabbitHandler
    @RabbitListener(queues = "${mq-brokers.trans_delay_queue}")
    public void getMessage(String message) {
        log.info("开始消费延迟队列trans_delay_queue消息{}",message);
        if (StringUtils.isBlank(message)){
            return;
        }
        log.info("消费延迟队列trans_delay_queue消费中，消息{}",message);
 
       //levyAccountRechargeService.checkInitAndPendingRecharge(Long.valueOf(message));
        log.info("消费延迟队列trans_delay_queue结束，消息{}",message);
    }
 
 
}
```

结果
运行一下，看看控制台的输出







虽然设置延迟60秒的消息先发送，依然是30秒的消息先消费。消息消费时间只与设置的延迟时间有关，与谁先发送并无关系。



## 四：总结
死信队列与延迟插件比较  
* 死信队列：如果消息发送到死信队列并超过了设置的存活时间，就会被转发到设置好的处理超时消息的队列当中去，利用该特性可以实现延迟消息。这种实现方式需要两个队列，比较麻烦。另外，当各个消息的延迟时间不同时，先发送的消息会阻塞后面的消息。

* 延迟插件：通过安装插件自定义交换机，让交换机拥有延迟发送消息的能力，从而实现延迟消息。一个交换机，一个队列即可实现。可以任意设置消息的延迟时间，会按照设定的时间进行消费。

目前我们系统已经安装了插件，需要时推荐使用延迟插件实现延迟消费。
