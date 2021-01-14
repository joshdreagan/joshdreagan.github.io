---
title: Upgrading AMQ 6 to AMQ 7
tags:
  - amq
  - activemq
  - artemis
cover: /2017/12/01/upgrading_amq_6_to_amq_7/post-bg.jpg
thumbnail: /2017/12/01/upgrading_amq_6_to_amq_7/post-bg.jpg
date: 2017-12-01 15:30:03
---


So [Red Hat AMQ 7](https://developers.redhat.com/products/amq/overview/) has been out for a while now, and there have already been a lot of customers who are understandably eager to upgrade to the latest and greatest version. Many of them have been reaching out and asking for help/instructions on how to migrate. So far I've been replying that I already blogged about that here: {% post_link decommissioning_jboss_a-mq_brokers %}. But that reply has been met with some confusion. So I figured I'd write up a more concrete set of instructions.<!-- more -->

Let's start by stating the problem: "I have a current, production system that is utilizing AMQ 6 and I'd like to upgrade it to AMQ 7 with as little downtime as possible". Well... such things do require a bit of planning, but it's not as difficult as it might seem. The easiest way to upgrade any app is _usually_ to do a rolling upgrade. I emphasize "_usually_" because every customer has a slightly different case, architecture, requirements, or other wrenches that can make things more difficult. But for the purpose of this blog, let's focus on the most typical case.

For the first step, let's walk through upgrading the brokers. First, we'll need to install AMQ 7 and create a broker instance. No need to enumerate the steps here since that's already been covered in the docs: [[https://access.redhat.com/documentation/en-us/red_hat_jboss_amq/7.0/html/using_amq_broker/](https://access.redhat.com/documentation/en-us/red_hat_jboss_amq/7.0/html/using_amq_broker/)]. For the purpose of this post, we'll assume that you plan to match your existing setup (ie, if you currently have a master/slave pair, you will install a new AMQ 7 master/slave pair). If you're installing to new hardware, you can go ahead and start the newly created broker instance(s). Otherwise, we'll have to shut down the existing AMQ 6 broker before starting the AMQ 7 broker. Yes... you could modify the ports and bring it up alongside your AMQ 6 broker, but you risk causing a contention for resources if you do so. So best not to tempt fate...

Next, let's talk about the clients. One of the awesome features in AMQ 7 is that is supports all of the same protocols that AMQ 6 did (in addition to a couple more). This means that your existing clients (with their existing client libraries) can seamlessly connect to the new AMQ 7 brokers. All you'll need to do is give the clients the new broker URL and they can immediately begin producing/consuming. And you don't even need to do that if you installed to the same hardware and bound to the same port. In fact, if your clients are using the "failover" protocol (and you didn't update the host/port), they will automatically switch over as soon as you take down the AMQ 6 broker and bring the AMQ 7 broker online. Neat! <img style="display: inline; height: 15px;" src="{% asset_path "banana_dance.gif" %}"/> It's worth noting that this will not be the case forever. Eventually, the OpenWire format (and potentially other formats) will be deprecated and removed from support. But that is a long ways away. So you'll have plenty of time to go back through and update all of your client applications with the newer client libraries as time/budget permits.

But what about those in-flight messages? The messages that had been accepted by the old AMQ 6 instance, but had not yet been delivered to a consumer client. Well, that is exactly what I wrote {% post_link decommissioning_jboss_a-mq_brokers "my previous blog post" %} about. Those messages cannot be lost. So we need to "drain" them from the old AMQ 6 KahaDB store and send them to the new AMQ 7 broker. Luckily, due to the fact that the AMQ 7 broker can still speak OpenWire, you can use the exact same drainer code that I provided in that blog: [[https://github.com/joshdreagan/activemq-drainer](https://github.com/joshdreagan/activemq-drainer)]. Super neat! <img style="display: inline; height: 15px;" src="{% asset_path "carlton_dance.gif" %}"/>

That's it! Rinse and repeat for each broker instance/pair. You can do it all at once, or in a "rolling" fashion. Up to you...

So to summarize, here are the high-level steps that you will perform:

* Install the new AMQ 7 broker/instance.
* Stop the AMQ 6 broker.
* Start the AMQ 7 broker.
* Update your clients with the new broker URL (only necessary if you installed to new hardware or otherwise changed the host/port).
* Drain the messages from the AMQ 6 KahaDB to the new AMQ 7 broker.
* Eventually plan on upgrading your client libraries.
* Optionally delete the old AMQ 6 installation once you're satisfied that the upgrade has completed successfully.

Did I cover every possible use case, permutation, and complication? No... But this should be a good starting point. And if you need more guidance, well... that's what we have [Red Hat Consulting](https://www.redhat.com/en/services/consulting) for.
