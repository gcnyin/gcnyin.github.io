---
layout: post
title: "对vproxy的分析"
date: 2021-01-03 17:26
categories:
- network
---

[vproxy](https://github.com/wkgcass/vproxy)是一款由Java编写的网络应用，零依赖、高性能且功能丰富。本文分析的是`dev`分支。本文在撰写时得到了vproxy作者的帮助，在此一并表示感谢。

> 注意！为了更好的阅读体验，请在电脑上阅读。

<!-- more -->

vproxy一共有6个模块，`base`, `core`, `extended`, `lib`, `test`, `app`。抛开用于测试的test模块，依赖关系如下图。base模块是最底层的，我们从这里开始。

![vproxy-modules](/images/vproxy-modules.png)

> 这里一并附上vproxy作者给出的架构图供大家参考。

![vproxy-architecture](/images/vproxy-architecture.jpg)

## Base

base里有很多package，重点在于vproxybase和vfd里。这里抽象出了FD与EventLoop的概念，是vproxy里的核心。FD是`File Descriptor`的缩写，中文翻译为“文件描述符”，它代表了一个打开的文件，可能是某个具体的磁盘文件或socket连接。

这是FD源码。可以看到它只是对jdk中的channel进行了简单的封装。关于里面提到的loop有必要稍微多说一点。Loop即是EventLoop，是reactor模式中的核心。reactor模型是关于并发的一种建模，核心概念是一个EventLoop不停地获取事件，并分发给对应的handler去处理。一般而言，一条EventLoop对应一条Thread。

```java
package vfd;

import vproxybase.selector.SelectorEventLoop;

import java.io.IOException;
import java.net.SocketOption;
import java.nio.channels.Channel;

public interface FD extends Channel {
    void configureBlocking(boolean b) throws IOException;

    <T> void setOption(SocketOption<T> name, T value) throws IOException;

    /**
     * @return the real fd object
     */
    FD real();

    /**
     * @param loop the loop which this fd is attaching to
     * @return true if the fd allows attaching, false otherwise
     */
    default boolean loopAware(SelectorEventLoop loop) {
        return true;
    }
}
```

下面列出了所有`FD`的subclass。可以看到种类众多的FD，有tcp, udp, kcp, file, posix。抽象出`FD`的好处也显而易见了，那就是屏蔽底层不同实现，复用上层逻辑。这些复用的逻辑具体是什么，后面会涉及。

![fd-subclass](/images/fd-subclass.png)

## Core

`core`里面有许多核心的东西，主要有`socks server`, `dns server`, `tcp load balancer`, `proxy`, `security`, `connection pool`等。

> 除此之外还有一整套网络栈实现，包括2、3、4层，统称为`vswitch`。这套东西只有在很特别的情况下才会使用，所以这里略过，感兴趣的读者可以自行研究，包括在其基础上实现各种拥塞算法。

```
/vproxy/core/src/main/java $ tree -L 2
├── module-info.java
├── vproxy
│   ├── component
│   ├── dns
│   ├── fstack
│   ├── package-info.java
│   ├── pool
│   ├── socks
│   └── util
└── vswitch
    └── 略
```

## Lib

`lib`里大多是一些应用层的东西，比如`HTTP client`, `HTTP server`, `Stream server`(这个其实就是`Network server`，处理裸socket), 还有一些HTTP Route的东西。

## Extended

`extended`里有一个非常关键的模块——`websocks`。这是一个作者自定义的网络应用层协议，其实就是将`socks`协议与`http`协议整合在一起，根据连接前几个字节判断属于哪个。懂行的朋友可能知道这是用来干什么的。

*未完待续*
