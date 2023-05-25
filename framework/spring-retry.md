spring-retry的使用与风险

### **目录**

一、背景引入：
二、使用方式：
1.POM依赖
2.启用@Retryable
3.在方法上添加@Retryable
4.@Recover
三、使用风险：
1.幂等控制
2.♣线程阻塞
3.无效重试
4.异常丢失
四、总结


一、背景引入：
有些场景需要我们对一些业务处理失败情况下的任务进行重试，这时候就用到了重试框架，常用的有spring-retry和guava-retry，今天讲spring-retry。

二、使用方式：
1.POM依赖
<dependency>
<groupId>org.springframework.retry</groupId>
<artifactId>spring-retry</artifactId>
</dependency>
<!--由于该组件是依赖于 AOP 给你的，所以还需要引入这个依赖：->
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-aop</artifactId>
<version>2.6.3</version>
</dependency>



2.启用@Retryable
@EnableRetry
@SpringBootApplication
public class HelloApplication {

public static void main(String[] args) {
SpringApplication.run(HelloApplication.class, args);
}

}

3.在方法上添加@Retryable
@Service
@AllArgsConstructor
@Slf4j
public class IRetryServiceImpl implements IRetryService {

@Retryable(value = Exception.class,maxAttempts = 3,backoff = @Backoff(delay = 3600,multiplier = 1.5))
@Override
public Boolean test(Integer code) {
log.info("current-time===============>"+ DateUtil.date());
if(code<0)
{
throw new RuntimeException("数字不能那个小于0");
}
return Boolean.TRUE;
}
}

简单解释一下注解中几个参数的含义：

value：抛出指定异常才会重试
include：和value一样，默认为空，当exclude也为空时，默认所有异常
exclude：指定不处理的异常
maxAttempts：最大重试次数，默认3次
backoff：重试等待策略，默认使用@Backoff，@Backoff的value默认为1000L，我们设置为2000L；multiplier（指定延迟倍数）默认为0，表示固定暂停1秒后进行重试，如果把multiplier设置为1.5，则第一次重试为2秒，第二次为3秒，第三次为4.5秒。
当重试耗尽时还是失败，会出现什么情况呢？

当重试耗尽时，RetryOperations可以将控制传递给另一个回调，即RecoveryCallback。Spring-Retry还提供了@Recover注解，用于@Retryable重试失败后处理方法。如果不需要回调方法，可以直接不写回调方法，那么实现的效果是，重试次数完了后，如果还是没成功没符合业务判断，就抛出异常。

4.@Recover
@Recover
public Boolean recover(Exception e，Integer code)
{log.info("回调方法执行！！！！"+code);
log.info("{}"+e.getMessage());
//记日志到数据库 或者调用其余的方法
return Boolean.FALSE;
}

可以看到传参里面写的是 Exception e，这个是作为回调的接头暗号（重试次数用完了，还是失败，我们抛出这个Exception e通知触发这个回调方法）。对于@Recover注解的方法，需要特别注意的是：

方法的返回值必须与@Retryable方法一致
方法的第一个参数，必须是Throwable类型的，建议是与@Retryable配置的异常一致，其他的参数，需要哪个参数，写进去就可以了（@Retryable方法中有的）
该回调方法与重试方法写在同一个实现类里面

注意：@Retryable基于aop实现，方法内的调用无效。

三、使用风险：
优点不重要的，重要的是缺点。

这里重点讨论，这个框架使用多带来的问题。

1.幂等控制
既然是重试，就要保证这个代码执行多遍不会有问题。比如，插入数据的操作，不要插入多条。

2.♣线程阻塞
spring-retry源码如下：

public void backOff(BackOffContext backOffContext) throws BackOffInterruptedException {
    ExponentialBackOffPolicy.ExponentialBackOffContext context = (ExponentialBackOffPolicy.ExponentialBackOffContext)backOffContext;

    try {
        long sleepTime = context.getSleepAndIncrement();
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Sleeping for " + sleepTime);
        }

        this.sleeper.sleep(sleepTime);
    } catch (InterruptedException var5) {
        throw new BackOffInterruptedException("Thread interrupted while sleeping", var5);
    }
}
public class ThreadWaitSleeper implements Sleeper {
    public ThreadWaitSleeper() {
    }

    public void sleep(long backOffPeriod) throws InterruptedException {
        Thread.sleep(backOffPeriod);
    }
}
spring-retry框架重试原理，当前线程睡眠一定时间后，然后重新调用。这就有了一定风险，如果重试次数多，且间隔时间长，则会导致线程长时间阻塞。

风险点1，调用方超时。

风险点2，如果大量请求重试，大量线程阻塞，则会导致线程资源耗尽。这个风险太大了。。。。。

3.无效重试
如果我们重试策略捕获的是Exception异常，而我们的代码问题、或者参数传递问题，导致有一个空指针异常。这会出现什么情况呢？

无用的重试，重试n次后，还是空指针异常。不仅起不到重试成功的效果，还会对资源造成浪费。

结合上一点线程阻塞想一想，这个风险真的很可怕。

推荐使用方式：重试捕获的Exception尽量小。

4.异常丢失
@Recover的使用，相当于捕获异常，如果没有在这个方法里抛出这个异常，那么调用方是感知不到异常的。

需要根据具体的业务场景，正确使用@Recover



四、总结


spring-retry是个很优雅的重试框架，推荐大家在适当的场景正确使用。使用时，尽量规避掉可能的风险。

测试环境没问题，生产环境也没问题，并不代表真的没有问题，只是问题还没有爆发。。。。
