# 优雅解决消息堆积问题-动态调整线程池

### 背景：

在某个固定的时间点有一个“消息发送”的服务频繁出现报警的情况，经过查看监控后确认是消费Kafka消息数量小于发送方发来的Kafka消息的数量，此时会出现**消息积压**，但是消息积压持续时间不长，很快就会恢复正常水平。

### 问题分析：

一旦出现消息积压的情况，一定要考虑下面几个方面

* 对业务会产生什么样的影响
* 什么时候出现的，积压了多长时间，积压了多少量？
* 消息积压的根本原因是什么？是consumer端消费逻辑有问题？还是consumer消费的速度落后于消息生产的速度（性能问题）？

> 在实习期间感触之一就是出现问题的时候不是第一时间去找是哪里错了，而是迅速确定该问题造成的影响，根据严重程度去指定解决方案，及时高效止损永远是第一位！

根据上面提到的几点我进行了如下的分析

1. 业务影响：会出现业务方发送信息的时间与用户接收到的信息时间出现Gap（业务上可以接受）
2. 出现时间：非公司流量高峰期（流量高峰期可能会出现所依赖的服务出现故障，导致自身服务受影响），积压了10min左右，积压量还在可控范围内

根据上面的分析，基本心里面就有数了，这个报警不会有什么大的影响，接下来就安心排查问题吧

### 排查思路：

1. 查看消费逻辑

   1. 该服务主要是使用**线程池批量消费**各个业务方发来的Kafka消息，消费逻辑就是简单的封装为满足微信公众号展示内容的格式即可，没有什么复杂的逻辑，批量消费耗时也比较短。所以从消费逻辑上来说可优化的空间比较少，可以确定无bug

2. 提升消费能力
   1. 目前我们的服务还是部署在裸机上（48核192G内存3TSSD的机器），因为运维有cgroups的限制所以实际上我们只能用到\(8核6G的资源\)
   2. 目前的业务线程池最大的核心线程数是17，线程池最大的容量是40，任务队列的最大容量是100

{% hint style="info" %}
根据上面的查看，我把重点放在通过调整线程池的大小，来提升消费能力。但是最重要的问题来了，如何确定线程池的参数呢？现在的参数确实是按照理论公式来的，我应该如何证明当前的参数不合理呢？
{% endhint %}

## 线程池工作原理

1. 如果当前运行的线程，少于corePoolSize，则创建一个新的线程来执行任务。
2. 如果运行的线程等于或多于 corePoolSize，将任务加入 BlockingQueue。
3. 如果 BlockingQueue 内的任务超过上限，则创建新的线程来处理任务。
4. 如果创建的线程数是当前运行的线程超出 maximumPoolSize，任务将被拒绝策略拒绝。

所以，如果理论上来说非高峰期的话，线程池中执行任务的线程数应该是小于等于核心线程数。在高峰期的情况下，只有当BlockingQueue打满的情况，才会启用非核心线程执行任务。很明显我们目的就是尽量的减少启用非核心线程来执行任务，避免触发拒绝策略。

### 如何查看运行时线程数的变化

![&#x83B7;&#x53D6;&#x6838;&#x5FC3;&#x7EBF;&#x7A0B;&#x6570;&#x548C;&#x6700;&#x5927;&#x7EBF;&#x7A0B;&#x6570;](.gitbook/assets/image%20%285%29.png)

![&#x83B7;&#x53D6;&#x4EFB;&#x52A1;&#x961F;&#x5217;&#x7684;&#x4FE1;&#x606F;](.gitbook/assets/image%20%287%29.png)

从ThreadPoolExecutor的源码中可以看到，我们所需要的属性都支持实时的监控，所以我们只需要通过这些属性上报当前线程池的执行情况即可。

#### 





