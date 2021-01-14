---
title: Streaming in the Cloud With Camel and Strimzi
tags:
  - fuse
  - camel
  - kafka
  - strimzi
  - openshift
  - spring-boot
cover: /2019/05/30/streaming_in_the_cloud_with_camel_and_strimzi/post-bg.jpg
thumbnail: /2019/05/30/streaming_in_the_cloud_with_camel_and_strimzi/post-bg.jpg
date: 2019-05-30 17:23:43
---


So I was at one of my favorite customers several months back, and was demo'ing our new [AMQ Streams](https://access.redhat.com/products/red-hat-amq#streams) product (aka [Strimzi](https://strimzi.io/)). They asked for some example [Apache Camel](https://camel.apache.org/) apps to show them how to securely connect to their newly installed Kafka on OpenShift. Not an unreasonable request... But try as I might, I could not find any existing examples. Sure, I found examples of Camel talking to Kafka, but not doing so securely. I found examples of Camel talking to Kafka _with_ authz, but not running in OpenShift. I even found examples of plain Java clients running in OpenShift __and__ doing authz, but not using Camel. So I offered to create a set of examples for them "as soon as I got some free time". Six months later... here we go! ;)<!-- more -->

For the purpose of this blog, let's go ahead and assume that you've already installed OpenShift. Let's also assume that you've installed Strimzi onto your OpenShift environment and have a Kafka cluster running. There's no need to cover either of those topics since they're both thoroughly outlined in the existing documentation. Instead, let's focus a bit on authentication and authorization. And more specifically, how those work in Strimzi. 

At present, there are two mechanisms for authentication (TLS or SCRAM-SHA-512), and one for authorization (simple). So we'll need to both enable "simple" authorization on the broker, and choose/enable one of the authorization mechanisms on the listener. It's actually really easy! You can just follow the docs [here](https://strimzi.io/docs/master/#assembly-kafka-authentication-and-authorization-deployment-configuration-kafka). So now my brokers expect me to authenticate? Great... How do I do that? Well, you'll need to create and apply a "KafkaUser" definition. In said "KafkaUser" definition, you'll specify both the authentication type (which should match the authentication type that you've chosen for your listener), and the authorization roles. The authorization roles are what gives permissions to a specific resource. So, for instance, whether or not a user has access to read or write to a topic. They are detailed in the docs [here](https://strimzi.io/docs/master/#simple_authorization_2). Once you've applied your "KafkaUser" definition to OpenShift, the "[User Operator](https://strimzi.io/docs/master/#assembly-user-operator-str)" will detect it and generate some resources for you automagically. Neat! But what does it generate? Well that depends on the authentication type you've chosen. Let's start with SCRAM since it's the easiest. 

__SCRAM-SHA-512__

When you create a "KafkaUser" with a "scram-sha-512" authentication type, the "User Operator" will generate an OpenShift "Secret" with the same name. So, if I've defined a user named "bob", I will see a secret named "bob" that has a key called "password". Those are the credentials that I'll use to connect. Seems straighforward... But how do I get that into my app? The easiest way is to inject the password into an environment variable in your container. You can do this in your `deployment.yml` file for your app. Here's an example:

```yaml deployment.yml
#...
- name: KAFKA_USER_PASSWORD
  valueFrom:
    secretKeyRef:
      name: bob
      key: password
#...
```

If I include the above snippet into my `deployment.yml`, I will have a `KAFKA_USER_PASSWORD` environment variable (containing my generated password) available to me inside of my container. So I can just reference it in my `application.yml` file like below.

```yaml application.yml
#...
camel:
  component:
    kafka.configuration:
      security-protocol: SASL_PLAINTEXT
      sasl-mechanism: SCRAM-SHA-512
      sasl-jaas-config: org.apache.kafka.common.security.scram.ScramLoginModule required username="bob" password="${KAFKA_USER_PASSWORD}";
#...
```

Now, when my Camel app connects to Kafka, it will connect as my specified user, using the auto-generated password. And, if desired, I can have OpenShift trigger a reload of my app if the password is updated/regenerated. Cool beans! On to a slightly more complex case...

__TLS__

When using "tls" as my authentication type, I (as a client) will need both the broker's public key, as well as my user's public & private keys. Such is the nature of mutual auth... Similarly though, those will be generated for me by the various operators. But it's a little more complicated than the SCRAM case. How so? Well, just like before, I can inject the secret values into environment variables as shown below. So no issues yet.

```yaml deployment.yml
#...
- name: KAFKA_CLUSTER_CRT
  valueFrom:
    secretKeyRef:
      name: my-cluster-cluster-ca-cert
      key: ca.crt
- name: KAFKA_USER_CRT
  valueFrom:
    secretKeyRef:
      name: alice
      key: user.crt
- name: KAFKA_USER_KEY
  valueFrom:
    secretKeyRef:
      name: alice
      key: user.key
#...
```

But I can't just use those key/cert values directly. I actually need to create a keystore & truststore, and then import those keys into their appropriate stores. Well, how do I go about that? One solution would be to use a custom container image and start script as per [Jakub's example](https://github.com/strimzi/client-examples). While this solution is very clever (as is Jakub :)), I wondered if there was a way to do it all within my app code. A quick Google search and, lo and behold, I find [env-keystore](https://github.com/heroku/env-keystore)! This handy little library will let me easily create Java keystores using arbitrary keys/certs from string values. So now I can actually use those injected environment variable values. What's more, it can write the stores out to a file once I've created them. And since it uses [Bouncy Castle](https://www.bouncycastle.org/), it can handle both Java formatted keys/certs as well as OpenSSL formatted ones (which is what Strimzi will generate for you). Good stuff! So if I include the below dependency...

```xml pom.xml
<dependency>
  <groupId>com.heroku.sdk</groupId>
  <artifactId>env-keystore</artifactId>
  <version>x.x.x</version>
</dependency>
```

And add a little bit of initialization code...

```java KafkaComponentCustomizer.java
package org.apache.camel.examples;

//import ...

@Component 
public class KafkaComponentCustomizer implements ComponentCustomizer<KafkaComponent> {

  @Autowired private KafkaComponentConfiguration kafkaConfiguration;
  @Value("#{systemEnvironment['KAFKA_CLUSTER_CRT']}") private String kafkaClusterCrt;
  @Value("#{systemEnvironment['KAFKA_USER_KEY']}") private String kafkaUserKey;
  @Value("#{systemEnvironment['KAFKA_USER_CRT']}") private String kafkaUserCrt;

  @Override
  public void customize(KafkaComponent component) {
    try {
        BasicKeyStore truststore = new BasicKeyStore(kafkaClusterCrt, kafkaConfiguration.getConfiguration().getSslTruststorePassword());
        truststore.store(Paths.get(kafkaConfiguration.getConfiguration().getSslTruststoreLocation()));

        BasicKeyStore keystore = new BasicKeyStore(kafkaUserKey, kafkaUserCrt, kafkaConfiguration.getConfiguration().getSslKeystorePassword());
        keystore.store(Paths.get(kafkaConfiguration.getConfiguration().getSslKeystoreLocation()));
    } catch (IOException | GeneralSecurityException e) {
      throw new RuntimeException(e);
    }
  }
}
```

My app can now grab those generated keys/certs from the OpenShift secrets, inject them into env variables, use those values to generate the client keystore/trustore, then reference those stores when it makes its connection to the brokers. More difficult than SCRAM, but still not too bad. Although...

__Syncing the secrets__

When the various operators generate secrets for you, they will do so in the namespace where the resources live. That's all fine and good if my apps are colocated alongside my Kafka cluster. But what if I want to put my cluster in one namespace (let's say one called "strimzi"), and my client apps in another (let's say it's called "fuse")? Unfortunately, OpenShift currently won't allow me to access secrets between namespaces. So I'd have to copy the values from the generated secrets in my "strimzi" namespace into manually created secrets in my "fuse" namespace where my client apps live. That sounds super error prone. What do I do if those secrets update/regenerate? Certificates do expire right? I'll have to make sure that when they update, I'll go update all the copies. That's a recipe for disaster! If only there was a way to syncronize those secrets between namespaces automatically. A little bit of searching and you'll likely stumble across [AppsCode Kubed](https://github.com/appscode/kubed). 

Kubed is an operator that, once installed, will monitor and sync any secrets you specify (among other things). It will keep them updated if the source secret changes, and even remove the copies if the source secret is deleted. And it can do this across as many namespaces as you need. So no need to worry if you have multiple apps in multiple namespaces. Perfect right!? Well, one minor hitch...

The way Kubed works, you have to apply an OpenShift annotation to the secret that you want sync'd. It will then find and monitor any secrets with said annotation, and then sync them to namespaces that have the appropriate label. Unfortunately, if the Strimzi operators see that the secret has been changed in any way (like say, adding an annotation), they will squash those changes and get things back in sync with their configs. Which really __is__ what they should do in that case... There's currently a JIRA issue open for adding the ability to specify annotations on the generated secrets via the Strimzi configs. And once that enhancement is made, Kubed will definitely be the way to go. But what can we do in the meantime?

It's not great, but one option is to use a "CronJob" that will simply execute some bash commands to sync the secrets. Clunky? Yes. Ideal? No. Works? Yes. Here's an example one that I wrote to sync the broker's public key. I installed it into the "fuse" namespace, and it sync'd my secret from the "strimzi" namespace, as needed, every second.

```yaml secret-sync-cronjob.yml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: camel-kafka-authz-secret-sync
spec:
  schedule: "* * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: camel-kafka-authz-sa
          restartPolicy: Never
          containers:
          - name: cluster-ca-cert-sync
            image: openshift/origin-cli
            command: ["bash",  "-c", "export SRC=\"$(oc extract -n strimzi secrets/my-cluster-cluster-ca-cert --keys=ca.crt --to=-)\"; export DST=\"$(oc extract -n fuse secrets/my-cluster-cluster-ca-cert --keys=ca.crt --to=-)\"; if [ -n \"$SRC\" ] && [ \"$DST\" != \"$SRC\" ]; then echo 'Values differ. Syncing...'; oc create secret generic -n fuse --dry-run -o yaml my-cluster-cluster-ca-cert --from-literal=\"ca.crt=$SRC\" | oc apply -f -; fi;"]
```

So once again, we've managed to solve all the worlds problems. Or maybe none of them... :) Either way, if you're looking for more than just snippets, take a look at the full source code for this example: https://github.com/joshdreagan/camel-kafka-authz.
