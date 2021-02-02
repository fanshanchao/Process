# 消息队列之ActiveMQ

> 阅读本文需要有一定的消息队列基础，至少得知道消息队列是干嘛的，为什么要用它，用它能解决什么问题。
>
> 本文偏向原理性说明，不会有详细的使用教程

## 前言

​		面对目前网上这么多消息队列，Kfaka、RabbitMQ、ActiveMQ、RocketMQ等等。全部学完，时间成本太高了。所以必须先从里面先选一个学习，入门再说。在选择ActiveMQ之前犹豫了很久，担心学了以后用不上，而学其它的消息队列又没有找到好的视频教程，所以一直没有下定决定学哪个消息队列。但是在看了尚硅谷周阳老师的前几节视频就想通了，不应该局限于某种消息队列上。其实各种消息队列要解决的问题差不多，原理实现也是遵循消息队列的思想。只是在实现的细节上不同，从而导致每种消息队列有各自适用的场景和优缺点。

​		所以我们应该在理解了消息队列的思想以及要解决的问题后，再选择一个消息队列作为入门学习。至于这个消息队列的选择，也不用太过于纠结。理解了消息队列的思想以及掌握了一门消息队列后，等到实际要用的适合再学其它消息队列也是来的及的。

​		一门技术是很容易被时间淘汰的，我们要学到一门技术里面蕴含的思路（原理）才不会跟着技术被时间淘汰。这也是本文为什么没有详细使用教程内容的原因之一。

## 消息队列的思想以及要解决的问题

​		消息队列有三个非常重要的特性：异步，解耦，削峰。这三个特性网上也有很有资料详细的说过，我这里就简单的说下说下这三个特性。

1. **异步：**涉及到某个业务操作可以丢到消息队列中去，让处理这个业务操作的系统/模块异步去消费处理。而不用像原来那样必须等到处理这个业务操作的系统/模板处理完才能进行下一步。

2. **解耦：**两个系统之间的交互通过消息队列来进行交互，减少两个系统之前的耦合性，甚至感受不到对方系统的存在。

3. **削峰：**某个系统高峰期时请量求非常高，如果这个时候全部都打到系统上，并且进行处理，那么系统肯定抗不住这么大压力。这个时候我们可以选择把请求转换成消息然后丢进消息队列中，让处理这些请求的系统按照自己能够处理的并发量慢慢从消息队列中消费消息。

   消息队列的这三个特性是非常强大的，但同时也带来了很多的问题，先来看下这些问题是什么，然后再带着这些问题去学习ActiveMQ，看看ActiveMQ是如何解决这些问题的。

   1. 如何解决消息发送一致性？

      > 所谓的一致性就是产生消息的业务操作和消息发送的一致性。不能业务操作成功了，然后消息发送失败。反之亦然。

   2. 如何解决消息中间间与使用者的强依赖关系？

      > 前面我们说到了消息队列能对两个系统进行解耦，但是同时也带来了新的问题：消息中间件和系统之间的强依赖关系又该如何解决？

   3. 如何选择消息模型？

      > 点对对和发布/订阅者模式的选择

   4. 如何选择消息订阅者消息的方式？

      > 也就是该选择持久化还是非持久化订阅

   5. 在持久化订阅的前提下，消息队列是如何保证消息的可靠性的？

   6. 如何防止消息的重复产生和消费？

   7. 如何保证消息的顺序性？

   8. 该选择Push还是Pull的方式对消息进行消费？

## JMS规范

​		JMS规范是Java EE中的一个关于消息的规范。ActiveMQ就是对这个规范的落地产品。

下图是JMS编码总体架构图，ActiveMQ的使用也是遵循这个规范架构图：

> 关于JMS更详细的概述可以看[这篇博客](https://www.cnblogs.com/jaycekon/p/6220200.html)

![image-20210102163216601](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210102163216601.png)

## 如何解决消息发送一致性？

​		前面说了，所谓的一致性就是产生消息的业务操作和消息发送的一致性。不能业务操作成功了，然后消息发送失败。反之亦然。业务操作的代码和发送消息的代码一般有两种写法：

```java
void fun1(){
    //业务操作
    //发送消息
}
void fun2(){
    //发送消息
    //业务操作
}
```

​		实际开发中一般我们都会选择用第一种方式，第一种方式出现问题的概率也是相对低的。但是不怕一万就怕万一，万一发送消息失败了我们该怎么办？

### JMS规范	

​	上面说过的JMS规范其实是有XA接口可以解决问题的，可以保证发送消息到消息中间件及业务操作的事务保证，但是这个事务是分布式事务。分布式事务会有一些比较难处理的问题：

1. 开销和复杂性都会增加。
2. 对业务操作有限制，要求业务操作的资源必须支持XA协议。这是个非常大的限制。

### 最终一致性方案

​		这个方案分为发送消息的**正向流程**和检查业务操作的**反向流程**。先来看正向流程的流程图：

![image-20210129231554864](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210129231554864.png)

​		这个流程比起最开始那个简单的流程要多了很多步骤，带来的异常也是更加多了，我们来分析一下每一步操作可能发生异常的情况：

1. 第一步将消息发送给消息中间件，这一步如果出现了问题，后面的业务操作不会做，消息也不会被存储到消息中间件。所以不会出现不一致的问题。

2. 第二步存储消息，这一步如果出现了问题，无论是消息存储失败还是网络问题，可能造成的结果有两个。

   * 消息中间件失效，那么业务应用是收不到消息返回结果的
   * 消息中间件插入失败，并且还可以将存储结果返回给业务应用，这时消息中间件是没有存储消息的

3. 第三步返回存储信息结果，这一步如果出现了问题，可能是网络，消息中间件的问题，也可能是业务应用在返回结果的时候出现了问题。

   * 第一种情况，如果是业务系统本身没有问题，那么业务系统在没有收到结果的情况下会把消息当作发送失败来处理，这个时候如果消息中间件入库成功，那么就会**导致不一致的问题**。
   * 第二种情况，如果是业务系统有问题，如果中间件存储消息成功，那么就会出现**不一致**的问题，如果存储失败，则还是一致的。

4. 第四步，业务应用进行业务操作，这一步不会有太大问题。

5. 第五步，业务应用发送业务操作结果给消息中间件。如果这一步出现问题，那么消息中间件将不知道如何处理已经入库的消息，可能会造成不一致。

6. 第六步，根据业务操作结果更新消息状态。这一步出现问题也是可能出现不一致的问题。

   上面这些异常情况看起来很复杂，其实可以归类为三种问题：

   1. 业务操作未进行，消息未存储
   2. 业务操作未进行，消息状态为待处理
   3. 业务操作已进行，消息状态为待处理

   第一种问题本身就是一致的，无需做额外处理。第二三种异常情况都需要了解到业务操作的结果，才能处理存储在消息中间件的待处理状态的消息。所以我们如果可以了解到业务操作的结果，就能解决上面流程的异常情况。检查业务操作结果的反向流程由此而来。

   ![image-20210129235325093](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210129235325093.png)

   ​	上图是检查业务操作的反向流程，当然也有很多异常情况。但是前面三步都是查询，失败了就失败了，后面最后一步如果失败了，定时重复这个反向流程即可。

   #### 最终一致性方案和传统方案的对比

   | 传统方案                  | 最终一致性方案                  |
   | ------------------------- | ------------------------------- |
   | 1. 业务操作               | 1. 发送消息给消息中间件         |
   | 2. 发送消息给消息中间件   | 2. 消息中间件入库消息           |
   | 3. 消息中间件入库消息     | 3. 消息中间件返回结果           |
   | 4. 消息中间件返回入库结果 | 4. 业务操作                     |
   |                           | 5. 发送业务操作结果给消息中间件 |
   |                           | 6. 消息中间件更改存储消息的状态 |

   ​		最终一致性方案只比传统方案多了两步，也就是第五步和第六步。多了一次网络IO和更新消息状态的IO。前面四步都一样，只是顺序不同。整体上，最终一致性的方案开销并不大，此外还可以用以下方案优化：

   1. 最终一致性方案的整个流程可以封装在一个调用里面，然后把业务操作封装成一个对象传进来，这样就可以更好的控制整个流程。
   2. 应用程序提供两个接口，一个使用传统方案（不支持发送一致性），一个使用最终一致性方案。

   ### ActiveMQ如何解决消息发送一致性？

   ​		ActiveMQ是JMS规范的产品，我们前面讲过了JMS在有些场景下可以解决消息发送一致性的问题，所以ActiveMQ肯定是可以使用JMS这一套也解决这个问题。ActiveMQ官方文档上也提供了JMS操作（发送消息）和Spring中的JDBC操作在同一个事务的教程。感兴趣的同学可以看[官网的教程](https://activemq.apache.org/jms-and-jdbc-operations-in-one-transaction)。在这套方案中Spring使用的是JtaTransactionManager事务管理器，使用JMS这套分布式事务的方案性能是比较低的，一般来说都不会用这种方案，实际上网上对于这种分布式事务使用的教程也很少。而且这套方案还必须要求你消息是同步发送，虽然保证了一致性，但是响应时间确大大降低。

   ​		那上面讲的最终一致性方案是否适用于ActiveMQ呢？使用过ActiveMQ的同学会发现，ActiveMQ是没有消息状态未待处理这个概念的，那么后面那个补偿机制就实现不了，目前消息队列中只有RocketMQ支持这种。其实可以反向思考一下，我们可以把消息状态在生产者端保存，把业务操作+记录消息状态到本地数据库这两个步骤放在一个本地事务中。ActiveMQ处理详细步骤如下：

   1. 执行业务逻辑。

   2. 记录消息状态为待确定到本地数据库，这一步和执行业务逻辑在一个事务里面。

   3. 发送成功后将本地数据库消息状态更新为成功，如果失败/不确定不做处理。后续通过一个定时任务去轮询重复数据库中失败/不确定的消息。

   

​		这个方案其实有个弊端，按现在的逻辑，每个消息生产者都需要去数据库维护一个表去记录消息状态，然后定时任务处理失败/未确定的消息。我们完全可以提取出来一个服务出来专门用来做消息状态管理。这个方案的核心思路就是只要上游业务操作成功了，那么就得保证消息得要消息队列中去。

​	

   ## 如何解决消息中间间与使用者的强依赖关系？

   ​		过多的依赖消息中间件会有一个问题，如果消息中间件出现问题，那么就会导致业务操作无法继续下去，即便当时业务应用的资源都是可用的。解决这个问题的思路有三种：

   1. 消息中间件保证百分百可用。这个肯定是不可能的。
   
   2. 对于消息中间件中影响业务操作进行的部分，使其可靠性与业务自身的可靠性相同。
   
   3. 提供弱依赖的支持，能够较好保证一致性。
   
      对于第二种思路，其实就是业务操作成功了，就需要消息能够入库成功。这个入库成功可以是有延迟的，只要能保证入库就可以了。这个方案是不是有点熟悉？其实就是上面在说ActiveMQ可以如何解决消息发送一致性的时候用的最终一致性的方案。方案流程图入下所示：
   
      ![image-20210131213856223](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210131213856223.png)
   
      但是这个方案对业务应用有一些影响：
   
      1. 业务需要自己去承载消息数据。这个前面说过，可以通过将这部分独立出来一个服务来解决这个问题。
      2. 需要业务操作的对象必须是一个数据库
      3. 需要确认需要发送的消息的内容。
      
      
      ActiveMQ和其它消息队列都可以使用上面这种方案解决消息中间间与使用者的强依赖关系，但同时也会带来新的问题，我们应该根据实际的业务场景做出最正确的选择。

## 如何选择消息模型？

​		在JMS中有两种消息模型，Queue（点对对）和Topic（发布/订阅）两种模型。那我们应该如何选择消息模型呢？对于简单的业务场景，我们根据需要选择Queue/Topic即可。相对复杂的业务场景，选择消息模型不仅仅是简单的选择Queue和Topic，还有可能会有以下需求：

1. 消息发送方和接收方都是集群

2. 同一个消息的接收方可能有多个集群进行消息的处理。也就是两个集群需要处理相同的消息。

3. 不同集群对同一条消息的处理不能互相干扰

​		对于第一种需求，因为JMS是可以一个进程有多个Connection连接到JMS Server（消息中间件），所以对于消息发送方和接收方都是集群可以直接支持。而ActiveMQ是符合JMS规范的产品，那肯定也可以直接支持。所以继续看第二个和第三个需求。

​		对于第二种和第三种需求，我们可以把集群和集群之前对消息的消费当做Topic模型来处理，而集群内部的各个具体应用实例对消息的消费当做Queue模型来处理。如果我们使用的不是JMS规范的消息队列，那么可以引入ClusterId，用这个id来标识不同的集群，而集群内部的各个应用实例的连接使用同样的ClusterId。当服务器端进行调度时，根据ClusterId进行连接的分组，在不同ClusterId之前保证消息的独立投递，而拥有同样Id的连接则共同消费这些消息。而如果是JMS规范的消息队列，也就是ActiveMQ这种，可以使用另外一种做法，把JMS的Topic和Queue按照一样的思路级联起来使用。

![image-20210131222937521](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210131222937521.png)

​		使用上图这种方式相对来说比较复杂，根据自己业务场景选择。我觉得用JMS的这套解决方式还是不太优雅，还是应该考虑按集群Id分组那种方式，就是有可能消息队列不支持，需要自己去实现。

## 如何选择消息订阅者消息的方式？

​		其实就是在持久化和非持久化订阅中做出选择。对于非持久化订阅，在应用启动时会确认订阅关系，这个时候可以收到消息，但是如果应用停止了，那么订阅关系也不存在了，停止的这段时间的消息是不会为应用保存的。而持久化订阅，订阅关系一旦确认，除非应用显式的取消订阅关系，否则这个订阅将一直存在，也就是可以接受到所有的消息。正常情况下我们要做到可靠都应该选择持久化订阅这种方式。

​		ActiveMQ这两种订阅方式都是支持的，我们只需要在订阅的时候选择持久化订阅即可。这里多说一下，Queue模式默认就是会把消息持久化到磁盘。

## 在持久化订阅的前提下，消息队列是如何保证消息的可靠性的？

​		消息从发送端应用到接受端应用，中间有三个阶段需要保证可靠。分别是消息发送端发送消息到消息中间件，消息中间件存储消息，消息中间件发送消息至接受应用端。三个阶段都可靠既可以保证最终消息的可靠。

### 消息发送端的可靠性保证

​		这一步没什么好说的，只要注意消息发送者和消息中间件之间的调用返回结果的清晰设定，以及对于返回结果的处理。只有消息中间件明确返回成功才能确定消息可靠到达消息中间件了，其它情况都表示发送这个操作失败了。这一步所有的消息队列都可以做到，ActiveMQ也没有什么特别之处。

### 消息存储的可靠性保证

​		这一步其实就是选择一个存储引擎来保存消息，例如使用文件保存，使用数据库保存等等。ActiveMQ目前支持了以下几种存储方式：

1. **AMQ Mesage Store：**这是一种文件存储形式，它具有写入速度快和容易恢复的特点。消息存储再一个个文件中文件的默认大小为32M，当一个文件中的消息已经全部被消费，那么这个文件将被标识为可删除，在下一个清除阶段，这个文件被删除。AMQ适用于ActiveMQ5.3之前的版本。

2. **KahaDB消息存储(默认)：**基于日志文件，从ActiveMQ5.4开始默认的持久化插件。

3. **JDBC存储：**基于MySQL

4. **LevelDB消息存储：**知道即可。

5. **JDBC Message Store with ActiveMQ Journal：**知道即可。

​		这里其实还有一个问题，就是扩容。扩容分为对于消息中间件自身的扩容和消息存储的扩容。对于消息中间件的扩容目前ActiveMQ是支持的，也就是搭建一个高可用的集群，可以使用zookeeper+replicated-leveldb-store的主从集群。而对于消息存储的扩容，在使用ActiveMQ时，也可以使用MySQL搭建一个集群。

### 消息投递的可靠性保证

​		这一步和消息发送端发送消息一样，处理比较简单，但是必须要注意一点，消息中间件必须要收到接收者确认已消费消息的信号才能删除消息。ActiveMQ也是有一个签收确认机制来解决这个问题的，这个签收确认机制其实就是JMS规范的一部分。

## 如何防止消息的重复产生和重复消费？

​		消息发送端重试会导致的消息重复产生。可以在重试发送消失时使用同样的消息Id，而不要在消息中间件端产生的消息Id，这样可以避免这类情况的产生。

​		而对于消息的重复消费，可以使用分布式事务来解决，不过这个方式很复杂，成本也很高。另一种方式是要求消息消费者来处理重复，也就是处理幂等性。可以使用Redis来做到幂等性。

### JMS(ActiveMQ)的消息确认机制与消息重复的关系

​		在JMS中，消息接收端对收到的消息进行确认，有以下几种方式：

1.  AUTO_ACKNOWLEDGE：这是自动确认的方式。但是确认时可能消息还没来得及处理或者没处理完，所以这种确认方式对于消息投递处理来说是不可靠的。

2. CLIENT_ACKNOWLEDGE：这是客户端自己确认。这种把控制方式交给了接受消息的客户端应用。

3. DUPS_OK_ACKNOWLEDGE：这种方式是在消息接收方的消息处理函数执行结束后进行确认，一方面保证了消息一定是处理结束后才进行确认，另一方面也不需要客户端主动调用方法去确认了。

   采用DUPS_OK_ACKNOWLEDGE或CLIENT_ACKNOWLEDGE模式并且处理消息前没有确认的话，都可能导致消息被重复发送至消息接收者。

   采用AUTO_ACKNOWLEDGE或CLIENT_ACKNOWLEDGE模式并且到了接受端就立刻进行确认，而如果接收端如果问题，那么就没有机会处理这个消息，也就是消息至多被消费一次。

## 如何保证消息的顺序性？

### 使用消息优先级

​		正常来说消息是先到先投递，但是有时候需要根据优先级来确定投递顺序，优先级高的消息即使到达消息中间件的时间较晚，也是可以被优先调度的。

​		目前ActiveMQ是支持优先级属性配置的，先在启动配置文件中配置某个队列是按优先级进行投递的。然后生产者发送消息的时候指定消息的优先级即可。但是要注意ActiveMQ的消息优先级只是一个理论上的概念，并不能保证优先级高的消息一定被消费者优先消费！也就是说ActiveMQ并不能保证消费的顺序性！

### 订阅者消息处理顺序和分级订阅

​		正常情况下，消息的多个订阅者之前是独立的，它们对消息的处理并不会互相造成影响。但有些时候我们会想要某些订阅者处理结束后再让其它订阅者进行处理。针对这种场景，可以设定优先处理的订阅集群，也就是设置订阅者消息处理顺序的优先级。还有一种方案是分级订阅，也就是再来一层中间件，如下图：

​		![image-20210201224532849](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210201224532849.png)

​		对于第一种方案，我还没有看到ActiveMQ可以支持该方案的资料，但是ActiveMQ可以给队列的消费者设置优先级，优先级高的消费者会优先消费信息。优先级高的消费者的prefetch buffer达到上限后将会交给下一个优先级的消费者进行处理。

> 这里补充一下ActiveMQ的prefetch机制：ActiveMQ 通过 prefetch 机制来提高性能，这意味这客户端的内存里可能会缓存一定数量的消息。缓存消息的数量由prefetch limit 来控制。当某个 consumer 的 prefetch buffer 已经达到上限，那么 broker 不会再向 consumer 分发消息，直到 consumer 向 broker 发送消息的确认。prefetch buffer还衍生出一个问题，消息积压！有时间可以了解下怎么处理。

### 保证局部顺序

​		局部顺序是指在大量消息中，和某个业务操作有关的消息之前有顺序，而多个业务操作之前的消息没有顺序。关于ActiveMQ局部顺序的实现可以看[这篇博客](https://cloud.tencent.com/developer/article/1190900)。

### ActiveMQ关于顺序的一些特性

​		ActiveMQ 从 4.x 版本起开始支持 Exclusive Consumer （或者说 Exclusive Queues）。 Broker 会从多个 consumers 中挑选一个 consumer 来处理 queue 中所有的消息，从而保证了消息的有序处理。如果这个 consumer 失效，那么 broker会自动切换到其它的 consumer。 

​		ActiveMQ还支持Message Groups。也就是消息分组。具有相同JMSGroupID的消息会分发到相同的consumber。这也是一种有序，另外，这其实也算一种负载均衡的机制。Message Groups的处理步骤如下：

1. 一个消息被分发到 consumer 之前，broker 首先检查消息 JMSXGroupID 属性

2. 如果存在，那么 broker 会检查是否有某个 consumer 拥有这个 message group

3. 如果没有，那么 broker 会选择一个 consumer，并将它关联到这个 message group。此后，这个 consumer 会接收这个 message group 的所有消息。直到Conumber被关闭/Message Group被关闭。

   ​	ActiveMQ可以支持保证不同的Topic consumer能够以想要的顺序接受消息。需要在配置文件中开启Strict Order Dispatch Policy。

## 该选择Push还是Pull的方式对消息进行消费？

​		Push和Pull方式的对比：

|                | Push                                             | Pull                                                         |
| -------------- | ------------------------------------------------ | ------------------------------------------------------------ |
| 数据传输状态   | 保存在服务端                                     | 保存在消费端                                                 |
| 传输失败，重试 | 服务端需要维护每次传输状态，遇到失败情况需要重试 | 不需要                                                       |
| 数据传输实时性 | 非常实时                                         | 默认的短论询方式的实时性依赖于Pull间隔时间，间隔越大实时性越低。长轮询模式的实时性与Push一致。 |
| 流控机制       | 服务端需要根据订阅者的消费能力做流控             | 消费端可以根据自身消费能力决定是否去Pull消息                 |

​		两种方式各有利弊，ActiveMQ使用的是Push方式，Pull方式需要我们开发自己去维护消息队列，开发者不太友好。

## 安装使用过程中遇到的问题

1. 开放虚拟机Centos7的端口后，win10能够ping通，但是不能通过浏览器成功访问8161端口。

   **解决方法**：把conf/jetty.xml目录里面的ip改成Centos7的ip地址。但是这样在Centos访问也需要用这个ip地址来访问。

2. 61616端口提供了JMS服务。

##  参考资料

1. 《大型网站系统与Java中间件实践》
2. [美团技术团队-消息队列](https://zhuanlan.zhihu.com/p/21649950)
3. [Spring+ActiveMQ 事务](https://www.jianshu.com/p/6ec17a57f115)
4. [柔性事务：可靠消息最终一致性](https://www.jianshu.com/p/ff7dab25be19)
5. [MQ消息最终一致性解决方案](https://juejin.cn/post/6844903951448408071)