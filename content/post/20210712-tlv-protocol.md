---
title: 浅谈TLV协议的实现
date: 2021-07-12T22:10:50+08:00
tags:
- network
---

今天稍微聊点应用层网络协议设计。

# 简介

众所周知，tcp是一种面向字节流的协议，可以看作一条无尽的水流。如果对水流的内容不加区分，便完全不知道各字节所代表的含义，进而无法处理。如此看来，设计一个能清晰划分字节流边界的应用层协议就显得非常必要。这便涉及到今天所讲的TLV协议。

TLV全称[Type–length–value](https://en.wikipedia.org/wiki/Type%E2%80%93length%E2%80%93value)，是大多数应用层协议的设计思路。主要内容包括

1. 最前面的若干位字节表明传统的是否为该协议（本文暂定为4个字节）
2. 紧跟着若干字节（本文暂定为4个字节，也就是Java中的int），代表后续value的字节长度
3. 最后是代表value的字节，其长度为2中所获得的数量

整个系统的状态机如图所示。

![tlv-state-machine](/tlv-state-machine.png)

# 实现

## 不同状态的Handler

用nio具体编写时，可以先抽象出一个`Handler`接口，可以接受一个`byte`进行处理。再添加三个实现（私有方法），代表0、1、2这三种状态对应的`Handler`。

```java
interface Handler {
  int feed(byte b);
}

private int readType(byte b) {
    // TODO
}

private int readLength(byte b) {
    // TODO
}

private int readValue(byte b) {
    // TODO
}
```

## 状态的定义

接着定义三种状态对应的code和一个`Handler`数组，这样后面可以用当前状态去取对应的`Handler`（`handlers[state]`）。`state`表示当前状态，初始值当然是`TYPE_STATE`。

```java
static final int TYPE_STATE = 0;
static final int LENGTH_STATE = 1;
static final int VALUE_STATE = 2;

final Handler[] handlers = new Handler[]{
    this::readType,
    this::readLength,
    this::readValue
};

int state = TYPE_STATE;
```

## 对外接口

对外接口是`read()`，供外层在读事件就绪时调用。这里会不停地一个字节一个字节的读取ByteBuffer，并调用当前状态的`Handler`去处理。如果返回`-1`代表处理失败，需要关闭连接。

```java
public void read() throws IOException {
  while (true) {
    int i = channel.read(buffer);
    if (i == -1) {
      close();
      return;
    }
    if (i == 0) return;

    buffer.flip();
    while (buffer.hasRemaining()) {
      byte b = buffer.get();
      int f = handlers[state].feed(b);
      if (f == -1) {
        close();
        return;
      }
    }
    buffer.clear();
  }
}
```

## 接受到的数据

接着定义若干`byte[]`，代表每个状态接收的字节，同时还有若干`Pointer`代表当前填充到第几个字节。注意，代表`value`的`data`并没有初始化，因为它的长度是根据`length`确定的。

```java
final byte[] type = new byte[4];
byte typePointer = 0;

final byte[] length = new byte[4];
byte lengthPointer = 0;

int dataLength = 0;
byte[] data;
```

## readType Handler

接下来就可以实现之前提到的三个`Handler`了。先看`readType`，每次一次`byte`进来，它就会填充到`type`(byte[])的下一位，并判断是否满足长度，如果不满足则等待下一个byte。如果长度够4位了，但内容和约定的不一样，会返回`-1`，代表内容有误，供上层处理。如果长度和内容正确，那么状态就转移到`LENGTH_STATE`上，由下一个`readLength Handler`处理。

```java
private int readType(byte b) {
  type[typePointer] = b;
  if (typePointer != 3) {
    typePointer++;
    return 0;
  }
  typePointer = 0;
  if (!Arrays.equals(type, new byte[]{0x01, 0x23, 0x45, 0x67})) {
    return -1;
  }
  state = LENGTH_STATE;
  return 0;
}
```

## readLength Handler

`readLength Handler`和上面也是类似的，额外多了一步，将读取的value长度保存到`dataLength`字段中，并将状态转移到`VALUE_STATE`上。

```java
private int readLength(byte b) {
  length[lengthPointer] = b;
  if (lengthPointer != 3) {
    lengthPointer++;
    return 0;
  }
  lengthPointer = 0;
  dataLength = ByteBuffer.wrap(length).getInt();
  if (dataLength == 0) {
    return -1;
  }
  data = new byte[dataLength];
  state = VALUE_STATE;
  return 0;
}
```

## readValue Handler

`readValue Handler`将字节存储起来，直到数量和`readLength`中获得的`dataLength`值一样，之后就可以进行正常的业务逻辑处理，我这里是简单地print所有数据。

```java
private int readValue(byte b) {
  data[dataPointer] = b;
  if (dataPointer != dataLength - 1) {
    dataPointer++;
    return 0;
  }
  dataPointer = 0;
  process();
  state = TYPE_STATE;
  return 0;
}

private void process() {
  if (data == null) return;
  String s = new String(data, StandardCharsets.UTF_8);
  System.out.println(s);
}
```

# 总结

tlv协议的实现大致就是这样。实现一个状态机，根据输入，判断是否转移到下一步状态。本文的实现比较简单，`value`是作为请求的`body`存在的，如果需要，还可以加上`Header`或timeout处理之类额外的功能。

原文代码在[这里](https://github.com/gcnyin/raw-nio/blob/master/src/main/java/com/github/gcnyin/rawnio/tlv)，里面除了TLV还有其他java nio的实践，欢迎交流。

参考资料

- https://en.wikipedia.org/wiki/Type%E2%80%93length%E2%80%93value
