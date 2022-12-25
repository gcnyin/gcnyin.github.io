---
title: "mini-parser小轮子"
date: 2019-11-06T18:00:00+08:00
tags:
- compiler
---

[代码链接](https://github.com/gcnyin/compiler/blob/master/mini-parser/index.ts)

demo如下，定义好rule后进行parse。

```javascript
const rule = AndRule.of(
  [
    TextRule.of("hello", "HELLO_RULE"),
    OrRule.of(
      [TextRule.of(", "), TimesRule.of(3, TextRule.of(" ", "SPACE"), "TIMES")],
      "NO_NAME"
    ),
    TextRule.of("world", "WORLD"),
    OneOrMoreRule.of(TextRule.of("!"), "SAMPLE")
  ],
  "HELLO_WORLD"
);

console.log(JSON.stringify(rule.accept("hello   world!!!")));
```

结果为：

```json
{
  "contain": true,
  "group": {
    "groups": [
      { "text": "hello", "name": "HELLO_RULE" },
      {
        "groups": [
          {
            "groups": [
              { "text": " ", "name": "SPACE" },
              { "text": " ", "name": "SPACE" },
              { "text": " ", "name": "SPACE" }
            ],
            "name": "TIMES"
          }
        ],
        "name": "NO_NAME"
      },
      { "text": "world", "name": "WORLD" },
      {
        "groups": [
          { "text": "!", "name": null },
          { "text": "!", "name": null },
          { "text": "!", "name": null }
        ],
        "name": "SAMPLE"
      }
    ],
    "name": "HELLO_WORLD"
  }
}
  
```

当然，这离真正可用还有很远：

![红红姐的教导](/honghongjiedejiaodao.jpg)
