---
title: "最近搭的一套离线数仓架构"
date: 2022-08-07T21:18:13+08:00
tags:
- data warehouse
---

最近在搭公司的离线数仓，整理出了一批很实用的组件，在这里分享下。

- spark-sql
- seatunnel
- k8s
- clickhouse
- hive metastore
- dolphin-scheduler

使用seatunnel+spark将数据从业务数据库导入clickhouse中，使用dolphin-scheduler调度clickhouse SQL生产dw层和ads层数据，最后再通过seatunnel将ads层数据写回业务系统数据库。hive metastore存储spark sql中的表信息。整个系统搭在k8s上，每个组件基本都有helm charts或者operator，极大地减少了部署难度。
