---
title: RocketMQ事务相关源码
toc: true
date: 2018-11-28 14:49:14
tags:
---
`源码基于4.3.2`
## 1. 客户端发送事务消息的部分
源码位置 `org.apache.rocketmq.example.transaction.TransactionProducer`
```java
    //当RocketMQ发现`Prepared消息`时，会根据这个Listener实现的策略来决断事务
    TransactionListener transactionListener = new TransactionListenerImpl();
    // 构造事务消息的生产者
    TransactionMQProducer producer = new TransactionMQProducer("please_rename_unique_group_name");
    
    ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread thread = new Thread(r);
            thread.setName("client-transaction-msg-check-thread");
            return thread;
        }
    });

    producer.setExecutorService(executorService);
    // 设置事务决断处理类
    producer.setTransactionListener(transactionListener);
    producer.start();

    String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
    for (int i = 0; i < 10; i++) {
        try {
            //构建消息
            Message msg =
                new Message("TopicTest1234", tags[i % tags.length], "KEY" + i,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
            //发送消息
            SendResult sendResult = producer.sendMessageInTransaction(msg, null);
            System.out.printf("%s%n", sendResult);

            Thread.sleep(10);
        } catch (MQClientException | UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }

    for (int i = 0; i < 100000; i++) {
        Thread.sleep(1000);
    }
    producer.shutdown();

```
