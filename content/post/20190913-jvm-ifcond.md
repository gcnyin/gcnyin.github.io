---
title: "JVM if<cond>指令笔记"
date: 2019-09-13T12:35:00+08:00
tags:
- java
---

```
format: if<cond> branchByte1 branchByte2
```

从栈中弹出一个值，和0进行比较，根据指令的不同，有不同的比较方法得出一个值，如果为假，则顺序执行后面的指令。那为真的呢？：

branchByte1 branchByte2 都是 0x00 - 0xFF 之间的一个 unsigned 值，可以用他俩算出来offset作为true branch的入口，公式为：

```
(branchbyte1 << 8) | branchbyte2
```

例子：

```
116: bipush        1
118: ifeq          132
121: getstatic     #12      // Field java/lang/System.out:Ljava/io/PrintStream;
124: bipush        1
126: invokevirtual #28      // Method java/io/PrintStream.println:(Z)V
129: goto          140
132: getstatic     #12      // Field java/lang/System.out:Ljava/io/PrintStream;
135: bipush        0
```

行中ifeq 132的二进制表示为99 00 0E。99代表该指令，00和0E代表两个操作数。

计算偏移量：`(0x00 << 8) | 0x0E` 得 14

就是说ifeq开头为118，加上14个偏移量为132，所以132是 true branch 的入口。

问题：

一、为什么需要两个操作数呢？

（一）通过观察 java 编译后的 class 文件，我发现 branchByte1 都是 0x00，完全没有必要为这样一个固定的值设计一个位置呀。

自答：如果 true branch 非常长（true branch 里的指令长度超过了255，那么 branchByte1 是会增长的。

（二）`(branchbyte1 << 8) | branchbyte2`的结果还是8位，难道不可以用一个 byte 来表示，为什么需要用两个 byte 计算后得到结果呢？

自答：如上所述，需要两个来处理 condition 很长的情况。
