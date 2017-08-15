---
title: Bridging Apache Artemis
tags:
  - activemq
  - artemis
  - jboss
  - a-mq
  - camel
  - fuse
banner: post-bg.jpg
permalink: bridging_apache_artemis
---


In a perfect world, every project would be a greenfield project and you could pick and choose the latest/greatest broker every time. And of course, if given the choice you'd pick [Apache Artemis](https://activemq.apache.org/artemis/)... ;) But in the real world, it is much more common that customers will have a very large existing codebase and will have to transition to the latest/greatest over some much longer period of time. And during that period, it's helpful to be able to set up some sort of bridge between your brokers so that your old and new code can still communicate.<!-- more -->

Before we get started though, there are a few things to clear up... The term "bridge" can mean may things. Artemis has something called a "[Core Bridge](https://activemq.apache.org/artemis/docs/latest/core-bridges.html)". Core bridges do __not__ use the JMS API, and are used only for bridging Artemis instances together (either explicitly, or implicitly when using cluster connections). While useful, Core bridges are not the focus of this post. Instead, we're going to talk about how you might bridge Artemis to another JMS broker (ie, [IBM MQ](https://www.ibm.com/software/products/en/ibm-mq)).

There is a baked in JMS bridge implementation in the Artemis codebase that you can use if you'd like. You can check out the docs on it here: [[https://activemq.apache.org/artemis/docs/latest/jms-bridge.html](https://activemq.apache.org/artemis/docs/latest/jms-bridge.html)]. It's pretty basic, but functional if you don't require any kind of transformations or other flexibility. To use it, simply instantiate an instance of `org.apache.activemq.artemis.jms.bridge.JMSBridge`, and pass it the required arguments (ie, source/destination `org.apache.activemq.artemis.jms.bridge.ConnectionFactoryFactory` instances, source/destination `org.apache.activemq.artemis.jms.bridge.DestinationFactory` instances, source/destination credentials, ...). Once you have your instance, you can call `start()` on it to begin consuming/producing messages. There's even an example included in the distribution: [[jms-bridge](https://github.com/apache/activemq-artemis/tree/master/examples/features/standard/jms-bridge)]. Seems pretty straightforward right? Well... maybe not. If you're running Artemis standalone (like most people), you don't really have a good way to load extra code. That is, there's no "hot deploy" folder where you can just drop a jar file with this config in it and have it automatically startup and shutdown with the broker. There has been some effort to create a plugin framework in the newer versions, but a quick look at the code showed that it still doesn't provide hooks into start/stop or other broker lifecycle events. So what do we do?

As it turns out, there is an embedded [Jetty](http://www.eclipse.org/jetty/) instance that serves the web admin console. And it already starts/stops with the server. So we can just piggyback on that to bootstrap our code. But if I'm going through all of this effort to bootstrap some code, I might as well use something with a bit more power (ie, [Apache Camel](http://camel.apache.org/)). That way, if I have any need to perform message transformations, set unique headers for idempotent consumption, wiretap audit logs, or do anything else outside of simply copying messages, I can do so. Sounds simple enough, but how do I actually do it?

First, you'll need to create a WAR containing all of your dependencies as well as your Spring XML defined Camel routes. We'll refer to this as `jms-bridge.war`... In the Camel routes, you'll place your actual JMS bridge definitions (as many as you need). In the `WEB-INF/web.xml`, you'll boostrap the Spring ApplicationContext using the `org.springframework.web.context.ContextLoaderListener` class. This is a pretty standard way to bootstrap a Spring application into a servlet container.

Second, you'll copy `jms-bridge.war` file into the `$ARTEMIS_HOME/web` folder.

Finally, you'll add a definition to the `$ARTEMIS_INSTANCE/etc/bootstrap.xml` file as shown below:

{% codeblock lang:xml %}
<broker xmlns="http://activemq.org/schema">
  ...
  <web bind="http://localhost:8161" path="web">
    ...
    <app url="jms-bridge" war="jms-bridge.war"/>
  </web>
</broker>
{% endcodeblock %}

*Notice the use of `$ARTEMIS_HOME` and `$ARTEMIS_INSTANCE` in the second and third steps... `$ARTEMIS_HOME` refers to the location that you unzipped the Artemis distribution. `$ARTEMIS_INSTANCE` refers to the directory of your broker instance (created with the "`artemis create ...`" command.*

Now when you start the broker instance, it will load up and start the Camel routes and begin bridging. Pretty cool! But we bundled up all of our dependencies inside our WAR file (ie, all of the IBM MQ JMS client libs). And we also bundled our Camel routes definition inside the WAR as well. And since the WAR file sits in the `$ARTEMIS_HOME/web` directory, its configurations apply to any and all of the instances that we create/run on that machine. Not to mention the fact that every change to my Camel routes will require a build/deploy/restart. All of this kind of sucks and we can do better... 

If you dig into the code a bit more, you'll see that the "main" that spins up the Artemis broker will actually add several directories, JARs, & ZIPs to the classpath. Specifically, it will add the `$ARTEMIS_HOME/etc` & `$ARTEMIS_INSTANCE/etc` directories. It will then scan through the `$ARTEMIS_HOME/lib` & `$ARTEMIS_INSTANCE/lib` directories, and add any JARs or ZIPs that it finds to the classpath (sorted by name). So that means that we can take all of our use-case specific stuff (ie, all of the IBM MQ JMS client libs) and put them in the `$ARTEMIS_INSTANCE/lib` folder. It also means that we can take the Camel routes configuration, and put it in the `$ARTEMIS_INSTANCE/etc` directory. Now we have a completely generic WAR that we deploy in the `$ARTEMIS_HOME/lib` directory. That WAR can be activated on an instance defined basis as needed. And each instance can have the dependencies and configuration that it requires completely independent of other instances. Additionally, I have the added benefit of being able to edit the Camel routes configuration and apply my changes with only a restart (ie, no re-build/re-deploy required). Neat! Want to see what it looks like? Take a look at this sample project: [[https://github.com/joshdreagan/artemis-jms-bridge](https://github.com/joshdreagan/artemis-jms-bridge)].

