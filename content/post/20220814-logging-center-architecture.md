---
title: "一套充分利用本地机器的日志中心架构"
date: 2022-08-14T23:09:08+08:00
tags:
- logging
---

![logging-architecture](/logging-architecture.png)

1. 服务搭在k8s上，起一个DaemonSet使用Fluent-Bit收集所有日志，发到Kafka（这个kafka可以是云服务商提供的，或者是自建的，但无论如何要能被本地机器访问）。
2. 本地机器也搞一个k8s集群，先搭ElasticSearch，再由Logstash收集Kafka中的日志（有插件）插入到es里，最后由Kibana读取。

具体命令懒得写了，大家自己研究吧。
