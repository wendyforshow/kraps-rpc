# kraps-rpc
[![Build Status](https://travis-ci.org/neoremind/kraps-rpc.svg?branch=master)](https://travis-ci.org/neoremind/kraps-rpc)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/net.neoremind/kraps-rpc_2.11/badge.svg)](https://maven-badges.herokuapp.com/maven-central/net.neoremind/kraps-rpc_2.11)
[![codecov](https://codecov.io/gh/neoremind/kraps-rpc/branch/master/graph/badge.svg)](https://codecov.io/gh/neoremind/kraps-rpc)
[![Hex.pm](https://img.shields.io/hexpm/l/plug.svg)](http://www.apache.org/licenses/LICENSE-2.0)


Kraps-rpc is a RPC framework split from [Spark](https://github.com/apache/spark), you can regard it as `spark-rpc` with the word *spark* reversed. 

This module is mainly for studying how RPC works in Spark, as people know that Spark consists many distributed components, such as driver, master, executor, block manager, etc, and they communicate with each other through RPC. In Spark project the functionality is sealed in `Spark-core` module. Kraps-rcp splits the core RCP part from it, not including security and streaming download feature.

The module is based on Spark 2.1 version, which eliminate [Akka](http://akka.io/) due to [SPARK-5293](https://issues.apache.org/jira/browse/SPARK-5293).

## 0 Dependency

You can configure you project by including dependency from below, currently only work with **scala 2.11**.

Maven:

```
<dependency>
    <groupId>net.neoremind</groupId>
    <artifactId>kraps-rpc_2.11</artifactId>
    <version>1.0.0</version>
</dependency>
```

SBT:

```
"net.neoremind" % "kraps-rpc_2.11" % "1.0.0"
```

To learn more dependencies, please go to *Dependency tree* section.

## 1 How to run

The following examples can be found in [kraps-rpc-example](https://github.com/neoremind/kraps-rpc/tree/master/kraps-rpc-example)

### 1.1 Create an endpoint

Creating an endpoint which contains the business logic you would like to provide as a RPC service. Below shows a simple example of a hello world echo service.

```
class HelloEndpoint(override val rpcEnv: RpcEnv) extends ThreadSafeRpcEndpoint {

  override def onStart(): Unit = {
    println("start hello endpoint")
  }

  override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
    case SayHi(msg) => {
      println(s"receive $msg")
      context.reply(s"hi, $msg")
    }
    case SayBye(msg) => {
      println(s"receive $msg")
      context.reply(s"bye, $msg")
    }
  }

  override def onStop(): Unit = {
    println("stop hello endpoint")
  }
}


case class SayHi(msg: String)

case class SayBye(msg: String)

```

`RpcEndpoint` is where to receive and handle requests, as “actor” notation in akka. `RpcEndpoint` does differentiate message “need-not-reply” from “need-reply”. Former one is much like UDP message(send and forget), latter one follows tcp way, waiting for one response.

```
  /**
   * Process messages from [[RpcEndpointRef.send]] or [[RpcCallContext.reply)]]. If receiving a
   * unmatched message, [[SparkException]] will be thrown and sent to `onError`.
   */
  def receive: PartialFunction[Any, Unit] = {
    case _ => throw new SparkException(self + " does not implement 'receive'")
  }

  /**
   * Process messages from [[RpcEndpointRef.ask]]. If receiving a unmatched message,
   * [[SparkException]] will be thrown and sent to `onError`.
   */
  def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
    case _ => context.sendFailure(new SparkException(self + " won't reply anything"))
  }
```

One `RpcCallContext` is provided for endpoint to seprate endpoing logic from message transfer process. Providing function to reply result or send failure information:

- reply(response: Any) : reply one message
- sendFailure(e: Throwable) : send failure

Also a serious status callback is provided(even more abundant status than actor) :

- onError
- onConnected
- onDisconnected
- onNetworkError
- onStart
- onStop
- stop

### 1.2 Run server

There are a couple of steps to create a RPC server which provide `HelloEndpoint` service.

1. Create `RpcEnvServerConfig`, `RpcConf` is where you can specify some parameters for the server, will be discussed the below section, `hello-server` is just a simple name, no real use later. Host and port must be specified. Note that is server can not bind on port, it will try to increase the port value by one and try next.
2. Create `RpcEnv` which launches the server via TCP socket at localhost on port 52345.
3. Create `HelloEndpoint` and setup it with an identifier of `hello-service`, the name is for client to call and route into the correct service.
4. `awaitTermination` will block the thread and make server run without exiting JVM.

```
import net.neoremind.kraps.RpcConf
import net.neoremind.kraps.rpc._
import net.neoremind.kraps.rpc.netty.NettyRpcEnvFactory

object HelloworldServer {

  def main(args: Array[String]): Unit = {
    val config = RpcEnvServerConfig(new RpcConf(), "hello-server", "localhost", 52345)
    val rpcEnv: RpcEnv = NettyRpcEnvFactory.create(config)
    val helloEndpoint: RpcEndpoint = new HelloEndpoint(rpcEnv)
    rpcEnv.setupEndpoint("hello-service", helloEndpoint)
    rpcEnv.awaitTermination()
  }
}
```

### 1.3 Client call

#### 1.3.1 Asynchronous invocation

Creating `RpcEnv` is the same as above, and here use `setupEndpointRef` to create a stub to call remote server at localhost on port 52345 and route to `hello-service`.

`Future` is used here for asynchronous invocation.

```
import net.neoremind.kraps.RpcConf
import net.neoremind.kraps.rpc.{RpcAddress, RpcEndpointRef, RpcEnv, RpcEnvClientConfig}
import net.neoremind.kraps.rpc.netty.NettyRpcEnvFactory
import scala.concurrent.{Await, Future}
import scala.concurrent.duration.Duration
import scala.concurrent.ExecutionContext.Implicits.global

object HelloworldClient {

  def main(args: Array[String]): Unit = {
    import scala.concurrent.ExecutionContext.Implicits.global
    val rpcConf = new RpcConf()
    val config = RpcEnvClientConfig(rpcConf, "hello-client")
    val rpcEnv: RpcEnv = NettyRpcEnvFactory.create(config)
    val endPointRef: RpcEndpointRef = rpcEnv.setupEndpointRef(RpcAddress("localhost", 52345), "hell-service")
    val future: Future[String] = endPointRef.ask[String](SayHi("neo"))
    future.onComplete {
      case scala.util.Success(value) => println(s"Got the result = $value")
      case scala.util.Failure(e) => println(s"Got error: $e")
    }
    Await.result(future, Duration.apply("30s"))
  }
}
```

#### 1.3.2 Synchronous invocation

Creating `RpcEnv` is the same as above, and here use `setupEndpointRef` to create a stub to call remote server at localhost on port 52345 and route to `hello-service`.

Use `askWithRetry` instead of `ask` to call in sync way. 

*Note that in latest Spark version the method signature has changed to `askSync`.*

```
object HelloworldClient {

  def main(args: Array[String]): Unit = {
    import scala.concurrent.ExecutionContext.Implicits.global
    val rpcConf = new RpcConf()
    val rpcConf = new RpcConf()
    val config = RpcEnvClientConfig(rpcConf, "hello-client")
    val rpcEnv: RpcEnv = NettyRpcEnvFactory.create(config)
    val endPointRef: RpcEndpointRef = rpcEnv.setupEndpointRef(RpcAddress("localhost", 52345), "hello-service")
    val result = endPointRef.askWithRetry[String](SayBye("neo"))
    println(result)
  }
}
```

## 2. About RpcConf

`RpcConf` is simply `SparkConf` in Spark, there are a couple of parameters to specify. They are listed below, for most of them you can reference to [Spark Configuration](http://spark.apache.org/docs/2.1.0/configuration.html). For example, you can specify parameter in the following way.

```
val rpcConf = new RpcConf()
rpcConf.set("spark.rpc.lookupTimeout", "2s") 
```

| Configuration                          | Description                              |
| -------------------------------------- | ---------------------------------------- |
| spark.rpc.lookupTimeout                | Timeout to use for RPC remote endpoint lookup, whenever a call is made the client will always ask the server whether specific endpoint exists or not, this is for the asking timeout, default is 120s |
| spark.rpc.askTimeout                   | Timeout to use for RPC ask operations, default is 120s |
| spark.rpc.numRetries                   | Number of times to retry connecting, default is 3 |
| spark.rpc.retry.wait                   | Number of milliseconds to wait on each retry, default is 3s |
| spark.rpc.io.numConnectionsPerPeer     | Number of concurrent connections between two nodes for fetching data. For reusing, used on client side to build client pool, please always set to 1 |
| spark.rpc.netty.dispatcher.numThreads  | Actor Inbox dispatcher thread pool size, default is 8 |
| spark.rpc.connect.threads              | Because TransportClientFactory.createClient is blocking, we need to run it in this thread pool, to implement non-blocking send/ask. Every remote address will have one client to serve, this is the pool used to create client. |
| spark.rpc.serialization.stream.factory | By default kraps-rpc and spark rpc use java native serialization, but its performance is not good, here kraps-rpc provide an alternative to use FST serialization which is 99% compatible with java but provide a better performance in both time consuming and byte size. |

## 3. More examples

Please find more in [test cases](https://github.com/neoremind/kraps-rpc/blob/master/kraps-rpc/src/test/scala/com/neoremind/kraps/RpcTest.scala).

## 4. Dependency tree

```
[INFO] +- org.apache.spark:spark-network-common_2.11:jar:2.1.0:compile
[INFO] |  +- io.netty:netty-all:jar:4.0.42.Final:compile
[INFO] |  +- org.apache.commons:commons-lang3:jar:3.5:compile
[INFO] |  +- org.fusesource.leveldbjni:leveldbjni-all:jar:1.8:compile
[INFO] |  +- com.fasterxml.jackson.core:jackson-databind:jar:2.6.5:compile
[INFO] |  +- com.fasterxml.jackson.core:jackson-annotations:jar:2.6.5:compile
[INFO] |  +- com.google.code.findbugs:jsr305:jar:1.3.9:compile
[INFO] |  +- org.apache.spark:spark-tags_2.11:jar:2.1.0:compile
[INFO] |  \- org.spark-project.spark:unused:jar:1.0.0:compile
[INFO] +- de.ruedigermoeller:fst:jar:2.50:compile
[INFO] |  +- com.fasterxml.jackson.core:jackson-core:jar:2.8.8:compile
[INFO] |  +- org.javassist:javassist:jar:3.21.0-GA:compile
[INFO] |  +- org.objenesis:objenesis:jar:2.5.1:compile
[INFO] |  \- com.cedarsoftware:java-util:jar:1.9.0:compile
[INFO] |     +- commons-logging:commons-logging:jar:1.1.1:compile
[INFO] |     \- com.cedarsoftware:json-io:jar:2.5.1:compile
[INFO] +- org.scala-lang:scala-library:jar:2.11.8:compile
[INFO] +- org.slf4j:slf4j-api:jar:1.7.7:compile
[INFO] +- org.slf4j:slf4j-log4j12:jar:1.7.7:compile
[INFO] |  \- log4j:log4j:jar:1.2.17:compile
[INFO] +- com.google.guava:guava:jar:15.0:compile
```

## 5. Ackownledgement

The development of Kraps-rpc is inspired by Spark. Kraps-rpc with Apache2.0 Open Source License retains all copyright, trademark, author’s information from Spark.



