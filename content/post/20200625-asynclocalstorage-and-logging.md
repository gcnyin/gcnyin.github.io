---
title: "AsyncLocalStorage与日志追踪"
date: 2020-06-25T19:26:00+08:00
tags:
- nodejs
---

最近在思考node.js如何做服务间与服务内部的日志追踪，一个很简单的实现就是在HTTP request header里添加一个字段x-trace-id来标识唯一性，打印日志时添加x-trace-id的值。但如何保存这个状态呢？

我看到的一些框架和应用会将traceId作为参数传递到应用的各个函数/方法里，调用`logger.info(traceId, "XXX")`来实现打印traceId的功能。这样的写法并不优雅，但node.js又不像其他语言框架，比如Spring MVC，可以用过ThreadLocal来保存这个状态，那如何保存这个状态呢？其实node.js已经有了这样一个API：`AsyncLocalStorage`。

`AsyncLocalStorage`属于async_hooks模块，node.js v14引入，后被反向移植到v12上。下面用一个express应用来说明它的用法。

```typescript
import { AsyncLocalStorage } from "async_hooks"
import express from "express"
import { v4 as uuid4 } from "uuid"

export const app = express()
const t = new AsyncLocalStorage<string>()

app.use((req, _, next) => {
  const traceId = req.header('x-trace-id')
  t.run(traceId ?? uuid4(), next)
})

app.get('/', async function (_, res) {
  console.log('start', t.getStore())
  // 模拟耗时操作
  await new Promise((resolve) => {
    setTimeout(() => resolve(), 100)
  })
  console.log('end', t.getStore())
  res.send('Hello World!')
})

app.listen(3000)
```

这里我们写了一个middleware，将HTTP Request Header里的信息保存到AsyncLocalStorage实例中，需要打印日志时使用`getStore`方法获取。

我们期望的行为是：一个请求进来后，保存并打印它的traceId，之后进行了一些异步操作，操作结束后，我们能通过`AsyncLocalStorage`获取到保存的traceId。下面我们就使用ab命令验证一下。

```bash
ab -n 6 -c 2 http://localhost:3000/
```

从日志中我们看出确实是成功地将状态保存了下来。

```
start 1c9de33e-84a0-4827-ae82-eae9dd8dec60
end 1c9de33e-84a0-4827-ae82-eae9dd8dec60
start 9c1f6eab-5c95-4a5f-b2f4-2f1a026684f5
start 61a32915-5316-4066-bfb0-382c41a2a1aa
end 9c1f6eab-5c95-4a5f-b2f4-2f1a026684f5
end 61a32915-5316-4066-bfb0-382c41a2a1aa
start bb1cb174-0f42-4dc6-9eab-85ccaef9d08c
start 3ef04aba-dc0c-4c02-aebf-850dbf8c2e31
end bb1cb174-0f42-4dc6-9eab-85ccaef9d08c
end 3ef04aba-dc0c-4c02-aebf-850dbf8c2e31
start a39d34be-da8a-44f6-b3b4-3eb6eb3501ef
end a39d34be-da8a-44f6-b3b4-3eb6eb3501ef
```

这样的写法比起将logger作为参数在各个函数间传递的做法，可读性、可维护性更强，对业务代码的侵入也更低。

以上是服务内日志追踪的办法。

那如何做服务与服务之间的追踪呢？其实也很简单，将traceId注入到http-client的header中，或者更进一步，重写require函数，调用http/https模块时，将traceId注入。这里就不详细展开。
