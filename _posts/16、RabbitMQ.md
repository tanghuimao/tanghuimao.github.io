---
layout:     post
title:      微秒级别的RabbitMq
subtitle:   兔子就是跑得快！
date:       2023-08-04
author:     HM
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - RabbitMq
    - 消息中间件
---

# RabbitMQ

1、项目中为什么要使用消息中间件？

2、项目中为什么使用RocketMQ而不是RabbitMQ？ 

3、系统TPS有多少？引入消息中间件之后，系统一定不会被撑爆了吗？

4、消息中间件中出现大量消息堆积，会产生什么后果？

5、如何发现出现了大量消息的堆积？采取了哪些应急措施？问题产生的根源是什么？如何避免

**课程主要内容**

消息中间件概述：

- 分布式系统中如何进行远程通信

- 为什么要使用消息中间件？市场上有哪些产品？有什么优缺点？该用哪个

- JMS规范和AMQP协议

RabbitMQ部分：

- RabbitMQ架构、环境准备和整合

- 高级特性如消息的可靠性保障、死信队列、延迟队列等

- RabbitMQ的集群、运维

- 源码分析，解析RabbitMQ的启动过程、交换器的实现、队列的实现等

## 1、消息中间件概述

### 1.1、分布式架构通信

#### 1.1.1、 分布式架构通信原理

![](../img/spring-cloud/spring-cloud-img-4.png)

根据实际业务，把系统拆分成合适的、独立部署的模块，模块之间相互独立。

优点：分布式、松耦合、扩展灵活、可重用。

SOA架构系统中，使用Dubbo和Zookeeper进行服务间的远程通信。

优点：Dubbo使用自定义的TCP协议，可以让请求报文体积更小，或者使用HTTP2协议，也可以减少报文的体积，提高传输效率。

![](../img/spring-cloud/spring-cloud-img-5.png)

SpringCloud中使用Feign解决服务之间远程通信的问题。

Feign：轻量级RESTful的HTTP服务客户端，广泛应用于Spring Cloud中。符合面向接口化的编程习惯。本质：封装了HTTP调用流程，类似Dubbo的服务调用。

多用于同步远程调用。

RPC主要基于TCP/UDP协议，HTTP协议是应用层协议，是构建在传输层协议TCP之上的，RPC效率更高，RPC长连接：不必每次通信都像HTTP一样三次握手，减

少网络开销；

HTTP服务开发迭代更快：在接口不多，系统与系统之间交互比较少的情况下，HTTP就显得更加方便；相反，在接口比较多，系统与系统之间交互比较多的情况

下，HTTP就没有RPC有优势。

#### 1.1.2、分布式同步通信的问题

电商项目中，如果后台添加商品信息，该信息放到数据库。我们同时，需要更新搜索引擎的倒排索引（全文索引）同时，假如有商品页面的静态化处理，也需要更

新该页面信息。

![](../img/RabbitMQ/RabbitMQ-img-1.png)

- **方式一：**

  可以在后台添加商品的方法中，如果数据插入数据库成功，就调用更新倒排索引的方法，接着调用更新静态化页面的方法。

  ```java
  Long goodsId = addGoods(goods); 
  if (goodsId != null) { 
      refreshInvertedIndex(goods); 
      refreshStaticPage(goods); 
  }
  ```

  问题：

  - 假如更新倒排索引失败，该怎么办？

  - 假如更新静态页面失败怎么办？

  解决方式：

  - 如果更新倒排索引失败，重试
  - 如果更新静态页面失败，重试

  ```java
  public Long saveGoods() { 
      Long goodsId = addGoods(goods); 
      if (goodsId != null) { 
          // 调用递归的方法，实现重试 
          boolean indexFlag = refreshInvertedIndex(goods); 
          // 调用递归的方法，实现重试 
          boolean pageFlag = refreshStaticPage(goods); 
      } 
  }
  private boolean refreshInvertedIndex(Goods goods) { 
      // 调用服务的方法
      boolean flag = indexService.refreshIndex(goods); 
      if (!flag) { 
          refreshInvertedIndex(goods); 
      } 
  }
  private boolean refreshStaticPage(Goods goods) {
      // 调用服务的方法 
      boolean flag = staticPageService.refreshStaticPage(goods); 
      if (!flag) { 
          refreshStaticPage(goods); 
      } 
  }
  ```

  以上代码在执行中的问题：

  1. 如果相应的更新一直失败，岂不是一直死循环直到调用栈崩溃？

  2. 如果相应的更新一直在重试，在重试期间，添加商品的方法调用是不是一直阻塞中？

  3. 如果添加商品的时候并发量很大，效率岂不是很低？

  或许可以加上迭代的等待时间，迭代的次数加以限制，减少CPU消耗。

  或许还可以加上多线程，同时执行更新的操作，减少执行的时间。

  但是都是基于该调用一定在可见的时间内调用成功。归根到底，是同步调用处理不当。这个问题在分布式架构中尤为严重。

- **方式二：**

  可以先执行添加商品的方法，商品添加成功，将更新索引和更新静态页面的任务缓存到一个公共的位置，然后由相应的服务从该位置获取任务来执行。

  ```java
  Long goodsId = addGoods(goods); 
  if (goodsId != null) { 
      goodsTaskService.cache(goods); 
  }
  ```

  此时，由于添加商品仅仅是将数据插入数据库，然后将任务信息缓存，调用立刻返回。对于添加商品方法的调用，不会存在线程阻塞，不会存在调用栈崩溃。

  由于更新倒排索引的的服务和更新静态页面的服务要从公共的缓存或者叫任务池中取出任务并执行，它们也会有执行失败的问题，也需要重试。如果一直更新

  失败，也需要一个方式来处理。比如如果更新失败，则每隔3秒钟重试一次，重试三次都失败则放弃执行。然后将错误结果放到另一个公共的地方，等待后续

  的补偿，无论是手工还是自动的。

  还有问题：

  1. 这个公共的任务池，会不会宕机？会不会服务不可用？如何解决？

  2. 你一定确信消息发送到任务池了吗？

  3. 如果在向任务池发送任务失败该如何处理？

  4. 如果重试的时候发送成功了，但是实际上发送了多次，更新倒排索引服务和更新静态页面服务会不会重复执行？

  5. 如果重复执行，最终结果会不会不一样？

  看来真是解决了一个问题，引进来三个问题。

  - 如果上述的问题都由我们从0开始解决，开发难度可想而知。

  - 分布式服务中，由于业务拆分，应用也需要拆分，甚至数据库分库分表。

  - 但是完成一个业务处理，往往要设计到多个模块之间的协调处理。此时模块之间，服务与服务之间以及客户端与服务端之间的通信将变得非常复杂。

#### 1.1.3、分布式异步通信模式

![](../img/RabbitMQ/RabbitMQ-img-2.png)

比较典型的“生产者消费者模式”，可以跨平台、支持异构系统，通常借助消息中间件来完成。

优点：系统间解耦，并具有一定的可恢复性，支持异构系统，下游通常可并发执行，系统具备弹性。服务解耦、流量削峰填谷等

缺点：消息中间件存在一些瓶颈和一致性问题，对于开发来讲不直观且不易调试，有额外成本。

使用异步消息模式需要注意的问题：

1. 哪些业务需要同步处理，哪些业务可以异步处理？

2. 如何保证消息的安全？消息是否会丢失，是否会重复？

3. 请求的延迟如何能够减少？

4. 消息接收的顺序是否会影响到业务流程的正常执行？

5. 消息处理失败后是否需要重发？如果重发如何保证幂等性？

### 1.2、消息中间件简介

#### 1.2.1、消息中间件概念

维基百科对消息中间件的解释：面向消息的系统（消息中间件）是在分布式系统中完成消息的发送和接收的基础软件。

消息中间件也可以称消息队列，是指用高效可靠的消息传递机制进行与平台无关的数据交流，并基于数据通信来进行分布式系统的集成。通过提供消息传递和消息

队列模型，可以在分布式环境下扩展进程的通信。

消息中间件就是在通信的上下游之间截断：break it，Broker

然后利用中间件解耦、异步的特性，构建弹性、可靠、稳定的系统。

体会一下：“必有歹人从中作梗”，”定有贵人从中相助“

应用场景：**异步处理**、**流量削峰**、限流、缓冲、排队、**最终一致性**、**消息驱动**等需求的场景都可以使用消息中间件。

![](../img/RabbitMQ/RabbitMQ-img-3.png)

#### 1.2.2、自定义消息中间件

并发编程领域经典面试题：**请使用java代码来实现“生产者消费者模式”。**

BlockingQueue（阻塞队列）是java中常见的容器，在多线程编程中被广泛使用。

当队列容器已满时生产者线程被阻塞，直到队列未满后才可以继续put；

当队列容器为空时，消费者线程被阻塞，直至队列非空时才可以继续take。

![](../img/RabbitMQ/RabbitMQ-img-4.png)

- 实体类

  ```java
  public class KouZhao {
      private Integer id;
      private String type;
    	//省略get/set   
  }
  ```

- 生产者

  ```java
  package com.thm;
  import java.util.concurrent.BlockingQueue;
  public class Provider implements Runnable {
      //阻塞队列
      private BlockingQueue<KouZhao> queue;
      private Integer index = 0;
      public Provider(BlockingQueue<KouZhao> queue) {
          this.queue = queue;
      }
      @Override
      public void run() {
          while (true) {
              try {
                  Thread.sleep(200);
                  KouZhao kouZhao = new KouZhao();
                  kouZhao.setId(index++);
                  kouZhao.setType("N95");
                  if (queue.remainingCapacity() > 0) {
                      queue.add(kouZhao);
                      System.out.println("我在生产口罩，当前库存是：" + queue.size());
                  } else {
                      System.out.println("我的仓库已经堆满了" + queue.size() + "个口罩，快来买口罩啊！");
                  }
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
          }
      }
  }
  ```

- 消费者

  ```java
  package com.thm;
  import java.util.concurrent.BlockingQueue;
  public class Consumer implements Runnable {
      //阻塞队列
      private BlockingQueue<KouZhao> queue;
  
      public Consumer(BlockingQueue<KouZhao> queue) {
          this.queue = queue;
      }
      @Override
      public void run() {
          while (true) {
              try {
                  Thread.sleep(300);
                  long startTime = System.currentTimeMillis();// 获取开始时间
                  KouZhao kouZhao = queue.take();
                  long endTime = System.currentTimeMillis();// 获取结束时间
                  System.out.println("我消费了口罩：" + kouZhao + ", 等到货时我阻 塞了" + (endTime - startTime) + "ms");
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
          }
      }
  }
  ```

- 测试类

  ```java
  public class BootStrap {
      public static void main(String[] args) {
          BlockingQueue<KouZhao> queue = new ArrayBlockingQueue<KouZhao>(20);
          new Thread(new Provider(queue)).start();
          new Thread(new Consumer(queue)).start();
      }
  }
  ```

#### 1.2.3、主流消息中间件及选型

当前业界比较流行的开源消息中间件包括：ActiveMQ、RabbitMQ、RocketMQ、Kafka、ZeroMQ等，其中应用最为广泛的要数RabbitMQ、RocketMQ、Kafka

这三款。

Redis在某种程度上也可以是实现类似“Queue”和“Pub/Sub”的机制，严格意义上不算消息中间件。

- **选取原则**

  首先，产品应该是开源的。开源意味着如果队列使用中遇到bug，可以很快修改，而不用等待开发者的更新。

  其次，产品必须是近几年比较流行的，要有一个活跃的社区。这样遇到问题很快就可以找到解决方法。同时流行也意味着bug较少。流行的产品一般跟周边系

  统兼容性比较好。

  最后，作为消息队列，要具备以下几个特性：

  - 消息传输的可靠性：保证消息不会丢失。

  - 支持集群，包括横向扩展，单点故障都可以解决。

  - 性能要好，要能够满足业务的性能需求。

- **RabbitMQ**

  RabbitMQ开始是用在电信业务的可靠通信的，也是少有的几款支持AMQP协议的产品之一。

  优点：

  1. 轻量级，快速，部署使用方便

  2. 支持灵活的路由配置。

     RabbitMQ中，在生产者和队列之间有一个交换器模块。根据配置的路由规则，生产者发送的消息可以发送到不同的队列中。路由规则很灵活，还可以自

     己实现。

  3. RabbitMQ的客户端支持大多数的编程语言。

  缺点：

  1. 如果有大量消息堆积在队列中，性能会急剧下降

  2. RabbitMQ的性能在Kafka和RocketMQ中是最差的，每秒处理几万到几十万的消息。如果应用要求高的性能，不要选择RabbitMQ。 

  3. RabbitMQ是Erlang开发的，功能扩展和二次开发代价很高。

- **RocketMQ**

  RocketMQ是一个开源的消息队列，使用java实现。借鉴了Kafka的设计并做了很多改进。

  RocketMQ主要用于有序，事务，流计算，消息推送，日志流处理，binlog分发等场景。经过了历次的双11考验，性能，稳定性可可靠性没的说。

  RocketMQ几乎具备了消息队列应该具备的所有特性和功能。

  

  java开发，阅读源代码、扩展、二次开发很方便。

  对电商领域的响应延迟做了很多优化。在大多数情况下，响应在毫秒级。如果应用很关注响应时间，可以使用RocketMQ。

  性能比RabbitMQ高一个数量级，每秒处理几十万的消息。

  缺点：跟周边系统的整合和兼容不是很好。

- **Kafka**

  Kafka的可靠性，稳定性和功能特性基本满足大多数的应用场景。

  跟周边系统的兼容性是数一数二的，尤其是大数据和流计算领域，几乎所有相关的开源软件都支持Kafka。

  Kafka高效，可伸缩，消息持久化。支持分区、副本和容错。

  

  Kafka是Scala和Java开发的，对批处理和异步处理做了大量的设计，因此Kafka可以得到非常高的性能。它的异步消息的发送和接收是三个中最好的，但是跟

  RocketMQ拉不开数量级，每秒处理几十万的消息。如果是异步消息，并且开启了压缩，Kafka最终可以达到每秒处理2000w消息的级别。

  

  但是由于是异步的和批处理的，延迟也会高，不适合电商场景。

|                        | RabbitMQ   | RocketMQ                               | Kafka                          |
| ---------------------- | ---------- | -------------------------------------- | ------------------------------ |
| 单机吞吐量             | 1w量级     | 10w量级                                | 10w量级                        |
| 开发语言               | Erlang     | Java                                   | Java和Scala                    |
| 消息延迟               | 微秒       | 毫秒                                   | 毫秒                           |
| 消息丢失               | 可能性很低 | 参数优化后可以0丢失                    | 参数优化后可以0丢失            |
| 消费模式               | 推拉       | 推拉                                   | 拉取                           |
| 主题数量对吞吐量的影响 | /          | 几百上千个主题会对吞吐量有一个小的影响 | 几十上百个主题会极大影响吞吐量 |
| 可用性                 | 高（主从） | 很高（主从）                           | 很高（分布式）                 |

#### 1.2.4、消息中间件应用场景

消息中间件的使用场景非常广泛，比如，12306购票的排队锁座，电商秒杀，大数据实时计算等。

##### 1.2.4.1、电商秒杀案例

比如6.18，活动从0:00开始，仅限前 200 名，秒杀即将开始时，用户会疯狂刷新 APP或者浏览器来保证自己能够尽早的看到商品。

- 当秒杀开始前，用户在不断的刷新页面，系统应该如何应对高并发的读请求呢？

- 在秒杀开始时，大量并发用户瞬间向系统请求生成订单，扣减库存，系统应该如何应对高并发的写请求呢？

**系统应该如何应对高并发的读请求**

- 使用缓存策略将请求挡在上层中的缓存中
- 能静态化的数据尽量做到静态化
- 加入限流（比如对短时间之内来自某一个用户，某一个IP、某个设备的重复请求做丢弃处理）

**系统应该如何应对高并发的写请求**

生成订单，扣减库存，用户这些操作不经过缓存直达数据库。如果在 1s内，有 1 万个数据连接同时到达，系统的数据库会濒临崩溃。如何解决这个问题呢？

我们可以使用 **消息队列**。

消息队列的作用：

- 削去秒杀场景下的峰值写流量——**流量削峰**

  **削去秒杀场景下的峰值写流量**

  **将秒杀请求暂存于消息队列**，业务服务器响应用户“秒杀结果正在处理中。。。”，释放系统资源去处理其它用户的请求。

  **削峰填谷**，削平短暂的流量高峰，消息堆积会造成请求延迟处理，但秒杀用户对于短暂延迟有一定容忍度。

  秒杀商品有 1000 件，处理一次购买请求的时间是 500ms，那么总共就需要 500s 的时间。这时你部署 10 个队列处理程序，那么秒杀请求的处理时间就是 50s，也就是说用户需要等待 50s 才可以看到秒杀的结果，这是可以接受的。这时会并发 10 个请求到达数据库，并不会对数据库造成很大的压力。

- 通过异步处理简化秒杀请求中的业务流程——**异步处理**

  **通过异步处理简化秒杀请求中的业务流程**

  先处理主要的业务，异步处理次要的业务。

  如主要流程是**生成订单**、**扣减库存**；次要流程比如购买成功之后会给用户**发优惠券**，**增加用户的积分**。

  此时秒杀只要处理生成订单，扣减库存的耗时，发放优惠券、增加用户积分异步去处理了。

- 解耦，实现秒杀系统模块之间松耦合——**解耦**

  **解耦**，实现秒杀系统模块之间松耦合

  将秒杀数据同步给数据团队，有两种思路：

  1. 使用 HTTP 或者 RPC 同步调用，即提供一个接口，实时将数据推送给数据服务。

     系统的耦合度高，如果其中一个服务有问题，可能会导致另一个服务不可用。

  2. 使用消息队列

     将数据全部发送给消息队列，然后数据服务订阅这个消息队列，接收数据进行处理。

##### 1.2.4.2、拉勾B端C端数据同步案例

拉勾网站分B端和C端，B端面向企业用户，C端面向求职者。这两个模块业务处理逻辑不同，数据库表结构不同，实际上是处于解耦的状态。

但是各自又需要对方的数据，需要共享：如

1. 当C端求职者在更新简历之后，B端企业用户如何尽早看到该简历更新？

2. 当B端企业用户发布新的职位需求后，C端用户如何尽早看到该职位信息？

无论是B端还是C端，都有各自的搜索引擎和缓存，B端需要获取C端的更新以更新搜索引擎和缓存；C端需要获取B端的更新以更新C端的搜索引擎与缓存。

**如何解决B端C端数据共享的问题？**

解决方式：

1. 同步方式：B端和C端通过RPC或WebService的方式发布服务，让对方来调用，以获取对方的信息。求职者每更新一次简历，就调用一次B端的服务，进行数据

   的同步；B端企业用户每更新职位需求，就调用C端的服务，进行数据的同步。

   使用同步方式，B端和C端耦合比较紧密，如果其中一个服务有问题，可能会导致另一个服务不可用。比如C端的RPC挂掉，企业用户有可能无法发布新的职位

   信息，因为发布了对方也看不到；B端的RPC挂掉，求职者可能无法更新简历，因为即使简历更新了，对方也看不到。

2. 异步方式：使用消息队列，B端将更新的数据发布到消息队列，C端将更新的数据发布到消息队列，B端订阅C端的消息队列，C端订阅B端的消息队列。

   使用消息队列的异步方式，对B端C端进行解耦，只要消息队列可用，双方都可以将需要同步的信息发送到消息队列，对方在收到消息队列推送来的消息的时

   候，各自更新自己的搜索引擎，更新自己的缓存数据。

##### 1.2.4.3、支付宝购买电影票

![](../img/RabbitMQ/RabbitMQ-img-5.png)

如上图，用户在支付宝购买了一张电影票后很快就收到消息推送和短信（电影院地址、几号厅、座位号、场次时间等），同时用户会积累一定的会员积分。

这里，交易系统并不需要一直等待消息送达等动作都完成后才返回成功，允许一定延迟和瞬时不一致（最终一致性），而且后面两个动作通常可以并发执行。

如果后期监控大盘想要获取实时交易数据，只需要新增个消费者程序并订阅该消息即可，交易系统对此并不感知，松耦合。

### 1.3、 JMS规范和AMQP协议

#### 1.3.1、JMS经典模式详解

JMS即Java消息服务（Java Message Service）应用程序接口，是一个Java平台中关于面向消息中间件（MOM，Message oriented Middleware）的API，用于在

两个应用程序之间，或分布式系统中发送消息，进行异步通信。与具体平台无关的API，绝大多数MOM提供商都支持。

它类似于JDBC(Java Database Connectivity)。

JMS是JEE平台的标准消息传递API。它可以在商业和开源实现中使用。每个实现都包括一个JMS服务器，一个JMS客户端库，以及用于管理消息传递系统的其他特定于实现的组件。 JMS提供程序可以是消息传递服务的独立实现，也可以是非JMS消息传递系统的桥梁。

JMS客户端API是标准化的，因此JMS应用程序可在供应商的实现之间移植。但是：

1. 底层消息传递实现未指定，因此JMS实现之间没有互操作性。除非存在桥接技术，否则想要共享消息传递的Java应用程序必须全部使用相同的JMS实现。

2. 如果没有供应商特定的JMS客户端库来启用互操作性，则非Java应用程序将无法访问JMS。 

3. AMQP 0-9-1是一种消息传递协议，而不是像JMS这样的API。任何实现该协议的客户端都可以访问支持AMQP 0-9-1的代理。

4. 协议级的互操作性允许以任何编程语言编写且在任何操作系统上运行的AMQP 0-9-1客户端都可以参与消息传递系统，而无需桥接不兼容的服务器实现。

##### 1.3.1.1、JMS消息

消息是JMS中的一种类型对象，由两部分组成：报文头和消息主体。

报文头包括消息头字段和消息头属性。字段是JMS协议规定的字段，属性可以由用户按需添加。

JMS报文头全部字段：

| 字段名称         | 含义                                                         |
| ---------------- | ------------------------------------------------------------ |
| JMSDestination   | JMSDestination 字段包含了消息要发送到的目的地。              |
| JMSDeliveryMode  | JMSDeliveryMode 字段包含了消息在发送的时候指定的投递模式。   |
| JMSMessageID     | 该字段包含了服务器发送的每个消息的唯一标识。                 |
| JMSTimestamp     | 该字段包含了消息封装完成要发往服务器的时间。不是真正向服务器发送的时间，因为真正的发送时间，可能会由于事务或客户端消息排队而延后。 |
| JMSCorrelationID | 客户端使用该字段的值与另一条消息关联。一个典型的场景是使用该字段将响应消息与请求消息关联。JMSCorrelationID可以包含如下值：<br/>- 服务器规定的消息ID<br/>- 应用指定的字符串<br/>- 服务器原生的byte[]值 |
| JMSReplyTo       | 该字段包含了在客户端发送消息的时候指定的Destination。即对该消息的响应应该发送到该字段指定的Destination。设置了该字段值的消息一般期望收到一个响应。 |
| JMSRedelivered   | 如果这个字段是true，则告知消费者应用这条消息已经发送过了，消费端应用应该小心别重复处理了。 |
| JMSType          | 消息发送的时候用于标识该消息的类型。具体有哪些类型，由JMS实现厂商决定。 |
| JMSExpiration    | 发送消息时，其到期时间将计算为send方法上指定的生存时间值与当前GMT值之和。 从send方法返回时，消息的JMSExpiration标头字段包含此值。 收到消息后，其JMSExpiration标头字段包含相同的值。 |
| JMSPriority      | JMS定义了一个十级优先级值，最低优先级为0，最高优先级为9。 此外，客户端应将优先级0-4视为正常优先级，将优先级5-9视为快速优先级。JMS不需要服务器严格执行消息的优先级排序； 但是，它应该尽力在普通消息之前传递加急消息。 |

消息主体则携带着应用程序的数据或有效负载。

根据有效负载的类型来划分，可以将消息分为几种类型：

1. 简单文本(TextMessage)

2. 可序列化的对象(ObjectMessage)

3. 属性集合(MapMessage)

4. 字节流(BytesMessage)

5. 原始值流(StreamMessage)

6. 无有效负载的消息(Message)。

##### 1.3.1.2、体系架构

JMS由以下元素组成：

1. JMS供应商产品

   JMS接口的一个实现。该产品可以是Java的JMS实现，也可以是非Java的面向消息中间件的适配器。

2. JMS Client

   生产或消费基于消息的Java的应用程序或对象。

3. JMS Producer

   创建并发送消息的JMS客户。

4. JMS Consumer

   接收消息的JMS客户。

5. JMS Message

   包括可以在JMS客户之间传递的数据的对象

6. JMS Queue

   缓存消息的容器。消息的接受顺序并不一定要与消息的发送顺序相同。消息被消费后将从队列中移除。

7. JMS Topic

   Pub/Sub模式。

##### 1.3.1.3、对象模型

1. ConnectionFactory 接口（连接工厂）

   用户用来创建到JMS提供者的连接的被管对象。JMS客户通过可移植的接口访问连接，这样当下层的实现改变时，代码不需要进行修改。管理员在JNDI名字空

   间中配置连接工厂，这样，JMS客户才能够查找到它们。根据消息类型的不同，用户将使用队列连接工厂，或者主题连接工厂。

2. Connection 接口（连接）

   连接代表了应用程序和消息服务器之间的通信链路。在获得了连接工厂后，就可以创建一个与JMS提供者的连接。根据不同的连接类型，连接允许用户创建会

   话，以发送和接收队列和主题到目标。

3. Destination 接口（目标）

   目标是一个包装了消息目标标识符的被管对象，消息目标是指消息发布和接收的地点，或者是队列，或者是主题。JMS管理员创建这些对象，然后用户通过

   JNDI发现它们。和连接工厂一样，管理员可以创建两种类型的目标，点对点模型的队列，以及发布者/订阅者模型的主题。

4. Session 接口（会话）

   表示一个单线程的上下文，用于发送和接收消息。由于会话是单线程的，所以消息是连续的，就是说消息是按照发送的顺序一个一个接收的。会话的好处是它

   支持事务。如果用户选择了事务支持，会话上下文将保存一组消息，直到事务被提交才发送这些消息。在提交事务之前，用户可以使用回滚操作取消这些消

   息。一个会话允许用户创建消息，生产者来发送消息，消费者来接收消息。

5. MessageConsumer 接口（消息消费者）

   由会话创建的对象，用于接收发送到目标的消息。消费者可以同步地（阻塞模式），或（非阻塞）接收队列和主题类型的消息。

6. MessageProducer 接口（消息生产者）

   由会话创建的对象，用于发送消息到目标。用户可以创建某个目标的发送者，也可以创建一个通用的发送者，在发送消息时指定目标。

7. Message 接口（消息）

   是在消费者和生产者之间传送的对象，也就是说从一个应用程序传送到另一个应用程序。一个消息有三个主要部分：

   - 消息头（必须）：包含用于识别和为消息寻找路由的操作设置。
   - 一组消息属性（可选）：包含额外的属性，支持其他提供者和用户的兼容。可以创建定制的字段和过滤器（消息选择器）。
   - 一个消息体（可选）：允许用户创建五种类型的消息（文本消息，映射消息，字节消息，流消息和对象消息）。

![](../img/RabbitMQ/RabbitMQ-img-6.png)

##### 1.3.1.4、模式

Java消息服务应用程序结构支持两种模式：

1. 点对点也叫队列模式

2. 发布/订阅模式

- 在点对点或队列模型下

  一个生产者向一个特定的**队列**发布消息，一个消费者从该队列中读取消息。这里，生产者知道消费者的队列，并直接将消息发送到消费者的队列，概括为：

  - 一条消息只有一个消费者获得
  - 生产者无需在接收者消费该消息期间处于运行状态，接收者也同样无需在消息发送时处于运行状态。

  - 每一个成功处理的消息要么自动确认，要么由接收者手动确认。

  ![](../img/RabbitMQ/RabbitMQ-img-7.png)

- 发布/订阅模式

  - 支持向一个特定的主题发布消息。
  - 0或多个订阅者可能对接收特定消息主题的消息感兴趣。
  - 发布者和订阅者彼此不知道对方。
  - 多个消费者可以获得消息

  在发布者和订阅者之间存在时间依赖性。

  - 发布者需要建立一个主题，以便客户能够订阅。
  - 订阅者必须保持持续的活动状态以接收消息，否则会丢失未上线时的消息。
  - 对于持久订阅，订阅者未连接时发布的消息将在订阅者重连时重发。

  ![](../img/RabbitMQ/RabbitMQ-img-8.png)

##### 1.3.1.5、传递方式

JMS有两种传递消息的方式。

- 标记为NON_PERSISTENT的消息最多投递一次，而标记为PERSISTENT的消息将使用暂存后再转送的机理投递。

- 如果一个JMS服务下线，持久性消息不会丢失，等该服务恢复时再传递。默认的消息传递方式是非持久性的。使用非持久性消息可能降低内务和需要的存储

  器，当不需要接收所有消息时使用。

#### 1.3.2、JMS在应用集群中的问题

生产中应用基本上都是以集群部署的。在Queue模式下，消息的消费没有什么问题，因为不同节点的相同应用会抢占式地消费消息，这样还能分摊负载。

如果使用Topic广播模式？对于一个消息，不同节点的相同应用都会收到该消息，进行相应的操作，这样就重复消费了。

![](../img/RabbitMQ/RabbitMQ-img-9.png)

- 方案一：选择Queue模式，创建多个一样的Queue，每个应用消费自己的Queue。

  弊端：浪费空间，生产者还需要关注下游到底有几个消费者，违反了“解耦”的初衷。

- 方案二：选择Topic模式，在业务上做散列，或者通过分布式锁等方式来实现不同节点间的竞争。

  弊端：对业务侵入较大，不是优雅的解决方法。

ActiveMQ通过“虚拟主题”解决了这个问题。

生产中似乎需要结合这两种模式：即不同节点的相同应用间存在竞争，会部分消费（P2P），而不同的应用都需要消费到全量的消息（Topic）模式。这样就可

以避免重复消费。

JMS规范文档(**jms-1_1-fr-spec.pdf**)下载地址：https://download.oracle.com/otndocs/jcp/7195-jms-1.1-fr-spec-oth-JSpec/

#### 1.3.3、AMQP协议剖析

##### 1.3.3.1、 协议架构

AMQP全称高级消息队列协议（Advanced Message Queuing Protocol），是一种标准，类似于JMS，兼容JMS协议。目前RabbitMQ主流支持AMQP 0-9-1，3.8.4

版本支持AMQP 1.0。

![](../img/RabbitMQ/RabbitMQ-img-10.png)

##### 1.3.3.2、 AMQP中的概念

- **Publisher：**消息发送者，将消息发送到Exchange并指定RoutingKey，以便queue可以接收到指定的消息。

- **Consumer：**消息消费者，从queue获取消息，一个Consumer可以订阅多个queue以从多个queue中接收消息。

- **Server：**一个具体的MQ服务实例，也称为Broker。

- **Virtual host：**虚拟主机，一个Server下可以有多个虚拟主机，用于隔离不同项目，一个Virtualhost通常包含多个Exchange、Message Queue。

- **Exchange：**交换器，接收Producer发送来的消息，把消息转发到对应的Message Queue中。

- **Routing key：**路由键，用于指定消息路由规则（Exchange将消息路由到具体的queue中），通常需要和具体的Exchange类型、Binding的Routing key结合

  起来使用。

- **Bindings：**指定了Exchange和Queue之间的绑定关系。Exchange根据消息的Routing key和Binding配置（绑定关系、Binding、Routing key等）来决

  定把消息分派到哪些具体的queue中。这依赖于Exchange类型。

- **Message Queue：**实际存储消息的容器，并把消息传递给最终的Consumer。

##### 1.3.3.3、AMQP 传输层架构

- **简要概述**

  - AMQP是一个二进制的协议，信息被组织成数据帧，有很多类型。数据帧携带协议方法和其他信息。所有数据帧都拥有基本相同的格式：帧头，负载，帧

    尾。数据帧负载的格式依赖于数据帧的类型。

  - 我们假定有一个可靠的面向流的网络传输层（TCP/IP或等价的协议）。

  - 在一个单一的socket连接中，可能有多个相互独立的控制线程，称为“channel”。每个数据帧使用通道号码编号。通过数据帧的交织，不同的通道共享一个

    连接。对于任意给定通道，数据帧严格按照序列传输。

  - 我们使用小的数据类型来构造数据帧，如bit，integer，string以及字段表。数据帧的字段做了轻微的封装，不会让传输变慢或解析困难。根据协议规范机

    械地生成成数据帧层相对简单。

  - 线级别的格式被设计为可伸缩和足够通用，以支持任意的高层协议（不仅是AMQP）。我们假定AMQP会扩展，改进以及随时间的其他变化，并要求wire-

    level格式支持这些变化。

- **数据类型**

  AMQP 使用的数据类型如下：

  - Integers（数值范围1-8的十进制数字）：用于表示大小，数量，限制等，整数类型无符号的，可以在帧内不对齐。
  - Bits（统一为8个字节）：用于表示开/关值。
  - Short strings：用于保存简短的文本属性，字符串个数限制为255，8个字节
  - Long strings：用于保存二进制数据块。
  - Field tables：包含键值对，字段值一般为字符串，整数等。

- **协议协商**

  AMQP客户端和服务端进行协议协商。意味着当客户端连接上之后，服务端会向客户端提出一些选项，客户端必须能接收或修改。如果双方都认同协商的结

  果，继续进行连接的建立过程。协议协商是一个很有用的技术手段，因为它可以让我们断言假设和前置条件。

  在AMQP中，我们需要协商协议的一些特殊方面：

  1、 真实的协议和版本。服务器可能在同一个端口支持多个协议。

  2、 双方的加密参数和认证方式。这是功能层的一部分。

  3、 数据帧最大大小，通道数量以及其他操作限制。

  对限制条件的认同可能会导致双方重新分配key的缓存，避免死锁。每个发来的数据帧要么遵守认同的限制，也就是安全的，要么超过了限制，此时另一方出

  错，必须断开连接。出色地践行了“要么一切工作正常，要么完全不工作”的RabbitMQ哲学。

  协商双方认同限制到一个小的值，如下：

  1. 服务端必须告诉客户端它加上了什么限制。

  2. 客户端响应服务器，或许会要求对客户端的连接降低限制。

- **数据帧界定**

  TCP/IP是流协议，没有内置的机制用于界定数据帧。现有的协议从以下几个方面来解决：

  1. 每个连接发送单一数据帧。简单但是慢。

  2. 在流中添加帧的边界。简单，但是解析很慢。

  3. 计算数据帧的大小，在每个数据帧头部加上该数据帧大小。这简单，快速，AMQP的选择。

##### 1.3.3.4、AMQP客户端实现JMS客户端

RabbitMQ的JMS客户端用RabbitMQ Java客户端实现，既与JMS API兼容，也与AMQP 0-9-1协议兼容。

局限性

RabbitMQ JMS客户端不支持某些JMS 1.1功能：

- JMS客户端不支持服务器会话。
- XA事务支持接口未实现。
- RabbitMQ JMS主题选择器插件支持主题选择器。队列选择器尚未实现。
- 支持RabbitMQ连接的SSL和套接字选项，但仅使用RabbitMQ客户端提供的（默认）SSL连接协议。
- RabbitMQ不支持JMS NoLocal订阅功能，该功能禁止消费者接收通过消费者自己的连接发布的消息。可以调用包含NoLocal参数的方法，但该方法将被忽略。

RabbitMQ使用amqp协议，JMS规范仅对于Java的使用作出的规定，跟其他语言无关，协议是语言无关的，只要语言实现了该协议，就可以做客户端。如此，则不

同语言之间互操作性得以保证。

AMQP协议文档下载地址：https://www.amqp.org/sites/amqp.org/files/amqp0-9-1.zip

## 2、RabbitMQ架构与实战

### 2.1、RabbitMQ介绍、概念、基本架构

#### 2.1.1、RabbitMQ介绍

RabbitMQ，俗称“兔子MQ”（可见其轻巧，敏捷），是目前非常热门的一款开源消息中间件，不管是互联网行业还是传统行业都广泛使用（最早是为了解决电信行

业系统之间的可靠通信而设计）。

- 高可靠性、易扩展、高可用、功能丰富等
- 支持大多数（甚至冷门）的编程语言客户端。
- RabbitMQ遵循AMQP协议，自身采用Erlang（一种由爱立信开发的通用面向并发编程的语言）编写。
- RabbitMQ也支持MQTT等其他协议。

RabbitMQ具有很强大的插件扩展能力，官方和社区提供了非常丰富的插件可供选择：https://www.rabbitmq.com/community-plugins.html

#### 2.1.2、RabbitMQ整体逻辑架构

![](../img/RabbitMQ/RabbitMQ-img-11.png)

#### 2.1.3、RabbitMQ Exchange类型

RabbitMQ常用的交换器类型有： fanout 、 direct 、 topic 、 headers 四种。

- **Fanout**

  会把所有发送到该交换器的消息路由到所有与该交换器绑定的队列中（广播），如图：

  ![](../img/RabbitMQ/RabbitMQ-img-12.png)

- **Direct**

  direct类型的交换器路由规则很简单，它会把消息路由到那些BindingKey和RoutingKey完全匹配的队列中，如下图：

  ![](../img/RabbitMQ/RabbitMQ-img-13.png)

- **Topic**

  ![](../img/RabbitMQ/RabbitMQ-img-14.png)

  topic类型的交换器在direct匹配规则上进行了扩展，也是将消息路由到BindingKey和RoutingKey相匹配的队列中，这里的匹配规则稍微不同，它约定：

  BindingKey和RoutingKey一样都是由"."分隔的字符串；BindingKey中可以存在两种特殊字符*****和“#”，用于模糊匹配，其中*****用于匹配一个单词，"#"用于匹配

  多个单词（可以是0个）。

- **Headers**

  headers类型的交换器不依赖于路由键的匹配规则来路由信息，而是根据发送的消息内容中的headers属性进行匹配。在绑定队列和交换器时指定一组键值

  对，当发送的消息到交换器时，RabbitMQ会获取到该消息的headers，对比其中的键值对是否完全匹配队列和交换器绑定时指定的键值对，如果匹配，消息

  就会路由到该队列。headers类型的交换器性能很差，不实用。

#### 2.1.4、RabbitMQ数据存储

RabbitMQ消息有两种类型：

- 持久化消息

  持久化消息在到达队列时写入磁盘，同时会内存中保存一份备份，当内存吃紧时，消息从内存中清除。这会提高一定的性能。

- 非持久化消息

  非持久化消息一般只存于内存中，当内存压力大时数据刷盘处理，以节省内存空间。

**注意：这两种消息都会被写入磁盘。**

RabbitMQ存储层包含两个部分：

- 队列索引
- 消息存储

![](../img/RabbitMQ/RabbitMQ-img-15.png)

- **队列索引：rabbit_queue_index**

  索引维护队列的落盘消息的信息，如存储地点、是否已被给消费者接收、是否已被消费者ack等。每个队列都有相对应的索引。

  ![](../img/RabbitMQ/RabbitMQ-img-16.png)

  索引使用顺序的段文件来存储，后缀为.idx，文件名从0开始累加，每个段文件中包含固定的segment_entry_count 条记录，默认值是**16384**。每个index从磁

  盘中读取消息的时候，**至少要在内存中维护一个段文件**，所以设置 queue_index_embed_msgs_below 值得时候要**格外谨慎**，一点点增大也可能会引起内存爆

  炸式增长。

- **消息存储：rabbit_msg_store**

  消息以**键值对的形式**存储到文件中，一个虚拟主机上的所有队列使用同一块存储，每个节点只有一个。存储分为持久化存储（msg_store_persistent）和短暂

  存储（msg_store_transient）。持久化存储的内容在broker重启后不会丢失，短暂存储的内容在broker重启后丢失。store使用文件来存储，后缀为.rdq，经

  过store处理的所有消息都会以追加的方式写入到该文件中，当该文件的大小超过指定的限制（file_size_limit）后，将会关闭该文件并创建一个新的文件以供

  新的消息写入。文件名从0开始进行累加。在进行消息的存储时，RabbitMQ会在ETS（Erlang Term Storage）表中记录消息在文件中的位置映射和文件的相

  关信息。

  ![](../img/RabbitMQ/RabbitMQ-img-17.png)

- **消息**

  消息（包括消息头、消息体、属性）可以直接存储在index中，也可以存储在store中。最佳的方式是较小的消息存在index中，而较大的消息存在store中。这

  个消息大小的界定可以通过queue_index_embed_msgs_below 来配置，默认值为4096B。当一个消息小于设定的大小阈值时，就可以存储在index中，这样

  性能上可以得到优化。**一个完整的消息**大小小于这个值，就放到索引中，否则放到持久化消息文件中。

  rabbitmq.conf中的配置信息：

  ```properties
  # queue_index_embed_msgs_below = 4096 
  ## You can also set this size in memory units
  # queue_index_embed_msgs_below = 4kb
  ```

  如果消息小于这个值，就在索引中存储，如果消息大于这个值就在store中存储。

  - 大于这个值的消息存储于msg_store_persistent目录中的<num>.rdq文件中

  - 小于这个值的消息存储于<num>.idx索引文件中

    

  读取消息时，先根据消息的ID（msg_id）找到对应存储的文件，如果文件存在并且未被锁住，则直接打开文件，从指定位置读取消息内容。如果文件不存在或

  者被锁住了，则发送请求由store进行处理。

  ​	

  删除消息时，只是从ETS表删除指定消息的相关信息，同时更新消息对应的存储文件和相关信息。在执行消息删除操作时，并不立即对文件中的消息进行删

  除，也就是说消息依然在文件中，仅仅是标记为垃圾数据而已。当一个文件中都是垃圾数据时可以将这个文件删除。当检测到前后两个文件中的有效数据可以

  合并成一个文件，并且所有的垃圾数据的大小和所有文件（至少有3个文件存在的情况下）的数据大小的比值超过设置的阈值garbage_fraction（默认值0.5）

  时，才会触发垃圾回收，将这两个文件合并，执行合并的两个文件一定是逻辑上相邻的两个文件。合并逻辑：

  1. 锁定这两个文件

  2. 先整理前面的文件的有效数据，再整理后面的文件的有效数据

  3. 将后面文件的有效数据写入到前面的文件中
  4. 更新消息在ETS表中的记录
  5. 删除后面文件

  ![](../img/RabbitMQ/RabbitMQ-img-18.png)

- **队列结构**

  通常队列由rabbit_amqqueue_process和backing_queue这两部分组成，rabbit_amqqueue_process负责协议相关的消息处理，即接收生产者发布的消息、

  向消费者交付消息、处理消息的确认（包括生产端的confirm和消费端的ack）等。backing_queue是消息存储的具体形式和引擎，并向

  rabbit_amqqueue_process提供相关的接口以供调用。

  如果消息投递的目的队列是空的，并且有消费者订阅了这个队列，那么该消息会直接发送给消费者，不会经过队列这一步。当消息无法直接投递给消费者时，

  需要暂时将消息存入队列，以便重新投递。

  rabbit_variable_queue.erl 源码中定义了RabbitMQ队列的**4种状态**： 

  1. alpha：消息索引和消息内容都存内存，最耗内存，很少消耗CPU

  2. beta：消息索引存内存，消息内存存磁盘

  3. gama：消息索引内存和磁盘都有，消息内容存磁盘

  4. delta：消息索引和内容都存磁盘，基本不消耗内存，消耗更多CPU和I/O操作

  消息存入队列后，不是固定不变的，它会随着系统的负载在队列中不断流动，消息的状态会不断发送变化。

  持久化的消息，索引和内容都必须先保存在磁盘上，才会处于上述状态中的一种。gama状态只有持久化消息才会有的状态。

  在运行时，RabbitMQ会根据消息传递的速度定期计算一个当前内存中能够保存的最大消息数量（target_ram_count），如果alpha状态的消息数量大于此

  值，则会引起消息的状态转换，多余的消息可能会转换到beta、gama或者delta状态。区分这4种状态的主要作用是满足不同的内存和CPU需求。

  对于**普通没有设置优先级和镜像**的队列来说，backing_queue的默认实现是rabbit_variable_queue，其内部通过**5个子队列**Q1、Q2、delta、Q3、Q4来体现

  消息的各个状态。

  ![](../img/RabbitMQ/RabbitMQ-img-19.png)

  ![](../img/RabbitMQ/RabbitMQ-img-20.png)

  消费者获取消息也会引起消息的状态转换。

  当消费者获取消息时

  1. 首先会从Q4中获取消息，如果获取成功则返回。

  2. 如果Q4为空，则尝试从Q3中获取消息，系统首先会判断Q3是否为空，如果为空则返回队列为空，即此时队列中无消息。

  3. 如果Q3不为空，则取出Q3中的消息；**进而**再判断此时Q3和Delta中的长度，如果都为空，则可以认为 Q2、Delta、 Q3、Q4 全部为空，此时将Q1中的消

     息直接转移至Q4，下次直接从Q4 中获取消息。

  4. 如果Q3为空，Delta不为空，则将Delta的消息转移至Q3中，下次可以直接从Q3中获取消息。在将消息从Delta转移到Q3的过程中，是按照索引分段读取

     的，首先读取某一段，然后判断读取的消息的个数与Delta中消息的个数是否相等，如果相等，则可以判定此时Delta中己无消息，则直接将Q2和刚读取到

     的消息一并放入到Q3中，如果不相等，仅将此次读取到的消息转移到Q3。

  这里就有两处疑问，第一个疑问是：为什么Q3为空则可以认定整个队列为空？

  1. 试想一下，如果Q3为空，Delta不为空，那么在Q3取出最后一条消息的时候，Delta 上的消息就会被转移到Q3这样与 Q3 为空矛盾；

  2. 如果Delta 为空且Q2不为空，则在Q3取出最后一条消息时会将Q2的消息并入到Q3中，这样也与Q3为空矛盾；

  3. 在Q3取出最后一条消息之后，如果Q2、Delta、Q3都为空，且Q1不为空时，则Q1的消息会被转移到Q4，这与Q4为空矛盾。

  其实这一番论述也解释了另一个问题：为什么Q3和Delta都为空时，则可以认为 Q2、Delta、Q3、Q4全部为空？

  通常在负载正常时，如果消费速度大于生产速度，对于不需要保证可靠不丢失的消息来说，极有可能只会处于alpha状态。

  对于持久化消息，它一定会进入gama状态，在开启publisher confirm机制时，只有到了gama 状态时才会确认该消息己被接收，若消息消费速度足够快、内

  存也充足，这些消息也不会继续走到下一个状态。

- **为什么消息的堆积导致性能下降？**

  在系统负载较高时，消息若不能很快被消费掉，这些消息就会进入到很深的队列中去，这样会增加处理每个消息的平均开销。因为要花更多的时间和资源处

  理“堆积”的消息，如此用来处理新流入的消息的能力就会降低，使得后流入的消息又被积压到很深的队列中，继续增大处理每个消息的平均开销，继而情况变

  得越来越恶化，使得系统的处理能力大大降低。

  应对这一问题一般有3种措施：

  1. 增加prefetch_count的值，即一次发送多条消息给消费者，加快消息被消费的速度。

  2. 采用multiple ack，降低处理 ack 带来的开销

  3. 流量控制

### 2.2、安装和配置RabbitMQ

安装环境：

- 操作系统：**CentOS Linux release 7.9**

- Erlang：**erlang-23.0.2-1.el7.x86_64**

- RabbitMQ：**rabbitmq-server-3.8.4-1.el7.noarch**

RabbitMQ的安装需要首先安装Erlang，因为它是基于Erlang的VM运行的。

RabbitMQ需要的依赖：socat和logrotate，logrotate操作系统中已经存在了，只需要安装socat就可以了。

RabbitMQ与Erlang的兼容关系详见：https://www.rabbitmq.com/which-erlang.html

1. 安装依赖：

   ```sh
   yum install socat -y
   ```

2. 安装Erlang

   ```shell
   wget https://github.com/rabbitmq/erlang-rpm/releases/download/v23.0.2/erlang-23.0.2-1.el7.x86_64.rpm
   # 或者下载后上传
   rpm -ivh erlang-23.0.2-1.el7.x86_64.rpm
   ```

3. 安装RabbitMQ

   ```shell
   wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.5/rabbitmq-server-3.8.5-1.el7.noarch.rpm
   # 或者下载后上传
   rpm -ivh rabbitmq-server-3.8.5-1.el7.noarch.rpm
   ```

4. 启用RabbitMQ的管理插件

   ```apl
   cd /usr/lib/rabbitmq/lib/rabbitmq_server-3.8.5/sbin
   rabbitmq-plugins enable rabbitmq_management
   ```

5. 开启RabbitMQ

   ```apl
   chkconfig rabbitmq-server on  # 添加开机启动RabbitMQ服务
   systemctl start rabbitmq-server # 启动
   systemctl stop rabbitmq-server # 停止
   rabbitmq-server #前台启动
   rabbitmq-server -detached # 后台启动
   ```

6. 添加用户

   ```apl
   rabbitmqctl add_user root 123456
   ```

7. 给用户添加权限

   ```apl
   rabbitmqctl set_permissions root -p / ".*" ".*" ".*"
   ```

8. 给用户设置标签

   ```apl
   rabbitmqctl set_user_tags root administrator
   ```

   用户的标签和权限：

   | Tag           | Capabilities                                                 |
   | ------------- | ------------------------------------------------------------ |
   | (None)        | 没有访问management插件的权限                                 |
   | management    | 可以使用消息协议做任何操作的权限，加上：<br/>1. 可以使用AMQP协议登录的虚拟主机的权限<br/>2. 查看它们能登录的所有虚拟主机中所有队列、交换器和绑定的权限<br/>3. 查看和关闭它们自己的通道和连接的权限<br/>4. 查看它们能访问的虚拟主机中的全局统计信息，包括其他用户的活动 |
   | policymaker   | 所有management标签可以做的，加上：<br/>1. 在它们能通过AMQP协议登录的虚拟主机上，查看、创建和删除策略以及虚拟主机参数的权限 |
   | monitoring    | 所有management能做的，加上：<br/>1. 列出所有的虚拟主机，包括列出不能使用消息协议访问的虚拟主机的权限<br/>2. 查看其他用户连接和通道的权限<br/>3. 查看节点级别的数据如内存使用和集群的权限<br/>4. 查看真正的全局所有虚拟主机统计数据的权限 |
   | administrator | 所有policymaker和monitoring能做的，加上：<br/>1. 创建删除虚拟主机的权限<br/>2. 查看、创建和删除用户的权限<br/>3. 查看、创建和删除权限的权限<br/>4. 关闭其他用户连接的权限 |

9. 打开浏览器，**访问http://<安装了CentOS的VMWare虚拟机IP地址>:15672**

10. 使用刚才创建的用户登录

### 2.3、RabbitMQ常用操作命令

```shell
# 前台启动Erlang VM和RabbitMQ 
rabbitmq-server 
# 后台启动 
rabbitmq-server -detached
# 停止RabbitMQ和Erlang VM 
rabbitmqctl stop 
# 查看所有队列 
rabbitmqctl list_queues 
# 查看所有虚拟主机 
rabbitmqctl list_vhosts 
# 在Erlang VM运行的情况下启动RabbitMQ应用
rabbitmqctl start_app 
rabbitmqctl stop_app 
# 查看节点状态 
rabbitmqctl status 
# 查看所有可用的插件 
rabbitmq-plugins list 
# 启用插件 
rabbitmq-plugins enable <plugin-name> 
# 停用插件
rabbitmq-plugins disable <plugin-name>
# 添加用户 
rabbitmqctl add_user username password 
# 列出所有用户： 
rabbitmqctl list_users
# 删除用户：
rabbitmqctl delete_user username
# 清除用户权限：
rabbitmqctl clear_permissions -p vhostpath username 
# 列出用户权限：
rabbitmqctl list_user_permissions username 
# 修改密码： 
rabbitmqctl change_password username newpassword 
# 设置用户权限： 
rabbitmqctl set_permissions -p vhostpath username ".*" ".*" ".*" 
# 创建虚拟主机: 
rabbitmqctl add_vhost vhostpath
# 列出所以虚拟主机: 
rabbitmqctl list_vhosts 
# 列出虚拟主机上的所有权限: 
rabbitmqctl list_permissions -p vhostpath 
# 删除虚拟主机: 
rabbitmqctl delete_vhost vhost vhostpath 
# 移除所有数据，要在 rabbitmqctl stop_app 之后使用:
rabbitmqctl reset
```

### 2.4、RabbitMQ工作流程详解

#### 2.4.1、生产者发送消息的流程

1. 生产者连接RabbitMQ，建立TCP连接( Connection)，开启信道（Channel） 

2. 生产者声明一个Exchange（交换器），并设置相关属性，比如交换器类型、是否持久化等

3. 生产者声明一个队列井设置相关属性，比如是否排他、是否持久化、是否自动删除等

4. 生产者通过 bindingKey （绑定Key）将交换器和队列绑定（ binding ）起来

5. 生产者发送消息至RabbitMQ Broker，其中包含 routingKey （路由键）、交换器等信息

6. 相应的交换器根据接收到的 routingKey 查找相匹配的队列。

7. 如果找到，则将从生产者发送过来的消息存入相应的队列中。

8. 如果没有找到，则根据生产者配置的属性选择丢弃还是回退给生产者

9. 关闭信道。

10. 关闭连接。

#### 2.4.2、消费者接收消息的过程

1. 消费者连接到RabbitMQ Broker ，建立一个连接(Connection ) ，开启一个信道(Channel) 。 

2. 消费者向RabbitMQ Broker 请求消费相应队列中的消息，可能会设置相应的回调函数， 以及做一些准备工作

3. 等待RabbitMQ Broker 回应并投递相应队列中的消息， 消费者接收消息。

4. 消费者确认( ack) 接收到的消息。

5. RabbitMQ 从队列中删除相应己经被确认的消息。

6. 关闭信道。

7. 关闭连接。

#### 2.4.3、具体案例

- 创建maven工程

- 添加依赖

  ```xml
  <dependency>
      <groupId>com.rabbitmq</groupId>
      <artifactId>amqp-client</artifactId>
      <version>5.9.0</version>
  </dependency>
  ```

- 创建消息生产者

  ```java
  package com.thm;
  
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  public class Producer {
      //定义消息队列名称
      private final static String QUEUE_NAME = "queue.biz";
      //定义交换器名称
      private final static String EX_NAME = "ex.biz";
      //定义绑定key
      private final static String ROUTE_KEY = "hello.word";
      public static void main(String[] args) {
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          // 设置服务器主机名或IP地址
          connectionFactory.setHost("192.168.81.100");
          // 设置客户端与服务器的通信端口，默认值为5672
          connectionFactory.setPort(5672);
          // 设置用户名和密码
          connectionFactory.setUsername("root");
          connectionFactory.setPassword("123456");
          // 设置Erlang的虚拟主机名称
          connectionFactory.setVirtualHost("/");
          //获取连接
          try (Connection conn = connectionFactory.newConnection();
               //创建通道
               Channel channel = conn.createChannel()) {
              /**
               * 定义消息队列，队列名称，是否持久化，是否排他，是否自动删除，消息队列属性map
               */
              channel.queueDeclare(QUEUE_NAME, false, false, true, null);
              /**
               * 定义交换器，交换器名称，交换器类型，是否持久化，是否自动删除，交换器属性map
               */
              channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.DIRECT,false,true,null);
              /**
               * 将交换器和消息队列进行绑定，并指定路由键
               */
              channel.queueBind(QUEUE_NAME,EX_NAME,ROUTE_KEY);
              //发送消息 交换器名称，路由键，属性，消息体
              String message = "Hello World2!";
              channel.basicPublish(EX_NAME, ROUTE_KEY, null, message.getBytes());
              System.out.println(" [x] Sent '" + message + "'");
          } catch (Exception e) {
  
          }
      }
  }
  ```

- 创建拉模式消费者

  ```java
  package com.thm;
  
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  import com.rabbitmq.client.GetResponse;
  
  import java.io.IOException;
  import java.net.URISyntaxException;
  import java.security.KeyManagementException;
  import java.security.NoSuchAlgorithmException;
  import java.util.concurrent.TimeoutException;
  
  public class PullConsumer {
      //定义消息队列名称
      private final static String QUEUE_NAME = "queue.biz";
      //定义交换器名称
      private final static String EX_NAME = "ex.biz";
      //定义绑定key
      private final static String ROUTE_KEY = "hello.word";
      public static void main(String[] args) throws NoSuchAlgorithmException, KeyManagementException, URISyntaxException, IOException, TimeoutException {
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          /**
           * 使用uri形式，amqp：//账号：密码@IP地址:端口/虚拟主机
           * %2f  =  /
           */
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          //获取连接
          Connection connection = connectionFactory.newConnection();
          //创建通道
          Channel channel = connection.createChannel();
          /**
           * 从指定消息队列中获取消息，消息队列名称，是否自动确认
           */
          GetResponse getResponse = channel.basicGet(QUEUE_NAME, true);
          byte[] body = getResponse.getBody();
          System.out.println(new String(body));
          channel.close();
          connection.close();
      }
  }
  ```

- 创建推模式消费者

  ```java
  package com.thm;
  
  import com.rabbitmq.client.*;
  
  import java.io.IOException;
  import java.net.URISyntaxException;
  import java.security.KeyManagementException;
  import java.security.NoSuchAlgorithmException;
  import java.util.concurrent.TimeoutException;
  public class PushConsumer {
      //定义消息队列名称
      private final static String QUEUE_NAME = "queue.biz";
      //定义交换器名称
      private final static String EX_NAME = "ex.biz";
      //定义绑定key
      private final static String ROUTE_KEY = "hello.word";
      public static void main(String[] args) throws NoSuchAlgorithmException, KeyManagementException, URISyntaxException, IOException, TimeoutException {
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          /**
           * 使用uri形式，amqp：//账号：密码@IP地址:端口/虚拟主机
           * %2f  =  /
           */
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          //获取连接
          Connection connection = connectionFactory.newConnection();
          //创建通道
          Channel channel = connection.createChannel();
          /**
           * 确保MQ中存在该消息队列，没有则创建
           * 定义消息队列，队列名称，是否持久化，是否排他，是否自动删除，消息队列属性map
           */
          channel.queueDeclare(QUEUE_NAME, false, false, true, null);
          /**
           * 使用服务器生成的consumerTag启动本地，非排他的使用者。
           * 启动一个 仅提供了basic.deliver和basic.cancel AMQP方法（对大多数情形够用了）
           * 第一个参数：队列名称 autoAck – true 只要服务器发送了消息就表示消息已经被消费者确认; false服务 端等待客户端显式地发送确认消息
           * deliverCallback – 服务端推送过来的消息回调函数
           * cancelCallback – 客户端忽略该消息的回调方法
           * Returns: 服务端生成的consumerTag
           */
          channel.basicConsume(QUEUE_NAME,  (consumerTag, delivery) -> {
              System.out.println(new String(delivery.getBody()));
          },consumerTag -> { });
      }
  }
  ```

#### 2.4.4、Connection 和Channel关系

生产者和消费者，需要与RabbitMQ Broker 建立TCP连接，也就是Connection 。一旦TCP 连接建立起来，客户端紧接着创建一个AMQP 信道（Channel），每个

信道都会被指派一个唯一的ID。信道是建立在Connection 之上的虚拟连接， RabbitMQ 处理的每条AMQP 指令都是通过信道完成的。

![](../img/RabbitMQ/RabbitMQ-img-21.png)

为什么不直接使用TCP连接，而是使用信道？

- RabbitMQ 采用类似NIO的做法，复用TCP 连接，减少性能开销，便于管理。

- 当每个信道的流量不是很大时，复用单一的Connection 可以在产生性能瓶颈的情况下有效地节省TCP 连接资源。

当信道本身的流量很大时，一个Connection 就会产生性能瓶颈，流量被限制。需要建立多个Connection ，分摊信道。具体的调优看业务需要。

信道在AMQP 中是一个很重要的概念，大多数操作都是在信道这个层面进行的。

```apl
channel.exchangeDeclare 
channel.queueDeclare 
channel.basicPublish 
channel.basicConsume 
// ...
```

RabbitMQ 相关的API与AMQP紧密相连，比如channel.basicPublish 对应AMQP 的Basic.Publish命令。

#### 2.4.5、RabbitMQ工作模式详解

官网地址：https://www.rabbitmq.com/getstarted.htm

##### 2.4.5.1、direct Work Queue

生产者发消息，启动多个消费者实例来消费消息，每个消费者仅消费部分信息，可达到负载均衡的效果。（轮询）

- 添加依赖

  ```xml
  <dependency>
      <groupId>com.rabbitmq</groupId>
      <artifactId>amqp-client</artifactId>
      <version>5.9.0</version>
  </dependency>
  ```

- work queue 生产者

  ```java
  package com.thm;
  
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  public class WorkProducer {
      //定义消息队列名称
      private final static String QUEUE_NAME = "queue.wq";
      //定义交换器名称
      private final static String EX_NAME = "ex.wq";
      //定义绑定key
      private final static String ROUTE_KEY = "wq.wq";
      public static void main(String[] args) {
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          // 设置服务器主机名或IP地址
          connectionFactory.setHost("192.168.81.100");
          // 设置客户端与服务器的通信端口，默认值为5672
          connectionFactory.setPort(5672);
          // 设置用户名和密码
          connectionFactory.setUsername("root");
          connectionFactory.setPassword("123456");
          // 设置Erlang的虚拟主机名称
          connectionFactory.setVirtualHost("/");
          //获取连接
          try (Connection conn = connectionFactory.newConnection();
               //创建通道
               Channel channel = conn.createChannel()) {
              /**
               * 定义消息队列，队列名称，是否持久化，是否排他，是否自动删除，消息队列属性map
               */
              channel.queueDeclare(QUEUE_NAME, false, false, true, null);
              /**
               * 定义交换器，交换器名称，交换器类型，是否持久化，是否自动删除，交换器属性map
               */
              channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.DIRECT,false,true,null);
              /**
               * 将交换器和消息队列进行绑定，并指定路由键
               */
              channel.queueBind(QUEUE_NAME,EX_NAME,ROUTE_KEY);
              //发送消息 交换器名称，路由键，属性，消息体
              for (int i = 0; i < 15; i++) {
                  channel.basicPublish(EX_NAME, ROUTE_KEY, null, ("工作队列模式消息：" + i).getBytes());
              }
          } catch (Exception e) {
  
          }
      }
  }
  ```

- work queue 消费者

  启动多个实例接收，轮询接收消息
  
  ```java
  package com.thm;
  
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  
  import java.io.IOException;
  import java.net.URISyntaxException;
  import java.security.KeyManagementException;
  import java.security.NoSuchAlgorithmException;
  import java.util.concurrent.TimeoutException;
  public class WorkPushConsumer {
      //定义消息队列名称
      private final static String QUEUE_NAME = "queue.wq";
      public static void main(String[] args) throws NoSuchAlgorithmException, KeyManagementException, URISyntaxException, IOException, TimeoutException {
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          /**
           * 使用uri形式，amqp：//账号：密码@IP地址:端口/虚拟主机
           * %2f  =  /
           */
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          //获取连接
          Connection connection = connectionFactory.newConnection();
          //创建通道
          Channel channel = connection.createChannel();
          /**
           * 确保MQ中存在该消息队列，没有则创建
           * 定义消息队列，队列名称，是否持久化，是否排他，是否自动删除，消息队列属性map
           */
          channel.queueDeclare(QUEUE_NAME, false, false, true, null);
          /**
           * 使用服务器生成的consumerTag启动本地，非排他的使用者。
           * 启动一个 仅提供了basic.deliver和basic.cancel AMQP方法（对大多数情形够用了）
           * 第一个参数：队列名称 autoAck – true 只要服务器发送了消息就表示消息已经被消费者确认; false服务 端等待客户端显式地发送确认消息
           * deliverCallback – 服务端推送过来的消息回调函数
           * cancelCallback – 客户端忽略该消息的回调方法
           * Returns: 服务端生成的consumerTag
           */
          channel.basicConsume(QUEUE_NAME,  (consumerTag, delivery) -> {
              System.out.println(new String(delivery.getBody()));
          },consumerTag -> {
              System.out.println(consumerTag);
          });
      }
}
  ```
  

##### 2.4.5.2、发布订阅模式

使用fanout类型交换器，routingKey忽略。每个消费者定义生成一个队列并绑定到同一个Exchange，每个消费者都可以消费到完整的消息。

消息广播给所有订阅该消息的消费者。

在RabbitMQ中，生产者不是将消息直接发送给消息队列，实际上生产者根本不知道一个消息被发送到哪个队列。

生产者将消息发送给交换器。交换器非常简单，从生产者接收消息，将消息推送给消息队列。交换器必须清楚地知道要怎么处理接收到的消息。应该是追加到一个

指定的队列，还是追加到多个队列，还是丢弃。规则就是交换器类型。

![](../img/RabbitMQ/RabbitMQ-img-23.png)

交换器的类型前面已经介绍过了： direct 、 topic 、 headers 和 fanout 四种类型。发布订阅使用fanout。创建交换器，名字叫 ex.myFan： 

```java
//定义消息队列名称
private final static String EX_NAME = "ex.myFan";
//声明fanout类型的交换器
channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.FANOUT,false,true,null);
```

fanout 交换器很简单，从名字就可以看出来（用风扇吹出去），将所有收到的消息发送给它知道的所有的队列。

```shell
rabbitmqctl list_exchanges --formatter pretty_table
# 列出RabbitMQ的交换器，包括了 amq.* 的和默认的（未命名）的交换器
```

![](../img/RabbitMQ/RabbitMQ-img-24.png)

- 未命名交换器

  在前面的那里中我们没有指定交换器，但是依然可以向队列发送消息。这是因为我们使用了默认的交换器。

  ```java
  channel.basicPublish("", "hello", null, message.getBytes());
  ```

  第一个参数就是交换器名称，为空字符串。直接使用routingKey向队列发送消息，如果该routingKey指定的队列存在的话。routingKey = 消息队列名称

- 指定的交换器

  ```java
  channel.basicPublish("ex.myFan", "", null, message.getBytes());
  ```

**临时队列**

前面我们使用队列的名称，生产者和消费者都是用该名称来发送和接收该队列中的消息。

首先，我们无论何时连接RabbitMQ的时候，都需要一个新的，空的队列。我们可以使用随机的名字创建队列，也可以让服务器帮我们生成随机的消息队列名字。

其次，一旦我们断开到消费者的连接，该队列应该自动删除。

```java
String queueName = channel.queueDeclare().getQueue(); 
//上述代码我们声明了一个非持久化的、排他的、自动删除的队列，并且名字是服务器随机生成的。
//queueName一般的格式类似： amq.gen-JzTY20BRgKO-HjmUJj0wLg 。
```

**绑定**

![](../img/RabbitMQ/RabbitMQ-img-25.png)

在创建了消息队列和 fanout 类型的交换器之后，我们需要将两者进行**绑定**，让交换器将消息发送给该队列。

```java
channel.queueBind(queueName,"ex.myFan","");
```

此时， ex.myFan交换器会将接收到的消息追加到我们的队列中。

可以使用下述命令列出RabbitMQ中交换器的绑定关系：

```shell
rabbitmqctl list_bindings --formatter pretty_table
```

![](../img/RabbitMQ/RabbitMQ-img-26.png)

**发布订阅模式的整体代码如下：**

![](../img/RabbitMQ/RabbitMQ-img-27.png)

- 添加依赖

  ```xml
  <dependency>
      <groupId>com.rabbitmq</groupId>
      <artifactId>amqp-client</artifactId>
      <version>5.9.0</version>
  </dependency>
  ```

- 创建消息生产者

  ```java
  package com.lagou;
  
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  public class FanoutProducer {
      //定义交换器名称
      private final static String EX_NAME = "ex.myFan";
  
      public static void main(String[] args) throws Exception{
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          /**
           * 使用uri形式，amqp：//账号：密码@IP地址:端口/虚拟主机
           * %2f  =  /
           */
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          //获取连接
          Connection connection = connectionFactory.newConnection();
          //创建通道
          Channel channel = connection.createChannel();
          //声明fanout类型的交换器
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.FANOUT,false,true,null);
  
          for (int i = 0; i < 5; i++) {
              channel.basicPublish(EX_NAME,
                      //fanout类型的交换器不需要指定路由键
                      "",
                      null,
                      ("hello fanout:" + i).getBytes());
  
          }
          channel.close();
          connection.close();
      }
  }
  ```

- 创建消息消费者

  ```java
  package com.lagou;
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  import java.io.IOException;
  import java.net.URISyntaxException;
  import java.security.KeyManagementException;
  import java.security.NoSuchAlgorithmException;
  import java.util.concurrent.TimeoutException;
  public class OneConsumer {
      //定义交换器名称
      private final static String EX_NAME = "ex.myFan";
      public static void main(String[] args) throws NoSuchAlgorithmException, KeyManagementException, URISyntaxException, IOException, TimeoutException {
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          /**
           * 使用uri形式，amqp：//账号：密码@IP地址:端口/虚拟主机
           * %2f  =  /
           */
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          //获取连接
          Connection connection = connectionFactory.newConnection();
          //创建通道
          Channel channel = connection.createChannel();
          //声明fanout类型的交换器
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.FANOUT,false,true,null);
          //创建临时消息队列
          String QUEUE_NAME = channel.queueDeclare().getQueue();
          System.out.println("创建临时消息队列:" + QUEUE_NAME);
          //绑定交换器
          channel.queueBind(QUEUE_NAME,EX_NAME,"");
          /**
           * 使用服务器生成的consumerTag启动本地，非排他的使用者。
           * 启动一个 仅提供了basic.deliver和basic.cancel AMQP方法（对大多数情形够用了）
           * 第一个参数：队列名称 autoAck – true 只要服务器发送了消息就表示消息已经被消费者确认; false服务 端等待客户端显式地发送确认消息
           * deliverCallback – 服务端推送过来的消息回调函数
           * cancelCallback – 客户端忽略该消息的回调方法
           * Returns: 服务端生成的consumerTag
           */
          channel.basicConsume(QUEUE_NAME,  (consumerTag, delivery) -> {
              System.out.println("One     " + new String(delivery.getBody()));
          },consumerTag -> {
              System.out.println(consumerTag);
          });
      }
  }
  ```

##### 2.4.5.3、direct路由模式

使用 direct 类型的Exchange，发N条消费并使用不同的 routingKey ，消费者定义队列并将队列、 routingKey 、Exchange绑定。此时使用 direct 模式Exchagne

必须要 routingKey 完全匹配的情况下消息才会转发到对应的队列中被消费。

上一个模式中，可以将消息广播到很多接收者。

现在我们想让接收者只接收部分消息，如，我们通过直接模式的交换器将关键的错误信息记录到log文件，同时在控制台正常打印所有的日志信息。

**绑定**

```java
channel.queueBind(queueName, EXCHANGE_NAME, "black");
```

bindingKey 的作用与具体使用的交换器类型有关。对于 fanout 类型的交换器，此参数设置无效，系统直接忽略。

分布式系统中有很多应用，这些应用需要运维平台的监控，其中一个重要的信息就是服务器的日志记录，我们需要将不同日志级别的日志记录交给不同的应用处理。

**使用direct交换器**

我们需要使用 direct 类型的交换器。 direct 交换器的路由算法很简单：只要消息的routingKey 和队列的 bindingKey 对应，消息就可以推送给该队列。

![](../img/RabbitMQ/RabbitMQ-img-28.png)

上图中的交换器 X 是 direct 类型的交换器，绑定的两个队列中，一个队列的 bindingKey 是 orange ，另一个队列的 bindingKey 是 black 和 green 。

如此，则 routingKey 是 orange 的消息发送给队列Q1， routingKey 是 black 和 green 的消息发送给Q2队列，其他消息丢弃。

**多重绑定**

![](../img/RabbitMQ/RabbitMQ-img-29.png)

上图中，我们使用 direct 类型的交换器 X ，建立了两个绑定：队列Q1根据 bindingKey 的值black 绑定到交换器 X ，队列Q2根据 bindingKey 的值 black 绑定到交换器 X ；交换器 X 会将消息发送给队列Q1和队列Q2。交换器的行为跟 fanout 的行为类似，也是广播。

在案例中，我们将日志级别作为 routingKey 。

![](../img/RabbitMQ/RabbitMQ-img-30.png)

- 添加依赖

  ```xml
  <dependency>
      <groupId>com.rabbitmq</groupId>
      <artifactId>amqp-client</artifactId>
      <version>5.9.0</version>
  </dependency>
  ```

- 创建消息生产者

  ```java
  package com.thm;
  
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  
  import java.util.Arrays;
  import java.util.List;
  import java.util.Random;
  public class Producer {
  
      private static final List<String> LOG_LEVEL = Arrays.asList("INFO","WARN","ERROR");
  
      private static Random random = new Random();
      //定义交换器名称
      private final static String EX_NAME = "ex.route";
      public static void main(String[] args) throws Exception{
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          /**
           * 使用uri形式，amqp：//账号：密码@IP地址:端口/虚拟主机
           * %2f  =  /
           */
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          //获取连接
          Connection connection = connectionFactory.newConnection();
          //创建通道
          Channel channel = connection.createChannel();
          //声明交换器
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.DIRECT,false,true,null);
          for (int i = 0; i < 100; i++) {
              String level = LOG_LEVEL.get(random.nextInt(100) % LOG_LEVEL.size());
              //发送消息
              channel.basicPublish(EX_NAME,level,null,("这是【" + level + "】的消息").getBytes());
          }
          channel.close();
          connection.close();
      }
  }
  ```

- 创建INFO消息消费者

  ```java
  package com.thm;
  
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  public class InfoConsumer {
      //定义交换器名称
      private final static String EX_NAME = "ex.route";
      //定义消息队列名称
      private final static String QUEUE_NAME = "queue.info";
      //定义路由KEY
      private final static String ROUTE_KEY = "INFO";
      public static void main(String[] args) throws Exception{
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          /**
           * 使用uri形式，amqp：//账号：密码@IP地址:端口/虚拟主机
           * %2f  =  /
           */
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          //获取连接
          Connection connection = connectionFactory.newConnection();
          //创建通道
          Channel channel = connection.createChannel();
          //声明交换器
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.DIRECT,false,true,null);
          //声明消息队列
          channel.queueDeclare(QUEUE_NAME,false,false,true,null);
          //消息队列绑定
          channel.queueBind(QUEUE_NAME,EX_NAME,ROUTE_KEY);
          //接收消息
          channel.basicConsume(QUEUE_NAME,  (consumerTag, delivery) -> {
              System.out.println(new String(delivery.getBody()));
          },consumerTag -> {
              System.out.println(consumerTag);
          });
  
      }
  }
  ```

- 创建WARN消息消费者

  ```java
  package com.thm;
  
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  public class WarnConsumer {
      //定义交换器名称
      private final static String EX_NAME = "ex.route";
      //定义消息队列名称
      private final static String QUEUE_NAME = "queue.warn";
      //定义路由KEY
      private final static String ROUTE_KEY = "WARN";
      public static void main(String[] args) throws Exception{
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          /**
           * 使用uri形式，amqp：//账号：密码@IP地址:端口/虚拟主机
           * %2f  =  /
           */
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          //获取连接
          Connection connection = connectionFactory.newConnection();
          //创建通道
          Channel channel = connection.createChannel();
          //声明交换器
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.DIRECT,false,true,null);
          //声明消息队列
          channel.queueDeclare(QUEUE_NAME,false,false,true,null);
          //消息队列绑定
          channel.queueBind(QUEUE_NAME,EX_NAME,ROUTE_KEY);
          //接收消息
          channel.basicConsume(QUEUE_NAME,  (consumerTag, delivery) -> {
              System.out.println(new String(delivery.getBody()));
          },consumerTag -> {
              System.out.println(consumerTag);
          });
  
      }
  }
  ```

- 创建ERROR消息消费者

  ```java
  package com.thm;
  
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  public class ErrorConsumer {
      //定义交换器名称
      private final static String EX_NAME = "ex.route";
      //定义消息队列名称
      private final static String QUEUE_NAME = "queue.error";
      //定义路由KEY
      private final static String ROUTE_KEY = "ERROR";
      public static void main(String[] args) throws Exception{
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          /**
           * 使用uri形式，amqp：//账号：密码@IP地址:端口/虚拟主机
           * %2f  =  /
           */
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          //获取连接
          Connection connection = connectionFactory.newConnection();
          //创建通道
          Channel channel = connection.createChannel();
          //声明交换器
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.DIRECT,false,true,null);
          //声明消息队列
          channel.queueDeclare(QUEUE_NAME,false,false,true,null);
          //消息队列绑定
          channel.queueBind(QUEUE_NAME,EX_NAME,ROUTE_KEY);
          //接收消息
          channel.basicConsume(QUEUE_NAME,  (consumerTag, delivery) -> {
              System.out.println(new String(delivery.getBody()));
          },consumerTag -> {
              System.out.println(consumerTag);
          });
  
      }
  }
  ```

##### 2.4.5.4、Topic 主题模式

使用 topic 类型的交换器，队列绑定到交换器、 bindingKey 时使用通配符，交换器将消息路由转发到具体队列时会根据消息 routingKey 模糊匹配，比较灵活。

上个模式中，我们通过 direct 类型的交换器做到了根据日志级别的不同，将消息发送给了不同队列的。

这里有一个限制，加入现在我不仅想根据日志级别划分日志消息，还想根据日志来源划分日志，怎么做？

比如，我想监听cron服务发送的 error 消息，又想监听从kern服务发送的所有消息。此时可以使用RabbitMQ的主题模式（ Topic ）。

要想 topic 类型的交换器， routingKey 就不能随便写了，它必须得是点分单词。单词可以随便写，生产中一般使用消息的特征。

如：“stock.usd.nyse”，“nyse.vmw”，“quick.orange.rabbit”等。**该点分单词字符串最长255字节。**

bindingKey 也必须是这种形式。 topic 类型的交换器背后原理跟 direct 类型的类似：只要队列的 bindingKey 的值与消息的 routingKey 匹配，队列就可以收到该

消息。有两个不同：

1. `*`(star)匹配一个单词

2. `#` 匹配0到多个单词

![](../img/RabbitMQ/RabbitMQ-img-31.png)

上图中，我们发送描述动物的消息。消息发送的时候指定的 routingKey 包含了三个词，两个点。

第一个单词表示动物的速度，第二个是颜色，第三个是物种：`<speed>.<color>.<species>`。 

**案例**

- 添加依赖

  ```xml
  <dependency>
      <groupId>com.rabbitmq</groupId>
      <artifactId>amqp-client</artifactId>
      <version>5.9.0</version>
  </dependency>
  ```

- 创建生产者

  ```java
  package com.thm;
  
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  import java.util.Arrays;
  import java.util.List;
  import java.util.Random;
  public class Producer {
  
      private static final List<String> LOG_LEVEL = Arrays.asList("info","warn","error");
      private static final List<String> LOG_AREA = Arrays.asList("chengdu","shanghai","beijing");
      private static final List<String> LOG_BIZ = Arrays.asList("edu-online","biz-online","emp-online");
  
      private static Random random = new Random();
      //定义交换器名称
      private final static String EX_NAME = "ex.topic";
      public static void main(String[] args) throws Exception{
          ConnectionFactory connectionFactory = new ConnectionFactory();
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          Connection connection = connectionFactory.newConnection();
          Channel channel = connection.createChannel();
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.TOPIC,false,true,null);
          
          String level, area, biz, routeKey, message;
          for (int i = 0; i < 100; i++) {
              level = LOG_LEVEL.get(random.nextInt(100) % LOG_LEVEL.size());
              area = LOG_AREA.get(random.nextInt(100) % LOG_LEVEL.size());
              biz = LOG_BIZ.get(random.nextInt(100) % LOG_LEVEL.size());
              routeKey = area + "." + biz + "." + level;
              message = "LOG：【" + level + "】- 这是：【" + area + "】地区" + biz + "服务器发送的消息，MSG-INDEX = " + i;
              channel.basicPublish(EX_NAME,routeKey,null,message.getBytes());
          }
          channel.close();
          connection.close();
      }
  }
  ```

- 创建chengdu日志消费者

  ```java
  package com.thm;
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  public class ChengduConsumer {
      //定义交换器名称
      private final static String EX_NAME = "ex.topic";
      public static void main(String[] args) throws Exception{
          ConnectionFactory connectionFactory = new ConnectionFactory();
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          Connection connection = connectionFactory.newConnection();
          Channel channel = connection.createChannel();
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.TOPIC,false,true,null);
          //创建临时对象
          String queueName = channel.queueDeclare().getQueue();
          //查看chengdu所有信息  routingkey以chengdu开头所有日志信息
          channel.queueBind(queueName,EX_NAME,"chengdu.#");
          //接收消息
          channel.basicConsume(queueName,  (consumerTag, delivery) -> {
              System.out.println(new String(delivery.getBody()));
          },consumerTag -> {
              System.out.println(consumerTag);
          });
      }
  }
  ```

- 创建shanghai.biz-onliner日志消费者

  ```java
  package com.thm;
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  public class ShanghaiConsumer {
      //定义交换器名称
      private final static String EX_NAME = "ex.topic";
      public static void main(String[] args) throws Exception{
          ConnectionFactory connectionFactory = new ConnectionFactory();
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          Connection connection = connectionFactory.newConnection();
          Channel channel = connection.createChannel();
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.TOPIC,false,true,null);
          //创建临时对象
          String queueName = channel.queueDeclare().getQueue();
          //routingkey以shanghai.biz-online开头所有日志信息
          channel.queueBind(queueName,EX_NAME,"shanghai.biz-online.*");
          //接收消息
          channel.basicConsume(queueName,  (consumerTag, delivery) -> {
              System.out.println(new String(delivery.getBody()));
          },consumerTag -> {
              System.out.println(consumerTag);
          });
      }
  }
  ```

- 创建beijing.*.error日志消费者

  ```java
  package com.thm;
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  public class beijingConsumer {
      //定义交换器名称
      private final static String EX_NAME = "ex.topic";
      public static void main(String[] args) throws Exception{
          ConnectionFactory connectionFactory = new ConnectionFactory();
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          Connection connection = connectionFactory.newConnection();
          Channel channel = connection.createChannel();
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.TOPIC,false,true,null);
          //创建临时对象
          String queueName = channel.queueDeclare().getQueue();
          //routingkey   beijing.*.error 查看beijing所有错误信息
          channel.queueBind(queueName,EX_NAME,"beijing.*.error");
          //接收消息
          channel.basicConsume(queueName,  (consumerTag, delivery) -> {
              System.out.println(new String(delivery.getBody()));
          },consumerTag -> {
              System.out.println(consumerTag);
          });
      }
  }
  ```

- 创建#.error日志消费者

  ```java
  package com.thm;
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  public class ErrorConsumer {
      //定义交换器名称
      private final static String EX_NAME = "ex.topic";
      public static void main(String[] args) throws Exception{
          ConnectionFactory connectionFactory = new ConnectionFactory();
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          Connection connection = connectionFactory.newConnection();
          Channel channel = connection.createChannel();
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.TOPIC,false,true,null);
          //创建临时对象
          String queueName = channel.queueDeclare().getQueue();
          //routingkey以error结尾所有日志信息
          channel.queueBind(queueName,EX_NAME,"#.error");
          //接收消息
          channel.basicConsume(queueName,  (consumerTag, delivery) -> {
              System.out.println(new String(delivery.getBody()));
          },consumerTag -> {
              System.out.println(consumerTag);
          });
      }
  }
  ```

##### 2.4.5.5、 Spring xml整合RabbitMQ

**spring-amqp**是对AMQP的一些概念的一些抽象，**spring-rabbit**是对RabbitMQ操作的封装实现。

主要有几个核心类 RabbitAdmin 、 RabbitTemplate 、 SimpleMessageListenerContainer 等 RabbitAdmin 类完成对Exchange，Queue，Binding的操作，在

容器中管理了 RabbitAdmin 类的时候，可以对Exchange，Queue，Binding进行自动声明。

RabbitTemplate 类是发送和接收消息的工具类。

SimpleMessageListenerContainer 是消费消息的容器。

目前比较新的一些项目都会选择基于注解方式，而比较老的一些项目可能还是基于配置文件的。

**基于配置文件的整合**

- 创建maven项目

- 添加依赖

  ```xml
  <dependency>
      <groupId>org.springframework.amqp</groupId>
      <artifactId>spring-rabbit</artifactId>
      <version>2.2.7.RELEASE</version>
  </dependency>
  ```

- 添加配置文件

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns:rabbit="http://www.springframework.org/schema/rabbit"
         xsi:schemaLocation="http://www.springframework.org/schema/beans
         http://www.springframework.org/schema/beans/spring-beans.xsd
         http://www.springframework.org/schema/rabbit
         http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">
      <!--创建连接工厂-->
      <rabbit:connection-factory id="connectionFactory"
                                 host="192.168.81.100"
                                 port="5672"
                                 virtual-host="/"
                                 username="root"
                                 password="123456"/>
      <!--自动查找类型是Queue、Exchange、Binding的bean，并为用户向 RabbitMQ声明-->
      <rabbit:admin id="rabbitAdmin" connection-factory="connectionFactory"/>
      <!--创建一个rabbit的 template对象-->
      <rabbit:template id="rabbitTemplate" connection-factory="connectionFactory"/>
      <!--声明交换器-->
      <rabbit:direct-exchange id="myExchange" name="ex.spring.xml" durable="false" auto-delete="true">
          <rabbit:bindings>
              <!--exchange：其他绑定到该交换器的交换器名称-->
              <!--queue：绑定到该交换器的queue的bean名称 beanId-->
              <!--key：显式声明的路由key-->
              <rabbit:binding queue="myQueue" key="myQueue"/>
          </rabbit:bindings>
      </rabbit:direct-exchange>
      <!--声明一个消息队列-->
      <rabbit:queue id="myQueue" name="queue.spring.xml" durable="false"  exclusive="false" auto-delete="true"/>
  
      <!--配置自定义监听实现-->
      <bean id="messageListener" class="MyMessageListener"/>
      <!--推送消息模式，监听配置-->
      <rabbit:listener-container connection-factory="connectionFactory">
          <rabbit:listener ref="messageListener" queues="myQueue" exclusive="false"/>
      </rabbit:listener-container>
  </beans>
  ```

- 创建消息生产者

  ```java
  import org.springframework.amqp.core.Message;
  import org.springframework.amqp.core.MessageBuilder;
  import org.springframework.amqp.core.MessageProperties;
  import org.springframework.amqp.core.MessagePropertiesBuilder;
  import org.springframework.amqp.rabbit.core.RabbitTemplate;
  import org.springframework.context.support.ClassPathXmlApplicationContext;
  public class Producer {
      public static void main(String[] args) throws Exception {
          ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring-rabbit.xml");
          RabbitTemplate rabbitTemplate = context.getBean(RabbitTemplate.class);
          //封装消息参数
          MessageProperties build = MessagePropertiesBuilder
                  .newInstance()
                  .setContentEncoding("utf-8")
                  .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
                  .build();
          //发送消息
          for (int i = 0; i < 20; i++) {
              //封装消息
              Message message = MessageBuilder.withBody(("你好，rabbit spring xml" + i).getBytes("utf-8"))
                      .andProperties(build)
                      .build();
              rabbitTemplate.send("ex.spring.xml", "myQueue", message);
          }
          context.close();
      }
  }
  ```

- 创建pull模式消息消费者

  ```java
  import org.springframework.amqp.core.Message;
  import org.springframework.amqp.rabbit.core.RabbitTemplate;
  import org.springframework.context.support.ClassPathXmlApplicationContext;
  import java.io.UnsupportedEncodingException;
  public class PullConsumer {
      public static void main(String[] args) throws UnsupportedEncodingException {
          ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring-rabbit.xml");
          RabbitTemplate rabbitTemplate = context.getBean(RabbitTemplate.class);
          //直接拉取消息模式
          Message message = rabbitTemplate.receive("queue.spring.xml");
          System.out.println(new String(message.getBody(),message.getMessageProperties().getContentEncoding()));
          context.close();
      }
  }
  ```

- 创建push模式消息消费者

  - 创建监听实现类，并在配置文件中配置

    ```java
    import com.rabbitmq.client.Channel;
    import org.springframework.amqp.core.Message;
    import org.springframework.amqp.rabbit.listener.api.ChannelAwareMessageListener;
    public class MyMessageListener implements ChannelAwareMessageListener {
        @Override
        public void onMessage(Message message, Channel channel) throws Exception {
            System.out.println(new String(message.getBody(),message.getMessageProperties().getContentEncoding()));
        }
    }
    ```

  - 创建启动类

    ```java
    import org.springframework.context.support.ClassPathXmlApplicationContext;
    public class PushConsumer {
        public static void main(String[] args) {
            ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring-rabbit.xml");
        }
    }
    ```

##### 2.4.5.6、Spring annotation整合RabbitMQ

- 创建maven项目

- 添加依赖

  ```xml
  <dependency>
      <groupId>org.springframework.amqp</groupId>
      <artifactId>spring-rabbit</artifactId>
      <version>2.2.7.RELEASE</version>
  </dependency>
  ```

- 创建生产者

  - 创建配置类

    ```java
    package com.thm.producer;
    import org.springframework.amqp.core.*;
    import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
    import org.springframework.amqp.rabbit.connection.ConnectionFactory;
    import org.springframework.amqp.rabbit.core.RabbitAdmin;
    import org.springframework.amqp.rabbit.core.RabbitTemplate;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import java.net.URI;
    @Configuration
    public class ProducerRabbitConfig {
        //连接工厂
        @Bean
        public ConnectionFactory connectionFactory(){
            ConnectionFactory connectionFactory = new CachingConnectionFactory(URI.create("amqp://root:123456@192.168.81.100:5672/%2f"));
            return connectionFactory;
        }
        //RabbitTemplate
        @Bean
        public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory){
            RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
            return rabbitTemplate;
        }
        //RabbitAdmin
        @Bean
        public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory){
            RabbitAdmin rabbitAdmin = new RabbitAdmin(connectionFactory);
            return rabbitAdmin;
        }
        //queue
        @Bean
        public Queue queue(){
            Queue queue = QueueBuilder.nonDurable("queue.spring.anno").autoDelete().build();
            return queue;
        }
        //exchange
        @Bean
        public Exchange exchange(){
            FanoutExchange exchange = new FanoutExchange("ex.spring.anno", false, true);
            return exchange;
        }
        //binding
        @Bean
        public Binding binding(Queue queue,Exchange exchange){
            Binding binding = BindingBuilder.bind(queue).to(exchange).with("anno.key").noargs();
            return binding;
        }
    }
    ```

  - 创建启动类

    ```java
    package com.thm.producer;
    import org.springframework.amqp.core.Message;
    import org.springframework.amqp.core.MessageBuilder;
    import org.springframework.amqp.core.MessageProperties;
    import org.springframework.amqp.core.MessagePropertiesBuilder;
    import org.springframework.amqp.rabbit.core.RabbitTemplate;
    import org.springframework.context.annotation.AnnotationConfigApplicationContext;
    import org.springframework.context.support.AbstractApplicationContext;
    import java.io.UnsupportedEncodingException;
    
    public class Producer {
        public static void main(String[] args) throws UnsupportedEncodingException {
            AbstractApplicationContext context = new AnnotationConfigApplicationContext(ProducerRabbitConfig.class);
            RabbitTemplate rabbitTemplate = context.getBean(RabbitTemplate.class);
            //封装消息参数
            MessageProperties build = MessagePropertiesBuilder
                    .newInstance()
                    .setContentEncoding("utf-8")
                    .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
                    .build();
            //发送消息
            for (int i = 0; i < 20; i++) {
                //封装消息
                Message message = MessageBuilder.withBody(("你好，rabbit spring anno" + i).getBytes("utf-8"))
                        .andProperties(build)
                        .build();
                rabbitTemplate.send("ex.spring.anno", "anno.key", message);
            }
            context.close();
        }
    }
    ```

- 创建拉模式消息消费者

  - 创建配置文件

    ```java
    package com.thm.pullConsumer;
    
    import org.springframework.amqp.core.*;
    import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
    import org.springframework.amqp.rabbit.connection.ConnectionFactory;
    import org.springframework.amqp.rabbit.core.RabbitAdmin;
    import org.springframework.amqp.rabbit.core.RabbitTemplate;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import java.net.URI;
    @Configuration
    public class PullRabbitConfig {
        //连接工厂
        @Bean
        public ConnectionFactory connectionFactory(){
            ConnectionFactory connectionFactory = new CachingConnectionFactory(URI.create("amqp://root:123456@192.168.81.100:5672/%2f"));
            return connectionFactory;
        }
        //RabbitTemplate
        @Bean
        public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory){
            RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
            return rabbitTemplate;
        }
        //RabbitAdmin
        @Bean
        public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory){
            RabbitAdmin rabbitAdmin = new RabbitAdmin(connectionFactory);
            return rabbitAdmin;
        }
        //queue
        @Bean
        public Queue queue(){
            Queue queue = QueueBuilder.nonDurable("queue.spring.anno").autoDelete().build();
            return queue;
        }
        //exchange
        @Bean
        public Exchange exchange(){
            FanoutExchange exchange = new FanoutExchange("ex.spring.anno", false, true);
            return exchange;
        }
        //binding
        @Bean
        public Binding binding(Queue queue,Exchange exchange){
            Binding binding = BindingBuilder.bind(queue).to(exchange).with("anno.key").noargs();
            return binding;
        }
    }
    ```

  - 创建启动类

    ```java
    package com.thm.pullConsumer;
    import org.springframework.amqp.core.Message;
    import org.springframework.amqp.rabbit.core.RabbitTemplate;
    import org.springframework.context.annotation.AnnotationConfigApplicationContext;
    import org.springframework.context.support.AbstractApplicationContext;
    import java.io.UnsupportedEncodingException;
    public class PullConsumer {
        public static void main(String[] args) throws UnsupportedEncodingException {
            AbstractApplicationContext context = new AnnotationConfigApplicationContext(PullRabbitConfig.class);
            RabbitTemplate rabbitTemplate = context.getBean(RabbitTemplate.class);
            //直接拉取消息模式
            Message message = rabbitTemplate.receive("queue.spring.anno");
            System.out.println(new String(message.getBody(), message.getMessageProperties().getContentEncoding()));
            context.close();
        }
    }
    ```

- 创建push模式消息消费者

  - 创建监听类

    ```java
    package com.thm.pushConsumer;
    import org.springframework.amqp.rabbit.annotation.RabbitListener;
    import org.springframework.messaging.handler.annotation.Payload;
    import org.springframework.stereotype.Component;
    import java.io.UnsupportedEncodingException;
    @Component
    public class MyMessageListener {
        /**
         * channel 对象
         * message 直接操作原生amqp消息
         * @PayLoad 注解方法参数 该参数的值是消息体
         * @Header 注解方法参数  访问指定的消息头字段信息
         * @Headers 注解方法参数 获取所有消息头字段信息  参数类型是map集合
         * MessageHeader 参数类型 访问所有消息头字段信息
         * MessageHeaderAccessor 访问所有消息头字段信息
         */
    /*    @RabbitListener(queues = "queue.spring.anno)")
        public void onMessage(Message message) throws UnsupportedEncodingException {
            System.out.println(new String(message.getBody(),message.getMessageProperties().getContentEncoding()));
        }*/
    
        @RabbitListener(queues = "queue.spring.anno")
        public void onMessage(@Payload String message) {
            System.out.println(message);
        }
    }
    ```

  - 创建配置类

    ```java
    package com.thm.pushConsumer;
    
    import org.springframework.amqp.core.*;
    import org.springframework.amqp.rabbit.annotation.EnableRabbit;
    import org.springframework.amqp.rabbit.config.SimpleRabbitListenerContainerFactory;
    import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
    import org.springframework.amqp.rabbit.connection.ConnectionFactory;
    import org.springframework.amqp.rabbit.core.RabbitAdmin;
    import org.springframework.amqp.rabbit.core.RabbitTemplate;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.ComponentScan;
    import org.springframework.context.annotation.Configuration;
    import java.net.URI;
    @ComponentScan("com.thm.pushConsumer")
    @Configuration
    //启用rabbit的监听模式
    @EnableRabbit
    public class PushRabbitConfig {
        //连接工厂
        @Bean
        public ConnectionFactory connectionFactory(){
            ConnectionFactory connectionFactory = new CachingConnectionFactory(URI.create("amqp://root:123456@192.168.81.100:5672/%2f"));
            return connectionFactory;
        }
        //RabbitTemplate
        @Bean
        public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory){
            RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
            return rabbitTemplate;
        }
        //RabbitAdmin
        @Bean
        public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory){
            RabbitAdmin rabbitAdmin = new RabbitAdmin(connectionFactory);
            return rabbitAdmin;
        }
        //queue
        @Bean
        public Queue queue(){
            Queue queue = QueueBuilder.nonDurable("queue.spring.anno").autoDelete().build();
            return queue;
        }
        //messageListener
        @Bean("rabbitListenerContainerFactory")
        public SimpleRabbitListenerContainerFactory simpleRabbitListenerContainerFactory(ConnectionFactory connectionFactory){
            SimpleRabbitListenerContainerFactory simpleRabbitListenerContainerFactory = new SimpleRabbitListenerContainerFactory();
            //设置连接工厂
            simpleRabbitListenerContainerFactory.setConnectionFactory(connectionFactory);
            //设置确认方式  默认自动
            simpleRabbitListenerContainerFactory.setAcknowledgeMode(AcknowledgeMode.AUTO);
            //设置并发处理数和最大并发处理
            //simpleRabbitListenerContainerFactory.setConcurrentConsumers(10);
            //simpleRabbitListenerContainerFactory.setMaxConcurrentConsumers(10);
            //批量处理数
            //simpleRabbitListenerContainerFactory.setBatchSize(10);
            return simpleRabbitListenerContainerFactory;
        }
    }
    ```

  - 创建启动类

    ```java
    package com.thm.pushConsumer;
    import org.springframework.context.annotation.AnnotationConfigApplicationContext;
    public class PushConsumer {
        public static void main(String[] args) {
            new AnnotationConfigApplicationContext(PushRabbitConfig.class);
        }
    }
    ```

##### 2.4.5.7、Spring Boot整合RabbitMQ

- 添加starter依赖

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-amqp</artifactId>
      <version>2.3.5.RELEASE</version>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <version>2.3.5.RELEASE</version>
  </dependency>
  ```

- 创建配置文件application.yml

  ```yaml
  spring:
    application:
      name: spting-boot-rabbitmq
    rabbitmq:
      host: 192.168.81.100
      virtual-host: /
      username: root
      password: 123456
      port: 5672
  ```

- 创建配置类

  ```java
  package com.thm.producer;
  import org.springframework.amqp.core.*;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  @Configuration
  public class ProducerRabbitConfig {
  
      @Bean
      public Queue queue(){
          return QueueBuilder.nonDurable("queue.spring.boot").autoDelete().build();
      }
      @Bean
      public Exchange exchange(){
          return new DirectExchange("ex.spring.boot",false,true);
      }
      @Bean
      public Binding binding(Queue queue,Exchange exchange){
          Binding binding = BindingBuilder.bind(queue).to(exchange).with("boot.key").noargs();
          return binding;
      }
  }
  ```

- 创建测试类为生产者

  ```java
  import com.thm.Bootstrap;
  import org.junit.runner.RunWith;
  import org.springframework.amqp.core.Message;
  import org.springframework.amqp.core.MessageBuilder;
  import org.springframework.amqp.core.MessageProperties;
  import org.springframework.amqp.core.MessagePropertiesBuilder;
  import org.springframework.amqp.rabbit.core.RabbitTemplate;
  import org.springframework.boot.test.context.SpringBootTest;
  import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
  import javax.annotation.Resource;
  import java.io.UnsupportedEncodingException;
  @RunWith(SpringJUnit4ClassRunner.class)
  @SpringBootTest(classes = Bootstrap.class)
  public class Test {
      @Resource
      private RabbitTemplate rabbitTemplate;
  
      /**
       * 发送消息
       * @throws UnsupportedEncodingException
       */
      @org.junit.Test
      public void send() throws UnsupportedEncodingException {
          //封装消息参数
          MessageProperties build = MessagePropertiesBuilder
                  .newInstance()
                  .setContentEncoding("utf-8")
                  .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
                  .setHeader("hello","word")
                  .build();
          //发送消息
          for (int i = 0; i < 20; i++) {
              //封装消息
              Message message = MessageBuilder.withBody(("你好，rabbit spring boot" + i).getBytes("utf-8"))
                      .andProperties(build)
                      .build();
              rabbitTemplate.send("ex.spring.boot", "boot.key", message);
          }
      }
  }
  ```

- 创建消息监听器

  ```java
  package com.thm.comsumer;
  import org.springframework.amqp.rabbit.annotation.RabbitListener;
  import org.springframework.messaging.handler.annotation.Header;
  import org.springframework.messaging.handler.annotation.Payload;
  import org.springframework.stereotype.Component;
  @Component
  public class MyMessageListener {
      @RabbitListener(queues = "queue.spring.boot")
      public void onMessage(@Payload String message, @Header("hello") String value) {
          System.out.println(message);
          System.out.println("hello: " + value);
      }
  }
  ```

- 创建启动类

  当前启动类启动，使用消息监听模式，为消费者，测试类为消息生产者，先启动主类，再启动测试类

  ```java
  package com.thm;
  import org.springframework.boot.SpringApplication;
  import org.springframework.boot.autoconfigure.SpringBootApplication;
  @SpringBootApplication
  public class Bootstrap {
      public static void main(String[] args) {
          SpringApplication.run(Bootstrap.class,args);
      }
  }
  ```

## 3、RabbitMQ高级特性解析

### 3.1、消息可靠性

你用支付宝给商家支付，如果是个仔细的人，会考虑我转账的话，会不会把我的钱扣了，商家没有收到我的钱？

一般我们使用支付宝或微信转账支付的时候，都是扫码，支付，然后立刻得到结果，说你支付了多少钱，如果你绑定的是银行卡，可能这个时候你并没有收到支付

的确认消息。往往是在一段时间之后，你会收到银行卡发来的短信，告诉你支付的信息。

支付平台如何保证这笔帐不出问题？

![](../img/RabbitMQ/RabbitMQ-img-32.png)

支付平台必须保证数据正确性，保证数据并发安全性，保证数据最终一致性。

支付平台通过如下几种方式保证数据一致性：

- **分布式锁**

  这个比较容易理解，就是在操作某条数据时先锁定，可以用redis或zookeeper等常用框架来实现。 比如我们在修改账单时，先锁定该账单，如果该账单有并

  发操作，后面的操作只能等待上一个操作的锁释放后再依次执行。

  优点：能够保证数据强一致性。 

  缺点：高并发场景下可能有性能问题。

- **消息队列**

  消息队列是为了保证最终一致性，我们需要确保消息队列有ack机制客户端收到消息并消费处理完成后，客户端发送ack消息给消息中间件如果消息中间件超过

  指定时间还没收到ack消息，则定时去重发消息。

  比如我们在用户充值完成后，会发送充值消息给账户系统，账户系统再去更改账户余额。

  优点：异步、高并发 

  缺点：有一定延时、数据弱一致性，并且必须能够确保该业务操作肯定能够成功完成，不可能失败。

我们可以从以下几方面来保证消息的可靠性：

1. 客户端代码中的异常捕获，包括生产者和消费者

2. AMQP/RabbitMQ的事务机制

3. 发送端确认机制

4. 消息持久化机制

5. Broker端的高可用集群

6. 消费者确认机制

7. 消费端限流

8. 消息幂等性

#### 3.1.1、异常捕获机制

先执行行业务操作，业务操作成功后执行行消息发送，消息发送过程通过try catch 方式捕获异常，在异常处理的代码块中执行行回滚业务操作或者执行行重发操

作等。这是一种最大努力确保的方式，并无法保证100%绝对可靠，因为这里没有异常并不代表消息就一定投递成功。

![](../img/RabbitMQ/RabbitMQ-img-33.png)

另外，可以通过**spring.rabbitmq.template.retry.enabled=true** 配置开启发送端的重试

#### 3.1.2、AMQP/RabbitMQ的事务机制

没有捕获到异常并不能代表消息就一定投递成功了。

一直到事务提交后都没有异常，确实就说明消息是投递成功了。但是，这种方式在性能方面的**开销比较大**，一般也不推荐使用。

![](../img/RabbitMQ/RabbitMQ-img-34.png)

#### 3.1.3、发送端确认机制

RabbitMQ后来引入了一种轻量量级的方式，叫**发送方确认(publisher confirm)机制**。生产者将信道设置成confirm(确认)模式，一旦信道进入confirm 模式，所

有在该信道上⾯面发布的消息都会被指派一个唯一的ID(从1 开始)，一旦消息被投递到所有匹配的队列之后（如果消息和队列是持久化的，那么确认消息会在消息

持久化后发出），RabbitMQ 就会发送一个确认(Basic.Ack)给生产者(包含消息的唯一ID)，这样生产者就知道消息已经正确送达了。

![](../img/RabbitMQ/RabbitMQ-img-35.png)

RabbitMQ 回传给生产者的确认消息中的deliveryTag 字段包含了确认消息的序号，另外，通过设置channel.basicAck方法中的multiple参数，表示到这个序号之

前的所有消息是否都已经得到了处理了。生产者投递消息后并不需要一直阻塞着，可以继续投递下一条消息并通过回调方式处理理ACK响应。如果 RabbitMQ 因为

自身内部错误导致消息丢失等异常情况发生，就会响应一条nack(Basic.Nack)命令，生产者应用程序同样可以在回调方法中处理理该 nack 命令。

- **同步确认**

  ```java
  package com.thm;
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  import java.io.IOException;
  import java.net.URISyntaxException;
  import java.security.KeyManagementException;
  import java.security.NoSuchAlgorithmException;
  import java.util.concurrent.TimeoutException;
  public class Producer {
      //定义消息队列名称
      private final static String QUEUE_NAME = "queue.pc";
      //定义交换器名称
      private final static String EX_NAME = "ex.pc";
      //定义绑定key
      private final static String ROUTE_KEY = "key.pc";
  
      public static void main(String[] args) throws Exception {
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          Connection connection = connectionFactory.newConnection();
          Channel channel = connection.createChannel();
          //开启发送方确认（同步确认）
          channel.confirmSelect();
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.DIRECT, true, false, null);
          channel.queueDeclare(QUEUE_NAME, true, false, false, null);
          channel.queueBind(QUEUE_NAME, EX_NAME, ROUTE_KEY);
          //发送消息
          String message = "hello PC";
          channel.basicPublish(EX_NAME, ROUTE_KEY, null, message.getBytes());
          try {
              channel.waitForConfirmsOrDie(5_000);
              System.out.println("消息被确认：message = " + message);
          } catch (IOException e) {
              System.err.println("消息被拒绝！ message = " + message);
          } catch (InterruptedException e) {
              System.err.println("在不是Publisher Confirms的通道上使用该方法");
          } catch (TimeoutException e) {
              System.err.println("等待消息确认超时！ message = " + message);
          }
          channel.close();
          connection.close();
      }
  }
  ```

  waitForConfirm方法有个重载的，可以自定义timeout超时时间，超时后会抛TimeoutException。类似的有几个waitForConfirmsOrDie方法，Broker端在返
  
  回nack(Basic.Nack)之后该方法会抛出java.io.IOException。需要根据异常类型来做区别处理理， TimeoutException超时是属于第三状态（无法确定成功还是
  
  失败），而返回Basic.Nack抛出IOException这种是明确的失败。上面的代码主要只是演示confirm机制，实际上还是同步阻塞模式的，性能并不不是太好。
  
- 同步批量确认

  实际上，我们也可以通过“批处理理”的方式来改善整体的性能（即批量量发送消息后仅调用一次waitForConfirms方法）。正常情况下这种批量处理的方式效

  率会高很多，但是如果发生了超时或者nack（失败）后那就需要批量量重发消息或者通知上游业务批量回滚（因为我们只知道这个批次中有消息没投递成功，

  而并不知道具体是那条消息投递失败了，所以很难针对性处理），如此看来，批量重发消息肯定会造成部分消息重复。

  ```java
  package com.thm;
  
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  import java.io.IOException;
  import java.util.concurrent.TimeoutException;
  public class ProducerBatch {
      //定义消息队列名称
      private final static String QUEUE_NAME = "queue.pc";
      //定义交换器名称
      private final static String EX_NAME = "ex.pc";
      //定义绑定key
      private final static String ROUTE_KEY = "key.pc";
  
      public static void main(String[] args) throws Exception {
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          Connection connection = connectionFactory.newConnection();
          Channel channel = connection.createChannel();
          //开启发送方确认（同步确认）
          channel.confirmSelect();
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.DIRECT, true, false, null);
          channel.queueDeclare(QUEUE_NAME, true, false, false, null);
          channel.queueBind(QUEUE_NAME, EX_NAME, ROUTE_KEY);
          //发送消息
          String message = "hello PC batch";
          //批量确认处理数量
          int batchSize = 10;
          int outstandingConfirms = 0;
          for (int i = 0; i < 105; i++) {
              channel.basicPublish(EX_NAME, ROUTE_KEY, null, message.getBytes());
              outstandingConfirms++;
              if (outstandingConfirms == batchSize) {
                  channel.waitForConfirmsOrDie(5_000);
                  System.out.println("批消息确认");
                  outstandingConfirms = 0;
              }
          }
          if (outstandingConfirms > 0) {
              channel.waitForConfirmsOrDie(5_000);
              System.out.println("批消息确认");
          }
          channel.close();
          connection.close();
      }
  }
  ```

- 异步回调确认

  另外，我们可以通过异步回调的方式来处理Broker的响应。addConfirmListener 方法可以添加ConfirmListener 这个回调接口，这个 ConfirmListener 接口

  包含两个方法:handleAck 和handleNack，分别用来处理 RabbitMQ 回传的 Basic.Ack 和 Basic.Nack。

  ```java
  package com.thm;
  import com.rabbitmq.client.*;
  import java.util.concurrent.ConcurrentNavigableMap;
  import java.util.concurrent.ConcurrentSkipListMap;
  
  public class ProducerAsyn {
      //定义消息队列名称
      private final static String QUEUE_NAME = "queue.pc";
      //定义交换器名称
      private final static String EX_NAME = "ex.pc";
      //定义绑定key
      private final static String ROUTE_KEY = "key.pc";
  
      public static void main(String[] args) throws Exception {
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          Connection connection = connectionFactory.newConnection();
          Channel channel = connection.createChannel();
          //开启发送方确认（同步确认）
          channel.confirmSelect();
          channel.exchangeDeclare(EX_NAME, BuiltinExchangeType.DIRECT, true, false, null);
          channel.queueDeclare(QUEUE_NAME, true, false, false, null);
          channel.queueBind(QUEUE_NAME, EX_NAME, ROUTE_KEY);
          //确认消息MAP
          ConcurrentNavigableMap<Long, String> outstandingConfirms = new ConcurrentSkipListMap<>();
          //消息确认成功回调
          ConfirmCallback cleanOutstandingConfirms = (sequenceNumber, multiple) -> {
              if (multiple) {
                  System.out.println("小于等于 " + sequenceNumber + " 的消息都被确认了");
                  // 获取map集合的子集
                  ConcurrentNavigableMap<Long, String> headMap = outstandingConfirms.headMap(sequenceNumber, true);
                  // 清空子集
                  headMap.clear();
              } else {
                  System.out.println(sequenceNumber + " 对应的消息被确认");
                  String removed = outstandingConfirms.remove(sequenceNumber);
              }
          };
          //添加消息确认监听
          channel.addConfirmListener(cleanOutstandingConfirms, (sequenceNumber, multiple) -> {
              if (multiple) {
                  System.out.println("小于等于 " + sequenceNumber + " 的消息都不确认了");
                  ConcurrentNavigableMap<Long, String> headMap = outstandingConfirms.headMap(sequenceNumber, true);
              } else {
                  System.out.println(sequenceNumber + " 对应的消息不确认");
                  outstandingConfirms.remove(sequenceNumber);
              }
          });
          //发送消息
          String message = "hello PC batch";
          //批量确认处理数量
          for (int i = 0; i < 1000; i++) {
              //获取下一条即将发送消息得ID
              long nextPublishSeqNo = channel.getNextPublishSeqNo();
              channel.basicPublish(EX_NAME, QUEUE_NAME, null, (message + i).getBytes());
              System.out.println("序列号为：" + nextPublishSeqNo + "的消息已经发送了：" + (message + i));
              outstandingConfirms.put(nextPublishSeqNo, (message + i));
          }
          Thread.sleep(10000);
          channel.close();
          connection.close();
      }
  }
  ```

- **springboot案例**

  - pom.xml添加依赖

    ```xml
    <dependency> 
        <groupId>org.springframework.boot</groupId> 
        <artifactId>spring-boot-starter-amqp</artifactId> 
    </dependency> 
    <dependency> 
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    ```

  -  application.properties添加RabbitMQ配置信息

    ```properties
    spring.application.name=publisherconfirm 
    spring.rabbitmq.host=192.168.81.100 
    spring.rabbitmq.virtual-host=/ 
    spring.rabbitmq.username=root 
    spring.rabbitmq.password=123456 
    spring.rabbitmq.port=5672 
    spring.rabbitmq.publisher-confirm-type=correlated 
    spring.rabbitmq.publisher-returns=true
    ```

  - RabbitConfig类 

    ```java
    package com.lagou.rabbitmq.demo.config;
    
    import org.springframework.amqp.core.*;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    
    @Configuration
    public class RabbitConfig {
        @Bean
        public Queue queue() {
            Queue queue = new Queue("q.biz", false, false, false, null);
            return queue;
        }
    
        @Bean
        public Exchange exchange() {
            Exchange exchange = new DirectExchange("ex.biz", false, false, null);
            return exchange;
        }
    
        @Bean
        public Binding binding() {
            return BindingBuilder.bind(queue()).to(exchange()).with("biz").noargs();
        }
    }
    ```

  - BizController类 

    ```java
    package com.lagou.rabbitmq.demo.controller;
    
    import org.springframework.amqp.core.Message;
    import org.springframework.amqp.core.MessageProperties;
    import org.springframework.amqp.rabbit.connection.CorrelationData;
    import org.springframework.amqp.rabbit.core.RabbitTemplate;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;
    
    import java.io.UnsupportedEncodingException;
    
    @RestController
    public class BizController {
        private RabbitTemplate rabbitTemplate;
    
        @Autowired
        public void setRabbitTemplate(RabbitTemplate rabbitTemplate) {
            this.rabbitTemplate = rabbitTemplate;
            this.rabbitTemplate.setConfirmCallback((correlationData, flag, cause) -> {
                if (flag) {
                    try {
                        System.out.println("消息确认：" + correlationData.getId() + " " + new String(correlationData.getReturnedMessage().getBody(), "utf-8"));
                    } catch (UnsupportedEncodingException e) {
                        e.printStackTrace();
                    }
                } else {
                    System.out.println(cause);
                }
            });
        }
    
        @RequestMapping("/biz")
        public String doBiz() throws UnsupportedEncodingException {
            MessageProperties props = new MessageProperties();
            props.setCorrelationId("1234");
            props.setConsumerTag("msg1");
            props.setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN);
            props.setContentEncoding("utf-8");
            CorrelationData cd = new CorrelationData();
            //props.setDeliveryMode(MessageDeliveryMode.NON_PERSISTENT); 
            // 1 
            // props.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
            // 2
            cd.setId("msg1");
            cd.setReturnedMessage(new Message("这是msg1的响 应".getBytes("utf-8"), null));
            Message message = new Message("这是等待确认的消 息".getBytes("utf-8"), props);
            rabbitTemplate.convertAndSend("ex.biz", "biz", message, cd);
            return "ok";
        }
    
        @RequestMapping("/bizfalse")
        public String doBizFalse() throws UnsupportedEncodingException {
            MessageProperties props = new MessageProperties();
            props.setCorrelationId("1234");
            props.setConsumerTag("msg1");
            props.setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN);
            props.setContentEncoding("utf-8");
            Message message = new Message("这是等待确认的消 息".getBytes("utf-8"), props);
            rabbitTemplate.convertAndSend("ex.bizFalse", "biz", message);
            return "ok";
        }
    }
    ```

  - 启动类

    ```java
    package com.lagou.rabbitmq.demo;
    
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    
    @SpringBootApplication
    public class RabbitmqDemo {
        public static void main(String[] args) {
            SpringApplication.run(RabbitmqDemo13.class, args);
        }
    }
    ```

#### 3.1.4、持久化存储机制

持久化是提高RabbitMQ可靠性的基础，否则当RabbitMQ遇到异常时（如：重启、断电、停机等）数据将会丢失。主要从以下几个方面来保障消息的持久性：

1. Exchange的持久化。通过定义时设置durable 参数为ture来保证Exchange相关的元数据不不丢失。

2. Queue的持久化。也是通过定义时设置durable 参数为ture来保证Queue相关的元数据不不丢失。

3. 消息的持久化。通过将消息的投递模式 (BasicProperties 中的 deliveryMode 属性)设置为 2即可实现消息的持久化，保证消息自身不丢失。

![](../img/RabbitMQ/RabbitMQ-img-36.png)

RabbitMQ中的持久化消息都需要写入磁盘（当系统内存不不足时，非持久化的消息也会被刷盘处理），这些处理动作都是在“持久层”中完成的。持久层是一个逻

辑上的概念，实际包含两个部分：

1. 队列索引(rabbit_queue_index)，rabbit_queue_index 负责维护Queue中消息的信息，包括消息的存储位置、是否已交给消费者、是否已被消费及Ack确认

   等，每个Queue都有与之对应的rabbit_queue_index。 

2. 消息存储(rabbit_msg_store)，rabbit_msg_store 以键值对的形式存储消息，它被所有队列列共享，在每个节点中有且只有一个。

下图中，

HOSTNAME/msg_stores/vhosts/$VHostId 这个路路径下包含 queues、msg_store_persistent、msg_store_transient 这 3 个目录，这是实际存储消息的位置。

其中queues目录中保存着rabbit_queue_index相关的数据，而msg_store_persistent保存着持久化消息数据，msg_store_transient保存着⾮非持久化相关的数

据。另外，RabbitMQ通过配置queue_index_embed_msgs_below可以根据消息大小决定存储位置，默认queue_index_embed_msgs_below是4096字节(包含消

息体、属性及headers)，小于该值的消息存在rabbit_queue_index中。

![](../img/RabbitMQ/RabbitMQ-img-37.png)

#### 3.1.5、消费端确认机制

如何保证消息被消费者成功消费？

前面我们讲了生产者发送确认机制和消息的持久化存储机制，然而这依然无法完全保证整个过程的可靠性，因为如果消息被消费过程中业务处理失败了但是消息却

已经出列了（被标记为已消费了），我们又没有任何重试，那结果跟消息丢失没什么分别。

RabbitMQ在消费端会有Ack机制，即消费端消费消息后需要发送Ack确认报文给Broker端，告知自己是否已消费完成，否则可能会一直重发消息直到消息过期

（AUTO模式）。

这也是我们之前一直在讲的“最终一致性”、“可恢复性” 的基础。

一般而言，我们有如下处理手段：

1. 采用NONE模式，消费的过程中自行捕获异常，引发异常后直接记录日志并落到异常恢复表，再通过后台定时任务扫描异常恢复表尝试做重试动作。如果业务

   不自行处理则有丢失数据的风险

2. 采用AUTO（自动Ack）模式，不主动捕获异常，当消费过程中出现异常时会将消息放回Queue中，然后消息会被重新分配到其他消费者节点（如果没有则还

   是选择当前节点）重新被消费，默认会一直重发消息并直到消费完成返回Ack或者一直到过期

3. 采用MANUAL（手动Ack）模式，消费者自行控制流程并手动调用channel相关的方法返回Ack

本小节的内容总结起来就如图所示，本质上就是“请求/应答”确认模式

![](../img/RabbitMQ/RabbitMQ-img-32.png)

**springboot完整案例**

- pom.xml

  ```xml
  <dependency> 
      <groupId>org.springframework.boot</groupId> 
      <artifactId>spring-boot-starter-amqp</artifactId> 
  </dependency> 
  <dependency> 
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId> 
  </dependency>
  ```

- application.properties

  ```properties
  spring.application.name=consumer_ack 
  spring.rabbitmq.host=192.168.81.100
  spring.rabbitmq.virtual-host=/ 
  spring.rabbitmq.username=root 
  spring.rabbitmq.password=123456 
  spring.rabbitmq.port=5672
  #最大重试次数 
  spring.rabbitmq.listener.simple.retry.max-attempts=5 
  #是否开启消费者重试（为false时关闭消费者重试， # 意思不是“不重试”，而是一直收到消息直到jack确认或者一直到超时） 
  spring.rabbitmq.listener.simple.retry.enabled=true 
  #重试间隔时间（单位毫秒） 
  spring.rabbitmq.listener.simple.retry.initial-interval=5000 
  # 重试超过最大次数后是否拒绝 
  spring.rabbitmq.listener.simple.default-requeue-rejected=false 
  #ack模式 全局
  spring.rabbitmq.listener.simple.acknowledge-mode=manual
  ```

- RabbitConfig

  ```java
  package com.lagou.rabbitmq.demo.config;
  import org.springframework.amqp.core.*;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  @Configuration
  public class RabbitConfig {
      @Bean
      public Queue queue() {
          return new Queue("q.biz", false, false, false, null);
      }
  
      @Bean
      public Exchange exchange() {
          return new DirectExchange("ex.biz", false, false, null);
      }
  
      @Bean
      public Binding binding() {
          return BindingBuilder.bind(queue()).to(exchange()).with("biz").noargs();
      }
  }
  ```

- MessageListener（push模式）

  ```java
  package com.lagou.rabbitmq.demo.listener;
  import com.rabbitmq.client.Channel;
  import org.springframework.amqp.rabbit.annotation.RabbitListener;
  import org.springframework.amqp.support.AmqpHeaders;
  import org.springframework.messaging.handler.annotation.Header;
  import org.springframework.messaging.handler.annotation.Payload;
  import org.springframework.stereotype.Component;
  import java.io.IOException;
  import java.util.Random;
  @Component
  public class MessageListener {
      private Random random = new Random();
      /** NONE模式，则只要收到消息后就立即确认（消息出列，标记已消费），有丢失数据 的风险
        * AUTO模式，看情况确认，如果此时消费者抛出异常则消息会返回到队列中
        * MANUAL模式，需要显式的调用当前channel的basicAck方法
        * @param channel
        * @param deliveryTag
        * @param message
        */
      @RabbitListener(queues = "q.biz", ackMode = "AUTO")
      @RabbitListener(queues = "q.biz", ackMode = "MANUAL")
      // @RabbitListener(queues = "q.biz", ackMode = "NONE")
      public void handleMessageTopic(Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag, @Payload String message) {
          System.out.println("RabbitListener消费消息，消息内容：" + message);
          try {
              if (random.nextInt(10) % 3 != 0) {
                  // 手动ack，deliveryTag表示消息的唯一标志，multiple表示是否是批量确认 
                  // channel.basicAck(deliveryTag, false);
                  // 手动nack，告诉broker消费者处理失败，最后一个参数表示是 否需要将消息重新入列
                  // channel.basicNack(deliveryTag, false, true);
                  // 手动拒绝消息。第二个参数表示是否重新入列
                  channel.basicReject(deliveryTag, true);
              } else { // 手动ack，deliveryTag表示消息的唯一标志，multiple表示是 否是批量确认
                  channel.basicAck(deliveryTag, false);
                  System.err.println("已确认消息：" + message);
              }
          } catch (IOException e) {
              e.printStackTrace();
          }
      }
  }
  ```

- 启动类

  ```java
  package com.lagou.rabbitmq.demo;
  
  import org.springframework.amqp.core.Message;
  import org.springframework.amqp.core.MessageProperties;
  import org.springframework.amqp.rabbit.core.RabbitTemplate;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.boot.ApplicationRunner;
  import org.springframework.boot.SpringApplication;
  import org.springframework.boot.autoconfigure.SpringBootApplication;
  import org.springframework.context.annotation.Bean;
  
  @SpringBootApplication
  public class RabbitmqDemo {
      @Autowired
      private RabbitTemplate rabbitTemplate;
  
      public static void main(String[] args) {
          SpringApplication.run(RabbitmqDemo.class, args);
      }
  	//等待5秒后向消息队列发送消息
      @Bean
      public ApplicationRunner runner() {
          return args -> {
              Thread.sleep(5000);
              for (int i = 0; i < 10; i++) {
                  MessageProperties props = new MessageProperties();
                  props.setDeliveryTag(i);
                  Message message = new Message(("消息：" + i).getBytes("utf-8"), props);
  				// this.rabbitTemplate.convertAndSend("ex.biz", "biz", "消息：" + i);
                  this.rabbitTemplate.convertAndSend("ex.biz", "biz", message);
              }
          };
      }
  }
  ```

- BizController （pull模式）

  ```java
  package com.lagou.rabbitmq.demo.controller;
  
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.GetResponse;
  import org.springframework.amqp.rabbit.core.ChannelCallback;
  import org.springframework.amqp.rabbit.core.RabbitTemplate;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.web.bind.annotation.RequestMapping;
  import org.springframework.web.bind.annotation.RestController;
  import java.util.Random;
  @RestController
  public class BizController {
      @Autowired
      private RabbitTemplate rabbitTemplate;
      private Random random = new Random();
  
      @RequestMapping("/biz")
      public String getBizMessage() {
          String message = rabbitTemplate.execute(new ChannelCallback<String>() {
              @Override
              public String doInRabbit(Channel channel) throws Exception {
                  final GetResponse getResponse = channel.basicGet("q.biz", false);
                  if (getResponse == null) return "你已消费完所有的消息";
                  String message = new String(getResponse.getBody(), "utf-8");
                  if (random.nextInt(10) % 3 == 0) {
                      channel.basicAck(getResponse.getEnvelope().getDeliveryTag(), false);
                      return "已确认的消息：" + message;
                  } else {
                      // 拒收一条消息 
                      channel.basicReject(getResponse.getEnvelope().getDeliveryTag(), true);
                      // 可以拒收多条消息 
                      channel.basicNack(getResponse.getEnvelope().getDeliveryTag(), false, true);
                      return "拒绝的消息：" + message;
                  }
              }
          });
          return message;
      }
  }
  ```

#### 3.1.6、消息限流

在电商的秒杀活动中，活动一开始会有大量并发写请求到达服务端，需要对消息进行削峰处理，如何削峰？

当消息投递速度远快于消费速度时，随着时间积累就会出现“**消息积压**”。消息中间件本身是具备一定的缓冲能力的，但这个能力是**有容量限制**的，如果长期运行并

没有任何处理，最终会导致*Broker*崩溃，而分布式系统的故障往往会发生上下游传递，连锁反应那就会很悲剧...

下面我将从多个角度介绍**QoS**与限流，防止上面的悲剧发生。

1. RabbitMQ 可以对**内存和磁盘使用量设置阈值**，当达到阈值后，**生产者将被阻塞(block)**，直到对应项指标恢复正常。全局上可以防止超大流量、消息积压等导

  致的Broker被压垮。当内存受限或磁盘可用空间受限的时候，服务器都会暂时阻止连接，服务器将暂停从发布消息的已连接客户端的套接字读取数据。连接心

  跳监视也将被禁用。所有网络连接将在rabbitmqctl和管理插件中显示为“已阻止”，这意味着它们尚未尝试发布，因此可以继续或被阻止，这意味着它们已发

  布，现在已暂停。兼容的客户端被阻止时将收到通知。

  在/etc/rabbitmq/rabbitmq.conf中配置磁盘可用空间大小：

  ![](../img/RabbitMQ/RabbitMQ-img-38.png)

2. RabbitMQ 还默认提供了一种基于**credit flow 的流控机制**，面向每一个连接进行流控。当单个队列达到最大流速时，或者多个队列达到总流速时，都会

  触发流控。触发单个链接的流控可能是因为connection、channel、queue的某一个过程处于flow状态，这些状态都可以从监控平台看到。

  ![](../img/RabbitMQ/RabbitMQ-img-39.png)

  ![](../img/RabbitMQ/RabbitMQ-img-40.png)

3. RabbitMQ中有一种QoS保证机制，可以**限制Channel上接收到的未被Ack的消息数量**，如果超过这个数量限制RabbitMQ将不会再往消费端推送消息。这是一

  种流控手段，可以防止大量消息瞬时从Broker送达消费端造成消费端巨大压力（甚至压垮消费端）。比较值得注意的是**QoS机制仅对于消费端推模式有效**，对

  拉模式无效。而且**不支持NONE Ack模式**。执行channel.basicConsume 方法之前通过 channel.basicQoS 方法可以设置该数量。消息的发送是异步的，消息

  的确认也是异步的。在消费者消费慢的时候，可以设置Qos的prefetchCount，它表示broker在向消费者发送消息的时候，一旦发送了prefetchCount个消息

  而没有一个消息确认的时候，就停止发送。消费者确认一个，broker就发送一个，确认两个就发送两个。换句话说，消费者确认多少，broker就发送多少，消

  费者等待处理的个数永远限制在prefetchCount个。

  如果对于每个消息都发送确认，增加了网络流量，此时可以批量确认消息。如果设置了multiple为true，消费者在确认的时候，比如说id是8的消息确认了，则

  在8之前的所有消息都确认了。

  ```java
  package com.thm;
  import com.rabbitmq.client.*;
  import java.io.IOException;
  public class Consumer {
      //定义交换器名称
      private final static String QUEUE_NAME = "queue.qos";
      public static void main(String[] args) throws Exception{
          ConnectionFactory connectionFactory = new ConnectionFactory();
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          Connection connection = connectionFactory.newConnection();
          Channel channel = connection.createChannel();
          channel.queueDeclare(QUEUE_NAME, false, false, true, null);
          //使用消费端限流处理,仅对push模式生效
          //表示qos是10个消息，最多有10个消息等待确认
          channel.basicQos(10);
          //表示最多有10个消息等待确认， 如果global为true表示只要使用当前channel的consumer都生效，false表示仅对当前consumer有效
          channel.basicQos(10,false);
          //第一参数表示未确认消息大小，rabbit未实现不用管
          channel.basicQos(1000,10,true);
          //接收消息
          channel.basicConsume(QUEUE_NAME, false,new DefaultConsumer(channel){
              @Override
              public void handleDelivery(String consumerTag,
                                         Envelope envelope,
                                         AMQP.BasicProperties properties,
                                         byte[] body) throws IOException {
                  //批量确认消息，减少每个消息都发送确认消息带来的网络负载
                  channel.basicAck(envelope.getDeliveryTag(),true);
              }
          });
      }
  }
  ```

生产者往往是希望自己产生的消息能快速投递出去，而当消息投递太快且超过了下游的消费速度时就容易出现消息积压/堆积，所以，从上游来讲我们应该在生

产端应用程序中也可以加入限流、应急开关等控制手段，避免超过Broker端的极限承载能力或者压垮下游消费者。

再看看下游，我们期望下游消费端能尽快消费完消息，而且还要防止瞬时大量消息压垮消费端（推模式），我们期望消费端处理速度是最快、最稳定而且还相

对均匀（比较理想化）。

**提升下游应用的吞吐量**和**缩短消费过程的耗时**，优化主要以下几种方式：

1. 优化应用程序的性能，缩短响应时间（需要时间）

2. 增加消费者节点实例（成本增加，而且底层数据库操作这些也可能是瓶颈）

3. 调整并发消费的线程数（线程数并非越大越好，需要大量压测调优至合理值）

   ```java
       @Bean("rabbitListenerContainerFactory")
       public SimpleRabbitListenerContainerFactory simpleRabbitListenerContainerFactory(ConnectionFactory connectionFactory){
           SimpleRabbitListenerContainerFactory simpleRabbitListenerContainerFactory = new SimpleRabbitListenerContainerFactory();
           //设置连接工厂
           simpleRabbitListenerContainerFactory.setConnectionFactory(connectionFactory);
           //设置确认方式  默认自动
           simpleRabbitListenerContainerFactory.setAcknowledgeMode(AcknowledgeMode.AUTO);
           //设置并发处理数和最大并发处理
           //simpleRabbitListenerContainerFactory.setConcurrentConsumers(10);
           //simpleRabbitListenerContainerFactory.setMaxConcurrentConsumers(10);
           //批量处理数
           //simpleRabbitListenerContainerFactory.setBatchSize(10);
           return simpleRabbitListenerContainerFactory;
       }
   }
   ```

#### 3.1.7、消息可靠性保障

在讲高级特性的时候几乎已经都涉及到了，这里简单回顾总结下：

1. 消息传输保障

2. 各种限流、应急手段

3. 业务层面的一些容错、补偿、异常重试等手段

**消息可靠传输**一般是业务系统接入消息中间件时**首要考虑的问题**，一般消息中间件的消息传输保障分为三个层级：

1. At most once：最多一次。消息可能会丢失，但绝**不会重复**传输

2. At least once：最少一次。消息绝不会丢失，但**可能会重复**传输

3. Exactly once：恰好一次。每条消息肯定会被**传输一次且仅传输一次**

RabbitMQ 支持其中的“**最多一次**”和“**最少一次**”。

- 其中“**最少一次**”投递实现需要考虑以下这个几个方面的内容：

  1. 消息生产者需要开启事务机制或者publisher confirm 机制，以确保消息可以可靠地传输到RabbitMQ 中。

  2. 消息生产者需要配合使用 mandatory 参数或者备份交换器来确保消息能够从交换器路由到队列中，进而能够保存下来而不会被丢弃。

  3. 消息和队列都需要进行持久化处理，以确保RabbitMQ 服务器在遇到异常情况时不会造成消息丢失。

  4. 消费者在消费消息的同时需要将autoAck 设置为false，然后通过手动确认的方式去确认已经正确消费的消息，以避免在消费端引起不必要的消息丢失。

- “**最多一次**”的方式就无须考虑以上那些方面，生产者随意发送，消费者随意消费，不过这样很难确保消息不会丢失。（估计有不少公司的业务系统都是这样

  的，想想都觉得可怕）

- “**恰好一次**”是**RabbitMQ** 目前无法保障的。

  考虑这样一种情况，消费者在消费完一条消息之后向RabbitMQ 发送确认Basic.Ack 命令，此时由于网络断开或者其他原因造成RabbitMQ 并没有收到这个确

  认命令，那么RabbitMQ 不会将此条消息标记删除。在重新建立连接之后，消费者还是会消费到这一条消息，这就造成了重复消费。

  再考虑一种情况，生产者在使用publisher confirm机制的时候，发送完一条消息等待RabbitMQ返回确认通知，此时网络断开，生产者捕获到异常情况，为了

  确保消息可靠性选择重新发送，这样RabbitMQ 中就有两条同样的消息，在消费的时候消费者就会重复消费。

#### 3.1.8、消息幂等性处理

刚刚我们讲到，追求高性能就无法保证消息的顺序，而追求可靠性那么就可能产生重复消息，从而导致重复消费...真是应证了那句老话：做架构就是权衡取舍。

RabbitMQ层面有实现“**去重机制**”来保证“**恰好一次**”吗？答案是并**没有**。而且这个在目前主流的消息中间件都没有实现。

借用淘宝沈洵的一句话：最好的解决办法就是不去解决。当为了在基础的分布式中间件中实现某种相对不太通用的功能，需要牺牲到性能、可靠性、扩展性时，并

且会额外增加很多复杂度，最简单的办法就是**交给业务自己去处理**。事实证明，很多业务场景下是可以容忍重复消息的。例如：操作日志收集，而对一些金融类的

业务则要求比较严苛。

一般解决重复消息的办法是，**在消费端**让我们消费消息的操作具备**幂等性**。

幂等性问题并不是消息系统独有，而是（分布式）系统中普遍存在的问题。例如：RPC框架调用超后会重试，HTTP请求会重复发起（用户手抖多点了几下按钮）

幂等（Idempotence）是一个数学上的概念，它是这样定义的：

如果一个函数f(x) 满足：f(f(x)) = f(x)，则函数f(x) 满足幂等性。这个概念被拓展到计算机领域，被用来描述一个操作、方法或者服务。

一个幂等操作的特点是，其**任意多次执行所产生的影响均与一次执行的影响相同**。一个幂等的方法，使用同样的参数，对它进行多次调用和一次调用，对系统产生

的影响是一样的。对于幂等的方法，**不用担心重复执行会对系统造成任何改变。**

业界对于幂等性的一些常见做法：

1. 借助数据库唯一索引，重复插入直接报错，事务回滚。还是举经典的转账的例子，为了保证不重复扣款或者重复加钱，我们这边维护一张“资金变动流水表”，

   里面至少需要交易单号、变动账户、变动金额等3个字段。我们选择交易单号和变动账户做联合唯一索引（单号是上游生成的可保证唯一性），这样如果同一

   笔交易发生重复请求时就会直接报索引冲突，事务直接回滚。现实中，数据库唯一索引的方式通常做为兜底保证；

2. 前置检查机制。这个很容易理解，并且有几种实现办法。还是引用上面转账的例子，当我在执行更改账户余额这个动作之前，我得先检查下资金变动流水表

   （或者Tair中）中是否已经存在这笔交易相关的记录了， select * from xxx where accountNumber=xxx and orderId=yyy ，如果已经存在，那么直接返回，

   否则执行正常的更新余额的动作。为了防止并发问题，我们通常需要借助**“排他锁**”来完成。在支付宝有一条铁律叫：**一锁、二判、三操作**。当然，我们**也可以**

   **使用乐观锁或CAS机制**，乐观锁一般会使用扩展一个版本号字段做判断条件

3. 唯一Id机制，比较通用的方式。对于每条消息我们都可以生成唯一Id，消费前判断Tair中是否存在（MsgId做Tair排他锁的key），消费成功后将状态写入Tair

   中，这样就可以防止重复消费了。

对于接口请求类的幂等性保证要相对更复杂，我们通常要求上游请求时传递一个类GUID的请求号（或TOKEN），如果我们发现已经存在了并且上一次请求处理结

果是成功状态的（有时候上游的重试请求是正常诉求，我们不能将上一次异常/失败的处理结果返回或者直接提示“请求异常”，如果这样重试就变得没意义了）则不

继续往下执行，直接返回“重复请求”的提示和上次的处理结果（上游通常是由于请求超时等未知情况才发起重试的，所以直接返回上次请求的处理结果就好了）。

如果请求ID都不存在或者上次处理结果是失败/异常的，那就继续处理流程，并最终记录最终的处理结果。这个请求序号由上游自己生成，上游通用需要根据请求

参数、时间间隔等因子来生成请求ID。同样也需要利用这个请求ID做分布式锁的KEY实现排他。

### 3.2、可靠性分析

- **Firehose**

  在使用任何消息中间件的过程中，难免会出现消息丢失等异常情况，这个时候就需要有一个良好的机制来跟踪记录消息的过程（轨迹溯源），帮助我们**排查问**

  **题**。在RabbitMQ 中可以使用Firehose 功能来实现消息追踪，Firehose 可以记录每一次发送或者消费消息的记录，方便RabbitMQ 的使用者进行调试、排错

  等。Firehose 的原理是将生产者投递给RabbitMQ 的消息，或者RabbitMQ 投递给消费者的消息按照指定的格式**发送到默认的交换器上**。这个默认的交换器的

  名称为 amq.rabbitmq.trace ，它是一个topic 类型的交换器。发送到这个交换器上的消息的路由键为 publish.{exchangename} 和 deliver. {queuename} 。

  其中 exchangename 和 queuename 为交换器和队列的名称，分别对应生产者投递到交换器的消息和消费者从队列中获取的消息。

  Firehose命令：

  ```shell
  #开启 其中[-p vhost]是可选参数，用来指定虚拟主机vhost。
  rabbitmqctl trace_on [-p vhost]
  #关闭
  rabbitmqctl trace_off [-p vhost]
  #同时需要建立队列进行接收数据，生产环境下不使用
  ```

  **Firehose** 默认情况下处于**关闭状态**，并且Firehose 的**状态是非持久化的**，会在RabbitMQ服务重启的时候还原成默认的状态。Firehose 开启之后多少会**影响**

  **RabbitMQ整体服务性能**，因为它会引起额外的消息生成、路由和存储。

- **rabbitmq_tracing**

  **rabbitmq_tracing** 插件相当于Firehose 的GUI 版本，它同样能跟踪RabbitMQ 中消息的流入流出情况。rabbitmq_tracing 插件同样会对流入流出的消息进行

  封装，然后将封装后的**消息日志存入相应的trace文件中**。

  ```shell
  #启用
  rabbitmq-plugins enable rabbitmq_tracing
  #禁用
  rabbitmq-plugins disable rabbitmq_tracing
  ```

  需要进行设置

  ![](../img/RabbitMQ/RabbitMQ-img-41.png)

  Name表示rabbitmq_tracing的一个条目的名称，

  Format可以选择Text或JSON，

  连接的用户名写root，密码写123456。

  Pattern：发布的消息：publish.<exname>

  Pattern：消费的消息：deliver.<queuename>

### 3.3、TTL机制

![](../img/RabbitMQ/RabbitMQ-img-42.png)

在京东下单，订单创建成功，等待支付，一般会给30分钟的时间，开始倒计时。如果在这段时间内用户没有支付，则默认订单取消。

该如何实现？

- 定期轮询（数据库等）

  用户下单成功，将订单信息放入数据库，同时将支付状态放入数据库，用户付款更改数据库状态。定期轮询数据库支付状态，如超过30分钟就将该订单取消。

  优点：设计实现简单

  缺点：需要对数据库进行大量的IO操作，效率低下。

- Timer

  ```java
  SimpleDateFormat simpleDateFormat = new SimpleDateFormat("HH:mm:ss");
  Timer timer = new Timer();
  TimerTask timerTask = new TimerTask() {
      @Override
      public void run() {
          System.out.println("用户没有付款，交易取消：" + simpleDateFormat.format(new Date(System.currentTimeMillis())));
          timer.cancel();
      }
  };
  // 10秒后执行timerTask timer.schedule(timerTask, 10 * 1000);
  System.out.println("等待用户付款："+simpleDateFormat.format(newDate(System.currentTimeMillis()))); 
  ```

  - 缺点：
    - Timers没有持久化机制.
    - Timers不灵活 (只可以设置开始时间和重复间隔，对等待支付貌似够用)
    - Timers 不能利用线程池，一个timer一个线程
    - Timers没有真正的管理计划

- ScheduledExecutorService

  ```java
  SimpleDateFormat format = new SimpleDateFormat("HH:mm:ss"); // 线程工
  ThreadFactory factory = Executors.defaultThreadFactory(); // 使用线程池 
  ScheduledExecutorService service = new ScheduledThreadPoolExecutor(10, factory); 
  System.out.println("开始等待用户付款10秒："+format.format(newDate())); 
  service.schedule(new Runnable() {
          @Override public void run () {
              System.out.println("用户未付款，交易取消：" + format.format(new Date()));
          }// 等待10s 单位秒 
  }, 10, TimeUnit.SECONDS);
  ```

  - 优点：可以多线程执行，一定程度上避免任务间互相影响，单个任务异常不影响其它任务。
  - 在高并发的情况下，不建议使用定时任务去做，因为太浪费服务器性能，不建议。

- Quartz

- Redis Zset

- JCronTab

- SchedulerX

- RabbitMQ

  使用TTL：TTL，Time to Live 的简称，即过期时间。

  RabbitMQ 可以**对消息和队列两个维度来设置TTL**。

  任何消息中间件的容量和堆积能力都是有限的，如果有一些消息总是不被消费掉，那么需要有一种过期的机制来做兜底。

  目前有两种方法可以设置消息的TTL。 

  1. 通过Queue属性设置，队列中所有消息都有相同的过期时间。

  2. 对消息自身进行单独设置，每条消息的TTL 可以不同。

  如果两种方法一起使用，则消息的TTL 以两者之间**较小数值为准**。通常来讲，消息在队列中的生存时间一旦超过设置的TTL 值时，就会变成“死信”(Dead 

  Message)，消费者默认就无法再收到该消息。当然，“死信”也是可以被取出来消费的，下一小节我们会讲解。

  - 原生案例

    ```java
    package com.thm;
    
    import com.rabbitmq.client.BuiltinExchangeType;
    import com.rabbitmq.client.Channel;
    import com.rabbitmq.client.Connection;
    import com.rabbitmq.client.ConnectionFactory;
    import java.net.URISyntaxException;
    import java.nio.charset.StandardCharsets;
    import java.security.KeyManagementException;
    import java.security.NoSuchAlgorithmException;
    import java.util.HashMap;
    import java.util.Map;
    public class Producer {
        public static void main(String[] args) throws NoSuchAlgorithmException, KeyManagementException, URISyntaxException {
            // 连接工厂
            ConnectionFactory connectionFactory = new ConnectionFactory();
            connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
            try(final Connection connection = connectionFactory.newConnection();
                final Channel channel = connection.createChannel()){
                // 创建队列（实际上使用的是AMQP default这个direct类型的交换器）
                // 设置队列属性
                Map<String, Object> arguments = new HashMap<>();
                // 设置队列消息的TTL 10秒
                arguments.put("x-message-ttl", 10000);
                // 设置队列的空闲存活时间（如该队列根本没有消费者，一直没有使用，队列可以存活多久） 15秒
                arguments.put("x-expires", 15000);
                channel.queueDeclare("queue.ttl",false,false,true,arguments);
                channel.exchangeDeclare("ex.ttl", BuiltinExchangeType.DIRECT,false,true,null);
                channel.queueBind("queue.ttl","ex.ttl","key.ttl");
    
                channel.basicPublish("ex.ttl","key.ttl",null,"过期消息".getBytes(StandardCharsets.UTF_8));
            } catch (Exception e){
                e.printStackTrace();
            }
        }
    }
    ```

    此外，还可以通过命令行方式设置全局TTL，执行如下命令：

    ```shell
    rabbitmqctl set_policy TTL ".*" '{"message-ttl":30000}' --apply-to queues
    ```

    **默认规则：**

    1. 如果不设置TTL，则表示此消息不会过期；

    2. 如果TTL设置为0，则表示除非此时可以直接将消息投递到消费者，否则该消息会被立即丢弃；

    注意理解 message-ttl 、 x-expires 这两个参数的区别，有不同的含义。但是这两个参数属性都遵循上面的默认规则。TTL相关的参数单位都是**毫秒**

  - **springboot案例**

    - pom.xml添加依赖

      ```xml
      <dependencies>
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-amqp</artifactId>
          </dependency>
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-web</artifactId>
          </dependency>
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-test</artifactId>
              <scope>test</scope>
              <exclusions>
                  <exclusion>
                      <groupId>org.junit.vintage</groupId>
                      <artifactId>junit-vintage-engine</artifactId>
                  </exclusion>
              </exclusions>
          </dependency>
          <dependency>
              <groupId>org.springframework.amqp</groupId>
              <artifactId>spring-rabbit-test</artifactId>
              <scope>test</scope>
          </dependency>
      </dependencies>
      ```

    - application.properties添加rabbitmq连接信息

      ```properties
      spring.application.name=ttl 
      spring.rabbitmq.host=192.168.81.100 
      spring.rabbitmq.virtual-host=/ 
      spring.rabbitmq.username=root 
      spring.rabbitmq.password=123456 
      spring.rabbitmq.port=5672
      ```

    - RabbitConfig类 

      ```java
      package com.lagou.rabbitmq.demo.config;
      import org.springframework.amqp.core.*;
      import org.springframework.context.annotation.Bean;
      import org.springframework.context.annotation.Configuration;
      import java.util.HashMap;
      import java.util.Map;
      @Configuration
      public class RabbitConfig {
          @Bean
          public Queue queueTTLWaiting() {
              Map<String, Object> props = new HashMap<>(); 
              // 对于该队列中的消息，设置都等待10s
              props.put("x-message-ttl", 10000);
              Queue queue = new Queue("q.pay.ttl-waiting", false, false, false, props);
              return queue;
          }
      
          @Bean
          public Queue queueWaiting() {
              Queue queue = new Queue("q.pay.waiting", false, false, false);
              return queue;
          }
      
          @Bean
          public Exchange exchangeTTLWaiting() {
              DirectExchange exchange = new DirectExchange("ex.pay.ttl- waiting", false, false);
              return exchange;
          }
      
          /**
            * 该交换器使用的时候，需要给每个消息设置有效期 
            * @return 
            */
          @Bean
          public Exchange exchangeWaiting() {
              DirectExchange exchange = new DirectExchange("ex.pay.waiting", false, false);
              return exchange;
          }
          
          @Bean
          public Binding bindingTTLWaiting() {
              return BindingBuilder.bind(queueTTLWaiting()).to(exchangeTTLWaiting()).wit h("pay.ttl-waiting").noargs();
          }
      
          @Bean
          public Binding bindingWaiting() {
              return BindingBuilder.bind(queueWaiting()).to(exchangeWaiting()).with("pay .waiting").noargs();
          }
      }
      ```

    - PayController类

      ```java
      package com.lagou.rabbitmq.demo.controller;
      
      import org.springframework.amqp.core.AmqpTemplate;
      import org.springframework.amqp.core.Message;
      import org.springframework.amqp.core.MessageProperties;
      import org.springframework.beans.factory.annotation.Autowired;
      import org.springframework.web.bind.annotation.RequestMapping;
      import org.springframework.web.bind.annotation.RestController;
      import java.io.UnsupportedEncodingException;
      import java.util.HashMap;
      import java.util.Map;
      
      @RestController
      public class PayController {
          @Autowired
          private AmqpTemplate rabbitTemplate;
      
          @RequestMapping("/pay/queuettl")
          public String sendMessage() {
              rabbitTemplate.convertAndSend("ex.pay.ttl-waiting", "pay.ttl-waiting", "发送了TTL-WAITING-MESSAGE");
              return "queue-ttl-ok";
          }
      
          @RequestMapping("/pay/msgttl")
          public String sendTTLMessage() throws UnsupportedEncodingException {
              // 设置消息的ttl
              MessageProperties properties = new MessageProperties();
              properties.setExpiration("5000");
              Message message = new Message("发送了WAITING- MESSAGE".getBytes("utf-8"), properties);
              rabbitTemplate.convertAndSend("ex.pay.waiting", "pay.waiting", message);
              return "msg-ttl-ok";
          }
      }
      ```

    - 启动类

      ```java
      package com.lagou.rabbitmq.demo;
      
      import org.springframework.boot.SpringApplication;
      import org.springframework.boot.autoconfigure.SpringBootApplication;
      
      @SpringBootApplication
      public class RabbitmqDemo {
          public static void main(String[] args) {
              SpringApplication.run(RabbitmqDemo07.class, args);
          }
      }
      ```

### 3.4、死信队列

用户下单，调用订单服务，然后订单服务调用派单系统通知外卖人员送单，这时候订单系统与派单系统采用 MQ异步通讯。

在定义业务队列时可以考虑指定一个 死信交换机，并绑定一个死信队列。当消息变成死信时，该消息就会被发送到该死信队列上，这样方便我们查看消息失败的

原因。

DLX，全称为Dead-Letter-Exchange，死信交换器。消息在一个队列中变成死信（Dead Letter）之后，被重新发送到一个特殊的交换器（DLX）中，同时，绑定

DLX的队列就称为“**死信队列**”。

以下几种情况导致消息变为死信：

1. 消息被拒绝（Basic.Reject/Basic.Nack），并且设置requeue参数为false； 

2. 消息过期；

3. 队列达到最大长度。

对于RabbitMQ 来说，DLX 是一个非常有用的特性。它可以处理异常情况下，消息不能够被消费者正确消费(消费者调用了Basic.Nack 或者Basic.Reject)而被置入

死信队列中的情况，后续分析程序可以通过消费这个死信队列中的内容来分析当时所遇到的异常情况，进而可以改善和优化系统。

- 原生案例

  ```java
  package com.thm;
  import com.rabbitmq.client.BuiltinExchangeType;
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  import java.net.URISyntaxException;
  import java.nio.charset.StandardCharsets;
  import java.security.KeyManagementException;
  import java.security.NoSuchAlgorithmException;
  import java.util.HashMap;
  import java.util.Map;
  public class Producer {
      public static void main(String[] args) throws NoSuchAlgorithmException, KeyManagementException, URISyntaxException {
          // 连接工厂
          ConnectionFactory connectionFactory = new ConnectionFactory();
          connectionFactory.setUri("amqp://root:123456@192.168.81.100:5672/%2f");
          try(final Connection connection = connectionFactory.newConnection();
              final Channel channel = connection.createChannel()){
              //创建死信交换器
              channel.exchangeDeclare("ex.dlx",BuiltinExchangeType.DIRECT,true);
              //创建死信队列
              channel.queueDeclare("queue.dlx",false,false,true,null);
              //绑定死信交换器和死信队列
              channel.queueBind("queue.dlx","ex.dlx","key.dlx");
              // 设置队列属性
              Map<String, Object> arguments = new HashMap<>();
              // 设置队列消息的TTL 10秒
              arguments.put("x-message-ttl", 10000);
              // 设置该队列所关联的死信交换器（当队列消息TTL到期后依然没有消费，则加入死信队列）
              arguments.put("x-dead-letter-exchange", "ex.dlx");
              // 设置该队列所关联的死信交换器的routingKey，如果没有特殊指定，使用原队列的 routingKey
              arguments.put("x-dead-letter-routing-key", "key.dlx");
              //创建业务队列
              channel.queueDeclare("queue.biz",false,false,true,arguments);
              //创建业务交换器
              channel.exchangeDeclare("ex.biz", BuiltinExchangeType.DIRECT,false,true,null);
              //绑定
              channel.queueBind("queue.biz","ex.biz","key.biz");
              //发送消息
              channel.basicPublish("ex.biz","key.biz",null,"转移死信队列的消息".getBytes(StandardCharsets.UTF_8));
          } catch (Exception e){
              e.printStackTrace();
          }
      }
  }
  ```

- **springboot案例**

  - pom.xml添加依赖

    ```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.amqp</groupId>
            <artifactId>spring-rabbit-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    ```

  - application.properties添加RabbitMQ连接信息

    ```properties
    spring.application.name=dlx 
    spring.rabbitmq.host=192.168.81.100 
    spring.rabbitmq.virtual-host=/ 
    spring.rabbitmq.username=root 
    spring.rabbitmq.password=123456 
    spring.rabbitmq.port=5672
    ```

  - RabbitConfig类

    ```java
    package com.lagou.rabbitmq.demo.config;
    
    import org.springframework.amqp.core.*;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    
    import java.util.HashMap;
    import java.util.Map;
    
    @Configuration
    public class RabbitConfig {
        @Bean
        public Queue queue() {
            Map<String, Object> props = new HashMap<>();
            // 消息的生存时间 10s 
            props.put("x-message-ttl", 10000);
            // 设置该队列所关联的死信交换器（当队列消息TTL到期后依然没有消费，则加 入死信队列）
            props.put("x-dead-letter-exchange", "ex.go.dlx");
            // 设置该队列所关联的死信交换器的routingKey，如果没有特殊指定，使用原 队列的routingKey 
            props.put("x-dead-letter-routing-key", "go.dlx");
            Queue queue = new Queue("q.go", true, false, false, props);
            return queue;
        }
    	//死信队列
        @Bean
        public Queue queueDlx() {
            Queue queue = new Queue("q.go.dlx", true, false, false);
            return queue;
        }
    
        @Bean
        public Exchange exchange() {
            DirectExchange exchange = new DirectExchange("ex.go", true, false, null);
            return exchange;
        }
    
        /**
        * 死信交换器 
        * @return 
        */
        @Bean
        public Exchange exchangeDlx() {
            DirectExchange exchange = new DirectExchange("ex.go.dlx", true, false, null);
            return exchange;
        }
    
        @Bean
        public Binding binding() {
            return BindingBuilder.bind(queue()).to(exchange()).with("go").noargs();
        }
    
        /**
        * 死信交换器绑定死信队列 
        * @return 
        */
        @Bean
        public Binding bindingDlx() {
            return BindingBuilder.bind(queueDlx()).to(exchangeDlx()).with("go.dlx").no args();
        }
    }
    ```

  - GoController类

    ```java
    package com.lagou.rabbitmq.demo.controller;
    import org.springframework.amqp.core.AmqpTemplate;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;
    
    @RestController
    public class GoController {
        @Autowired
        private AmqpTemplate rabbitTemplate;
    
        @RequestMapping("/go")
        public String distributeGo() {
            rabbitTemplate.convertAndSend("ex.go", "go", "送单到石景山x小区，请在10秒内接受任务");
            return "任务已经下发，等待送单。。。";
        }
    
        @RequestMapping("/notgo")
        public String getAccumulatedTask() {
            String notGo = (String) rabbitTemplate.receiveAndConvert("q.go.dlx");
            return notGo;
        }
    }
    ```

  - 主入口类

    ```java
    package com.lagou.rabbitmq.demo;
    
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    
    @SpringBootApplication
    public class RabbitmqDemo {
        public static void main(String[] args) {
            SpringApplication.run(RabbitmqDemo07.class, args);
        }
    }
    ```

### 3.5、延迟队列

延迟消息是指的消息发送出去后并不想立即就被消费，而是需要等（指定的）一段时间后才触发消费。

例如下面的业务场景：在支付宝上面买电影票，锁定了一个座位后系统默认会帮你保留15分钟时间，如果15分钟后还没付款那么不好意思系统会自动把座位释放

掉。怎么实现类似的功能呢？

1. 可以用定时任务每分钟扫一次，发现有占座超过15分钟还没付款的就释放掉。但是这样做很低效，很多时候做的都是些无用功；

2. 可以用分布式锁、分布式缓存的被动过期时间，15分钟过期后锁也释放了，缓存key也不存在了；

3. 还可以用延迟队列，锁座成功后会发送1条延迟消息，这条消息15分钟后才会被消费，消费的过程就是检查这个座位是否已经是“已付款”状态;

你在公司的协同办公系统上面预约了一个会议，邀请汪产品和陈序员今晚22点准时参加会有。系统还比较智能，除了默认发会议邀请的邮件告知参会者以外，**到了**

**今晚21:45分的时候（提前15分钟）就会通知提醒参会人员做好参会准备，会议马上开始...**

同样的，这也可以通过轮询“会议预定表”来实现，比如我每分钟跑一次定时任务看看当前有哪些会议即将开始了。当然也可以通过延迟消息来实现，预定会议以后

系统投递一条延迟消息，而这条消息比较特殊不会立马被消费，而是延迟到指定时间后再触发消费动作（发通知提醒参会人准备）。不过遗憾的是，在AMQP协议

和RabbitMQ中都没有相关的规定和实现。不过，我们似乎可以借助上一小节介绍的“死信队列”来变相的实现。

可以使用**rabbitmq_delayed_message_exchange**插件实现。

这里和TTL方式有个很大的不同就是TTL存放消息在死信队列(delayqueue)里，二基于插件存放消息在延时交换机里(x-delayed-message exchange)。 

![](../img/RabbitMQ/RabbitMQ-img-43.png)

1. 生产者将消息(msg)和路由键(routekey)发送指定的延时交换机(exchange)上 

2. 延时交换机(exchange)存储消息等待消息到期根据路由键(routekey)找到绑定自己的队列(queue)并把消息给它

3. 队列(queue)再把消息发送给监听它的消费者(customer） 

- 下载插件

  下载地址：https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases

  ![](../img/RabbitMQ/RabbitMQ-img-44.png)

- 安装插件

  将插件拷贝到rabbitmq-server的安装路径：/usr/lib/rabbitmq/lib/rabbitmq_server-3.8.4/plugins

- 启用插件

  ```shell
  rabbitmq-plugins list 
  rabbitmq-plugins enable rabbitmq_delayed_message_exchange
  ```

- 重启rabbitmq-server

  ```sh
  systemctl restart rabbitmq-server
  ```

- **springboot案例**
  - 添加依赖

    ```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.amqp</groupId>
            <artifactId>spring-rabbit-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    ```

  - application.properties中添加rabbitmq的配置

    ```properties
    spring.application.name=ttl 
    spring.rabbitmq.host=192.168.81.100 
    spring.rabbitmq.virtual-host=/ 
    spring.rabbitmq.username=root 
    spring.rabbitmq.password=123456 
    spring.rabbitmq.port=5672
    ```

  - RabbitMQ的对象配置

    ```java
    package com.lagou.rabbitmq.demo.config;
    import org.springframework.amqp.core.*;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import java.util.HashMap;
    import java.util.Map;
    @Configuration
    public class RabbitConfig {
        @Bean
        public Queue queue() {
            Queue queue = new Queue("q.delayed", false, false, false, null);
            return queue;
        }
    
        @Bean
        public Exchange exchange() {
            Map<String, Object> props = new HashMap<>();
            //x-delayed-type 真正交换器类型
            props.put("x-delayed-type", ExchangeTypes.FANOUT);
            //x-delayed-message 使用延时交换器插件
            Exchange exchange = new CustomExchange("ex.delayed", "x-delayed-message", true, false, props);
            return exchange;
        }
    
        @Bean
        public Binding binding() {
            return BindingBuilder.bind(queue()).to(exchange()).with("key.delayed").noa rgs();
        }
    }
    ```

  - 使用推消息模式接收延迟队列的广播

    ```java
    package com.lagou.rabbitmq.demo.listener;
    
    import com.rabbitmq.client.Channel;
    import org.springframework.amqp.core.Message;
    import org.springframework.amqp.rabbit.annotation.RabbitListener;
    import org.springframework.stereotype.Component;
    
    import java.io.IOException;
    
    @Component
    public class MeetingListener {
        @RabbitListener(queues = "q.delayed")
        public void broadcastMeetingAlarm(Message message, Channel channel) throws IOException {
            System.err.println("提醒：5秒后：" + new String(message.getBody(), "utf-8"));
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        }
    }
    ```

  - 开发RestController，用于向延迟队列发送消息，并指定延迟的时长

    ```java
    package com.lagou.rabbitmq.demo.controller;
    import org.springframework.amqp.core.AmqpTemplate;
    import org.springframework.amqp.core.Message;
    import org.springframework.amqp.core.MessageProperties;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.web.bind.annotation.PathVariable;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;
    import java.io.UnsupportedEncodingException;
    @RestController
    public class PublishController {
        @Autowired
        private AmqpTemplate rabbitTemplate;
    
        @RequestMapping("/prepare/{seconds}")
        public String toMeeting(@PathVariable Integer seconds) throws UnsupportedEncodingException {
            // RabbitMQ只会检查队列头部的消息是否过期，如果过期就放到死信队列
            // 假如第一个过期时间很长，10s，第二个消息3s，则系统先看第一个消息，等 到第一个消息过期，放到DLX
            // 此时才会检查第二个消息，但实际上此时第二个消息早已经过期了，但是并没 有先于第一个消息放到DLX。
            // 插件rabbitmq_delayed_message_exchange帮我们搞定这个。
            MessageProperties properties = new MessageProperties();
            properties.setHeader("x-delay", (seconds - 10) * 1000);
            Message message = new Message((seconds + "秒后召开销售部门会议。").getBytes("utf-8"), properties);
            // 如果不设置message的properties，也可以使用下述方法设置x-delay属性的值
            // rabbitTemplate.convertAndSend("ex.delayed", "key.delayed", message, msg -> {
            // // 使用定制的属性x-delay设置过期时间，也就是提前5s提醒
            // // 当消息转换完，设置消息头字段
            // msg.getMessageProperties().setHeader("x-delay", (seconds - 5) * 1000);
            // return msg;
            // });
            rabbitTemplate.convertAndSend("ex.delayed", "key.delayed", message);
            return "已经定好闹钟了，到时提前告诉大家";
        }
    }
    ```

  - 启动类

    ```java
    package com.lagou.rabbitmq.demo;
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    @SpringBootApplication
    public class RabbitmqDemo {
        public static void main(String[] args) {
            SpringApplication.run(RabbitmqDemo.class, args);
        }
    }
    ```


## 4、RabbitMQ集群与运维

### 4.1、集群方案原理

对于无状态应用（如普通的微服务）很容易实现负载均衡、高可用集群。而对于有状态的系统（如数据库等）就比较复杂。

#### 4.1.1、业界实践

- 主备模式：

  单活，容量对等，可以实现故障转移。使用独立存储时需要借助复制、镜像同步等技术，数据会有延迟、不一致等问题（CAP定律），使用共享存储时就不会

  有状态同步这个问题。

- 主从模式：

  一定程度的双活，容量对等，最常见的是读写分离。通常也需要借助复制技术，或者要求上游实现双写来保证节点数据一致。

- 主主模式：

  两边都可以读写，互为主备。如果两边同时写入很容易冲突，所以通常实现的都是“伪主主模式”，或者说就是主从模式的升级版，只是新增了主从节点的选举

  和切换。

- 分片集群：不同节点保存不同的数据，上游应用或者代理节点做路由，突破存储容量限制，分摊读写负载；典型的如MongoDB的分片、MySQL的分库分表、

  Redis集群。

- 异地多活：“两地三中心”是金融行业经典的容灾模式（有资源闲置的问题），“异地多活”才是王道。

#### 4.1.2、常用负载均衡算法

1. 随机

2. 轮询

3. 加权轮询

4. 最少活跃连接

5. 原地址/目标地址hash（一致性hash）

#### 4.1.3、集群中的经典问题

- 脑裂（可以通过协调器选举算法、仲裁节点等方式来解决）

- 网络分区、一致性、可用性（CAP）

相关景点的技术和工具：LVS、HAProxy、Nginx、KeepAlived、Heartbeat、DRBD、Corosync、Pacemaker、MMM/MHA、Galera、MGR等。

现在太多公司选择直接购买公有云服务，基本不用太关心很多基础设施和中间件的部署、运维细节。但是这些技术以及背后的原理是非常重要的。

#### 4.1.4、RabbitMQ分布式架构模式

##### 4.1.4.1、主备模式

也叫Warren（兔子窝）模式，同一时刻只有一个节点在工作（备份节点不能读写），当主节点发生故障后会将请求切换到备份节点上（主恢复后成为备份节

点）。需要借助HAProxy之类的（VIP模式）负载均衡器来做健康检查和主备切换，底层需要借助共享存储（如SAN设备）。这不是RabbitMQ官方或者开源社

区推荐方案，适用于访问压力不是特别大但是又有高可用架构需求（故障切换）的中小规模的系统来使用。首先有一个节点闲置，本身就是资源浪费，其次共

享存储往往需要借助硬件存储，或者分布式文件系统。

![](../img/RabbitMQ/RabbitMQ-img-45.png)

##### 4.1.4.2、Shovel铲子模式

Shovel是一个插件，用于实现跨机房数据复制，或者数据迁移，故障转移与恢复等。

如下图，用户下单的消费先是投递在Goleta Broker实例中，当Goleta实例达到触发条件后（例如：消息堆积数达到阈值）会将消息放到Goleta实例的

backup_orders备份队列中，并通过Shovel插件从Goleta的backup_orders队列中将消息拉取到Carpinteria实例存储。

![](../img/RabbitMQ/RabbitMQ-img-46.png)

![](../img/RabbitMQ/RabbitMQ-img-47.png)

使用Shovel插件后，模型变成了近端同步确认，远端异步确认的方式。

此模式支持WAN传输，并且broker实例的RabbitMQ、Erlang版本不要求完全一致。

Shovel的配置分静态模式（修改RabbitMQ配置）和动态模式（在控制台直接部署，重启后失效）

##### 4.1.4.3、RabbitMQ集群

RabbitMQ集群允许消费者和生产者在RabbitMQ单个节点崩溃的情况下继续运行，并可以通过添加更多的节点来线性扩展消息通信的吞吐量。当失去一个

RabbitMQ节点时，客户端能够重新连接到集群中的任何其他节点并继续生产和消费。

RabbitMQ集群中的所有节点都会备份所有的元数据信息，包括：

1. 队列元数据：队列的名称及属性；

2. 交换器：交换器的名称及属性；

3. 绑定关系元数据：交换器与队列或者交换器与交换器之间的绑定关系；

4. vhost元数据：为vhost内的队列、交换器和绑定提供命名空间及安全属性。

基于存储空间和性能的考虑，RabbitMQ集群中的各节点存储的消息是不同的（有点儿类似分片集群，各节点数据并不是全量对等的），各节点之间同步备份的仅

仅是上述元数据以及QueueOwner（队列所有者，就是实际创建Queue并保存消息数据的节点）的指针。当集群中某个节点崩溃后，该节点的队列进程和关联的

绑定都会消失，关联的消费者也会丢失订阅信息，节点恢复后（前提是消息有持久化）消息可以重新被消费。虽然消息本身也会持久化，但如果节点磁盘存储设备

发生故障那同样会导致消息丢失。

总的来说，该集群模式只能保证集群中的某个Node挂掉后应用程序还可以切换到其他Node上继续地发送和消费消息，但并无法保证原有的消息不丢失，所以并不

是一个真正意义的高可用集群。

![](../img/RabbitMQ/RabbitMQ-img-48.png)

这是RabbitMQ内置的集群模式，Erlang语言天生具备分布式特性，所以不需要借助类似Zookeeper之类的组件来实现集群（集群节点间使用cookie来进行通信验

证，所有节点都必须使用相同的 .erlang.cookie 文件内容），不同节点的Erlang、RabbitMQ版本必须一致。

##### 4.1.4.4、镜像队列模式

前面我们讲了，**RabbitMQ内置的集群模式有丢失消息的风险，“镜像队列”**可以看成是对内置默认集群模式的一种**高可用架构的补充**。可以将队列镜像（同步）到

集群中的其他broker上，相当于是**多副本冗余**。如果集群中的一个节点失效，队列能**自动地切换到集群中的另一个镜像节点上**以保证服务的可用性，而且消息不丢

失。

在RabbitMQ镜像队列中**所谓的master和slave都仅仅是针对某个queue而言的，而不是node**。一个queue第一次创建所在的节点是它的master节点，其他节点

为slave节点。如果master由于某种原因失效，**最先加入的slave会被提升为新的master**。

无论客户端请求到达master还是slave，最终数据都是从master节点获取。当请求到达master节点时，master节点直接将消息返回给client，同时master节点会

通过GM（Guaranteed Multicast）协议将queue的最新状态广播到slave节点。GM保证了广播消息的原子性，即要么都更新要么都不更新。当请求到达slave节点

时，slave节点需要将请求先重定向到master节点，master节点将消息返回给client，同时master节点会通过GM协议将queue的最新状态广播到slave节点。

很多同学可能就会疑惑，这样设计太傻叉了，slave完全是闲置的啊！干嘛不学习MySQL主从复制，起码可以搞个读写分离啊！其实业界很多HA架构实践中冗余资

源都是闲置的。前面我们讲了RabbitMQ镜像队列中的master、slave是Queue维度而并非Node维度，所以我们**可以交叉减少资源限制**，如下图所示：

![](../img/RabbitMQ/RabbitMQ-img-49.png)

##### 4.1.4.5、Federation联邦模式

Federation和Shovel类似，也是一个**实现跨集群、节点消息同步的插件**。支持联邦交换器、联邦队列（作用在不同级别）。

Federation插件允许你配置一个**exchanges federation**或者**queues federation**。

一个exchange/queues federation允许你从一个或者多个upstream接收信息，就是远程的exchange/queues。

![](../img/RabbitMQ/RabbitMQ-img-50.png)

无论是Federation还是Shovel都只是**解决消息数据传输的问题**（当然插件自身可能会一些应用层的优化），跨机房跨城市的这种网络延迟问题是客观存在的，不是

简单的通过什么插件可以解决的，一般需要借助昂贵的专线。

很多书籍和文章中存在误导大家的，可能会说Federation/Shovel可以解决延迟的问题，可以实现异地多活等等，其实这都是错误的。而且我可以负责人的告诉大

家，他们所谓的“异地多活”并非大厂最佳实践。

例如：使用Shovel构建集群，RabbitMQ和应用程序都选择双机房部署时，当杭州机房发生了消息积压后超出阈值部分的消息就会被转发到上海机房中，此时上海

机房的应用程序直接消费掉上海机房RabbitMQ的消息，这样看起来上海机房是可以分摊负载，而且一定程度上实现“双机房多活”的。但是数据库呢？选择两边都

部署还是仅部署在某个机房呢？两边同时写入是很容易造成冲突的，如果数据库仅仅部署在杭州机房，那么数据库也可能成为瓶颈导致消费速度依然上不去，只不

过是多了上海机房中的消费者实例节点而已。

而使用Federation模式呢？如果要真正要实现“双机房多活”那么应用程序也是多机房的，那某些Exchange/Queue中的消息会在两边机房都有，两边机房的应用程

序都会同时消息，那必然会造成重复消息！

##### 4.1.4.6、异地多活架构

| 方案       | 容量                                                         | 容灾                                                         | 成本                                               |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------------------------- |
| 异地多活   | [优]基于逻辑机房，容量可伸缩的云微架构<br/>[优]容量可异地伸缩 | [优]日常运行，容灾时可用性高。<br/>[劣]受城际网络故障影响，影响度取决于横向依赖程度 | [优]IDC、应用等成本在日常得到有效利用              |
| 两地三中心 | [劣]仅可部署在一个城市，容量伸缩有城市级瓶颈                 | [劣]灾备设施冷备等待，容灾时可用性低。                       | [劣]容灾设施等成本仅在容灾时才使用，且受限于可用性 |

### 4.2、单机多实例部署

单机版安装参照：2.2。

此处在单机版基础上 ，也就是一台Linux虚拟机上启动多个RabbitMQ实例，部署集群。

#### 4.2.1、在单个Linux虚拟机上运行多个RabbitMQ实例

- 多个RabbitMQ使用的端口号不能冲突
- 多个RabbitMQ使用的磁盘存储路径不能冲突
- 多个RabbitMQ的配置文件也不能冲突

在单个Linux虚拟机上运行多个RabbitMQ实例，涉及到RabbitMQ虚拟主机的名称不能重复，每个RabbitMQ使用的端口不能重复。

#### 4.2.2、RABBITMQ_NODE_PORT（5672）

用于设置RabbitMQ的服务发现，对外发布的其他端口在这个端口基础上计算得来：

| 端口         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| 4369         | epmd，RabbitMQ节点和CLI工具使用的对等发现服务                |
| 5672、5671   | 分别为不带TLS和带TLS的AMQP 0-9-1和1.0客户端使用              |
| 25672        | 用于节点间和CLI工具通信（Erlang分发服务器端口），并从动态范围分配（默认情况下限制为单个端口，计算为AMQP端口+ 20000）。一般这些端口不应暴露出去。 |
| 35672-35682  | 由CLI工具（Erlang分发客户端端口）用于与节点进行通信，并从动态范围（计算为服务器分发端口+ 10000通过服务器分发端口+ 10010）分配。 |
| 15672        | HTTP API客户端，管理UI和Rabbitmqadmin（仅在启用了管理插件的情况下） |
| 61613、61614 | 不带TLS和带TLS的STOMP客户端（仅在启用STOMP插件的情况下）     |
| 1883、8883   | 如果启用了MQTT插件，则不带TLS和具有TLS的MQTT客户端           |
| 15674        | STOMP-over-WebSockets客户端（仅在启用了Web STOMP插件的情况下） |
| 15675        | MQTT-over-WebSockets客户端（仅在启用Web MQTT插件的情况下）   |
| 15692        | Prometheus指标（仅在启用Prometheus插件的情况下）             |

#### 4.2.3、RABBITMQ_NODENAME

用于设置RabbitMQ节点名称， @ 前缀是用户名， @ 后缀是RabbitMQ所在的Linux主机的 hostname：

- 数据存储目录：

  ![](../img/RabbitMQ/RabbitMQ-img-51.png)

- 日志数据存储目录：

  ![](../img/RabbitMQ/RabbitMQ-img-52.png)

- RabbitMQ使用的环境变量：

  | 环境变量                                | 描述                                                         |
  | --------------------------------------- | ------------------------------------------------------------ |
  | **RABBITMQ_NODE_IP_ADDRESS**            | 将RabbitMQ绑定到一个网络接口。 如果要绑定多个网络接口，可以在配置文件中配置。 默认值：空字串。表示绑定到所有的网络接口。 |
  | **RABBITMQ_NODE_PORT**                  | 默认值：5672                                                 |
  | **RABBITMQ_DIST_PORT**                  | RabbitMQ节点之间通信以及节点和CLI工具通信用到的端口。 如果在配置文件中配置了kernel.inet_dist_listen_min 或者kernel.inet_dist_listen_max，则忽略该配置。 默认值：$RABBITMQ_NODE_PORT + 20000 |
  | **ERL_EPMD_ADDRESS**                    | epmd 使用的网络接口， epmd 用于节点之间以及节点和CLI之间的通信。 默认值：所有网络接口，包括和IPv4。 |
  | **ERL_EPMD_PORT**                       | epmd 使用的端口。 默认值：4369。                             |
  | **RABBITMQ_DISTRIBUTION_BUFFER_SIZE**   | 节点之间通信连接使用的发送数据缓冲区大小限制， 单位是KB。推荐使用小于64MB的值。 默认值：128000。 |
  | **RABBITMQ_IO_THREAD_POOL_SIZE**        | Erlang运行时的 I/O 用到的线程数。不推荐小于32的值。 默认值：128（Linux），64（Windows） |
  | **RABBITMQ_NODENAME**                   | RabbitMQ的节点名称。对于Erlang节点和机器，此名称应该唯一。 通过设置此值，可以在一台机器上多个RabbitMQ节点。 默认值：rabbit@$HOSTNAME（Unix-like），rabbit@%COMPUTERNAME%（Windows） |
  | **RABBITMQ_CONFIG_FILE**                | RabbitMQ主要配置文件的路径。例如 /etc/rabbitmq/rabbitmq.conf 或者 /data/configuration/rabbitmq.conf 是新格式的配置文件。 如果是老格式的配置文件，扩展名是 .config 或者不写。 默认值： 对于Unix： $RABBITMQ_HOME/etc/rabbitmq/rabbitmq Debian:/etc/rabbitmq/rabbitmq RPM： /etc/rabbitmq/rabbitmq MacOS(Homebrew)： ${install_prefix}/etc/rabbitmq/rabbitmq ， Homebrew的前缀通常是： /usr/local/ Windo%APPDATA%\RabbitMQ\rabbitmq |
  | **RABBITMQ_ADVANCED_CONFIG_FILE**       | RabbitMQ带 .config 的高级配置文件路径（基于Erlang配置）。例如， /data/rabbitmq/advanced.config 。<br/>默认值：<br/>Unix： $RABBITMQ_HOME/etc/rabbitmq/advanced<br/>Debian： /etc/rabbitmq/advanced<br/>RPM： /etc/rabbitmq/advanced<br/>MacOS(Homebrew)： ${install_prefix}/etc/rabbitmq/advanced ， 其中Homebrew前缀通常是 /etc/local/<br/>Windows： %APPDATA%\RabbitMQ\advanced 。 |
  | **RABBITMQ_CONF_ENV_FILE**              | 包含了环境变量定义的文件的目录（不使用 RABBITMQ_ 前缀）。 Windows上的文件名称与其他操作系同。 <br/>默认值：<br/>UNIX： $RABBITMQ_HOME/etc/rabbitmq/rabbitmq-env.conf Ubuntu和<br/>Debian： /etc/rabbitmq/rabbitmq-env.conf<br/>RPM： /etc/rabbitmq/rabbitmq-env.conf<br/>Mac(Homebrew)： ${install_prefix}/etc/rabbitmq/rabbitmq-env.conf ，Homebrew的前缀一般是 /usr/local <br/>Windows： %APPDATA%\RabbitMQ\rabbitmq-env-conf.bat |
  | **RABBITMQ_MNESIA_BASE**                | 包含了RabbitMQ服务器的节点数据库、消息存储以及 集群状态文件子目录的根目录。除非显式设置了RABBITMQ_MNESIA_DIR 的值。需要确保RabbitMQ用户 在该目录拥有读、写和创建文件以及子目录的该变量一般不要覆盖。一般覆盖 RABBITMQ_MNESIA_DIR 变量。 <br/>默认值： <br/>Unix： $RABBITMQ_HOME/var/lib/rabbitmq/mnesia <br/>Ubuntu和Debian： /var/lib/rabbitmq/mnesia/<br/>RPM： /var/lib/rabbitmq/plugins <br/>MacOS（Homebrew）：${install_prefix}/var/lib/rabbitmq/mnesia ， 其中Homebrew的前缀一般是 /usr/local<br/>Windows： %APPDATA%\RabbitMQ |
  | **RABBITMQ_MNESIA_DIR**                 | RabbitMQ节点存储数据的目录。该目录中包含了数据库、 消息存储、集群成员信息以及节点其他的持状态。 <br/>默认值： <br/>通用UNIX包： $RABBITMQ_MNESIA_BASE/$RABBITMQ_NODENAME <br/>Ubuntu和Debia：$RABBITMQ_MNESIA_BASE/$RABBITMQ_NODENAME <br/>RPM： $RABBITMQ_MNESIA_BASE/$RABBITMQ_NO<br/>MacOS (Homebrew)： ${install_prefix}/var/lib/rabbitmq/mnesia/$RABBITMQ_NODENAME ，Homebrew的前缀一般是 /usr/local <br/>Windows: %APPDATA%\RabbitMQ\$RABBITMQ_NODENAME |
  | **RABBITMQ_PLUGINS_DIR**                | 存放插件压缩文件的目录。RabbitMQ从此目录解压插件。 跟PATH变量语法类似，多个路径之间使用统的分隔符分隔 （Unix是 : ，Windows是';'）。插件可以安装到该变量指定的任何目录。 路径不要有符。 默认值： <br/>通用UNIX包： $RABBITMQ_HOME/plugins<br/>Ubuntu和Debian包： /var/lib/rabbitmq/plugins <br/>RPM： /var/lib/rabbitmq/plugins <br/>MacOS (Homebrew)： ${install_prefix}/Cellar/rabbitmq/${version}/plugins ， Homebrew的前缀一般是 /usr/l<br/>Windows： %RABBITMQ_HOME%\plugins |
  | **RABBITMQ_PLUGINS_EXPAND_DIR**         | 节点解压插件的目录，并将该目录添加到代码路径。该路径不要包含特殊字符。 <br/>默认值： <br/>UNIX： $RABBITMQ_MNESIA_BASE/$RABBITMQ_NODENAME-plugins-expand <br/>Ubuntu和Debian packages： $RABBITMQ_MNESIA_BASE/$RABBITMQ_NODENAME-plugins-expand RPM： $RABBITMQ_MNESIA_BASE/$RABBITMQ_NODENAME-plugins-expand <br/>MacOS (Homebrew)： ${install_prefix}/var/lib/rabbitmq/mnesia/$RABBITMQ_NODENAME-plugins-expand <br/>Window%APPDATA%\RabbitMQ\$RABBITMQ_NODENAME-plugins-expand |
  | **RABBITMQ_USE_LONGNAME**               | 当设置为 true 的时候，RabbitMQ会使用全限定主机名标记节点。 在使用全限定域名的环境中使用。重置节点， 不能在全限定主机名和短名之间切换。 默认值： false 。 |
  | **RABBITMQ_SERVER_CODE_PATH**           | 当启用运行时的时候指定的外部代码路径（目录）。 当节点启动的时候，这个是值传给 erl 的命令行默认值：(none) |
  | **RABBITMQ_CTL_ERL_ARGS**               | 当调用 rabbitmqctl 的时候传给 erl 的命令行参数。 可以给Erlang设置使用端口的范围： -kernel inet_dist_listen_min 35672 -kernel inet_dist_listen_max 35680 默认值：(none) |
  | **RABBITMQ_SERVER_ERL_ARGS**            | 当调用RabbitMQ服务器的时候 erl 的标准命令行参数。 仅用于测试目的。使用该环境变量会覆盖默认<br/>默认值： <br/>Unix*： +P 1048576 +t 5000000 + stbt db +zdbbl 128000 <br/>Windows：没有 |
  | **RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS** | 调用RabbitMQ服务器的时候传递给 erl 命令的额外参数。 该变量指定的变量追加到默认参数列表<（ RABBITMQ_SERVER_ERL_ARGS ）。 默认值： Unix*： 没有 Windows： 没有 |
  | **RABBITMQ_SERVER_START_ARGS**          | 调用RabbitMQ服务器的时候传给 erl 命令的额外参数。该变量不覆盖 RABBITMQ_SERVER_ERL_ARGS默认值：没有 |
  | **RABBITMQ_ENABLED_PLUGINS_FILE**       | 用于指定 enabled_plugins 文件所在的位置。默认： /etc/rabbitmq/enabled_plugins |

- 启动方式

  - 方式一

    ```shell
    export RABBITMQ_NODE_PORT=5672 
    export RABBITMQ_NODENAME=rabbit2 
    rabbitmq-server -detached
    export RABBITMQ_NODE_PORT=5673 
    export RABBITMQ_NODENAME=rabbit3 
    rabbitmq-server -detached
    export RABBITMQ_NODE_PORT=5674 
    export RABBITMQ_NODENAME=rabbit4 
    rabbitmq-server -detached
    ```

  - 方式二

    ```shell
    RABBITMQ_NODE_PORT=5672 RABBITMQ_NODENAME=rabbit2 rabbitmq-server -detached
    RABBITMQ_NODE_PORT=5673 RABBITMQ_NODENAME=rabbit3 rabbitmq-server -detached
    RABBITMQ_NODE_PORT=5674 RABBITMQ_NODENAME=rabbit4 rabbitmq-server -detached
    ```

  以上命令的运行主要考虑到环境变量的可见性问题。

#### 4.2.4、启动web控制台的管理插件

```shell
rabbitmq-plugins enable rabbitmq_management
```

rabbitmq从3.3.0开始禁止使用guest/guest权限通过除localhost外的访问。如果想使用guest/guest通过远程机器访问，需要在rabbitmq配置文件中设置

loopback_users为[]，当然也可以按我之前讲的命令自己创建用户。

```shell
vim /etc/rabbitmq/rabbitmq.conf
#配置guest远程登陆
loopback_users.guest=false
#配置管理界面访问端口 默认15672
management.tcp.port=15672
```

需要注意的是，如果要使用自定义位置的配置文件，需要目录属于rabbitmq 组。默认配置文件的位置：/etc/rabbitmq/rabbitmq.conf

我们在/opt/rabbitconf中创建三个配置文件：rabbit1.conf，rabbit2.conf，rabbit3.conf，其中注明三个RabbitMQ实例使用的rabbitmq_management插件使用

的端口号，以及开通guest远程登录系统的权限：

修改文件权限

```shell
chown :rabbit rabbitconf
chown :rabbit rabbit*.conf
```

| 文件路径                       | 内容                                                    |
| ------------------------------ | ------------------------------------------------------- |
| /opt/rabbitmqconf/rabbit1.conf | loopback_users.guest=false<br/>management.tcp.port=6001 |
| /opt/rabbitmqconf/rabbit2.conf | loopback_users.guest=false<br/>management.tcp.port=6002 |
| /opt/rabbitmqconf/rabbit3.conf | loopback_users.guest=false<br/>management.tcp.port=6003 |

- **启动命令:**

  | 节点名称 | 命令                                                         |
  | -------- | ------------------------------------------------------------ |
  | rabbit1  | RABBITMQ_NODENAME=rabbit1 RABBITMQ_NODE_PORT=5001 RABBITMQ_CONFIG_FILE=/opt/rabbitconf/rabbit1.conf rabbitmq-server -detached |
  | rabbit2  | RABBITMQ_NODENAME=rabbit2 RABBITMQ_NODE_PORT=5002 RABBITMQ_CONFIG_FILE=/opt/rabbitconf/rabbit2.conf rabbitmq-server -detached |
  | rabbit3  | RABBITMQ_NODENAME=rabbit3 RABBITMQ_NODE_PORT=5003 RABBITMQ_CONFIG_FILE=/opt/rabbitconf/rabbit3.conf rabbitmq-server -detached |

- **停止命令：**

  | 节点名称 | 命令                        |
  | -------- | --------------------------- |
  | rabbit1  | rabbitmqctl -n rabbit1 stop |
  | rabbit2  | rabbitmqctl -n rabbit2 stop |
  | rabbit3  | rabbitmqctl -n rabbit3 stop |

### 4.3、RabbitMQ集群管理

前面我们讲了几种RabbitMQ分布式/集群架构的模式，下面我们结合Rabbit集群+ 镜像队列，并借助HAProxy 实现负载均衡的集群。

- **在node1、node2、node3三台Linux虚拟机中安装RabbitMQ**

- **从node1拷贝.erlang.cookie到node2、node3的相应目录**

  如果没有该文件，手动创建/var/lib/rabbitmq/.erlang.cookie ，生成Cookie字符串，或者启动一次RabbitMQ自动生成该文件。生产中推荐使用第三方工具生成。

  - 我们首先在node2上启动单机版RabbitMQ，以生成Cookie文件，启动如果看不到文件（使用命令 ls -al ：查看隐藏文件）

  - 开始准备同步 .erlang.cookie 文件。RabbitMQ的集群依赖Erlang的分布式特性，需要保持Erlang Cookie一致才能实现集群节点的认证和通信，我们直接

    使用scp命令从node1远程传输

    ```shell
    scp .erlang.cookie root@node2:/var/lib/rabbitmq/
    scp .erlang.cookie root@node3:/var/lib/rabbitmq/
    ```

  - 修改node2和node3上该文件的所有者为rabbitmq:rabbitmq：

    ```shell
    chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
    ```

    注意.erlang.cookie文件权限为400：

    ![](../img/RabbitMQ/RabbitMQ-img-53.png)

- **使用下述命令启动node2和node3上的RabbitMQ**

  在node1上查看集群状态，并记录集群名称

  ```apl
  rabbitmqctl cluster_status
  ```

  分别执行如下命令：

  ```apl
  # 停止Erlang VM上运行的RabbitMQ应用，保持Erlang VM的运行 
  rabbitmqctl stop_app 
  # 移除当前RabbitMQ虚拟主机中的所有数据：重置 
  rabbitmqctl reset 
  # 将当前RabbitMQ的主机加入到rabbit@node1这个虚拟主机的集群中。一个节点也是集群。
  rabbitmqctl join_cluster rabbit@node1 
  # 启动当前Erlang VM上的RabbitMQ应用 
  rabbitmqctl start_app
  #查看集群状态（任意节点执行都可以） 
  rabbitmqctl cluster_status
  ```

  1. rabbit@node2 表示RabbitMQ节点名称，默认前缀就是 rabbit ， @ 之后是当前虚拟主机所在的物理主机 hostname 。 

  2. 注意检查下hostname要可以相互ping通 

  3. join_cluster默认是使用disk模式，后面可以加入参数--ram启用内存模式

  **移出集群节点使用**

  ```apl
  # 将虚拟主机（RabbitMQ的节点）rabbit@node3从集群中移除，但是rabbit@node3还保留集群信息
  # 还是会尝试加入集群，但是会被拒绝。可以重置rabbit@node3节点。 
  rabbitmqctl forget_cluster_node rabbit@node3
  #修改集群名称（任意节点执行都可以） 
  rabbitmqctl set_cluster_name 
  #查看集群状态（任意节点执行都可以） 
  rabbitmqctl cluster_status
  ```

  **在三个RabbitMQ节点上的任意一个添加用户，设置用户权限，设置用户标签，即可**

  ```apl
  rabbitmqctl add_user root 123456 
  rabbitmqctl set_permissions --vhost "/" root ".*" ".*" ".*" 
  rabbitmqctl set_user_tags --vhost "/" root administrator
  ```

  可以到web控制台查看集群信息，如果要看到所有RabbitMQ节点上的运行情况，都需要启用rabbitmq_management 插件。

### 4.4、RabbitMQ镜像集群配置

RabbitMQ中队列的内容是保存在单个节点本地的（声明队列的节点）。跟交换器和绑定不同，它们是对于集群中所有节点的。如此，则**队列内容存在单点故障**，

解决方式之一就是**使用镜像队列**。在多个节点上拷贝队列的副本。

每个镜像队列包含一个master，若干个镜像。

master存在于称为master的节点上。

所有的操作都是首先对master执行，之后广播到镜像。

这涉及排队发布，向消费者传递消息，跟踪来自消费者的确认等。

**镜像意味着集群，不应该WAN使用**。

发布到队列的消息会拷贝到该队列所有的镜像。消费者连接到master，当消费者对消息确认之后，镜像删除master确认的消息。

队列的镜像提供了高可用，但是没有负载均衡。

HTTP API和CLI工具中队列对象的字段原来使用的是slave代表secondaries，现在盖字段的存在仅是为了向后兼容，后续版本会移除。

可以使用策略随时更改队列的类型，可以首先创建一个非镜像队列，然后使用策略将其配置为镜像队列或者反过来。非镜像队列没有额外的基础设施，因此可以提

供更高的吞吐率。

master选举策略：

1. **最长的运行镜像升级为主镜像**，前提是**假定它与主镜像完全同步**。如果没有与主服务器同步的镜像，则仅存在于主服务器上的消息将丢失。 

2. 镜像认为所有以前的消费者都已突然断开连接。它重新排队已传递给客户端但正在等待确认的所有消息。这包括客户端已为其发出确认的消息，例如，确认是

   在到达节点托管队列主节点之前在线路上丢失了，还是在从主节点广播到镜像时丢失了。在这两种情况下，新的主服务器都别无选择，只能重新排队它尚未收

   到确认的所有消息。

3. 队列故障转移时请求通知的消费者将收到取消通知。当镜像队列发生了master的故障转移，系统就不知道向哪些消费者发送了哪些消息。已经发送的等待确认

   的消息会重新排队。

4. 重新排队的结果是，从队列重新使用的客户端必须意识到，他们很可能随后会收到已经收到（重复）的消息。

5. 当所选镜像成为主镜像时，在此期间发布到镜像队列的消息将不会丢失（除非在提升的节点上发生后续故障）。发布到承载队列镜像的节点的消息将路由到队

   列主服务器，然后复制到所有镜像。如果主服务器发生故障，则消息将继续发送到镜像，并在完成向主服务器的镜像升级后将其添加到队列中。

6. 即使主服务器（或任何镜像）在正在发布的消息与发布者收到的确认之间失败，由客户端使用发布者确认发布的消息仍将得到确认。从发布者的角度来看，发

   布到镜像队列与发布到非镜像队列没有什么不同。

给队列添加镜像要**慎重**。

| ha-mode | ha-params | 描述                                                         |
| ------- | --------- | ------------------------------------------------------------ |
| exactly | count     | 设置集群中队列副本的个数（镜像+master）。<br/>1表示一个副本；也就是master。如果master不可用，行为依赖于队列的持久化机制。<br/>2表示1个master和1个镜像。如果master不可用，则根据镜像推举策略从镜像中选出一个做master。<br/>如果节点数量比镜像副本个数少，则镜像覆盖到所有节点。<br/>如果count个数少于集群节点个数，则在一个镜像宕机后，会在其他节点创建出来一个镜像。<br/>将“exactly”模式与“ha-promote-on-shutdown”: “ always”一起使用可能很危险，因为队列可以在整个集群中迁移并在关闭时变得不同步。 |
| all     | （none）  | 镜像覆盖到集群中的所有节点。当添加一个新的节点，队列就会复制过去。这个配置很保守。一般推荐N/2+1个节点。在集群所有节点拷贝镜像会给集群所有节点施加额外的负载，包括网络IO，磁盘IO和磁盘空间使用。 |
| nodes   | nodeNames | 在指定node name的节点上复制镜像。node name就是在rabbitmqctl cluster_status命令输出中的node name。如果有不属于集群的节点名称，它不报错。如果指定的节点都不在线，则仅在客户端连接到的声明镜像的节点上创建镜像。 |

启用镜像队列命令（前提是在集群环境下）：

```apl
# 对/节点配置镜像队列，使用全局复制 
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}' 
# 配置过半（N/2 + 1）复制镜像队列   正则匹配
rabbitmqctl set_policy ha-halfmore "^queue.*a$" '{"ha-mode":"exactly", "ha-params":2}' 
# 指定优先级，数字越大，优先级越高 
rabbitmqctl set_policy --priority 1 ha-all "^" '{"ha-mode":"all"}'
```

其他细节可以查看官方文档：https://www.rabbitmq.com/ha.html

![](../img/RabbitMQ/RabbitMQ-img-54.png)

### 4.5、负载均衡-HAProxy

将客户端的连接和操作的压力分散到集群中的不同节点，防止单个或几台服务器压力过大成为访问的瓶颈，甚至宕机。

- HAProxy是一款开源免费，并提供高可用性、负载均衡以及基于TCP和HTTP协议的代理软件，可以支持四层、七层负载均衡，经过测试单节点可以支持10W左

  右并发连接。

- LVS是工作在内核模式（IPVS），支持四层负载均衡，实测可以支撑百万并发连接。

- Nginx支持七层的负载均衡（后期的版本也支持四层了），是一款高性能的反向代理软件和Web服务器，可以支持单机3W以上的并发连接。

这里我们使用HAProxy来做RabbitMQ的负载均衡，通过暴露VIP给上游的应用程序直接连接，上游应用程序不感知底层的RabbitMQ的实例节点信息。

- **安装HAproxy**

  ```apl
  yum install gcc -y 
  tar -zxf haproxy-2.1.0.tar.gz 
  cd haproxy-2.1.0 
  make TARGET=linux-glibc 
  make install mkdir /etc/haproxy 
  #赋权 添加用户组
  groupadd -r -g 149 haproxy 
  # 添加用户 
  useradd -g haproxy -r -s /sbin/nologin -u 149 haproxy 
  #创建haproxy配置文件 
  touch /etc/haproxy/haproxy.cfg
  #查看创建用户ID ，记录id在配置文件中使用
  id proxy
  ```

  或者：你过你觉得编译安装很麻烦，你可以简单的

  ```apl
  #使用yum安装的，那么haproxy默认在/usr/sbin/haproxy，且会自动创建配置文件/etc/haproxy/haproxy.cfg
  yum -y install haproxy
  #默认会创建proxy用户组和proxy用户
  #查看创建用户ID ，记录id在配置文件中使用
  id proxy
  ```

- **配置HAProxy**

  修改

  ```apl
  vim /etc/haproxy/haproxy.cfg
  ```

  配置文件都做了详细注释，如下：

  ```apl
  global
  	log 127.0.0.1 local0 info 
  	# 服务器最大并发连接数；如果请求的连接数高于此值，将其放入请求队列，等待其它连接被释 放； 
  	maxconn 5120 
  	# chroot /tmp 
  	# 指定用户 
  	uid 188 
  	# 指定组
  	gid 188 
  	# 让haproxy以守护进程的方式工作于后台，其等同于“-D”选项的功能 
  	# 当然，也可以在命令行中以“-db”选项将其禁用； 
  	daemon 
  	# debug参数 
  	quiet 
  	# 指定启动的haproxy进程的个数，只能用于守护进程模式的haproxy； 
  	# 默认只启动一个进程， 
  	# 鉴于调试困难等多方面的原因，在单进程仅能打开少数文件描述符的场景中才使用多进程模式； 
  	# nbproc 20 
  	nbproc 1 
  	pidfile /var/run/haproxy.pid
  defaults
  	log global 
  	# tcp：实例运行于纯TCP模式，第4层代理模式，在客户端和服务器端之间将建立一个全双工的连接，且不会对7层报文做任何类型的检查； 
  	# 通常用于SSL、SSH、SMTP等应用；
      mode tcp 
      option tcplog 
      option dontlognull 
      retries 3 
      option redispatch 
      maxconn 2000 
      # contimeout 5s 
      timeout connect 5s 
      # 客户端空闲超时时间为60秒则HA 发起重连机制 
      timeout client 60000 
      # 服务器端链接超时时间为15秒则HA 发起重连机制 
      timeout server 15000
      
  listen rabbitmq_cluster
  	# VIP，反向代理到下面定义的三台Real Server 
  	bind 192.168.81.103:5672 
  	#配置TCP模式 
  	mode tcp 
  	#简单的轮询 
  	balance roundrobin 
  	# rabbitmq集群节点配置 
  	# inter 每隔五秒对mq集群做健康检查，2次正确证明服务器可用，2次失败证明服务器不可用， 并且配置主备机制 
  	server rabbitmqNode1 192.168.81.100:5672 check inter 5000 rise 2 fall 2 
  	server rabbitmqNode2 192.168.81.101:5672 check inter 5000 rise 2 fall 2 
  	server rabbitmqNode3 192.168.81.102:5672 check inter 5000 rise 2 fall 2
  
  #配置haproxy web监控，查看统计信息 
  listen stats
  	bind 192.168.81.103:9000 
  	mode http 
  	option httplog 
  	# 启用基于程序编译时默认设置的统计报告 
  	stats enable 
  	#设置haproxy监控地址为http://192.168.81.103:9000/rabbitmq-stats 
  	stats uri /rabbitmq-stats
  	# 每5s刷新一次页面 
  	stats refresh 5s
  ```

- **启动和关闭HAProxy**

  ```apl
  #启动
  haproxy -f /etc/haproxy/haproxy.cfg
  #关闭
  kill -9 <pid>
  ```

- 测试基于springboot

  - 添加依赖

    ```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
            <version>2.3.5.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <version>2.3.5.RELEASE</version>
        </dependency>
    </dependencies>
    ```

  - 添加配置文件，直接连接HAproxy的VIP

    ```yaml
    spring:
      application:
        name: spting-boot-rabbitmq-haproxy
      rabbitmq:
        host: 192.168.81.103
        virtual-host: /
        username: root
        password: 123456
        port: 5672
    ```

  - 创建配置类

    ```java
    package com.thm.producer;
    
    import org.springframework.amqp.core.*;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    
    @Configuration
    public class ProducerRabbitConfig {
    
        @Bean
        public Queue queue(){
            //return QueueBuilder.nonDurable("queue.spring.boot.ha").exclusive().autoDelete().build();
            return new Queue("queue.spring.boot.ha",false,false,true);
        }
        @Bean
        public Exchange exchange(){
            return new DirectExchange("ex.spring.boot.ha",false,true);
        }
        @Bean
        public Binding binding(Queue queue,Exchange exchange){
            //Binding binding = BindingBuilder.bind(queue).to(exchange).with("ha.boot.key").noargs();
            //return binding;
            return new Binding("queue.spring.boot.ha",
                    Binding.DestinationType.QUEUE,
                    "ex.spring.boot.ha",
                    "ha.boot.key",
                    null);
        }
    }
    ```

  - 创建listener消费消息

    ```java
    package com.thm.comsumer;
    import org.springframework.amqp.rabbit.annotation.RabbitListener;
    import org.springframework.messaging.handler.annotation.Header;
    import org.springframework.messaging.handler.annotation.Payload;
    import org.springframework.stereotype.Component;
    @Component
    public class MyMessageListener {
    
        @RabbitListener(queues = "queue.spring.boot.ha")
        public void onMessage(@Payload String message, @Header("hello") String value) {
            System.out.println(message);
            System.out.println("hello: " + value);
        }
    }
    
    ```

  - 启动类

    ```java
    package com.thm;
    
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    
    @SpringBootApplication
    public class Bootstrap {
        public static void main(String[] args) {
            SpringApplication.run(Bootstrap.class,args);
        }
    }
    ```

  - 测试类发送消息（先启动启动类，再发送消息）   

    ```java
    import com.thm.Bootstrap;
    import org.junit.runner.RunWith;
    import org.springframework.amqp.core.Message;
    import org.springframework.amqp.core.MessageBuilder;
    import org.springframework.amqp.core.MessageProperties;
    import org.springframework.amqp.core.MessagePropertiesBuilder;
    import org.springframework.amqp.rabbit.core.RabbitTemplate;
    import org.springframework.boot.test.context.SpringBootTest;
    import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
    
    import javax.annotation.Resource;
    import java.io.UnsupportedEncodingException;
    @RunWith(SpringJUnit4ClassRunner.class)
    @SpringBootTest(classes = Bootstrap.class)
    public class Test {
        @Resource
        private RabbitTemplate rabbitTemplate;
    
        /**
         * 发送消息
         * @throws UnsupportedEncodingException
         */
        @org.junit.Test
        public void send() throws UnsupportedEncodingException {
            //封装消息参数
            MessageProperties build = MessagePropertiesBuilder
                    .newInstance()
                    .setContentEncoding("utf-8")
                    .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
                    .setHeader("hello","word")
                    .build();
            //发送消息
            for (int i = 0; i < 20; i++) {
                //封装消息
                Message message = MessageBuilder.withBody(("你好，rabbit spring boot haproxy" + i).getBytes("utf-8"))
                        .andProperties(build)
                        .build();
                rabbitTemplate.send("ex.spring.boot.ha", "ha.boot.key", message);
            }
        }
    }
    ```

### 4.6、监控

RabbitMQ自带的（Management插件）管理控制台功能比较丰富，不仅提供了Web UI界面，还暴露了很多HTTP API的能力。其中也具备基本的监控能力。此

外，自带的命令行工具（例如：rabbitmqctl ）也比较强大。

不过这些工具都不具备告警的能力。在实际的生产环境中，我们需要知道负载情况和运行监控状态（例如：系统资源、消息积压情况、节点健康状态等），而且当

发生问题后需要触发告警。像传统的监控平台Nagios、Zabbix等均提供了RabbitMQ相关的插件支持。

另外，当前云原生时代最热门的Prometheus监控平台也提供了rabbitmq_exporter，结合Grafana漂亮美观的dashboard（可以自定义，也可以在仓库选择一些

现有的），我们目前公司就是使用Prometheus + Grafana来监控RabbitMQ的，并实现了水位告警通知。

感兴趣可以自己研究。https://www.rabbitmq.com/prometheus.html

![](../img/RabbitMQ/RabbitMQ-img-55.png)