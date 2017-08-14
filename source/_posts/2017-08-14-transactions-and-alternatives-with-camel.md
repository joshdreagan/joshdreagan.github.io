---
title: Transactions and Alternatives with Camel
tags:
  - activemq
  - artemis
  - jboss
  - a-mq
  - camel
  - fuse
  - openshift
  - spring-boot
  - narayana
  - karaf
  - eap
  - wildfly
banner: post-bg.jpg
permalink: transactions_and_alternatives_with_camel
date: 2017-08-14 09:54:38
---


There are loads of use cases which require "all or nothing" processing. And there are a bunch of different strategies for accomplishing said result. Luckily for me, they've already been covered many times in tons of different blogs/books/articles. So for this post I'm just going to concentrate on a few of the strategies, and more specifically, how to do them with [Apache Camel](http://camel.apache.org/).<!-- more -->

## Transactions

The first (and most obvious) solution that I'd like to cover, is transactions. Usually, when people need to do a bunch of tasks in an atomic fashion, they simply use a transaction. This can be either a local transaction, or an XA one. Basically, you get to pawn off all of the complication onto a transaction manager and keep your application code clean. So it's a great option if you're using resources that can be transacted (ie, a relational database, or a JMS broker).

If you're only using a single resource, you can do a local transaction. Which is a nice balance of simplicity and speed. For instance, consuming from a queue, enriching with some extra data, and then producing to another queue on the same broker. The only thing you have to be cautious of is that you maintain a single thread throughout your processing. This really only gets tricky if you do something like a [Splitter EIP](http://camel.apache.org/splitter.html) and turn on the `parallelProcessing` option. If you need an example of configuring a local transaction, you can just take a look at the docs [[http://camel.apache.org/transactional-client.html](http://camel.apache.org/transactional-client.html)].

However, if you are using multiple resources, you will need to use an XA transaction. For instance, consuming from a queue, and inserting into a database. XA transaction managers are usually provided and configured by your container. If you're using [Red Hat JBoss Fuse on Karaf](https://developers.redhat.com/products/fuse/overview/), you'll likely use [Aries](http://aries.apache.org/). If you're on [Red Hat JBoss EAP](https://developers.redhat.com/products/eap/overview/), you'll use [Narayana](http://narayana.io/) (formerly JBoss TM). [Spring Boot](https://projects.spring.io/spring-boot/) provides no transaction manager out-of-the-box. Instead, it has hooks to auto-configure various TM's based on which one you've added as a dependency. In case you're curious and would like an example of Camel + XA + Spring Boot, take a look here [[https://github.com/joshdreagan/camel-spring-boot-xa](https://github.com/joshdreagan/camel-spring-boot-xa)].

Using an XA transaction manager does increase complexity a bit (at least from a configuration perspective), and comes with a handful of requirements and caveats:

The first requirement, is that you will (obviously) need to run and configure some sort of XA capable transaction manager. There are several options available (as outlined above). And because they all implement JTA, you can swap them out with no changes to your code. So you can shop around and find the one that works best.

Second, due to the requirement of a 2-phase commit, it will be significantly slower. This is usually a huge sticking point for a lot of people as they don't want to (or can't afford to) pay that performance penalty. Unfortunately, there is little that can be done about it. Or more accurately, I have never seen an XA transaction manager implementation that maintains the speed and simplicity of a non-XA one.

The third, and often overlooked, requirement is that you will need some sort of persistence. This is because, in the case of a crash, the recovery manager will attempt to pick up where things left off. And in order to survive a crash, we need persistence...

It's worth noting that this third requirement (persistence) makes HA a bit of a pain. As mentioned above, transaction managers will run some sort of recovery thread in the background so that they can (as the name would suggest) recover transactions that were not yet complete at the time of a crash. But they all (or at least all of the implementations I know of) can only have a single instance of the tx and recovery manager per object store (or more specifically, per tx manager id). So that means that, if I wanted to scale out my application (to make up for the added slowness of XA), each server would likely have its own persistent store. Most people don't even notice when they're using a server like [JBoss WildFly](http://wildfly.org/) because each instance will (by default) write its logs to a subfolder of its installation. This can be (in my opinion) __very__ dangerous because most people are unaware that that directory should be sitting on some sort of resilient storage. If, however, you're running on a platform like [OpenShift](https://www.openshift.com/), you will be immediately aware because all instances will share the same storage mount and configuration, and will simply fail to work properly. You _could_ use subfolders for each pod instance, and then create a separate recovery pod that would run independent of your application and would spin up recovery managers for each downed instance. In fact, I actually had an implementation working at one point. But it was quite clunky, and after a quick conversation with Hiram Chirino at one of our meetups, I concluded that he was working on a __way__ more elegant solution using [Stateful Sets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/). So for now, if you want to run XA transactions on OpenShift, I would either limit my app to a single instance (ie, no scaling), or wait a bit for Hiram's version.

## Idempotent Consumer

So what if I can't (or don't want to) use XA transactions? Perhaps I value speed over application simplicity... Perhaps I'm not dealing with "transactable" resources... Perhaps I'm running on OpenShift and can't wait on Hiram... ;) No matter what the reason, it'd be nice to have some other options. Luckily, as mentioned at the beginning of the post, there are oodles of options. So lets talk about one of the more popular ones... idempotent consumer.

With the [Idempotent Consumer EIP](http://www.enterpriseintegrationpatterns.com/patterns/messaging/IdempotentReceiver.html), you give up on trying to do things atomically, and instead favor eventual consistency. In other words, I write my application in such a way that retries will not hurt anything. So if I have a failure, I can just keep retrying until I eventually succeed all the way through.

To give a concrete example... Let's say that I wanted to read a file from an input directory, unmarshal/validate the contents, insert it as a record into a DB, and also write it out to a file in an output directory (perhaps in parallel). In this example, only the DB write can participate in a transaction. Both the consumption of and the production of files cannot. So local (or even XA) transactions are not an option. But if I put a simple check before writing the DB and also before writing the file, I can retry as many time as I'd like with no negative side-effects. To be very specific, if I was successful in writing to the DB, but failed on writing the file (maybe because the filesystem was temporarily full), I can just re-ingest the same file. The DB check will ensure that I don't try that step again. So I'll basically skip it and then try the file write again. Hopefully this time it succeeds...

There are a couple of ways that I can go about implementing this with Camel. One way would be to guard each step using the [Idempotent Consumer EIP](http://camel.apache.org/idempotent-consumer.html). With this EIP, I select a unique id (or rather, an expression to retrieve a unique id) for each message. When that message is received, its unique id will checked against an `org.apache.camel.spi.IdempotentRepository` (of which there are many implementations to choose from). If it already exists, it will simply be skipped. If not, it will be added and then passed on to the processor. Now, I can ingest/retry my data as many times as I want and be _relatively_ certain that each step will only be performed once. If you want an example of this pattern, take a look here: [[https://github.com/joshdreagan/camel-idempotent-consumer](https://github.com/joshdreagan/camel-idempotent-consumer)].

This pattern works great for most cases. It's easy to implement, and maintains pretty good performance. But sometimes it just isn't flexible or robust enough. More specifically, what do I do if I don't have a unique message id to key off of? Also, what happens if I have a system failure after adding something to the idempotent repository, but before performing my actual processing? Or what if I have an error during my processing, but suffer a system failure before I can remove the message id from my repository? Basically, I have situations where my idempotent repository could be out-of-sync with my actual processing. If I'm using the JDBC based implementation to guard a DB insert, I could use a local transaction to make sure both of those steps occur atomically. But that doesn't really help with my file writing use case. The margin for error is pretty small, and might be acceptable. But what if it's not? Luckily for us, Camel usually has more than one way to solve a problem...

The Idempotent Consumer EIP is really just a specialized version of the [Message Filter EIP](http://camel.apache.org/message-filter.html). The biggest difference, is that its expression will return a boolean dictating whether or not it skips or processes the message. So instead of just matching an id, I can perform any steps I'd like to determine if a message has already been processed before. Which solves my first issue... Using my example above, I would guard my DB insert with one filter check, and then my file write with another filter check. The DB filter check could query the DB to see if my message had already been inserted, and then skip processing if it had. My file writer check could similarly check to see if the destination file had already been written, and then skip processing if it had. It's a tiny bit more complicated to use, but since I'm not maintaining a separate idempotent repository, I don't have any chance of getting out-of-sync. So that solves my system failure issues... If you want an example of what this might look like, take a look at this example: [[https://github.com/joshdreagan/camel-filter-consumer](https://github.com/joshdreagan/camel-filter-consumer)].

As I said at the beginning of this post, there are several ways to tackle this issue. Of which, I've only covered a few. And in doing so, I used some fairly contrived use cases. But hopefully it's still worthwhile, and at a bare minimum maybe someone will find the code examples useful...
