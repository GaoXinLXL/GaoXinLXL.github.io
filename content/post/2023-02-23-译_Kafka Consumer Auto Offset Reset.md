---
title: 译_Kafka Consumer Auto Offset Reset

date: '2023-02-23'
categories:
    - 笔记
tags:
    - Kafka
---
> 原文：[https://www.lydtechconsulting.com/blog-kafka-auto-offset-reset.html](https://www.lydtechconsulting.com/blog-kafka-auto-offset-reset.html)
>

# 引言

当没有初始化的位移（offset）时，这个`auto.offset.reset` 配置定义了一个消费者（consumer）在某个主题（topic）的分区（partition）消费时，该如何去消费。当一个新的消费者组被定义并且首次监听某个主题时，我们通常会关注到这个配置。这个配置将会告诉组内的消费者们是从分区的开头还是结尾读取。

# 消费消息

每个 Kafka 消费者都属于一个消费者组（consumer group），由消费者的 `group.id` 配置分组在一起。一个消费者组将包含一个或多个消费者。消费者组中的消费者将被分配到主题分区以消费他们的消息。每个分区将只有一个消费者分配给它，尽管一个消费者可以分配给任意一个主题中的多个分区，并且类似地分配给它订阅的所有主题中的分区。

当一个新的消费者组首次被创建以及组内消费者被分配到主题分区时，它们必须被指定从哪里开始轮询（polling）消息。除非已经被告知从某个具体的位移开始轮询（一般不常见），否则有两种主要方式去指定。第一种方式，从分区最开始处开始读取消息，处理分区日志中存在的每一条信息。第二种方式，当消费者开始监听时，仅仅读取最新写入的信息。

# 配置

当消费者组没有初始化位移时，到底是从分区开头读取信息还是仅仅处理最新消息，这个选择由消费端的`auto.offset.reset` 配置决定。下表展示了该配置的取值情况。

| Value    | Usage                    |
| -------- | ------------------------ |
| earliest | 重置位移到最开始处，从分区最开始处消费。     |
| latest   | 重置位移到最新位置，从分区尾部开始消费（默认）。 |
| none     | 如果消费者组中无位移，则抛出异常         |

一旦消费者组写入了位移，则该配置不再生效。如果消费者组中的消费者被停止然后重新启动，他们将从最后一个偏移量开始消费。

# Earliest的作用

![image.png](https://s2.loli.net/2023/03/09/OUjVwpKyn2lQmhF.png)

将新消费者配置为`auto.offset.reset: earliest` 将导致它消费分区上的所有内容。在以下示例中，分区有两条消息“foo”和“bar”，这些消息将被使用：

当然，分区上可能包含数百万条信息，所有要确保明白数据量以及系统不被压垮。这些消息可以追溯到数周或数月之前，也可以追溯到系统的开始，具体取决于主题的保留期限。`retention.ms` 设置为 -1 意味着不会丢弃任何旧消息，因此将轮询所有消息。

# Latest的作用

将新消费者配置为 `auto.offset.reset: latest` 将导致它仅消费写入分区的新消息。在上述场景中，仅会消费偏移量 (3) 的新消息。已经存在的消息“foo”和“bar”将会被跳过。

![image.png](https://s2.loli.net/2023/03/09/xdaUGMX7ZzrkH29.png)

消费者是否应该配置为跳过现有消息应然取决于需求。

# 数据丢失风险

存在一种极端场景会导致数据的丢失，导致在可重试的异常情况下不会重新传递消息。此场景适用于尚未记录任何当前偏移量（或偏移量已被删除）的新消费者组。

- 两个消费者实例 A 和 B 加入一个新的消费者组。
- 消费者实例配置为`auto.offset.reset:latest`（即仅限新消息）。
- 消费者 A 从主题分区中消费一条新消息。
- 消费者 A 在消息处理完成之前死亡。标记消息被消费的消费者位移没有被更新。
- 消费者组再平衡（rebalances），消费者B被分配到主题分区。
- 由于没有有效位移，以及`auto.offset.reset` 设置的是`latest` ，这个消息是不会被消费的。

由于A已经读到了消息，因此期望在失败的情况下，这个消息能够重新分发给下一个该分区上的消费者。然而在该场景下，数据却实际上丢失了。

# 结论

消费者首次监听分区时，能够被配置为消费所以信息还是只消费新的信息。采取何种设置应取决于应用需求。如果要消费所以信息，则要考虑数据量以及处理这些数据要耗费的资源的影响。

**参考资料**

- [https://www.lydtechconsulting.com/blog-kafka-auto-offset-reset.html](https://www.lydtechconsulting.com/blog-kafka-auto-offset-reset.html)
- [https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#auto.offset.reset](https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#auto.offset.reset)
