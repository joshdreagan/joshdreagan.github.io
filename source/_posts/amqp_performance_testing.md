---
title: AMQP Performance Testing With JBoss A-MQ
cover: /2016/02/02/amqp_performance_testing/post-bg.png
thumbnail: /2016/02/02/amqp_performance_testing/post-bg.png
date: 2016-02-02 00:00:00
tags:
- activemq
- amq
---

I recently had a customer that wanted us to do some load testing of [Red Hat's JBoss A-MQ](http://www.jboss.org/products/amq/overview/) for them. In particular, this customer wanted the tests performed using the AMQP protocol instead of ActiveMQ's native OpenWire. From previous engagements, I knew that there would be a performance difference. But after a quick look I didn't see any blogs or posts on the subject. More specifically, I didn't see any posts that detailed how to run the tests yourself so that you could get real numbers in your own environment. So I figured I'd write up some steps and post my results for future reference.
<!-- more -->

> __Please note that the purpose of this blog post is not to give you a number of msg/s to expect or tell you the absolute best way to tune your broker. The purpose of this post is to show you how to run the tests yourself and give you some very rough idea of the performance difference between the two protocols. All of my testing was done on my laptop using the default configurations. You will probably get wildly different performance in your environment and will likely need to tune the broker specific to your use case to get the best performance possible.__

## Protocol Basics

Both [AMQP (Advanced Message Queuing Protocol)](https://www.amqp.org/) and [OpenWire](http://activemq.apache.org/openwire.html) are what we refer to as "wire-level protocols". They define how a client and broker will negotiate a connection, how they'll format messages, and how they'll communicate in general. ActiveMQ can speak many different wire-level protocols, but OpenWire and AMQP are supposed to be the fastest as they are both binary in nature. In both cases, you have multiple options as to what language your clients can be written in (ie, Java, .NET, C++, ...). It really doesn't matter as long as they are communicating via the same wire-level protocol as the broker. In this case, I'll be using Java simply because of the availability of test tools.

## Test Details

There are a ton of available test tools that you can use to drive traffic to your brokers. For this blog, I used the [ActiveMQ Perf Maven Plugin](http://activemq.apache.org/activemq-performance-module-users-manual.html). This is a great tool that comes with the ActiveMQ source code and allows you to run as both producers and consumers. It is easily configurable and gives you a ton of options to test various scenarios (ie, persistent/non-persistent, transacted/non-transacted, 1-n producer/consumer threads, ...). The only caveat to the tool is that it will load a connection factory that uses the OpenWire protocol. Luckily, it was written in such a way that we can extend it to use AMQP as well. To do that, we only need to create one class that knows how to load an AMQP connection factory. The source for the class is as follows:

{% codeblock lang:java %}
package org.jboss.examples.amqp.spi;

import org.apache.activemq.tool.spi.ReflectionSPIConnectionFactory;

public class AMQPReflectionSPIConnectionFactory extends ReflectionSPIConnectionFactory {

  @Override
  public String getClassName() {
    return "org.apache.qpid.jms.JmsConnectionFactory";
  }
}
{% endcodeblock %}

Then we just need to make sure that our custom class and the Qpid client libraries are on the Maven classpath.

{% codeblock lang:xml %}
<dependency>
  <groupId>${project.groupId}</groupId>
  <artifactId>${project.artifactId}</artifactId>
  <version>${project.version}</version>
</dependency>
<dependency>
  <groupId>org.apache.activemq</groupId>
  <artifactId>activemq-amqp</artifactId>
  <version>${activemq.version}</version>
</dependency>
<dependency>
  <groupId>org.apache.qpid</groupId>
  <artifactId>qpid-jms-client</artifactId>
  <version>${qpid.version}</version>
</dependency>
{% endcodeblock %}

The tricky part is that we need them on the classpath of the Maven plugin itself and not necessarily the classpath of the project. Take a look at the [pom.xml](https://github.com/joshdreagan/amqp-perf-test/blob/master/pom.xml) file to see what I mean. The full code for this test can be found at [https://github.com/joshdreagan/amqp-perf-test](https://github.com/joshdreagan/amqp-perf-test). Feel free to just clone it down and use it directly.

If you take a look at the [plugin's documentation](http://activemq.apache.org/activemq-performance-module-users-manual.html), you'll see several properties that can be set for the test as a whole, the consumer, the producer, and the connection factory. You can configure the properties using "-Dkey=value" arguments on the command line, or via a .properties file (or some combination of the two). If you plan to run the test multiple times, it's probably worth creating a .properties file. The only detail that I'll give here is that the connection factory properties are set via reflection and will be specific to the connection factory used. I mention this because we swap out the connection factory when we run the AMQP tests. To give a concrete example, when using the `org.apache.activemq.tool.spi.ActiveMQReflectionSPI` you will set the URI for the broker using `-Dfactory.brokerURL=tcp://localhost:61616`, but when using the `org.jboss.examples.amqp.spi.AMQPReflectionSPIConnectionFactory` you will set the URI for the broker using `-Dfactory.remoteURI=amqp://localhost:5672`. This is because the "setters" are named differently for the different connection factory implementations. The available options for the AMQP connection factory can be found in the [Qpid docs](https://qpid.apache.org/releases/qpid-jms-0.5.0/docs/index.html). All other options (ie, producer, consumers, & test) should be the same and are found on the [plugin docs](http://activemq.apache.org/activemq-performance-module-users-manual.html).

## Test Results

Here are some sample runs that I did. _Please note the disclaimer at the beginning of this blog post regarding performance results._

### Producer OpenWire

{% codeblock %}
$> mvn activemq-perf:producer -DsysTest.propsConfigFile=src/main/resources/tcp-producer.properties
OpenJDK 64-Bit Server VM warning: ignoring option MaxPermSize=2048m; support was removed in 8.0
[INFO] Scanning for projects...
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building ActiveMQ Perf: AMQP Perf Test 1.0.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO]
[INFO] --- activemq-perf-maven-plugin:5.11.0:producer (default-cli) @ amqp-perf-test ---
[INFO] Loading properties file: /home/jreagan/Development/Projects/joshdreagan/amqp-perf-test/src/main/resources/tcp-producer.properties
[INFO] Created: org.apache.activemq.ActiveMQConnectionFactory using SPIConnectionFactory: org.apache.activemq.tool.spi.ActiveMQReflectionSPI
[INFO] Sampling duration: 300000 ms, ramp up: 0 ms, ramp down: 0 ms
[INFO] Creating queue: queue://TEST.FOO
[INFO] Creating JMS Connection: Provider=ActiveMQ-5.11.0, JMS Spec=1.1
[INFO] Creating  producer to: queue://TEST.FOO with non-persistent delivery.
[INFO] Starting to publish 1024 byte(s) messages for 300000 ms
[INFO] Finished sending
[INFO] Client completed
#########################################
####    SYSTEM THROUGHPUT SUMMARY    ####
#########################################
System Total Throughput: 13347982
System Total Clients: 1
System Average Throughput: 44493.27333333333
System Average Throughput Excluding Min/Max: 44274.73333333333
System Average Client Throughput: 44493.27333333333
System Average Client Throughput Excluding Min/Max: 44274.73333333333
Min Client Throughput Per Sample: clientName=JmsProducer0, value=0
Max Client Throughput Per Sample: clientName=JmsProducer0, value=65562
Min Client Total Throughput: clientName=JmsProducer0, value=13347982
Max Client Total Throughput: clientName=JmsProducer0, value=13347982
Min Average Client Throughput: clientName=JmsProducer0, value=44493.27333333333
Max Average Client Throughput: clientName=JmsProducer0, value=44493.27333333333
Min Average Client Throughput Excluding Min/Max: clientName=JmsProducer0, value=44274.73333333333
Max Average Client Throughput Excluding Min/Max: clientName=JmsProducer0, value=44274.73333333333
[INFO] Created performance report: /home/jreagan/Development/Projects/joshdreagan/amqp-perf-test/./target/JmsProducer_numClients1_numDests1_all.xml
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 05:10 min
[INFO] Finished at: 2016-02-02T11:46:30-07:00
[INFO] Final Memory: 8M/235M
[INFO] ------------------------------------------------------------------------
{% endcodeblock %}

### Consumer OpenWire

{% codeblock %}
$> mvn activemq-perf:consumer -DsysTest.propsConfigFile=src/main/resources/tcp-consumer.properties
OpenJDK 64-Bit Server VM warning: ignoring option MaxPermSize=2048m; support was removed in 8.0
[INFO] Scanning for projects...
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building ActiveMQ Perf: AMQP Perf Test 1.0.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO]
[INFO] --- activemq-perf-maven-plugin:5.11.0:consumer (default-cli) @ amqp-perf-test ---
[INFO] Loading properties file: /home/jreagan/Development/Projects/joshdreagan/amqp-perf-test/src/main/resources/tcp-consumer.properties
[INFO] Created: org.apache.activemq.ActiveMQConnectionFactory using SPIConnectionFactory: org.apache.activemq.tool.spi.ActiveMQReflectionSPI
[INFO] Sampling duration: 300000 ms, ramp up: 0 ms, ramp down: 0 ms
[INFO] Creating queue: queue://TEST.FOO
[INFO] Creating JMS Connection: Provider=ActiveMQ-5.11.0, JMS Spec=1.1
[INFO] Creating non-durable consumer to: queue://TEST.FOO
[INFO] Starting to asynchronously receive messages for 300000 ms...
[INFO] Client completed
#########################################
####    SYSTEM THROUGHPUT SUMMARY    ####
#########################################
System Total Throughput: 13287310
System Total Clients: 1
System Average Throughput: 44291.03333333333
System Average Throughput Excluding Min/Max: 44074.76
System Average Client Throughput: 44291.03333333333
System Average Client Throughput Excluding Min/Max: 44074.76
Min Client Throughput Per Sample: clientName=JmsConsumer0, value=0
Max Client Throughput Per Sample: clientName=JmsConsumer0, value=64882
Min Client Total Throughput: clientName=JmsConsumer0, value=13287310
Max Client Total Throughput: clientName=JmsConsumer0, value=13287310
Min Average Client Throughput: clientName=JmsConsumer0, value=44291.03333333333
Max Average Client Throughput: clientName=JmsConsumer0, value=44291.03333333333
Min Average Client Throughput Excluding Min/Max: clientName=JmsConsumer0, value=44074.76
Max Average Client Throughput Excluding Min/Max: clientName=JmsConsumer0, value=44074.76
[INFO] Created performance report: /home/jreagan/Development/Projects/joshdreagan/amqp-perf-test/./target/JmsConsumer_numClients1_numDests1_equal.xml
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 05:00 min
[INFO] Finished at: 2016-02-02T11:46:19-07:00
[INFO] Final Memory: 7M/220M
[INFO] ------------------------------------------------------------------------
{% endcodeblock %}

### Producer AMQP

{% codeblock %}
$> mvn activemq-perf:producer -DsysTest.propsConfigFile=src/main/resources/amqp-producer.properties
OpenJDK 64-Bit Server VM warning: ignoring option MaxPermSize=2048m; support was removed in 8.0
[INFO] Scanning for projects...
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building ActiveMQ Perf: AMQP Perf Test 1.0.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO]
[INFO] --- activemq-perf-maven-plugin:5.11.0:producer (default-cli) @ amqp-perf-test ---
[INFO] Loading properties file: /home/jreagan/Development/Projects/joshdreagan/amqp-perf-test/src/main/resources/amqp-producer.properties
[INFO] Created: org.apache.qpid.jms.JmsConnectionFactory using SPIConnectionFactory: org.jboss.examples.amqp.spi.AMQPReflectionSPIConnectionFactory
[INFO] Sampling duration: 300000 ms, ramp up: 0 ms, ramp down: 0 ms
[INFO] Creating queue: queue://TEST.FOO
[INFO] Best match for SASL auth was: SASL-PLAIN
[INFO] Connection ID:bearkat-34938-1454439352850-0:2 connected to remote Broker: amqp://localhost:5672
[INFO] Creating JMS Connection: Provider=QpidJMS-0.5.0, JMS Spec=1.1
[INFO] Creating  producer to: TEST.FOO with non-persistent delivery.
[INFO] Starting to publish 1024 byte(s) messages for 300000 ms
[INFO] Finished sending
[INFO] Client completed
#########################################
####    SYSTEM THROUGHPUT SUMMARY    ####
#########################################
System Total Throughput: 3600145
System Total Clients: 1
System Average Throughput: 12000.483333333334
System Average Throughput Excluding Min/Max: 11947.51
System Average Client Throughput: 12000.483333333334
System Average Client Throughput Excluding Min/Max: 11947.51
Min Client Throughput Per Sample: clientName=JmsProducer0, value=1800
Max Client Throughput Per Sample: clientName=JmsProducer0, value=14092
Min Client Total Throughput: clientName=JmsProducer0, value=3600145
Max Client Total Throughput: clientName=JmsProducer0, value=3600145
Min Average Client Throughput: clientName=JmsProducer0, value=12000.483333333334
Max Average Client Throughput: clientName=JmsProducer0, value=12000.483333333334
Min Average Client Throughput Excluding Min/Max: clientName=JmsProducer0, value=11947.51
Max Average Client Throughput Excluding Min/Max: clientName=JmsProducer0, value=11947.51
[INFO] Created performance report: /home/jreagan/Development/Projects/joshdreagan/amqp-perf-test/./target/JmsProducer_numClients1_numDests1_all.xml
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 05:00 min
[INFO] Finished at: 2016-02-02T12:00:53-07:00
[INFO] Final Memory: 10M/128M
[INFO] ------------------------------------------------------------------------
{% endcodeblock %}

### Consumer AMQP

{% codeblock %}
$> mvn activemq-perf:consumer -DsysTest.propsConfigFile=src/main/resources/amqp-consumer.properties
OpenJDK 64-Bit Server VM warning: ignoring option MaxPermSize=2048m; support was removed in 8.0
[INFO] Scanning for projects...
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building ActiveMQ Perf: AMQP Perf Test 1.0.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO]
[INFO] --- activemq-perf-maven-plugin:5.11.0:consumer (default-cli) @ amqp-perf-test ---
[INFO] Loading properties file: /home/jreagan/Development/Projects/joshdreagan/amqp-perf-test/src/main/resources/amqp-consumer.properties
[INFO] Created: org.apache.qpid.jms.JmsConnectionFactory using SPIConnectionFactory: org.jboss.examples.amqp.spi.AMQPReflectionSPIConnectionFactory
[INFO] Sampling duration: 300000 ms, ramp up: 0 ms, ramp down: 0 ms
[INFO] Creating queue: queue://TEST.FOO
[INFO] Best match for SASL auth was: SASL-PLAIN
[INFO] Connection ID:bearkat-51184-1454439351110-0:2 connected to remote Broker: amqp://localhost:5672
[INFO] Creating JMS Connection: Provider=QpidJMS-0.5.0, JMS Spec=1.1
[INFO] Creating non-durable consumer to: TEST.FOO
[INFO] Starting to asynchronously receive messages for 300000 ms...
[INFO] Client completed
#########################################
####    SYSTEM THROUGHPUT SUMMARY    ####
#########################################
System Total Throughput: 3583507
System Total Clients: 1
System Average Throughput: 11945.023333333333
System Average Throughput Excluding Min/Max: 11899.69
System Average Client Throughput: 11945.023333333333
System Average Client Throughput Excluding Min/Max: 11899.69
Min Client Throughput Per Sample: clientName=JmsConsumer0, value=0
Max Client Throughput Per Sample: clientName=JmsConsumer0, value=13600
Min Client Total Throughput: clientName=JmsConsumer0, value=3583507
Max Client Total Throughput: clientName=JmsConsumer0, value=3583507
Min Average Client Throughput: clientName=JmsConsumer0, value=11945.023333333333
Max Average Client Throughput: clientName=JmsConsumer0, value=11945.023333333333
Min Average Client Throughput Excluding Min/Max: clientName=JmsConsumer0, value=11899.69
Max Average Client Throughput Excluding Min/Max: clientName=JmsConsumer0, value=11899.69
[INFO] Created performance report: /home/jreagan/Development/Projects/joshdreagan/amqp-perf-test/./target/JmsConsumer_numClients1_numDests1_equal.xml
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 05:01 min
[INFO] Finished at: 2016-02-02T12:00:51-07:00
[INFO] Final Memory: 12M/243M
[INFO] ------------------------------------------------------------------------
{% endcodeblock %}

That's it! You can see that OpenWire is quite a bit faster in this case. But please don't take my word for it... clone the project, tweak some configurations, try scaling the producers/consumers/brokers, and run the test yourself to see what results you'll get.
