---
title: "使用zio-http, quill与postgres开发web服务"
date: 2023-07-22T00:53:32+08:00
categories:
- 技术
tags:
- scala
---

[zio](https://zio.dev/)是用Scala语言开发的一套框架，核心功能是并发管理和资源管理。[zio-http](https://zio.dev/zio-http/)是zio生态中的http库。[quill](https://zio.dev/zio-quill/)是zio生态中的数据库操作库，支持主流关系型数据库。

基于sbt创建项目，并引入相关依赖。

```scala
    libraryDependencies ++= Seq(
      "dev.zio" %% "zio" % "2.0.15",
      "dev.zio" %% "zio-http" % "3.0.0-RC2",
      "io.getquill" %% "quill-jdbc-zio" % "4.6.1",
      "org.postgresql" % "postgresql" % "42.5.4"
    )
```

## 访问数据库

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
val dsLayer: ZLayer[Any, Throwable, DataSource] = Quill.DataSource.fromPrefix("db")
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

我们可以在程序中创建一个`case class`映射到这张表上，并通过`quill context`引入相关数据库操作符。

```scala
final case class User(userId: Int, username: String, password: String)

final case class UserRepository(quill: Quill.Postgres[CompositeNamingStrategy2[SnakeCase, PostgresEscape]]) {

  import quill._

  override def listUser: ZIO[Any, SQLException, Seq[User]] = run(query[User])
}

object UserRepository {
  val layer: ZLayer[Quill.Postgres[CompositeNamingStrategy2[SnakeCase, PostgresEscape]], Nothing, UserRepository] =
    ZLayer.fromFunction(UserRepository.apply _)
}
```

这里的`query[User]`会在编译器生成对应的SQL，尝试编译下即可在命令行里看到。

```
[info] /Users/gcnyin/dev/scala/zio-http-quill-demo/src/main/scala/example/repository/UserRepositoryImpl.scala:14:65: SELECT x."user_id" AS userId, x."username" AS username, x."password" AS password FROM "user" x
[info]   override def listUser: ZIO[Any, SQLException, Seq[User]] = run(query[User])
```

到这里，数据库访问的相关工作已经基本就绪。

## http服务

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

这里通过`Http.collect[Request]`的方式进行`pattern match`模式匹配设置路由和对应的处理逻辑。由于我们使用了`zio`，会换成`zio-http`提供的`Http.collectZio[Request]`。

```scala
import zio.json._

def httpApp(userRepository: UserRepository): Http[Any, Nothing, Request, Response] = {
  Http.collectZIO[Request] {
    case Method.GET -> Root / "user" / "list" =>
      val response = for {
        worlds <- userRepository.listUser.mapError(e => ErrorMsg("INTERNAL_ERROR", e.getMessage))
      } yield Response.json(worlds.toJson)
      response.catchAll(errorMsg => ZIO.succeed(Response.json(errorMsg.toJson)))
  }
}
```

这里展通过访问数据库获取`User`列表，进行序列化后返回`Response`。自定义的`ErrorMsg`代表请求失败时的响应。`toJson`是`zio-json`库提供的功能，将`case class`序化列成`json`。

完整代码请访问 https://github.com/gcnyin/zio-http-quill-demo 。
