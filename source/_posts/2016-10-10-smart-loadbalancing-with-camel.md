---
title: Smart LoadBalancing With Camel
tags:
  - fuse
  - camel
  - jboss
  - a-mq
  - activemq
banner: post-bg.jpg
permalink: smart_loadbalancing_with_camel
date: 2016-10-10 15:29:41
---

LoadBalancing is a fairly well-known concept these days. There are a ton of existing strategies out there (ie, RoundRobin, Random, Sticky, Weighted, ...), and there a ton of existing implementations that have been built using both hardware and software (ie, Apache HTTPD, HAProxy, f5, Layer7, ...). So why create another one? Well... although it's not likely to be very useful, I thought it might be neat to see if I could make one that utilized CPU load (or any metric) to do more intelligent routing.<!--more-->

Luckily, like many things in [Camel](http://camel.apache.org/), this is a fairly simple task. I just have to create my own `org.apache.camel.processor.loadbalancer.LoadBalancer` implementation, and I can make it do pretty much anything I want. For instance, [I might implement one to do dynamic discovery using Infinispan](http://joshdreagan.github.io/2015/12/04/custom_camel_loadbalancer_with_infinispan/) (shameless self-promotion :)). But I digress...

So let's break down the wish list: I want to be able to use the strategy for more than just HTTP. I'd like to be able to use any available metric. And I need the collection of said metric to occur asynchronously in the background (so I don't slow down my routing).

{% asset_img "smart-lb.png" "Smart LoadBalancer" %}

Take a look at the source code [https://github.com/joshdreagan/camel-smart-loadbalancer](https://github.com/joshdreagan/camel-smart-loadbalancer) to see how I did it. In my example, I load balanced HTTP calls and used JMX to collect CPU utilization. But you could just as easily use the exact same implementation to monitor [ActiveMQ](http://activemq.apache.org/) queue depth (or queue % full) and load balance between brokers. Or maybe monitor filesystem space and load balance FTP endpoints.

Like I said in the intro, this is probably not terribly useful in a real-world environment since the metrics will likely change at a faster rate than you would reasonably poll. But at the very least, hopefully someone will find it interesting. And maybe... just maybe... it will get Christian Posta to read my blog. :)
