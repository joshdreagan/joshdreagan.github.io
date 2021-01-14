---
title: Camel CXFRS Contract First
tags:
  - fuse
  - camel
  - cxf
cover: /2018/03/02/camel_cxfrs_contract_first/post-bg.jpg
thumbnail: /2018/03/02/camel_cxfrs_contract_first/post-bg.jpg
date: 2018-03-02 00:00:51
---


I've recently had a rash of customers who want to do contract driven REST development. That is, they want to define the contract for their service up front, and then generate their interfaces and models. And like all subjects, if I get asked about it enough times, I'll write it down in a blog. So here goes...<!-- more -->

First, why would someone want to generate code from their contract? Well... we certainly don't want to have to keep code and contract in sync manually. That would be extremely error prone (and would repeat the same mistakes we made in the early days of SOAP/WSDL). So that leaves two options: Either we generate the contract from the code (ie, code-first), or we generate the code from the contract (ie, contract-first). If we choose to go the code-first route, what we usually do is add some annotations (or add additional metadata via some language/library specific means), deploy the code to a running server, and then pull the contract by hitting (HTTP GET'ing) a special url. This seems all fine and good since it gives us a contract that is exposed for clients to consume. What more do we need right? As it turns out though, many organizations don't do development this way. In fact, many of them will want to develop their contract well before implementation takes place. Often times, that contract is even developed by a separate team. So in these cases, we need to go contract-first. Which means that we have to figure out a way to generate the code...

Now that we're through all of that "intro" business, let's talk about how one might actually accomplish this. There are many competing specs for defining REST contracts (ie, WADL, RAML, API Blueprint, ...), but the most popular (and the one I'll be targeting for this post) seems to be Swagger/OpenAPI. If you check out [Swagger's website](https://swagger.io/swagger-codegen/), you'll see that they've implemented some tooling to do this code generation. What's more, the tooling is actually pretty pluggable and can be used to generate implementation/model files for [many different languages](https://generator.swagger.io/api/gen/servers). It doesn't have the best documentation, and can be a bit buggy at times. But luckily it's open source. So we can just read through [the code](https://github.com/swagger-api/swagger-codegen) to figure out how it works. And after all, code is really the best documentation... ;) Here's a snippet of what the plugin configuration might look like:

```xml
<plugin>
  <groupId>io.swagger</groupId>
  <artifactId>swagger-codegen-maven-plugin</artifactId>
  <version>${swagger-codegen-maven-plugin.version}</version>
  <executions>
    <execution>
      <id>generate-sources</id>
      <goals>
        <goal>generate</goal>
      </goals>
      <configuration>
        <inputSpec>src/main/swagger/api.json</inputSpec>
        <language>jaxrs-cxf</language>
        <generateSupportingFiles>false</generateSupportingFiles>
        <modelPackage>org.apache.camel.examples.model</modelPackage>
        <apiPackage>org.apache.camel.examples.api</apiPackage>
        <output>${project.build.directory}/generated-sources</output>
        <generateApiTests>false</generateApiTests>
        <configOptions>
          <sourceFolder>swagger</sourceFolder>
          <implFolder>swagger</implFolder>
        </configOptions>
      </configuration>
    </execution>
  </executions>
</plugin>
```

Once we have the plugin properly configured, it will bind to the `generate-sources` lifecycle phase of our Maven build. Which means that the code it outputs will be on our classpath, and all we need to do is create our implementation. Here's a snippet of how you might do just that using [Apache Camel](http://camel.apache.org/) (the most awesome framework available!):

```Java
from("cxfrs:/camel?resourceClasses=org.apache.camel.examples.api.MyServiceApi")
  .routingSlip(simple("direct:${headers[operationName]}"))
;

from("direct:myServiceOperation")
  .log("This is where you should fill in your method implementations...")
;
```

If you want to see what a full project looks like, I've created an example that uses the above tooling to generate the JAXRS/CXF annotated Java interfaces/model files from a real API JSON file that I downloaded from Swagger's website. It then uses those generated files to create an implementation in Camel, and runs it on a [Spring Boot](https://projects.spring.io/spring-boot/) container. Give it a look here: [https://github.com/joshdreagan/camel-swagger-contract-first](https://github.com/joshdreagan/camel-swagger-contract-first).

So now that we know how to do contract-first REST development (and even have an example), we can just copy/paste any time we want to start a new project. But can we do better? I think so... As it turns out, Maven has a baked-in mechanism for creating templates to generate new projects. They're called Maven Archetypes. Usually, you just create an archetype project as defined by [the docs](https://maven.apache.org/archetype/maven-archetype-plugin/). The archetype project contains a fairly simple descriptor XML, and a bunch of [Velocity](http://velocity.apache.org/) templates for each of the files you want to be included in your generated project. This allows you to do simple property expansion (ie, `groupId`, `artifactId`, `package`, ...), conditionals, and even some looping inside of your template files. Once you build/install the archetype, you can crank out new projects by simply executing a command like this:

```
$ mvn archetype:generate -DarchetypeCatalog=local \
                         -DarchetypeGroupId=org.apache.camel.archetypes \
                         -DarchetypeArtifactId=my-custom-archetype \
                         -DarchetypeVersion=1.0.0-SNAPSHOT \
                         -DgroupId=com.mycompany \
                         -DartifactId=myproject \
                         -Dversion=1.0.0-SNAPSHOT
```

However, if you looked at the example project I gave you above, you'd notice that some of the files need values that are parsed from the Swagger API doc. So Velocity templates alone won't cut it... We could just leave those values blank (or put in some sort of placeholder), and then have the user fill them in after the project generation. But once again, we can do better... There is a little known (and horribly under-documented) feature that we can take advantage of to automate things. If you look under the ["Advanced Usage"](https://maven.apache.org/archetype/maven-archetype-plugin/advanced-usage.html) section of the archetype plugin docs, you will see a reference to a "Post-generation script" (toward the bottom of the page). Basically, you can just include a [Groovy](http://groovy-lang.org/) script named `archetype-post-generate.groovy` inside the `src/main/resources/META-INF/` folder, and its contents will be executed during project generation (after the Velocity templates have been applied, and the files copied into their destinations). And because Groovy is super awesome and incredibly powerful, we can use it do pretty much anything we want. For instance, parsing up a JSON file and using its values to replace extra placeholders in our project files... Neat! Here's an example archetype project to get you started: [https://github.com/joshdreagan/camel-archetype-spring-boot-cxfrs-contract-first](https://github.com/joshdreagan/camel-archetype-spring-boot-cxfrs-contract-first). Enjoy!
