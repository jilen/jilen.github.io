---

layout: post
categories: [scala]
tags: [scala, akka, play]

---

####  Goals  ####
******************

*  Create an akka cluster
*  Tell the member up/down event

<!--more-->

#### Project setup  #####
***************************

1.  Use `play new app` to create the app
2.  Add `akka-cluster` as sbt dependencies


> build.sbt
{% highlight scala %}
name := "cluster-demo"

version := "1.0-SNAPSHOT"

libraryDependencies ++= Seq(
  "com.typesafe.akka" %% "akka-cluster" % "2.2.3"
)

play.Project.playScalaSettings
{% endhighlight %}



####  Cluster setup  ####
***************************

**Configuration system**

Akka use [typesafe config](https://github.com/typesafehub/config) library to manage its config
We can create an config file
And Then load it use `com.typesafe.config.ConfigFactory.load`

`node.conf`(should be placed in the classpath)
{% highlight scala %}
akka {
  actor {
    provider = "akka.cluster.ClusterActorRefProvider"
  }
  remote {
    log-remote-lifecycle-events = off
    netty.tcp {
      hostname = "127.0.0.1"
      port = 0
    }
  }

  cluster {
    seed-nodes = [
      "akka.tcp://demo@127.0.0.1:2551",
      "akka.tcp://demo@127.0.0.1:2552"]

    auto-down-unreachable-after = 10s
  }
}
{% endhighlight %}

*The system name must match while creating*

**Create actor system**

`ActorSystem.apply` method will create an actor system
{% highlight scala %}
  import akka.cluster._
  import akka.actor._
  val cfg = ConfigFactory.load("node1.conf")
  val actorSystem = ActorSystem("demo", cfg)
  val cluster = Cluster(actorSystem)
{% endhighlight %}

Now the `ActorSystem` and `Cluster` are created

**Subscribe events**

We can subscribe cluster event after the cluster state changed

First, create a listener actor
{% highlight scala %}
import akka.cluster.ClusterEvent._
import play.api.Logger

class ClusterListener extends Actor {
  def receive = {
    case state: CurrentClusterState =>
    case MemberUp(member) =>
      Logger.info(s"Member is Up: ${member}")
    case UnreachableMember(member) =>
      Logger.warn(s"Member detected as unreachable: ${member}")
    case MemberRemoved(member, previousStatus) =>
      Logger.warn(s"Member is Removed: ${member.address} after ${previousStatus}")
    case _: ClusterDomainEvent => // ignore
  }
}
{% endhighlight %}

Then, suscribe the cluster event
{% highlight scala %}
def start() = {
    val listener = actorSystem.actorOf(Props[ClusterListener], "cluster-listener")
    cluster.subscribe(listener, classOf[ClusterDomainEvent])
    Logger.info("init cluster")
  }
{% endhighlight %}

*The start method should be called in Global.scala as a startup hook, because the `cluster` defined in `object` will be lazy initialized*

##### Setup use play's internal actory system #####
***************************************************
Play has an actor system called `application` by default.
So if we want to create cluster with that actor system, we should

1.  Move the conf to `application.conf`

2.  Then create the `ActorSystem` with play api

{% highlight scala %}
    import play.api.Play.current
    import play.api.libs.concurrent.Akka

    val actorSystem = Akka.system
{% endhighlight %}

There is nothing new indeed

##### Full example #####

**actors.ClusterDemo.scala**
{% highlight scala %}
package actors

import akka.actor._
import akka.cluster._
import akka.cluster.ClusterEvent._
import com.typesafe.config.ConfigFactory
import play.api.Logger

object ClusterDemo {
  val cfg = ConfigFactory.load("node.conf")
  val actorSystem = ActorSystem("demo", cfg)
  val cluster = Cluster(actorSystem)

  def start() = {
    val listener = actorSystem.actorOf(Props[ClusterListener], "cluster-listener")
    cluster.subscribe(listener, classOf[ClusterDomainEvent])
    Logger.info("init cluster")
  }

  def stop() = {
    actorSystem.shutdown()
  }
}

class ClusterListener extends Actor {
  def receive = {
    case state: CurrentClusterState =>
    case MemberUp(member) =>
      Logger.info(s"Member is Up: ${member}")
    case UnreachableMember(member) =>
      Logger.warn(s"Member detected as unreachable: ${member}")
    case MemberRemoved(member, previousStatus) =>
      Logger.warn(s"Member is Removed: ${member.address} after ${previousStatus}")
    case _: ClusterDomainEvent => // ignore
  }
}
{% endhighlight %}

**Global.scala**
{% highlight scala %}
import play.api._

object Global extends GlobalSettings {

  override def onStart(app: Application) {
    actors.ClusterDemo.start()
  }

  override def onStop(app: Application)  {
    actors.ClusterDemo.stop()
  }
}
{% endhighlight %}
