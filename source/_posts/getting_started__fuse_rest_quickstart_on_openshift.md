---
title: Getting Started - Fuse REST Quickstart On OpenShift
cover: /2015/05/07/getting_started__fuse_rest_quickstart_on_openshift/post-bg.png
thumbnail: /2015/05/07/getting_started__fuse_rest_quickstart_on_openshift/post-bg.png
date: 2015-05-07 00:00:00
tags:
- fuse
- openshift
- fabric8
- karaf
- camel
- cxf
---

This guide demonstrates step-by-step how to create a Fuse cluster and deploy one of the quickstart applications using Red Hat's OpenShift Online infrastructure.
<!-- more -->

The first thing you'll do is open up a browser and navigate to [OpenShift Online](https://openshift.redhat.com/). If you don't already have an account, you can sign up for a free one. However, the free account is restricted to 3 _small_ gears which will not run JBoss Fuse instances. So you will need to upgrade to a paid account to complete this guide.

# Step 1: Create A Domain/Namespace

After you've created an OpenShift Online account and logged in, you will end up at the landing page.

![Landing Page](landing_page.png)

Before creating any apps, you'll need to create a domain/namespace. If you already have a domain/namespace, and you'd like to re-use it, you can skip this step. This is a unique identifier that allows you to group applications. Click on the "__Settings__" tab at the top of the page.

![Settings Button](setting_button.png)

Enter your desired domain/namespace and click the "__Create__" button.

![Create Domain Button](create_domain.png)

Now you're ready to create your JBoss Fuse cluster.

# Step 2: Create A JBoss Fuse Cluster

To create a JBoss Fuse cluster, first navigate to the applications page by clicking the "__Applications__" button at the top of the screen.

![Applications Button](applications_button.png)

If you don't already have any applications, you will see a link to "__Create your first application now__" like in the picture below.

![Create Application Link](create_application_link.png)

Click the link and it will begin the "__Create Application__" workflow. On the first page, you will see a list of available _cartridges_.

![Create Application](create_application-01.png)

You'll want to find and click on the "__JBoss Fuse 6.1__" cartridge pictured below. If you can't find it, you can use the search box.

![Create Application](create_application-02.png)

Next, you'll configure the cartridge.

![Create Application](create_application-03.png)

Fill in the "__Public URL__" field with the desired application name. You'll notice that it automatically appends your domain/namespace that you created in __Step 1__.

![Create Application](create_application-04.png)

Select the desired "__Gear Size__". You must use at least a _medium_ sized gear to run the JBoss Fuse cartridge.

![Create Application](create_application-05.png)

When you're satisfied with your settings, click the "__Create Application__" button. Be patient, as it might take a minute or so to finish. __Do not navigate away (or refresh) the page!__

![Create Application](create_application-06.png)

Once your application has been successfully created, you will be presented with an informational screen like the one shown below. __Make sure to capture this information as you will need it later and there is no way to get it once you've left this page!__

![Create Application](create_application-07.png)

# Step 3: Provision A Child Container

Now that you have a JBoss Fuse cluster, you can provision a child container with the actual quickstart deployed. Click on the "__Applications__" tab at the top of the page, and you should see your newly created JBoss Fuse application.

![Application List](applications_list.png)

Click on that application, and it should open up the JBoss Fuse Management Console. You will need to login using the credentials that were displayed for you at the end of __Step 2__.

![Fuse Console](fuse_console_login.png)

After you login, you should see the JBoss Fuse Management Console's landing page.

![Fuse Console](fuse_console_landing_page.png)

Click on the "__Runtime__" tab to view the list of servers that are currently part of this cluster.

![Fuse Console](fuse_console_runtime_button.png)

At the moment, there should be only 1 and it is the Fabric Registry server (indicated by the little cloud icon).

![Fuse Console](fuse_console_container_list.png)

Click on the "__Create__" button toward the top right of the page to begin the "__Create Container__" workflow.

![Fuse Console](fuse_console_create_container_button.png)

Fill in the desired "__Container Name__" and "__Gear Size__" making sure to use at least a _medium_ or larger. You can enter your OpenShift Online credentials and click the "__Login to OpenShift__" button to have it fill in the "__OpenShift Domain__" field. Select the "__rest__" profile under the "__Example / Quickstarts__" folder.

![Fuse Console](fuse_console_create_container-01.png)

When done, click the "__Create And Start Container__" button. Be patient, as it might take a minute or so to finish. __Do not navigate away (or refresh) the page!__

![Fuse Console](fuse_console_create_container-02.png)

Once the container has been created, you will be redirected back to the "__Containers__" page. You will notice that there are now 2. One is the Fabric Registry server, and one is hosting your application.

![Fuse Console](fuse_console_create_container-03.png)

Click on the "__APIs__" tab to view the base URI of the REST service you've just deployed. It will be in the "__Location__" column.

![Fuse Console](fuse_console_apis.png)

# Step 4: Test The Application

Open a browser and navigate to the URI that you discovered in __Step 3__, appending `/customerservice/customers/123`. If successful, you should see some XML text come back.

If all went well, you've now got a running JBoss Fuse cluster with the REST quickstart deployed to a child instance.
