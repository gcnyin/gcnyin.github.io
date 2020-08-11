---
layout: post
title: "Logging tracing with AsyncLocalStorage"
date: 2020-06-25 12:01:00
categories:
  - nodejs
---

Recently, I am thinking about how node.js does log tracking between services and services. A very simple implementation is to add a field `traceId` in the HTTP request header to identify uniqueness, and add `traceId` when printing logs. But how to save this state?

<!-- more -->

Some frameworks and applications I have seen pass `traceId` as a parameter to various functions/methods of the application, and call `logger.info(traceId, "XXX")` to realize the function of printing traceId. This way of writing is not elegant, but unlike other language frameworks, such as Spring MVC, node.js can use `ThreadLocal` to save this state. How to save this state? In fact, node.js already has such an API: `AsyncLocalStorage`.

`AsyncLocalStorage` belongs to the async_hooks module, which was introduced in node.js v14 and then backported to v12. Let's use an express application to illustrate its usage.

```typescript
import { AsyncLocalStorage } from "async_hooks";
import express from "express";
import { v4 as uuid4 } from "uuid";

export const app = express();
const t = new AsyncLocalStorage<string>();

app.use((req, _, next) => {
  const traceId = req.header("x-trace-id");
  t.run(traceId ?? uuid4(), next);
});

app.get("/", async function (_, res) {
  console.log("start", t.getStore());
  // Simulate time-consuming operations
  await new Promise((resolve) => {
    setTimeout(() => resolve(), 100);
  });
  console.log("end", t.getStore());
  res.send("Hello World!");
});

app.listen(3000);
```

Here we write a middleware to save the information in the HTTP Request Header to the AsyncLocalStorage instance, and use the `getStore` method to get it when you need to print the log.

The desired behavior is: after a request comes in, save and print its traceId, and then perform some asynchronous operations. After the operation is over, we can get the saved traceId through `AsyncLocalStorage`. Let's use the `ab` command to verify.

```bash
ab -n 6 -c 2 http://localhost:3000/
```

From the log, we can see that the state was indeed successfully saved.

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

Compared with the method of passing `logger` as a parameter between various functions, this method is more readable and maintainable, and has lower intrusion to business code.
