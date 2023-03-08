---
title: 译_Kafka - difference between Log end offset(LEO) vs High Watermark(HW)

date: '2023-03-01'
categories:
    - 笔记
tags:
    - Kafka
---



> 原文链接：[https://stackoverflow.com/questions/39203215/kafka-difference-between-log-end-offsetleo-vs-high-watermarkhw](https://stackoverflow.com/questions/39203215/kafka-difference-between-log-end-offsetleo-vs-high-watermarkhw)
> 这里简单翻译一下其中一个回答。
>

让我们从Google上能够找到的一个最流行的`watermark`定义开始吧。

“HW就是被成功复制到所有日志副本的那个最后一个消息的位移。”

对于上述定义我是不太信服的，经过更深层次的探索，我发现了如下图片：

![https://i.imgtg.com/2023/03/01/VZUjP.png](https://img-blog.csdnimg.cn/img_convert/6eaed92f7c25ac46ca42453a00b72f43.png)

这出现了什么问题？图片最右边的`stuck follower` 并没有收到第4条消息。也许在google找到的第一个定义并不完备，作者想表达的实际意图是：“HW是被成功复制到所有ISR的最后一条消息的位移”。

在本能指引下，我发现了这篇[文章](https://rongxinblog.wordpress.com/2016/07/29/kafka-high-watermark/)，提供了`watermark` 在代码中如何被计算的细节。

我发现文章中的`watermark` 被定义的更加精确：

“HW被计算为分区ISR中的最小LEO，以及它是单调递增的。”

这个回答及附带的代码印证了我的直觉。

总的来说，我认为`watermark` 的细节定义展现了LEO和WH的区别。最后提交的位移和LEO在ISR-Fellow中可能重合，但就像第一个链接的图片所示，对Leader来说可能并非如此。
