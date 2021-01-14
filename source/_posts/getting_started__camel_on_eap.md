---
title: Getting Started - Camel On EAP
cover: /2015/04/10/getting_started__camel_on_eap/post-bg.png
thumbnail: /2015/04/10/getting_started__camel_on_eap/post-bg.png
date: 2015-04-10 00:00:00
tags:
- fuse
- jboss
- wildfly
- camel
- cxf
---

This guide demonstrates step-by-step how to create a sample Camel project in JBoss Developer Studio (JBDS) using the Maven archetypes that come out of the box. Specifically, we’ll be creating a contract-first based web services java project. We'll then convert that project to a WAR file so that it can be deployed to JBoss EAP. Finally, we'll deploy the project and verify that it is running.
<!-- more -->

Here are a few assumptions before you begin:

- You already have a supported JDK installed (ie, Oracle's JDK version 6 or 7)
- You already have Maven version 3.x installed
- You already have JDBS installed (tested with version 7.1.1, but should work with newer)
- You already have the "JBoss Integration and SOA Development" tools for JBDS installed
- You already have JBoss EAP installed (tested with version 6.3.0, but should work with others)

After you've met all the prerequisites, you can proceed with the guide.

# Step 1: Create A New Fuse Project

Start off by creating a new project. Click __File->New->Project...__ to start the "New Project" wizard.

![New Project](new_project-01.png)

Select "__Fuse Project__" as the project type and click "__Next >__".

![New Project](new_project-02.png)

Specify your preferred project location (or let it use the default) and click "__Next >__".

![New Project](new_project-03.png)

Select the appropriate Maven archetype from the list. The one we use for this example is "__io.fabric8:camel-cxf-contract-first-arch:1.0.0.redhat-379__". Enter your preferred _Group Id_, _Artifact Id_, _Version_, & _Package_. When done, click "__Finish__".

![New Project](new_project-04.png)

Your new project should be created. If this is the first time, it can take a few minutes as JBDS downloads all the required Maven dependencies.

# Step 2: Convert Project To WAR Format

First we need to change the packaging type in the Maven POM so that we will build a WAR file instead of the default JAR file. To do so, open up the project's __pom.xml__ file, click on the drop-down list next to "__Packaging:__", and select "__war__" as the type. Save your changes when done.

![Convert To WAR](convert_to_war-01.png)

Next, we need to tell the Eclipse Maven Plugin to update the project configuration so that JBDS will add the appropriate facets and display the project structure correctly. So first, right-click on the project and select "__Maven->Update Project...__" to start the "Update Maven Project" wizard.

![Maven Update Project](maven_update_project-01.png)

Make sure the project is selected and click "__OK__".

![Maven Update Project](maven_update_project-02.png)

You should notice that the project structure changes a bit.

Now we will need to add some Maven dependencies so that we can bootstrap our Camel Context in a Servlet container. Open up the project's __pom.xml__ file, and select the "__Dependencies__" tab.

![Add Dependencies](add_dependencies-01.png)

Click on the "__Add...__" button to start the "Select Dependency" wizard.

![Add Dependencies](add_dependencies-02.png)

Fill out the input boxes as shown below and click "__OK__" when done. _Don't forget to save your changes._

![Add Dependencies](add_dependencies-03.png)

Once we have all of our dependencies in place, we can create our deployment descriptor (ie, the __web.xml__ file). So right-click on the "__Deployment Descriptor:__" section in the project view and select "__Generate Deployment Descriptor Stub__". This will create a skeleton __web.xml__ file that you can use as a starting point.

![Create DD](create_dd-01.png)

Now double-click on the "__Deployment Descriptor:__" section in the project view to edit the __web.xml__ file. We will need to add several elements.

- A `context-param` element to tell Spring where the XML files are located.
- A `listener` element to load Spring's ServletContextListener which will bind the Spring Context to the Servlet's lifecycle.
- A `servlet` element to name the CXF Servlet.
- A `servlet-mapping` element to map the CXF Servlet to a URL pattern.

If you'd like, you can just copy/paste the following XML:

{% codeblock lang:xml %}
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee" xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd" version="3.0">
  <display-name>camel-cxf-contract-first</display-name>
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:META-INF/spring/*.xml</param-value>
  </context-param>
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
  <servlet>
    <servlet-name>CXFServlet</servlet-name>
    <servlet-class>org.apache.cxf.transport.servlet.CXFServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>CXFServlet</servlet-name>
    <url-pattern>/soap/*</url-pattern>
  </servlet-mapping>
</web-app>
{% endcodeblock %}

Save your changes and close the __web.xml__ file.

By default, the generated CXF configuration will boot up an embedded Jetty server and bind to "_http://localhost:9000/order/_". Since we will be deploying to a server that already has a Servlet container, we can let CXF use the provided one instead. So first, find the __camel-cxf.xml__ file in the "__Project Explorer__" tab and open it up.

![Change Bind Address](change_bind_address-01.png)

We simply need to change the `address` attribute of the `cxf:cxfEndpoint` element from "_http://localhost:9000/order/_" to "_/order/_".

![Change Bind Address](change_bind_address-02.png)

The result should look like the image below. Save your changes and close the file.

![Change Bind Address](change_bind_address-03.png)

Finally, you may have noticed that the "__Problems__" tab is showing some errors. This is because the Maven archetype that we used has a WSDL that contains 2 minor issues.

![Fix WSDL](fix_wsdl-01.png)

To fix the issues, we'll need to edit the WSDL file. To do so, either double-click on the error in the "__Problems__" tab, or on the WSDL in the "__Project Explorer__" tab to open the __order.wsdl__ file. Now locate the `wsdl:binding` element. It should be toward the bottom of the file. It should contain `wsdl:input` and `wsdl:output` elements. Both of which have a `soap:body` element with a `parts` attribute as shown below.

![Fix WSDL](fix_wsdl-02.png)

Simply remove the `parts` attribute. The resulting file should look like the image below. Save your changes and close the file.

![Fix WSDL](fix_wsdl-03.png)


That's it for the changes. The project should now be ready to deploy to JBoss EAP.

# Step 3: Deploy Project To JBoss EAP

If you don't have one already, create a new server configuration in JBDS. Click on the "__Servers__" tab. If you don't have any server configurations, you will see a link like the image below. Click on the link to start the "__New Server__" wizard.

![Create New Server](create_new_server-01.png)

Select "__JBoss Enterprise Application Platform 6.1+__" and click "__Next >__".

![Create New Server](create_new_server-02.png)

Click the "__Browse__" button and navigate to the root directory of your JBoss EAP installation. Click "__Next >__" when done.

![Create New Server](create_new_server-03.png)

Select your desired options, or leave the defaults and click "__Next >__".

![Create New Server](create_new_server-04.png)

Select the project from the "__Available:__" pane and click the "__Add__" button.

![Create New Server](create_new_server-05.png)

You should see the project move over to the "__Configured:__" pane as shown below.

![Create New Server](create_new_server-06.png)

Click the "__Finish__" button when done. You should now have a new server configuration as shown below.

![Create New Server](create_new_server-07.png)

Next, start the server. You can do so by clicking the <img src="{% asset_path "start_server-01.png" %}" style="display: inline"/> icon as shown below.

![Start Server](start_server-02.png)

If the server starts successfully, you should see no errors in your "__Console__" tab.

![Start Server](start_server-03.png)

Finally, open up a web browser (or use the one built into JBDS by clicking the <img src="{% asset_path "verify_deployment-01.png" %}" style="display: inline"/> icon) and navigate to "_http://localhost:8080/camel-cxf-contract-first/soap/order?wsdl_". If the everything was successful, you should see the WSDL contents displayed as shown in the image below.

![Verify Deployment](verify_deployment-02.png)

At this point, you have successfully created and deployed a Camel/CXF web service to JBoss EAP. You can do further testing using a tool like SoapUI (or any SOAP service testing tool).
