---
title: 使用akka踩的一些坑
date: 2021-08-12T11:06:25+08:00
tags:
- scala
- akka
---

最近在学习 akka，踩了很多坑，这里分享给大家。

## 使用 akka-stream 限制并发度

原代码如下。

```scala
  def fetchRlCnt(pageNumbers: Seq[Int]): Future[Int] = {
    val futures: Seq[Future[HttpResponse]] = pageNumbers
      .map(page => Http()
        .singleRequest(HttpRequest(uri = s"https://examples.org/mix_list/$page")))
    Future.sequence(futures)
      .map(_.map(Unmarshal(_).to[MixList]))
      .flatMap(Future.sequence(_))
      .map(_.map(_.data.rl.length).sum)
  }
```

本意是请求所有的分页内容，以为使用 singleRequest 同时请求所有的分页即可，没想到却出错了。

```
(WaitingForResponseEntitySubscription)]Response entity was not subscribed after 1 second.
Make sure to read the response `entity` body or call `entity.discardBytes()` on it
```

为什么会这样呢？

其实 akka-http 在 singleRequest 时，针对同一个 hostname 会创建一个连接池，如果有相同域名的请求可以提升请求速度。

但当函数参数`pages`足够大，超过了连接池最大并发请求数时，新进入连接池的请求就得不到处理，也就会出现超时的情况。

如何解决呢？

可以将请求放到 akka-stream 中，限制同时处理的请求个数。`mapAsyncUnordered`的第一个参数就是最大并发个数。

```scala
  def fetchRlCnt(pageNumbers: Seq[Int]): Future[Int] =
    Source(pageNumbers)
      .map(it => HttpRequest(uri = s"https://examples.org/mix_list/$it"))
      .mapAsyncUnordered(5)(Http().singleRequest(_).flatMap(Unmarshal(_).to[MixList]))
      .map(_.data.rl.length)
      .runFold(0)(_ + _)
```

这样一来，在 akka-stream 层面做了最大并发个数的限制，HTTP 连接池也就不会超时了。

## 使用 akka-http-json 代替 spray

akka-http 自带的 json 库是[spray](https://github.com/spray/spray-json)，需要手动创建一个`implicit JsonFormat`，不是很好用。

```scala
import akka.http.scaladsl.server.Directives
import akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport
import spray.json._

// domain model
final case class Item(name: String, id: Long)

// collect your json format instances into a support trait:
trait JsonSupport extends SprayJsonSupport with DefaultJsonProtocol {
  implicit val itemFormat = jsonFormat2(Item)
}

// use it wherever json (un)marshalling is needed
class MyJsonService extends Directives with JsonSupport {

  val route =
      get {
        pathSingleSlash {
          complete(Item("thing", 42)) // will render as JSON
        }
      }
}
```

[akka-http-json](https://github.com/hseeberger/akka-http-json)将许多 JSON 库与 akka-http 进行了集成，非常方便，我们这里选择比较易用的[circe](https://circe.github.io/circe/)。

```
"de.heikoseeberger" %% "akka-http-circe" % "1.37.0"
```

使用时引入必要的包即可。

```scala
import de.heikoseeberger.akkahttpcirce.FailFastCirceSupport._
import io.circe.generic.auto._

class StreamerRoutes(streamerRepository: ActorRef[StreamerActor.Command])
  (implicit val system: ActorSystem[_]) {
  implicit val timeout: Timeout =
    Timeout.create(system.settings.config.getDuration("myapp.routes.ask-timeout"))

  def getStreamerCount: Future[StreamerActor.StreamerCount] =
    streamerRepository.ask(StreamerActor.QueryStreamerCount)

  val routes: Route =
    path("streamerCount") {
      get {
        onSuccess(getStreamerCount)(complete(_))
      }
    }
}

```

可以看到，circe 不需要手动创建 Format 对象，能够自动处理序列化。

但到这里还不算完。这里的 StreamerCount 其实是 trait，有两个实现类。

```scala
sealed trait StreamerCount
final case class StreamerCountResult(datetime: Instant, count: Int) extends StreamerCount
final case class StreamerCountError(error: String) extends StreamerCount
```

circe 在序列化时，会默认将实现类的名称作为 key 放入 json 中。

```json
{
  "StreamerCountResult": {
    "datetime": "2021-08-13T10:52:50.301390Z",
    "count": 2843
  }
}
```

这虽然保留了类型信息方便反序列化，但与外部系统进行交互时，会很显得很多余。

想要去除这个 key 的包装，我们可以引入`circe-generic-extras`包。

```
"io.circe" %% "circe-generic-extras" % "0.14.1"
```

引入`io.circe.generic.extras.Configuration`并进行配置，再使用`import io.circe.generic.extras.auto._`替换`import io.circe.generic.auto._`即可。代码如下。

```scala
package app

import akka.actor.typed.scaladsl.AskPattern._
import akka.actor.typed.{ActorRef, ActorSystem}
import akka.http.scaladsl.model.{ContentTypes, HttpEntity}
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.server.Route
import akka.util.Timeout
import de.heikoseeberger.akkahttpcirce.FailFastCirceSupport._
import io.circe.generic.extras.auto._
import io.circe.generic.extras.Configuration

import scala.concurrent.Future

class StreamerRoutes(streamerRepository: ActorRef[StreamerActor.Command])
  (implicit val system: ActorSystem[_]) {
  implicit val timeout: Timeout =
    Timeout.create(system.settings.config.getDuration("myapp.routes.ask-timeout"))

  implicit val genDevConfig: Configuration =
    Configuration.default.withDiscriminator("_type")

  def getStreamerCount: Future[StreamerActor.StreamerCount] =
    streamerRepository.ask(StreamerActor.QueryStreamerCount)

  val routes: Route =
    path("streamerCount") {
      get {
        onSuccess(getStreamerCount)(complete(_))
      }
    }
}
```

返回的 Response 变成了我们期望的样子，类型信息保留在了`_type`字段上，未来反序列化时也不会问题。

```json
{
  "datetime": "2021-08-13T10:57:35.209627Z",
  "count": 2883,
  "_type": "StreamerCountResult"
}
```
