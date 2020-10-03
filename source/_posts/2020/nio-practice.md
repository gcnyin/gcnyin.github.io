---
layout: post
title: "Java NIO实践的一些感悟"
date: 2020-10-03 22:08:00
categories:
  - java
---

最近在[练习 Java NIO](https://github.com/gcnyin/raw-nio)，有点感悟，记录在此，讲到哪算哪。

好多人讲 reactor 上来噼里啪啦上一堆代码很不好，全都掉进细节里了。应该先看看[Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)和 reactor model[最初始的论文](http://www.dre.vanderbilt.edu/~schmidt/PDF/reactor-siemens.pdf)。里面讲到一个关键概念叫`Synchronous Event Demultiplexer`，可以对应到 Java 的`Selector`，这才是 reactor 和 proactor 之间最大的区别，一个同步选择事件，一个异步回调。

有个东西叫 Event Loop，听起来很玄乎，其实也很简单，就是在`while(true)`循环里不停调用`Selector::select`获得事件并处理的过种。上面讲到的 reactor，在 Java 里可以用一条线程、两条或多条线程来实现。一条线程的情况是，serverSocketChannel 和 socketChannel 用同一条。两条的情况是，serverSocketChannel 一条，socketChannel 一条。多条的情况是，serverSocketChannel 一条，socketChannel 多条且所有 socketChannel 均匀地分布在给定的的 Event Loop 上。

有了 Event Loop，就可以开始封装框架了，我的实现也正是基于此。抽象出 SocketHandler，以及将 SocketHandler 连起来的 Pipeline，这样就很像 Netty 了。其实大家写纯 nio，到最后都得封装出一个类似 netty 的框架，没办法，范式已经定了，最佳实践也有了，不论怎么写逃不出 netty 的手掌心。

框架有了，后面就是实现协议了。如果打算实现复杂协议，比如 HTTP2，难度会很大，这种情况还是直接用 netty 吧，不过简单的协议自己还是可以练手玩玩，比如基于 Type-Length-Value 的文件服务器。等这段时间忙完了，挑一两个常用的协议实现下。

最后再吐槽下，写业务的话没事别用 RPC，连个好用的 API 测试工具也没的。

## 参考资料

- http://www.dre.vanderbilt.edu/~schmidt/PDF/reactor-siemens.pdf 这篇是 reactor 模式诞生的论文
- http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf Scalable IO in Java
