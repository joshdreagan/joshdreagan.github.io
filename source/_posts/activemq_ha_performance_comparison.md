---
title: ActiveMQ HA Performance Comparison
tags:
  - activemq
  - amq
cover: /2017/03/15/activemq_ha_performance_comparison/post-bg.jpg
thumbnail: /2017/03/15/activemq_ha_performance_comparison/post-bg.jpg
date: 2017-03-15 14:07:01
---


A while back, I wrote a blog post on {% post_link ha_deployments_with_fuse %}. Surprisingly, it received a lot of interest. In particular, the JMS section. Apparently, there are tons of people trying to solve the cross-dc HA problem... Not surprisingly, everyone claims to have the highest possible failure requirements and asked me what I meant by "prepare to make some serious performance tradeoffs". So I figured I'd give some concrete numbers for comparison.<!-- more -->

_Before we get started though, here are some very important disclaimers: First, I did absolutely no tweaking of options or performance tuning. I just unzipped the [Red Hat JBoss A-MQ](https://developers.redhat.com/products/amq) distro, uncommented a user, and started it up. For the DRBD tests, I only changed the path where it stores the KahaDB to point to the replicated filesystem mount. Similarly, I just did a standard [DRBD](http://drbd.org) installation by following their getting started docs. I did not tweak or otherwise optimize the synchronization protocols in any way. Second, I used an 'm4.xlarge' [EC2 instance type](https://aws.amazon.com/ec2/instance-types/) and attached a standard SSD [EBS volume type](https://aws.amazon.com/ebs/details/). So more of a mid-level machine with average (or below) networking and storage performance._

__Given the above conditions, you should not take these numbers as a performance metric. You could likely get much greater performance if you tweaked some of the network settings and used more performant machine. These numbers are only meant to serve as a comparison of local storage vs replicated storage.__

Now that I've got the "disclaimers" out of the way, let me run down what I actually did...

First, I wanted to run a baseline (since this is not about performance numbers directly, but rather comparison). So I created an 'm4.xlarge' instance used to host my broker. We'll call it "broker 1". I also created a second 'm4.xlarge' instance in the same availability zone and region to host my client (both producer and consumer). We'll call it "client". Next, I installed [Red Hat JBoss A-MQ 6.3](https://developers.redhat.com/products/amq) on "broker 1". I also created a Maven project for my [ActiveMQ Perf Maven Plugin](http://activemq.apache.org/activemq-performance-module-users-manual.html) on "client". I then ran a client instance with 5 threads for the producer, and another instance with 5 threads for the consumer. _Note: Both clients were run from the same "client" EC2 machine._ I then repeated the test using 25 threads, and again using 50 threads. I tried to go above 50 threads, but the performance started to degrade. Likely due to a network or storage bottleneck for the machine/storage types I chose. All the numbers for these runs can be found in the table below in the "Baseline" row.

Once I had my baseline, I spun up another 'm4.xlarge' instance in the same region, but on another availability zone. We'll call this one "broker 2" I also created and attached an EBS volume (using a standard SSD) to both of my "broker" instances ("broker 1" and "broker 2"). I installed DRBD on the instances and configured it to replicate synchronously (using [Protocol C](https://www.linbit.com/drbd-user-guide/users-guide-drbd-8-4/#s-replication-protocols)) from "broker 1" to "broker 2". Once I brought them online and the initial sync was done, I ran the same tests as above and captured the numbers. The numbers can be found in the table below in the "DRBD (across availability zones)" row. As you can see from the results, we do get a performance drop (because we have to write the data twice), but it's not too bad since we're not having to replicate across a WAN yet.

Finally, I terminated the "broker 2" instance and recreated it in another region. So now, the "broker 1" and "client" instances were in the "US West (Oregon)" region. And the "broker 2" instance was in the "US East (N. Virginia)" region. I reconnected DRBD to replicate from "broker 1" to "broker 2" and waited for the initial sync to complete (this took a while). Once that was done, I re-ran the same tests and captured the output again. It can be found in the table below in the "DRBD (across regions)" row.

Now we see the "performance tradeoffs" I was talking about... In my tests, it was between 1-2 orders of magnitude slower than local storage. So I will repeat my statement from my previous blog... "You will not be processing large sets of data while synchronously replicating across a WAN."

<hr/>

__Performance Results__

|                                  | 5 Threads | 25 Threads | 50 Threads |
| -------------------------------- | --------: | ---------: | ---------: |
| Baseline                         |  2229 tps |   9456 tps |  12524 tps |
| DRBD (across availability zones) |  1636 tps |   7068 tps |   9133 tps |
| DRBD (across regions)            |    26 tps |    129 tps |    256 tps |

<hr/>

For reference, here are the links to the actual [ActiveMQ Perf Maven Plugin](http://activemq.apache.org/activemq-performance-module-users-manual.html) run outputs:

__5 Threads__

 - {% asset_link 5/baseline/producer.txt Baseline Producer %}
 - {% asset_link 5/baseline/consumer.txt Baseline Consumer %}
 - {% asset_link 5/drbd/producer-drbd.txt DRBD Producer (across availability zones) %}
 - {% asset_link 5/drbd/consumer-drbd.txt DRBD Consumer (across availability zones) %}
 - {% asset_link 5/drbd/producer-drbd-x-region.txt DRBD Producer (across regions) %}
 - {% asset_link 5/drbd/consumer-drbd-x-region.txt DRBD Consumer (across regions) %}

__25 Threads__

 - {% asset_link 25/baseline/producer.txt Baseline Producer %}
 - {% asset_link 25/baseline/consumer.txt Baseline Consumer %}
 - {% asset_link 25/drbd/producer-drbd.txt DRBD Producer (across availability zones) %}
 - {% asset_link 25/drbd/consumer-drbd.txt DRBD Consumer (across availability zones) %}
 - {% asset_link 25/drbd/producer-drbd-x-region.txt DRBD Producer (across regions) %}
 - {% asset_link 25/drbd/consumer-drbd-x-region.txt DRBD Consumer (across regions) %}

__50 Threads__

 - {% asset_link 50/baseline/producer.txt Baseline Producer %}
 - {% asset_link 50/baseline/consumer.txt Baseline Consumer %}
 - {% asset_link 50/drbd/producer-drbd.txt DRBD Producer (across availability zones) %}
 - {% asset_link 50/drbd/consumer-drbd.txt DRBD Consumer (across availability zones) %}
 - {% asset_link 50/drbd/producer-drbd-x-region.txt DRBD Producer (across regions) %}
 - {% asset_link 50/drbd/consumer-drbd-x-region.txt DRBD Consumer (across regions) %}
