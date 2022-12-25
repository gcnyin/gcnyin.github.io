---
title: "AWS SQS如何实现指数退避"
date: 2020-08-08T16:35:20+08:00
tags:
- aws
---

SQS 全称 Simple queue service，是 AWS 推出一款消息队列服务。按照 AWS 官方文档的说法，SQS 居有高吞吐、高可用的特性。从我个人的开发体验来看，SQS 是一款相当易用并且功能强大的消息队列服务，满足了生产使用的大部分需求。

> Amazon Simple Queue Service (SQS) 是一种完全托管的消息队列服务，可让您分离和扩展微服务、分布式系统和无服务器应用程序。SQS 消除了与管理和运营消息型中间件相关的复杂性和开销，并使开发人员能够专注于重要工作。借助 SQS，您可以在软件组件之间发送、存储和接收任何规模的消息，而不会丢失消息，并且无需其他服务即可保持可用。使用 AWS 控制台、命令行界面或您选择的 SDK 和三个简单的命令，在几分钟内即可开始使用 SQS。

# 什么是指数退避？

不过有一项功能 SQS 并没有原生实现，那就是“指数退避”。指数退避是指在消息处理失败时，消息队列会发送消息，直到消息处理成功或者超时。而针对每条消息，每次重试的间隔时间是指数级上升的，比如第一次重试 3 秒，第二次重试 9 秒，第三次重试 27 秒等等，以此类推。这个功能在系统设计中属于常见功能。

虽然 SQS 没有原生实现，但这我们可以利用 SQS 现有 API 来实现。本文关注使用的，就是 SQS 中 VisibilityTimeout 和 ApproximateReceiveCount 属性以`changeMessageVisibility`API。

# 什么是 VisibilityTimeout

VisibilityTimeout 这个概念源于 SQS 中的一个巧妙设计。

当一个消费者接收到 SQS 中的消息后，SQS 并不会立刻将这条消息从队列中删除，因为 SQS 并不知道消费者是否成功接受并处理了消息。如果想要删除这条消息，消费者必须在处理完消息后，手动调用 SQS 的`deleteMessage`API 去删除，这样 SQS 才会认为这条消费已经被成功消费，可以从队列中删除了。

可问题来了，如果消费者 A 接受到消息后，消息还在队列中，消费者 B 能不能同时接受并处理这条消息呢？如果经常有多个消费者同时处理同一个消息，系统不是就乱套了吗？

基于这种考虑，SQS 对每条消息设置了一个 VisibilityTimeout 的属性，这个属性的值代表了时间长度。如果消费者 A 获得了消息，那么这条消息在 VisibilityTimeout 这么长的时间内是不会被其他消费者读取的，只有超过了这个时间长度，其他消费者才可以读取。也正是基于这个设计，大家在配置 VisibilityTimeout 时长时，要比处理这条消息所花的时间稍长一点。

![](/sqs-visibility-timeout-diagram.png)

有了这个属性，还有操作它的`changeMessageVisibility`API，我们就可以构思出指数退避的大致方案了。每当消息处理失败时，就重新设置消息的 VisibilityTimeout，并且值是上一次的 N 倍（下面全部都设定为`2`）。伪代码如下：

```js
const message = receiveMessage();
try {
  process(message);
} catch (e) {
  const retriesCount = getRetriesCount(message);
  changeMessageVisibility(message, 2 ^ retriesCount);
}
```

# 什么是 ApproximateReceiveCount？

上面伪代码需要从消息中获取重试次数，如何获得呢？我们可以在从 SQS 接受消息时，指定额外的`Attributes`字段告知 SQS 返回相关属性。

请求

```
https://sqs.us-east-2.amazonaws.com/xxxxxxxxxxx/MyQueue/
?Action=ReceiveMessage
&AttributeName=All
```

响应

```xml
<ReceiveMessageResponse>
  <ReceiveMessageResult>
    <Message>
      <MessageId>5fea7756-0ea4-451a-a703-a558b933e274</MessageId>
      <ReceiptHandle>
        MbZj6wDWli+JvwwJaBV+3dcjk2YW2vA3+STFFljTM8tJJg6HRG6PYSasuWXPJB+Cw
        Lj1FjgXUv1uSj1gUPAWV66FU/WeR4mq2OKpEGYWbnLmpRCJVAyeMjeU5ZBdtcQ+QE
        auMZc8ZRv37sIW2iJKq3M9MFx1YvV11A2x/KSbkJ0=
      </ReceiptHandle>
      <MD5OfBody>fafb00f5732ab283681e124bf8747ed1</MD5OfBody>
      <Body>This is a test message</Body>
      <Attribute>
        <Name>SenderId</Name>
        <Value>195004372649</Value>
      </Attribute>
      <Attribute>
        <Name>SentTimestamp</Name>
        <Value>1238099229000</Value>
      </Attribute>
      <Attribute>
        <Name>ApproximateReceiveCount</Name>
        <Value>5</Value>
      </Attribute>
      <Attribute>
        <Name>ApproximateFirstReceiveTimestamp</Name>
        <Value>1250700979248</Value>
      </Attribute>
    </Message>
  </ReceiveMessageResult>
  <ResponseMetadata>
    <RequestId>b6633655-283d-45b4-aee4-4e84e0ae6afa</RequestId>
  </ResponseMetadata>
</ReceiveMessageResponse>
```

返回的响应即含有`ApproximateReceiveCount`，它就是伪代码中的`retriesCount`。

# 小结

本文基于 SQS 已有的 API，构建出一种简单的指数退避方案。除了已提到的内容，还要注意系统的幂等性，避免多次处理消息失败产生额外的影响。

# 参考资料

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ReceiveMessage.html
