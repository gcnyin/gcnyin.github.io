---
title: Flink是如何启动的
date: 2021-01-04 19:31:25
categories:
- flink
---

Flink可以使用命令行提交Job并执行，那么这个命令背后都执行了哪些操作呢？

这里有一个最简单的例子，可以看到调用了`flink`脚本。我们就以此为入口开始探究。

```
./bin/flink run examples/streaming/TopSpeedWindowing.jar
```

<!-- more -->

`bin/flink`是一个bash script，里面最关键的是最后的这一句。

```
exec $JAVA_RUN $JVM_ARGS $FLINK_ENV_JAVA_OPTS "${log_setting[@]}" \
    -classpath "`manglePathList "$CC_CLASSPATH:$INTERNAL_HADOOP_CLASSPATHS"`" \
    org.apache.flink.client.cli.CliFrontend "$@"
```

抛开冗杂的参数，核心命令是执行java命令。向classpath添加若干必要的flink-jar，启动`org.apache.flink.client.cli.CliFrontend`的`main`方法。

上面所举的例子，经过变换后执行的命令大致如下，`run examples/streaming/TopSpeedWindowing.jar`作为参数被完整地传递给了`CliFrontend`。`$@`是bash中一种特殊用法，用于将一个脚本所接受的参数全部传给另一个脚本。

```
java -classpath flink-dist.jar \
    org.apache.flink.client.cli.CliFrontend run examples/streaming/TopSpeedWindowing.jar
```

顺藤摸瓜来`CliFrontend`到内部，`main`方法首先获取了configuration的目录以及对应的配置文件（如果没有特别指定会读取`./conf/flink-conf.yaml`），并用这两个参数创建一个`CliFrontend`实例。接着进入`parseAndRun`方法。

进入parseAndRun会将传递的第一个参数`run`作为具体要执行的动作，在源码中有一个静态变量`ACTION_RUN`值为`run`。将剩下的参数，也就是`examples/streaming/TopSpeedWindowing.jar`传递给run方法运行。

```java
    public int parseAndRun(String[] args) {

        // check for action
        // 略

        // get action
        String action = args[0];

        // remove action from parameters
        final String[] params = Arrays.copyOfRange(args, 1, args.length);

        try {
            // do action
            switch (action) {
                case ACTION_RUN:
                    run(params);
                    return 0;
```

*未完待续*
