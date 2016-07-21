---
title: Getting Started - First Fuse Project
banner: post-bg.png
permalink: getting_started__first_fuse_project
date: 2015-04-09 00:00:00
tags:
- fuse
- karaf
- camel
- jboss
---


This guide demonstrates step-by-step how to create a sample Camel project in JBoss Developer Studio (JBDS) using the Maven archetypes that come out of the box. Specifically, we’ll be creating a Spring based file poller project. We'll then deploy the project to a JBoss Fuse server and verify that it is running properly. <!-- more -->

Here are a few assumptions before you begin:

- You already have a supported JDK installed (ie, Oracle's JDK version 6 or 7)
- You already have Maven version 3.x installed
- You already have JDBS installed (tested with version 7.1.1, but should work with newer)
- You already have the "JBoss Integration and SOA Development" tools for JBDS installed
- You already have JBoss Fuse version 6.1 installed

After you've met all the prerequisites, you can proceed with the guide.

# Step 1: Create A New Fuse Project

Start off by creating a new project. Click __File->New->Project...__ to start the "New Project" wizard.

{% asset_img "new_project-01.png" [New Project] %}

Select "__Fuse Project__" as the project type and click "__Next >__".

{% asset_img "new_project-02.png" [New Project] %}

Specify your preferred project location (or let it use the default) and click "__Next >__".

{% asset_img "new_project-0.png" [New Project] %}

Select the appropriate Maven archetype from the list. The one we use for this example is "__org.apache.camel.archetypes:camel-archetype-spring:2.12.0.redhat-610379__". Enter your preferred _Group Id_, _Artifact Id_, _Version_, & _Package_. When done, click "__Finish__". 

{% asset_img "new_project-04.png" [New Project] %}

Your new project should be created. If this is the first time, it can take a few minutes as JBDS downloads all the required Maven dependencies.

# Step 2: Run The Project Locally

To run the project locally, right click on the Camel Context and select __Run As->Local Camel Context (without tests)__.

{% asset_img "run_local-01.png" [Run Local] %}

This will run the project using the Camel Maven Plugin which uses a simple embedded, containerless runtime. The project contains some sample files in the "`src/data`" directory that will be picked up immediately once the Camel Context has successfully started. You should see output similar to the one below in the "__Console__" tab.

{% asset_img "run_local-02.png" [Run Local] %}

When you're satisfied that everything is running properly, you can shut down the local runtime by clicking the clicking the <img src="{% asset_path "run_local-03.png" %}" style="display: inline"/> icon in the "__Console__" tab.


# Step 3: Deploy Project To JBoss Fuse

If you don't have one already, create a new server configuration in JBDS. Click on the "__Servers__" tab. If you don't have any server configurations, you will see a link like the image below. Click on the link to start the "__New Server__" wizard.

{% asset_img "create_new_server-01.png" [Create New Server] %}

Select "__JBoss Fuse 6.1 Server__" and click "__Next >__".

{% asset_img "create_new_server-02.png" [Create New Server] %}

Click the "__Browse__" button and navigate to the root directory of your JBoss Fuse installation. Click "__Next >__" when done.

{% asset_img "$2" [$1] %}

Enter the appropriate details/credentials for your JBoss Fuse server and click "__Finish__". 

{% asset_img "create_new_server-04.png" [Create New Server] %}

You should now have a new server configuration as shown below. 

{% asset_img "create_new_server-05.png" [Create New Server] %}

Next, start the server. You can do so by clicking the <img src="{% asset_path "start_server-01.png" %}" style="display: inline"/> icon as shown below.

{% asset_img "start_server-02.png" [Start Server] %}

You _may_ get a warning about SSH keys like the one shown below. If so, just click "__Yes__" to accept the new key.

{% asset_img "start_server-03.png" [Start Server] %}

If the server starts successfully, you should see the JBoss Fuse shell shown below.

{% asset_img "start_server-04.png" [Start Server] %}

Next we'll build the project and deploy it to the running server. To build the project, right-click at the project root and select __Run As->Maven install__.

{% asset_img "build_project-01.png" [Build Project] %}

You will see the output of the Maven build in the "__Console__" tab. If there were no errors, you should the a __BUILD SUCCESS__ message.

{% asset_img "build_project-02.png" [Build Project] %}

Now select the "__Shell__" tab again to switch back to the JBoss Fuse console. Enter the following command to deploy your project (make sure to substitute the Maven _Group Id_, _Artifact Id_, & _Version_ for the ones you specified in __Step 1__):

{% codeblock %}
osgi:install -s mvn:com.mycompany/spring-camel/1.0.0-SNAPSHOT
{% endcodeblock %}

Your bundle should install and give you a _Bundle ID_. This _Bundle ID_ is the identifier you will use to interact with that bundle in the future (ie, to uninstall, stop, start, ...).

{% asset_img "deploy_project-01.png" [Deploy Project] %}

You can also type the following command to verify that there are no failures:

{% codeblock %}
osgi:list
{% endcodeblock %}

You should see your newly installed bundle as shown below.

{% asset_img "deploy_project-02.png" [Deploy Project] %}

Finally, open up a file browser (or use a terminal) and copy the sample files from "`camel-spring/src/data`" to the "`src/data`" directory inside the JBoss Fuse installation folder. Now, switch back to the "__Shell__" tab and type the following command in the JBoss Fuse console:

{% codeblock %}
log:display
{% endcodeblock %}

You should see the log statements printed toward the bottom of the log.

{% asset_img "deploy_project-03.png" [Deploy Project] %}

At this point, you have successfully created and deployed a Spring based file poller project to JBoss Fuse.
