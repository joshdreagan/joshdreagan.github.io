---
title: Getting Started - First Fuse Project
cover: /2015/04/09/getting_started__first_fuse_project/post-bg.png
thumbnail: /2015/04/09/getting_started__first_fuse_project/post-bg.png
date: 2015-04-09 00:00:00
tags:
- fuse
- karaf
- camel
---

This guide demonstrates step-by-step how to create a sample Camel project in JBoss Developer Studio (JBDS) using the Maven archetypes that come out of the box. Specifically, we’ll be creating a Spring based file poller project. We'll then deploy the project to a JBoss Fuse server and verify that it is running properly.
<!-- more -->

Here are a few assumptions before you begin:

- You already have a supported JDK installed (ie, Oracle's JDK version 6 or 7)
- You already have Maven version 3.x installed
- You already have JDBS installed (tested with version 7.1.1, but should work with newer)
- You already have the "JBoss Integration and SOA Development" tools for JBDS installed
- You already have JBoss Fuse version 6.1 installed

After you've met all the prerequisites, you can proceed with the guide.

# Step 1: Create A New Fuse Project

Start off by creating a new project. Click __File->New->Project...__ to start the "New Project" wizard.

![New Project](new_project-01.png)

Select "__Fuse Project__" as the project type and click "__Next >__".

![New Project](new_project-02.png)

Specify your preferred project location (or let it use the default) and click "__Next >__".

![New Project](new_project-03.png)

Select the appropriate Maven archetype from the list. The one we use for this example is "__org.apache.camel.archetypes:camel-archetype-spring:2.12.0.redhat-610379__". Enter your preferred _Group Id_, _Artifact Id_, _Version_, & _Package_. When done, click "__Finish__".

![New Project](new_project-04.png)

Your new project should be created. If this is the first time, it can take a few minutes as JBDS downloads all the required Maven dependencies.

# Step 2: Run The Project Locally

To run the project locally, right click on the Camel Context and select __Run As->Local Camel Context (without tests)__.

![Run Local](run_local-01.png)

This will run the project using the Camel Maven Plugin which uses a simple embedded, containerless runtime. The project contains some sample files in the "`src/data`" directory that will be picked up immediately once the Camel Context has successfully started. You should see output similar to the one below in the "__Console__" tab.

![Run Local](run_local-02.png)

When you're satisfied that everything is running properly, you can shut down the local runtime by clicking the clicking the <img src="{% asset_path "run_local-03.png" %}" style="display: inline"/> icon in the "__Console__" tab.


# Step 3: Deploy Project To JBoss Fuse

If you don't have one already, create a new server configuration in JBDS. Click on the "__Servers__" tab. If you don't have any server configurations, you will see a link like the image below. Click on the link to start the "__New Server__" wizard.

![Create New Server](create_new_server-01.png)

Select "__JBoss Fuse 6.1 Server__" and click "__Next >__".

![Create New Server](create_new_server-02.png)

Click the "__Browse__" button and navigate to the root directory of your JBoss Fuse installation. Click "__Next >__" when done.

![Create New Server](create_new_server-03.png)

Enter the appropriate details/credentials for your JBoss Fuse server and click "__Finish__".

![Create New Server](create_new_server-04.png)

You should now have a new server configuration as shown below.

![Create New Server](create_new_server-05.png)

Next, start the server. You can do so by clicking the <img src="{% asset_path "start_server-01.png" %}" style="display: inline"/> icon as shown below.

![Start Server](start_server-02.png)

You _may_ get a warning about SSH keys like the one shown below. If so, just click "__Yes__" to accept the new key.

![Start Server](start_server-03.png)

If the server starts successfully, you should see the JBoss Fuse shell shown below.

![Start Server](start_server-04.png)

Next we'll build the project and deploy it to the running server. To build the project, right-click at the project root and select __Run As->Maven install__.

![Build Project](build_project-01.png)

You will see the output of the Maven build in the "__Console__" tab. If there were no errors, you should the a __BUILD SUCCESS__ message.

![Build Project](build_project-02.png)

Now select the "__Shell__" tab again to switch back to the JBoss Fuse console. Enter the following command to deploy your project (make sure to substitute the Maven _Group Id_, _Artifact Id_, & _Version_ for the ones you specified in __Step 1__):

{% codeblock %}
osgi:install -s mvn:com.mycompany/spring-camel/1.0.0-SNAPSHOT
{% endcodeblock %}

Your bundle should install and give you a _Bundle ID_. This _Bundle ID_ is the identifier you will use to interact with that bundle in the future (ie, to uninstall, stop, start, ...).

![Deploy Project](deploy_project-01.png)

You can also type the following command to verify that there are no failures:

{% codeblock %}
osgi:list
{% endcodeblock %}

You should see your newly installed bundle as shown below.

![Deploy Project](deploy_project-02.png)

Finally, open up a file browser (or use a terminal) and copy the sample files from "`camel-spring/src/data`" to the "`src/data`" directory inside the JBoss Fuse installation folder. Now, switch back to the "__Shell__" tab and type the following command in the JBoss Fuse console:

{% codeblock %}
log:display
{% endcodeblock %}

You should see the log statements printed toward the bottom of the log.

![Deploy Project](deploy_project-03.png)

At this point, you have successfully created and deployed a Spring based file poller project to JBoss Fuse.
