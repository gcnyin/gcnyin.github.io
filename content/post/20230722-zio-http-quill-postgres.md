---
title: "使用zio-http, quill与postgres开发web服务"
date: 2023-07-22T00:53:32+08:00
categories:
- 技术
tags:
- scala
- zio
---

> 本文面向有一定`scala`和`zio`基础的读者。

[zio](https://zio.dev/)是用Scala语言开发的一套框架，核心功能是并发管理和资源管理，近年来在`scala`社区中逐渐流行。[zio-http](https://zio.dev/zio-http/)是zio生态中的http库，原本是非官方项目，前段时间获得了`zio`官方支持。[quill](https://zio.dev/zio-quill/)是zio生态中的数据库操作库，支持主流关系型数据库，当初转投`zio`社区还引发了不小的风波。

本文旨在提供一套基于`zio`生态的http服务快速开发方式。

## 访问数据库

引入依赖

```scala
    libraryDependencies ++= Seq(
      "dev.zio" %% "zio" % "2.0.15",
      "io.getquill" %% "quill-jdbc-zio" % "4.6.1",
      "org.postgresql" % "postgresql" % "42.5.4"
    )
```

向`resources`目录里添加`application.conf`文件，填写数据库连接配置。

```
db {
  dataSourceClassName = org.postgresql.ds.PGSimpleDataSource
  dataSource.user = postgres
  dataSource.portNumber = 5432
  dataSource.password = password
  connectionTimeout = 30000
}
```

创建`Main.scala`作为程序入口，引入`ZIODefaultApp`。

```scala
import zio.{Scope, ZIO, ZIOAppArgs, ZIOAppDefault}

object Main extends ZIOAppDefault {
  override def run: ZIO[ZIOAppArgs with Scope, Any, Any] = ???
}
```

使用`quill`读取上面的db配置，创建数据源`DataSource`。这里的`ZLayer`是`zio`提供的依赖管理工具，类型参数有三个，第一个代表了依赖的项，第二个代表了创建依赖过程中可能抛出的错误类型，第三个代表了最终创建出的项。这里第一个参数是`Any`代表不依赖其他任何项就可以创建`DataSource`。

```scala
val dsLayer: ZLayer[Any, Throwable, DataSource] =
  Quill.DataSource.fromPrefix("db")
```

接口创建`quill context`用于编译和运行时使用。通过ZLayer的类型参数可以很容易地看出它依赖了`DataSource`才能成功创建`quill context`。

```scala
val quillLayer: ZLayer[DataSource, Nothing, Quill.Postgres[CompositeNamingStrategy2[SnakeCase, PostgresEscape]]] =
      Quill.Postgres.fromNamingStrategy(CompositeNamingStrategy2(SnakeCase, PostgresEscape))
```

我们通过`>>>`操作符将上一个`ZLayer`的结果传递给下一个`ZLayer`，组合出新的`ZLayer`。可以看到`dsLayer`的第三个类型参数与`quillLayer`的第一类型参数相抵消，我们就得到了一个不需要任何依赖就能创建`quill context`的`ZLayer`。

```scala
val contextLayer: ZLayer[Any, Throwable, Quill.Postgres[CompositeNamingStrategy2[SnakeCase, PostgresEscape]]] =
  dsLayer >>> quillLayer
```

有了`quill context`就可以进行数据库访问。假设有这样一张表。

```sql
create table "user"
(
    user_id  serial primary key,
    username varchar(255) not null unique,
    password varchar(255) not null
);
```

我们可以在程序中创建一个同名`case class`映射到这张表上，并通过`quill context`引入相关数据库操作符。

```scala
final case class User(userId: Int, username: String, password: String)

final case class UserRepository(quill: Quill.Postgres[CompositeNamingStrategy2[SnakeCase, PostgresEscape]]) {

  import quill._

  def listUser: ZIO[Any, SQLException, Seq[User]] = run(query[User])
}

object UserRepository {
  val layer: ZLayer[Quill.Postgres[CompositeNamingStrategy2[SnakeCase, PostgresEscape]], Nothing, UserRepository] =
    ZLayer.fromFunction(UserRepository.apply _)
}
```

这里的`query[User]`会在编译器生成对应的SQL，尝试编译下即可在命令行里看到。

```
[info] /zio-http-quill-demo/src/main/scala/example/repository/UserRepository.scala:14:65: SELECT x."user_id" AS userId, x."username" AS username, x."password" AS password FROM "user" x
[info]   def listUser: ZIO[Any, SQLException, Seq[User]] = run(query[User])
```

到这里，数据库访问的相关工作已经基本就绪。

## http服务

引入依赖

```scala
    libraryDependencies ++= Seq(
      "dev.zio" %% "zio-http" % "3.0.0-RC2",
      "dev.zio" %% "zio-json" % "0.6.0",
    )
```

`zio-http`底层使用`netty`，有较好的性能，编写时也很方便。下面是一个最简单的demo。

```scala
import zio._
import zio.http._

object Main extends ZIOAppDefault {

  val app: App[Any] = 
    Http.collect[Request] {
      case Method.GET -> Root / "text" => Response.text("Hello World!")
    }

  override val run =
    Server.serve(app).provide(Server.default)
}
```

这里`Http.collect[Request]`通过`pattern match`模式匹配设置路由和对应的处理逻辑。由于我们使用了`zio`，会换成`zio-http`提供的`Http.collectZio[Request]`。

```scala
import zio.json._

def httpApp(userRepository: UserRepository): Http[Any, Nothing, Request, Response] = {
  Http.collectZIO[Request] {
    case Method.GET -> Root / "user" / "list" =>
      val response = for {
        worlds <- userRepository
          .listUser
          .mapError(e => ErrorMsg("INTERNAL_ERROR", e.getMessage))
      } yield Response.json(worlds.toJson)
      response
        .catchAll(errorMsg => ZIO.succeed(
          Response.json(errorMsg.toJson)
        ))
  }
}
```

这里展通过访问数据库获取`User`列表，进行序列化后返回`Response`。自定义的`ErrorMsg`代表请求失败时的响应。`toJson`是`zio-json`库提供的功能，将`case class`序化列成`json`。

## 日志

引入依赖

```scala
    libraryDependencies ++= Seq(
      "dev.zio" %% "zio-logging" % "2.1.13",
      "dev.zio" %% "zio-logging-slf4j2" % "2.1.13",
      "org.slf4j" % "slf4j-api" % "2.0.7",
      "ch.qos.logback" % "logback-classic" % "1.4.8"
    )
```

在`Main.scala`中将`ZIO.log`的默认实现替换为`zio-logging`提供的组件。

```scala
import zio.logging.backend.SLF4J

object Main extends ZIOAppDefault {
  override val bootstrap: ZLayer[ZIOAppArgs, Any, Any] =
    Runtime.removeDefaultLoggers >>> SLF4J.slf4j
}
```

日志接口使用`slf4j`，实现使用`logback`。同时`zio`有自己的`log`操作符，我们添加`zio-logging-slf4j2`依赖将背后的实现替换为`slf4j`，这样我们在调用`ZIO.log("xxx")`时就会使用`logback`。

## zio-http middleware

`zio-http middleware`是`zio-http`功能的扩展点，如果我们想要实现打印请求响应、链路追踪、超时和重试等功能，`zio-http middleware`就是一个非常良好的实现方式。

简单写一个在`response`里添加响应时间的`middleware`并通过`@@`操作符绑定到`httpApp`上。

```scala
import zio.http._
import zio.{Trace, ZIO}

class ResponseTimeMiddleware extends RequestHandlerMiddleware.Simple[Any, Nothing] {
  override def apply[R1 <: Any, Err1 >: Nothing](
      handler: Handler[R1, Err1, Request, Response]
  )(implicit trace: Trace): Handler[R1, Err1, Request, Response] =
    Handler.fromFunctionZIO[Request] { request =>
      for {
        startTime <- ZIO.succeed(System.currentTimeMillis())
        response <- handler.runZIO(request)
        endTime <- ZIO.succeed(System.currentTimeMillis())
      } yield response.addHeader(Header.Custom("X-Response-Time", s"${endTime - startTime}"))
    }
}

object Main extends ZIOAppDefault {
  override val run = {
    val responseTimeMiddleware = new ResponseTimeMiddleware()
    Server.serve(app @@ responseTimeMiddleware).provide(Server.default)
  }
}
```

## 实现一个简单的日志链路追踪

日志链路追踪是web后端服务常见的需求之一，我们已经了解了`zio-http middleware`和`zio-logging`的基础用法，现在结合二者的能力实现请求级别的日志链路追踪。

首先要复习下`zio`中的一个概念`ZIOAspect`，正如其名字aspect名字“切面”所言，可以包裹住一个`ZIO`调用并进行某种操作，常见的有日志、重试等。

```scala
import zio.http._
import zio.logging.LogAnnotation
import zio.{Trace, ZIO}

import java.util.UUID

class LoggingMiddleware extends RequestHandlerMiddleware.Simple[Any, Nothing] {
  override def apply[R1 <: Any, Err1 >: Nothing](
      handler: Handler[R1, Err1, Request, Response]
  )(implicit trace: Trace): Handler[R1, Err1, Request, Response] =
    Handler.fromFunctionZIO[Request] { request =>
      val h = for {
        _ <- ZIO.log(request.url.path.toString())
        response <- handler.runZIO(request)
      } yield response
      h @@ LogAnnotation.TraceId(UUID.randomUUID())
    }
}
```

这里调用`LogAnnotation.TraceId()`就是创建了一个`ZIOAspect`，具体作用是向这个`Aspect`中的日志上下文添加了`traceId=XXX`的信息，这样被包裹的`ZIO`操作中的`ZIO.log`就可以拿到相关信息并添加到日志中。

日志系统的底层是`logback`，`zio-logging`会向`%kvp`中添加我们传递的`traceId`上下文，我们将`%kvp`添加进`logback.xml`的`pattern`中。

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>[%level] [%kvp] - %msg %n</pattern>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="STDOUT"/>
  </root>
</configuration>
```

添加后，我们发起请求后就可以看到携带有`traceId`的日志，第一条日志是`middleware`打印的。并且我们在`Http.collectZIO[Request]`中调用`ZIO.log`也会打印`traceId`，因为已经被`ZIOAspect`包裹起来了，可以拿到上下文，第二条日志就是如此。

```
[INFO] [trace_id="2dbf5bd2-a856-4d21-945c-b42a08f3bdc0"] - /user/list
[INFO] [trace_id="2dbf5bd2-a856-4d21-945c-b42a08f3bdc0"] - return 2 users
```

需要注意，我们在`middleware`添加的`ZIOAspect`上下文是请求级别的，请求之间并不共享。

## end

想要获取完整代码请访问 https://github.com/gcnyin/zio-http-quill-demo
