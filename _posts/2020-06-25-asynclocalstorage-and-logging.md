---
layout: post
title:  "AsyncLocalStorage与日志追踪"
date:   2020-06-25 12:00:00 +0800
tags: [nodejs]
category: nodejs
---

最近在思考node.js如何做服务间与服务内部的日志追踪，一个很简单的实现就是在HTTP request header里添加一个字段x-trace-id来标识唯一性，打印日志时添加x-trace-id的值。其他语言框架，比如Spring MVC，可以用过ThreadLocal来保存这个状态，但node.js是单线程异步的，如何保存这个状态呢？经过一番查找，发现了AsyncLocalStorage。

AsyncLocalStorage，用于在异步调用之前保存状态，属于async_hooks模块，node.js v14引入，后被反向移植到老版本上。下面用一个express 应用来说明它的用法。

```typescript
import { AsyncLocalStorage } from "async_hooks"
import express from "express"
import { v4 as uuid4 } from "uuid"

export const app = express()

const traceIdStorage = new AsyncLocalStorage<string>()

app.use((req, _res, next) => {
    const traceId = req.header('x-trace-id')
    traceIdStorage.run(traceId ?? uuid4(), next)
})

app.get('/', async function (_req, res) {
    console.log('start', traceIdStorage.getStore())
    await new Promise((resolve) => { // 模拟耗时操作
        setTimeout(() => {
            resolve()
        }, 100)
    })
    console.log('end', traceIdStorage.getStore())
    res.send('Hello World!')
})

app.listen(3000, () => console.log('start at http://localhost:3000'))
```

可以自己写一个express middleware，将http request header里的信息保存起来，需要打印日志时直接.getStore()就能获取。这就是服务内日志追踪的办法。

那如何做服务与服务之间的追踪呢？其实也很简单，将traceId注入到http-client的header中，或者更进一步，重写require函数，调用http/https模块时，将traceId注入。这样就可以将traceId在不同服务间传递。

社区里已经有一些库封装了相关操作，比如cls-rtracer，但是它把ID生成算法写死了，用的是UUID v1，如果有定制化的需要，还是自己写一个吧。

爽！
