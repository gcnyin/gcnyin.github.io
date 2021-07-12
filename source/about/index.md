---
layout: about
title: About
date: 2018-09-02 12:35:00
---

网名菠萝头，又名箱子。

# 教育经历

西北大学，本科，2014年至2018年

# 专业技能

编程语言：Java/ Scala/ TypeScript/ JavaScript/ Python。

web开发：Spring/ Playframework/ Netty/ Express.js。

数据库：MySQL/ Doris。MySQL应用于OLTP场景，Doris应用于OLAP场景。

中间件：Kafka/ Redis。

软件工程：测试驱动开发、领域驱动开发。日常开发中习惯使用测试驱动开发，并在组内推广。

DevOps：docker/ k8s/ jenkins/ CICD。有完整的CICD系统搭建经验。

云平台：AWS，有一些使用经验，包括SQS, Lambda, SNS等。

# 工作经历

美团。2020年至今。职位为数据系统开发，主要负责实时数据仓库的开发。

ThoughtWorks。2018年至2020年。职位为软件开发工程师，主要承担后端开发。

# 开源项目及作品

1. https://github.com/gcnyin/raw-nio 一个tcp负载均衡器，内置http server。基于reactor模型、使用原始Java NIO编写的EventLoop和TCP Load Balancer。现已支持random, round robin, min connection count三种负载均衡策略。内置的http server可以用类似vertx风格的api编写服务器逻辑。
2. https://github.com/gcnyin/slisp 一个编译到JVM的简单的lisp方言。通过编写antlr4 g8文件，生成java代码，进行词法分析与语法分析。通过分析抽象语法树，抽象出“变量定义”，“打印变量”，“加减乘除”，“if语句”，“条件判断”等语言特性。基于栈式虚拟机的设计，将所有操作压入栈中。最终通过asm库生成字节码，运行在jvm上。
