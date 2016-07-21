---
title: Getting Started - Fuse REST Quickstart On OpenShift
banner: post-bg.png
permalink: getting_started__fuse_rest_quickstart_on_openshift
date: 2015-05-07 00:00:00
tags:
- fuse
- openshift
- fabric8
- karaf
- camel
- cxf
- jboss
---


This guide demonstrates step-by-step how to create a Fuse cluster and deploy one of the quickstart applications using Red Hat's OpenShift Online infrastructure.<!-- more -->

The first thing you'll do is open up a browser and navigate to [OpenShift Online](https://openshift.redhat.com/). If you don't already have an account, you can sign up for a free one. However, the free account is restricted to 3 _small_ gears which will not run JBoss Fuse instances. So you will need to upgrade to a paid account to complete this guide.

# Step 1: Create A Domain/Namespace

After you've created an OpenShift Online account and logged in, you will end up at the landing page.

{% asset_img "landing_page.png" [Landing Page] %}

Before creating any apps, you'll need to create a domain/namespace. If you already have a domain/namespace, and you'd like to re-use it, you can skip this step. This is a unique identifier that allows you to group applications. Click on the "__Settings__" tab at the top of the page.

{% asset_img "setting_button.png" [Settings Button] %}

Enter your desired domain/namespace and click the "__Create__" button.

{% asset_img "create_domain.png" [Create Domain Button] %}

Now you're ready to create your JBoss Fuse cluster.

# Step 2: Create A JBoss Fuse Cluster

To create a JBoss Fuse cluster, first navigate to the applications page by clicking the "__Applications__" button at the top of the screen.

{% asset_img "applications_button.png" [Applications Button] %}

If you don't already have any applications, you will see a link to "__Create your first application now__" like in the picture below.

{% asset_img "create_application_link.png" [Create Application Link] %}

Click the link and it will begin the "__Create Application__" workflow. On the first page, you will see a list of available _cartridges_.

{% asset_img "create_application-01.png" [Create Application] %}

You'll want to find and click on the "__JBoss Fuse 6.1__" cartridge pictured below. If you can't find it, you can use the search box.

{% asset_img "create_application-02.png" [Create Application] %}

Next, you'll configure the cartridge.

{% asset_img "create_application-03.png" [Create Application] %}

Fill in the "__Public URL__" field with the desired application name. You'll notice that it automatically appends your domain/namespace that you created in __Step 1__.

{% asset_img "create_application-04.png" [Create Application] %}

Select the desired "__Gear Size__". You must use at least a _medium_ sized gear to run the JBoss Fuse cartridge.

{% asset_img "create_application-05.png" [Create Application] %}

When you're satisfied with your settings, click the "__Create Application__" button. Be patient, as it might take a minute or so to finish. __Do not navigate away (or refresh) the page!__

{% asset_img "create_application-06.png" [Create Application] %}

Once your application has been successfully created, you will be presented with an informational screen like the one shown below. __Make sure to capture this information as you will need it later and there is no way to get it once you've left this page!__

{% asset_img "create_application-07.png" [Create Application] %}

# Step 3: Provision A Child Container

Now that you have a JBoss Fuse cluster, you can provision a child container with the actual quickstart deployed. Click on the "__Applications__" tab at the top of the page, and you should see your newly created JBoss Fuse application.

{% asset_img "applications_list.png" [Application List] %}

Click on that application, and it should open up the JBoss Fuse Management Console. You will need to login using the credentials that were displayed for you at the end of __Step 2__.

{% asset_img "fuse_console_login.png" [Fuse Console] %}

After you login, you should see the JBoss Fuse Management Console's landing page.

{% asset_img "fuse_console_landing_page.png" [Fuse Console] %}

Click on the "__Runtime__" tab to view the list of servers that are currently part of this cluster.

{% asset_img "fuse_console_runtime_button.png" [Fuse Console] %}

At the moment, there should be only 1 and it is the Fabric Registry server (indicated by the little cloud icon).

{% asset_img "fuse_console_container_list.png" [Fuse Console] %}

Click on the "__Create__" button toward the top right of the page to begin the "__Create Container__" workflow.

{% asset_img "fuse_console_create_container_button.png" [Fuse Console] %}

Fill in the desired "__Container Name__" and "__Gear Size__" making sure to use at least a _medium_ or larger. You can enter your OpenShift Online credentials and click the "__Login to OpenShift__" button to have it fill in the "__OpenShift Domain__" field. Select the "__rest__" profile under the "__Example / Quickstarts__" folder.

{% asset_img "fuse_console_create_container-01.png" [Fuse Console] %}

When done, click the "__Create And Start Container__" button. Be patient, as it might take a minute or so to finish. __Do not navigate away (or refresh) the page!__

{% asset_img "fuse_console_create_container-02.png" [Fuse Console] %}

Once the container has been created, you will be redirected back to the "__Containers__" page. You will notice that there are now 2. One is the Fabric Registry server, and one is hosting your application.

{% asset_img "fuse_console_create_container-03.png" [Fuse Console] %}

Click on the "__APIs__" tab to view the base URI of the REST service you've just deployed. It will be in the "__Location__" column.

{% asset_img "fuse_console_apis.png" [Fuse Console] %}

# Step 4: Test The Application

Open a browser and navigate to the URI that you discovered in __Step 3__, appending `/customerservice/customers/123` like in the picture below. If successful, you should see some XML text come back.

{% asset_img "application_test.png" [Application Test] %}

If all went well, you've now got a running JBoss Fuse cluster with the REST quickstart deployed to a child instance.
