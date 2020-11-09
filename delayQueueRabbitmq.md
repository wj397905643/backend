# rabbitmq delay queue

[TOC]

## 问题分析
对于延迟队列不管是 AMQP 协议或者 RabbitMQ 本身是不支持的，之前有介绍过如何使用 RabbitMQ 死信队列（DLX） + TTL 的方式来模拟实现延迟队列，这也是通常的一种做法
1.将Message指定TTL后放入队列中.
2.等超时后, Message放入死信队列.
3.死信队列将Message转发到目标队列.

左侧队列 queue1 分别两条消息 msg1、msg2 过期时间都为 1s，输出顺序为 msg1、msg2 是没问题的
右侧队列 queue2 分别两条消息 msg1、msg2 注意问题来了，msg2 的消息过期时间为 1S 而 msg1 的消息过期为 2S，你可能想谁先过期就谁先消费呗，显然不是这样的，因为这是在同一个队列，必须前一个消费，第二个才能消费，所以就出现了时序问题。


```java
long delayTime = 5000;
byte[] messageBodyBytes = "delayed payload".getBytes("UTF-8");
Map<String, Object> headers = new HashMap<String, Object>();
headers.put("x-delay", delayTime);
AMQP.BasicProperties.Builder props = new AMQP.BasicProperties.Builder().headers(headers);
channel.basicPublish("my-exchange", "", props.build(), messageBodyBytes);
```

延迟队列生产者使用 x-delay 设置延迟时长，单位 ms。经测试，此 delayTime 设置过长（未超过 long 限制）时，延迟队列失效，会立刻被消费掉。
此处应该是该插件有问题，多次尝试后，发现一个有趣的现象，设置 delayTime = 2147483648l 2 时出现此问题，delayTime = 2147483648l 2 - 1 时正常，说明该插件使用了 4 个字节用来表示延时时常，delayTime设置过大溢出后，变为负数，导致该消息立即被执行。因而该插件最多只能支持延迟 49 天左右。
rabbitmq-delayed-message-exchange文档Performance Impact一节已明确说明了这个限制。

