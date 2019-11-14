---
layout: post
title:  "给博客添加tags功能"
date:   2019-11-13 18:00:00 +0800
tags: [Jekyll]
---

[这篇文章](https://codinfox.github.io/dev/2015/03/06/use-tags-and-categories-in-your-jekyll-based-github-pages/)介绍了如何添加tags。

我把它化用了一下。

在项目根目录下添加`tags.md`，将上文代码复制过来，删除一些多余的链接和样式。

不过原文代码有两点需要注意，一是对字符没有转义，如果文章题目里有HTML的特殊字符，不能正确处理；二是日期显示对国人不够友好，用的是`30 May 2017`这样的形式。

第一个问题，Jekyll使用了Liquid模板引擎，Liquid中提供了`escape`filter，添加上就好了。

第二个问题，使用Liquid的`date`filter即可，我这用的是`%Y-%m-%d"`这种格式。