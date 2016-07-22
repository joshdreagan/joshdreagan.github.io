---
title: Fuse Fabric Offline CI/CD
banner: post-bg.png
permalink: fuse_fabric_offline_ci__cd
date: 2015-08-04 00:00:00
tags:
- fuse
- karaf
- camel
- cxf
- fabric8
- jboss
---

This is a follow-on to the [excellent blog post](http://blog.christianposta.com/fabric8/fuse-fabric-profile-migration-for-continuous-delivery/) that my colleague Christian Posta wrote on doing CI/CD with Fuse Fabric. In particular, this post will extend upon his write-up by showing how to migrate Fabric8 profiles and Maven artifacts into environments which are disconnected from the development environment.
<!-- more -->

> The sample code for this blog can be found at [https://github.com/joshdreagan/fuse-fabric-promotion-build](https://github.com/joshdreagan/fuse-fabric-promotion-build). This is just a forked version of [Christian's original sample code](/home/jreagan/Development/Projects/joshdreagan/joshdreagan.github.io/_posts/2015-08-04-fuse_fabric_offline_ci__cd.markdown) with the changes outlined below.

Most of the steps and information found in [Christian's original blog post](http://blog.christianposta.com/fabric8/fuse-fabric-profile-migration-for-continuous-delivery/) are still relevant. In fact, all of the steps leading up to the "__Environment build__" and it's use of the Fabric8 Maven Plugin's "_branch_" goal are exactly the same.

## The Problem

The issue with the "_branch_" goal is that it assumes connectivity to a running Fabric8 ensemble so that it can directly branch it's internal Git repo and upload the profile zips. In addition, the Fabric8 machines will also need connectivity back to the Maven repository (ie, Nexus or Artifactory) where the artifacts are stored so that it can pull them all down and install them once the profile is applied to a container. This is not always feasible due to the fact that some companies will restrict connectivity between environments.

For many companies, it is very common that there will be restricted access between development and production environments due to various security constraints. Furthermore, some production environments (ie, defense and classified networks) will have absolutely no access whatsoever to the development environment or even to the internet. Therefore, it will often be necessary to package and transfer all the necessary Fabric8 profile zips and Maven artifacts via a completely offline (and usually read-only) media like a CD-R. This will of course limit the amount of automation that you can achieve, but can easily be accomplished with a few extra steps.

## The Solution

The simplest solution is to build a zip file containing everything we need so that we will require no outside access during installation. This can easily be accomplished using the Maven Assembly Plugin. Once we have the zip file, we can transfer it to the target environments, unzip it, import our Fabric8 profiles, and proceed with our installation. Since all the artifacts are included, installation/provisioning should proceed as smoothly as if we were connected to the internet.

## Modify the Environment build

The first step is to modify the "__Environment build__" by adding a Maven Assembly Plugin configuration.

Add the following Maven Assembly Plugin configuration to your project:

{% codeblock lang:xml %}
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-assembly-plugin</artifactId>
  <configuration>
    <attach>false</attach>
    <descriptors>
      <descriptor>src/assembly/repository.xml</descriptor>
    </descriptors>
  </configuration>
</plugin>
{% endcodeblock %}

Make sure to set the "_attach_" configuration to `false`. Failure to do so could result in the zip file (which could be quite large) being inadvertently uploaded to your Maven repository. Next, you'll need to define the assembly descriptor as follows:

{% codeblock lang:xml %}
<assembly xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3 http://maven.apache.org/xsd/assembly-1.1.3.xsd">
  <id>repository</id>
  <formats>
    <format>zip</format>
  </formats>
  <includeBaseDirectory>false</includeBaseDirectory>
  <repositories>
    <repository>
      <includeMetadata>true</includeMetadata>
    </repository>
  </repositories>
</assembly>
{% endcodeblock %}

You can optionally add an "_execution_" for the Maven Assembly Plugin and bind it to a lifecycle. I prefer to invoke it directly as follows:

{% codeblock %}
mvn assembly:single
{% endcodeblock %}

The output should be a zip file containing an offline Maven repository with all of the Fabric8 profile zips as well as the required Maven dependencies. You can now download/transfer that zip file via whatever means you have available to your target environment.

Once you've transferred the zip file, you simply unzip it into the "~/.m2/repository" folder on each of the Fabric ensemble machines. __Make sure to place it in the user home of the user that runs the Fuse instances.__ You can then import the Fabric8 profile zips with the following Karaf CLI commands:

{% codeblock %}
fabric:profile-import --version 1.1 mvn:com.redhat.demo.promotion/eip/2.0/zip/profile
fabric:profile-import --version 1.1 mvn:com.redhat.demo.promotion/rest/1.5/zip/profile
{% endcodeblock %}

Since the profile zips now exist in the "~/.m2/repository" folder, they will be discovered and installed by the Fabric Maven Proxy without needing to go online.

Once you apply the profiles to a container (or upgrade a container to the new version), the provisioned container instances will reach back to the Fabric ensemble to retrieve the necessary Maven artifacts. Similarly to the profile zips, they will be discovered and installed by the Fabric Maven Proxy from its "~/.m2/repository" folder and will be transferred over without ever needing to go online.
