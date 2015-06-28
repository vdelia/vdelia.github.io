---
layout: post
title:  "Getting started with Scala"
date:   2015-06-26 23:58:58
categories: [scala, intro]
---

Why Scala?  Because some of the most remarkable open source projects of the last years, like [Akka][akka] and [Spark][spark], are written in scala.
 It has a big community (which keeps growing), and a lot of hype. Simply stated: if you are interested in distributed systems you cannot ignore Scala.

These are my notes on Scala.

## Install

**You need a JDK installed. I assume you have it.**

The simplest way to be up and running is [SBT][sbt].


### SBT

SBT is the build tool and package manager for Scala. It allows you to ...

On the [download page][sbt-download] you can find an installer for windows, and .tar.gz archives
for other platforms. For Linux Ubuntu/Mint there are PPA repositories availables. For Mac `$ brew install sbt`
just works fine. I suggest you to check the installation instructions [here][sbt-install].





### IntelliJ

[sbt-install]: http://www.scala-sbt.org/release/tutorial/Setup.html
[akka]: http://akka.io
[spark]: https://spark.apache.org
[sbt]: http://www.scala-sbt.org/
[sbt-download]:  http://www.scala-sbt.org/download.html
