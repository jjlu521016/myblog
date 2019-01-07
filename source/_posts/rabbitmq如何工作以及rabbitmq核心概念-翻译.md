---
title: rabbitmq如何工作以及rabbitmq核心概念(翻译)
toc: true
date: 2019-01-07 11:02:52
tags:
 - rabbitmq
---
> 原文出处: https://dzone.com/articles/how-rabbitmq-works-and-rabbitmq-core-concepts-1  

在本文中，我们将学习什么是[RabbitMQ](http://www.javaguides.net/p/rabbitmq-java-tutorial-with-examples.html)，它是如何工作的，以及[RabbitMQ](http://www.javaguides.net/p/rabbitmq-java-tutorial-with-examples.html)的核心概念。
<!-- more -->
RabbitMQ是一个开源的消息代理软件。它接受来自生产者的消息并将其传递给消费者。它就像一个中间人，可以用来减少Web应用服务器的负载和投递时间。
## RabbitMQ是如何工作的  
我们简单的看下RabbitMQ是如何工作的。
我们首先熟悉rabbitmq的几个重要概念:  
+ 生产者(Producer)：发送消息的应用。
+ 消费者(Consumer)：接收消息的应用。
+ 队列(Queue)：存储消息的缓冲区。
+ 消息(Message)：通过RabbitMQ从生产者发送给消费者的信息。
+ 连接(Connection)：连接是应用程序和RabbitMQ代理之间的TCP连接。
+ 通道(Channel)：通道是连接内部的虚拟连接。当您发布或使用队列中的消息时，都是通过通道完成的。
+ 交换机(Exchange)：接收来自生产者的消息，并根据交换类型定义的规则将它们推送到队列中。要接收消息，需要将队列绑定到至少一个交换。
+ 绑定(Binding)：绑定是队列和交换之间的链接。
+ 路由密钥(Routing key)：路由密钥是Exchange用来决定如何将消息路由到队列的密钥。路由密钥类似于邮件的地址。

**Producers**向代理发送/发布消息->**Consumers**从代理接收消息。**RabbitMQ**充当生产者和消费者之间的通信中间件，即使它们在不同的机器上运行。
当生产者向队列中发送消息时，它不会直接发送，而是使用交换机发送。下面的设计演示了三个主要组件是如何相互连接的。
交换代理负责将消息路由到不同队列。以便消息可以从生产者接收到交换，然后再次转发到队列。这就是所谓的“发布”方法。  

<img src="/image/rabbitmq/rabbitmq1.png" />

将从队列中提取和使用消息；这称为“使用”。

## 发送消息到多个队列
通过拥有更复杂的应用程序，我们将拥有多个队列。因此消息将在多个队列中发送它。
<img src="/image/rabbitmq/rabbitm-multiple-queues.png"/>
将消息发送到多个队列交换通过绑定和路由键连接到队列。绑定是为将队列连接到交换而设置的“链接”。路由密钥是一个消息属性。在决定如何将消息路由到队列时（取决于交换类型），交换可能会查看此键。

## 交换机
消息不是直接通过队列直接发送，相反，生产者通过交换机发送消息。交换机负责将消息路由到不同的队列。交换机接受来自生产者应用程序的消息，并在绑定和路由键的帮助下将它们路由到消息队列。绑定连接着队列和交换机。
<img src="/image/rabbitmq/exchanges-bidings-routing-keys.png"/>  

## RabbitMQ中的消息流  
+ 生产者发布一个消息到交换机。当创建交换机时，必须指定其类型。稍后将详细解释不同类型的交换。
+ 交换机接收消息后立马负责消息的路由。根据交换类型，交换会考虑不同的消息属性，例如路由密钥。
+ 必须创建从交换机到队列的绑定。在本例中，我们看到两个绑定到来自交换机的两个不同队列。交换机根据消息属性将消息路由到队列中。
+ 消息一直在队列中，直到被消费者处理
+ 消费者处理消息。

## 交换机的类型  
<img src="/image/rabbitmq/exchanges-topic-fanout-direct.png"/>
1. 直接类型(Direct)：直接交换机根据消息路由密钥将消息传递到队列。
2. 多播类型(fanout): 多播交换机将消息路由到绑定到它的所有队列。
3. 主题类型(Topic): 主题交换在路由密钥和绑定中指定的路由模式之间进行通配符匹配。
4. 头类型(Headers): 头交换机使用消息头属性进行路由。
   
# RabbitMQ核心概念   
这里有一些重要的概念需要在我们深入研究rabbitmq之前进行描述。
+ 生产者(Producer): 发送消息的应用。
+ 消费者(Consumer)：接收消息的应用。
+ 队列(Queue): 存储消息的缓冲区。
+ 消息(Message)：通过RabbitMQ从生产者发送给消费者的信息。
+ 连接(Connection)：连接是应用程序和RabbitMQ代理之间的TCP连接。
+ 通道(Channel)：通道是连接内部的虚拟连接。当您发布或使用队列中的消息时，都是通过通道完成的。
+ 交换机(Exchange)：接收来自生产者的消息，并根据交换类型定义的规则将它们推送到队列中。要接收消息，需要将队列绑定到至少一个交换。
+ 绑定(Binding)：绑定是队列和交换之间的链接。
+ 路由密钥(Routing key)：路由密钥是Exchange用来决定如何将消息路由到队列的密钥。路由密钥类似于邮件的地址。
+ AMQP: AMQP(Advanced Message Queuing Protocol)是RabbitmQ消息之间的协议。
+ 用户(Users): 可以使用给定的用户名和密码连接到RabbitmQ。可以为每个用户分配权限，例如在实例中读取、写入和配置权限。
  
一旦我们熟悉RabbitMQ的核心概念和了解RabbitMQ如何工作，现在让我们用下面的文章来亲身体验rabbitmq：
> [RabbitMQ Java HelloWorld Example](http://www.javaguides.net/2018/12/rabbitmq-java-helloworld-example.html)  - 在这篇文章中，我们将会学到在java的Hello world 示例中如何使用RabbitMQ。

> [RabbitMQ Tutorial with Publish/Subscribe Example](http://www.javaguides.net/2018/12/rabbitmq-tutorial-with-publishsubscribe-example.html) - 在本教程中，我们将查看rabbitmq的概述，然后我们将逐步开发一个发布/订阅示例。

查看完整的RabbitMQ教程，这里[here](http://www.javaguides.net/p/rabbitmq-java-tutorial-with-examples.html)有具体示例。

# 参考
+ https://www.rabbitmq.com

+ https://www.rabbitmq.com/tutorials/tutorial-one-java.html