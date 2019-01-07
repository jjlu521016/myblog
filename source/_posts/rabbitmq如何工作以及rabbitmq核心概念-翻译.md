---
title: rabbitmq如何工作以及rabbitmq核心概念(翻译)
toc: true
date: 2019-01-07 11:02:52
tags:
 - rabbitmq
---
> 原文出处: https://dzone.com/articles/how-rabbitmq-works-and-rabbitmq-core-concepts-1
在本文中，我们将学习什么是[RabbitMQ](http://www.javaguides.net/p/rabbitmq-java-tutorial-with-examples.html)，它是如何工作的，以及[RabbitMQ](http://www.javaguides.net/p/rabbitmq-java-tutorial-with-examples.html)的核心概念。

RabbitMQ是一个开源的消息代理软件。它接受来自生产者的消息并将其传递给消费者。它就像一个中间人，可以用来减少Web应用服务器的负载和投递时间。
## RabbitMQ是如何工作的  
让我们简单的看下RabbitMQ是如何工作的。
让我们首先熟悉rabbitmq的几个重要概念:
+ 生产者(Producer)：发送消息的应用。
+ 消费者(Consumer)：接收消息的应用。
+ 队列(Queue)：存储消息的缓冲区。
+ 消息(Message)：通过RabbitMQ从生产者发送给消费者的信息。
+ 连接(Connection)：连接是应用程序和RabbitMQ代理之间的TCP连接。
+ 通道(Channel)：通道是连接内部的虚拟连接。当您发布或使用队列中的消息时，都是通过通道完成的。
+ 交换(Exchange)：接收来自生产者的消息，并根据交换类型定义的规则将它们推送到队列中。要接收消息，需要将队列绑定到至少一个交换。
+ 绑定：绑定是队列和交换之间的链接。
+ 路由密钥(Routing key)：路由密钥是Exchange用来决定如何将消息路由到队列的密钥。路由密钥类似于邮件的地址。

**Producers**向代理发送/发布消息->**Consumers**从代理接收消息。**RabbitMQ**充当生产者和消费者之间的通信中间件，即使它们在不同的机器上运行。
当生产者向队列中发送消息时，它不会直接发送，而是使用交换发送。下面的设计演示了三个主要组件是如何相互连接的。
交换代理负责将消息路由到不同队列。以便消息可以从生产者接收到交换，然后再次转发到队列。这就是所谓的“发布”方法。  

<image src="/image/rabbitmq/rabbitmq1.png" />

将从队列中提取和使用消息；这称为“使用”。

## 发送消息到多个队列
通过拥有更复杂的应用程序，我们将拥有多个队列。因此消息将在多个队列中发送它。
<image src="/rabbitm-multiple-queues.png"/>
将消息发送到多个队列交换通过绑定和路由键连接到队列。绑定是为将队列连接到交换而设置的“链接”。路由密钥是一个消息属性。在决定如何将消息路由到队列时（取决于交换类型），交换可能会查看此键。