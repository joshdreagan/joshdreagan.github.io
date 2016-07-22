---
title: Custom Camel LoadBalancer With Infinispan
banner: post-bg.png
permalink: custom_camel_loadbalancer_with_infinispan
date: 2015-12-04 00:00:00
tags:
- camel
- infinispan
- wildfly
---

Apache Camel is a pretty full-featured EIP implementation framework. It has several existing strategies for load-balancing right out of the box. Round Robin, Random, Sticky, Weighted Round Robin, Weighted Random,... the list goes on and on. But being that it's a very well written and pluggable framework, it also gives you the ability to drop in your own custom strategies should you find that none of the existing ones meet your specific needs. So for this post, I created a custom Camel Load Balancer implementation utilizing an Infinispan cache to dynamically discover and load-balance between destination endpoints.
<!-- more -->

> The sample code for this post can be found at [https://github.com/joshdreagan/infinispan-discovery](https://github.com/joshdreagan/infinispan-discovery)

## Why?

Why would I do such a thing? Well... there's a few good reasons.

First, all of the existing load-balancer strategies work on a static list. So if I know all of my endpoints ahead of time, no problem. I just code them into my Camel route. But what if my list of endpoints changes between environments? Maybe I could use properties. Well... only if the number of endpoints is static. Which brings me to the next reason...

All of the existing load-balancer strategies are configured at startup. So what do I do if my list changes dynamically at runtime? Let's say that I want to do service discovery and load-balance between the currently active/registered backend services. If you're familiar with Camel, you might be thinking "Why not just use the Camel Fabric Component? It does dynamic load-balancing and service discovery. Problem solved right?". If all of my services are running in containers that are managed by Fabric8, that is a viable solution. But what if I want to discover some endpoints that are running on JBoss EAP instances. Or what if I'm not running a Fabric8 ensemble at all?

Finally, the most important reason... Because I can. :)

## Implementation Details

Creating a custom Camel Load Balancer implementation is fairly straight forward. You just create a class and implement the `LoadBalancer` interface. There's even a base class (`LoadBalancerSupport`) that you can extend that will take care of some of the boilerplate coding for you. You then just fill in the details of how it picks the next endpoint from its internal list. Pretty simple right?

In my case, however, I'm not actually coming up with my own strategy for how to pick endpoints. I'm really just augmenting some existing strategy with a dynamic list of endpoints. So to be more specific, I'm not interested in implementing my own flavor of the Random, Round Robin, Sticky, ... strategies. No need to reinvent that wheel. Instead, I just want to decorate those existing strategies and provide them with some additional capabilities. So I use the decorator pattern. That allows me to ignore all the tom-foolery of the load-balancing itself and concentrate on the portion that I really want. The dynamic service discovery.

Here's my custom load-balancer class (or at least the important parts):

{% codeblock lang:java %}
package org.apache.camel.processor.loadbalancer;

// Import statements removed for brevity.

public class JCacheLoadBalancer extends ServiceSupport implements LoadBalancer, CamelContextAware, InitializingBean {

  private final Logger log = LoggerFactory.getLogger(JCacheLoadBalancer.class);

  private static final LoadBalancer DEFAULT_DELEGATE = new RoundRobinLoadBalancer();

  private Cache<String, Set<String>> registry;
  private String groupId;
  private LoadBalancer delegate;
  private UriPreProcessor uriPreProcessor;
  private Boolean throwExceptionIfEmpty;

  private Map<String, Processor> processorMap;
  private CacheEntryListenerConfiguration<String, Set<String>> registryListenerConfiguration;

  private CamelContext camelContext;

  // Getters and setters removed for brevity.

  @Override
  public void addProcessor(Processor processor) {
    delegate.addProcessor(processor);
  }

  @Override
  public void removeProcessor(Processor processor) {
    delegate.removeProcessor(processor);
  }

  @Override
  public List<Processor> getProcessors() {
    return delegate.getProcessors();
  }

  @Override
  public boolean process(Exchange exchange, AsyncCallback callback) {
    if ((getProcessors() == null || getProcessors().isEmpty()) && throwExceptionIfEmpty) {
      if (throwExceptionIfEmpty) {
        exchange.setException(new LoadBalancerUnavailableException(String.format("No URIs found for service '%s'.", groupId)));
      }
      callback.done(true);
      return true;
    } else {
      return delegate.process(exchange, callback);
    }
  }

  @Override
  public void process(Exchange exchange) throws Exception {
    if ((getProcessors() == null || getProcessors().isEmpty()) && throwExceptionIfEmpty) {
      throw new LoadBalancerUnavailableException(String.format("No URIs found for service '%s'.", groupId));
    }
    delegate.process(exchange);
  }

  @Override
  protected void doStart() throws Exception {
    if (delegate == DEFAULT_DELEGATE) {
      ((Service) delegate).start();
    }
    registry.registerCacheEntryListener(registryListenerConfiguration);
    processUris(registry.get(groupId));
  }

  @Override
  protected void doStop() throws Exception {
    registry.deregisterCacheEntryListener(registryListenerConfiguration);
    if (delegate == DEFAULT_DELEGATE) {
      ((Service) delegate).stop();
    }
  }

  @Override
  public void afterPropertiesSet() throws Exception {
    Objects.requireNonNull(registry, "The registry property must not be null.");
    Objects.requireNonNull(groupId, "The groupId property must not be null.");
    Objects.requireNonNull(camelContext, "The camelContext property must not be null.");

    if (delegate == null) {
      delegate = DEFAULT_DELEGATE;
    }

    if (throwExceptionIfEmpty == null) {
      throwExceptionIfEmpty = true;
    }

    processorMap = new HashMap<>();
    registryListenerConfiguration = new MutableCacheEntryListenerConfiguration<>(new LookupCacheListenerFactory(), null, false, false);
  }

  private void processUris(Set<String> uris) throws Exception {
    if (uris == null) {
      uris = new HashSet<>();
    }
    for (String uri : processorMap.keySet()) {
      if (!uris.contains(uri)) {
        log.info(String.format("Removing uri: %s", uri));
        Processor processor = processorMap.remove(uri);
        removeProcessor(processor);
        if (processor instanceof Producer) {
          camelContext.removeEndpoint(((Producer) processor).getEndpoint());
        }
      }
    }
    for (String uri : uris) {
      if (!processorMap.containsKey(uri)) {
        log.info(String.format("Adding uri: %s", uri));
        String processedUri = uri;
        if (uriPreProcessor != null) {
          processedUri = uriPreProcessor.process(uri);
        }
        Processor processor = camelContext.getEndpoint(processedUri).createProducer();
        processorMap.put(uri, processor);
        addProcessor(processor);
      }
    }
  }

  private class LookupCacheListener implements CacheEntryCreatedListener<String, Set<String>>,
          CacheEntryUpdatedListener<String, Set<String>>,
          CacheEntryRemovedListener<String, Set<String>>,
          CacheEntryExpiredListener<String, Set<String>>,
          Serializable {

    // All listener methods removed for brevity. They just delegate to the onEvent method anyway.

    public void onEvent(Iterable<CacheEntryEvent<? extends String, ? extends Set<String>>> events) throws CacheEntryListenerException {
      for (CacheEntryEvent<? extends String, ? extends Set<String>> event : events) {
        log.debug(String.format("Got a cache event: %s", event));
        try {
          processUris(event.getValue());
        } catch (Exception e) {
          throw new CacheEntryListenerException(e);
        }
      }
    }
  }

  private class LookupCacheListenerFactory implements Factory<LookupCacheListener> {

    @Override
    public LookupCacheListener create() {
      return new LookupCacheListener();
    }
  }
}
{% endcodeblock %}

You can see that it's just delegating most of the methods (ie, the `addProcessor`, `getProcessor`, and `removeProcessor` methods) to whatever existing implementation that it's decorating. The actual methods that do the load-balancing (ie, the `process` methods) do a little bit of work, but end up just delegating as well. So I didn't actually have to do any algorithm work and I still get to use all the existing strategies. Pretty neat!

In addition to a delegate `LoadBalancer` implementation, this class expects that you will give it a fully-configured jCache instance. In my example, I used Infinispan. But I could have just as easily used any other spec compliant implementation. Here's my Infinispan configuration:

{% codeblock lang:xml %}
<?xml version="1.0" encoding="UTF-8"?>
<infinispan xmlns="...">

  <jgroups>
    <stack-file name="external-file" path="default-configs/default-jgroups-tcp.xml"/>
  </jgroups>
  <cache-container default-cache="default">
    <local-cache name="registry-cache"/>
    <transport cluster="registry-cluster" stack="external-file"/>
    <replicated-cache name="registry-cache" mode="SYNC" />
  </cache-container>

</infinispan>
{% endcodeblock %}

Now let's get to the part that's actually doing some work. The `LookupCacheListener` class just implements the various `CacheListener` interfaces from the jCache API. If it gets any events on the cache entry containing our endpoints, it simply updates the delegate's internal list of processors. So as services come and go they can register their URIs in the cache, our listener will be notified, and our list of available load-balancer endpoints will be updated.

The final piece to discuss for this load-balancer implementation is the `UriPreProcessor`. This is an interface that I created to allow an implementation to customize the URI in some way before adding it to the list. The idea is that other services that are registering themselves might not know that they're going to be invoked from a Camel endpoint. So they likely won't add options like `bridgeEndpoint=true` to the URI. An implementation of this interface would allow you to add such options on their behalf. Here's the interface itself:

{% codeblock lang:java %}
package org.apache.camel.processor.loadbalancer;

public interface UriPreProcessor {

  public String process(String uri) throws Exception;
}
{% endcodeblock %}

And here's a sample implementation that adds the options:

{% codeblock lang:java %}
package org.jboss.poc.greeter.camel;

// Import statements removed for brevity.

public class GreeterServiceUriPreProcessor implements UriPreProcessor {

  @Override
  public String process(String uri) throws Exception {
    Map<String, Object> options = new HashMap<>();
    options.put("bridgeEndpoint", true);
    return URISupport.appendParametersToURI(uri, options);
  }
}
{% endcodeblock %}

Now all that's left is to actually use it in my Camel routes. To do so, I declare it just like any other bean. Then I use the `custom` element in my `loadBalance` to `ref` it. Looks something like this:

{% codeblock lang:xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="...">

  <bean id="cachingProvider"
        class="javax.cache.Caching" factory-method="getCachingProvider"/>

  <bean id="cacheManager"
        factory-bean="cachingProvider" factory-method="getCacheManager">
    <constructor-arg value="META-INF/infinispan/infinispan-clustered.xml"/>
    <constructor-arg>
      <bean class="org.springframework.util.ClassUtils"
            factory-method="getDefaultClassLoader"/>
    </constructor-arg>
  </bean>

  <bean id="registryCache"
        factory-bean="cacheManager" factory-method="getCache">
    <constructor-arg value="registry-cache"/>
  </bean>

  <bean id="jCacheLoadBalancer"
        class="org.apache.camel.processor.loadbalancer.JCacheLoadBalancer"
        init-method="start" destroy-method="stop">
    <property name="registry" ref="registryCache"/>
    <property name="groupId"
              value="/services/soap-http/{http://poc.jboss.org/greeter}GreeterService"/>
    <property name="delegate" ref="randomLoadBalancer"/>
    <property name="uriPreProcessor" ref="greeterServiceUriPreProcessor"/>
  </bean>

  <bean id="greeterServiceUriPreProcessor"
        class="org.jboss.poc.greeter.camel.GreeterServiceUriPreProcessor"/>

  <bean id="randomLoadBalancer"
        class="org.apache.camel.processor.loadbalancer.RandomLoadBalancer"/>

  <camelContext id="greeterGateway"
                trace="false" xmlns="http://camel.apache.org/schema/spring">

    <route id="proxyRoute">
      <from uri="jetty:http://localhost:9000/gateway/GreeterService?matchOnUriPrefix=true&amp;bridgeEndpoint=true"/>
      <onException>
        <exception>org.apache.camel.processor.loadbalancer.LoadBalancerUnavailableException</exception>
        <handled>
          <constant>true</constant>
        </handled>
        <setHeader headerName="CamelHttpResponseCode">
          <constant>404</constant>
        </setHeader>
        <setBody>
          <constant>NOT FOUND</constant>
        </setBody>
      </onException>
      <loadBalance>
        <custom ref="jCacheLoadBalancer"/>
      </loadBalance>
    </route>

  </camelContext>

</beans>
{% endcodeblock %}

That's it for the Camel side of things. Now let's discuss how to get some services registered.

In my example, I just created a simple JAX-WS service in JBoss WildFly. Here's the code so you can see how simple it is:

{% codeblock lang:java %}
package org.jboss.poc.greeter.impl;

// Import statements removed for brevity.

@WebService(endpointInterface = "org.jboss.poc.greeter.Greeter",
            serviceName = "GreeterService",
            portName = "GreeterServicePort")
public class EnglishGreeter implements Greeter {

  @Override
  public String getGreeting(String name) {
    String greeting = null;
    try {
      greeting = String.format("Hello %s from %s!", name, InetAddress.getLocalHost().getHostAddress());
    } catch (Exception e) {
      greeting = String.format("Hello %s! I was unable to find my IP :(...", name);
    }
    return greeting;
  }
}
{% endcodeblock %}

For this service, I created a `ServletContextListener` to register/unregister it's URI to/from the jCache.

{% codeblock lang:java %}
package org.jboss.poc.greeter.impl;

// Import statements removed for brevity.

@WebListener()
public class GreeterServiceRegistrar implements ServletContextListener {

  private static final String SERVICE_NAMESPACE = "http://poc.jboss.org/greeter";
  private static final String SERVICE_NAME = "GreeterService";

  private static final String REGISTRY_CACHE_NAME = "registry-cache";

  private final Logger log = LoggerFactory.getLogger(GreeterServiceRegistrar.class);

  @Inject
  private DefaultCacheManager cacheManager;

  @Override
  public void contextInitialized(ServletContextEvent sce) {
    final String key = getServiceKey();
    final String uri = getServiceURI(sce);

    if (cacheManager != null) {
      log.info(String.format("Registering service: {'%s': '%s'}", key, uri));

      Cache<String, Set<String>> cache = cacheManager.getCache(REGISTRY_CACHE_NAME);
      Set<String> uris = cache.getOrDefault(key, new HashSet<String>());
      uris.add(uri);
      cache.put(key, uris);
    } else {
      log.warn(String.format("Unable to register service: {'%s': '%s'}. CacheManager is null.", key, uri));
    }
  }

  @Override
  public void contextDestroyed(ServletContextEvent sce) {
    final String key = getServiceKey();
    final String uri = getServiceURI(sce);

    if (cacheManager != null) {
      log.info(String.format("Unregistering service: {'%s': '%s'}", key, uri));
      Cache<String, Set<String>> cache = cacheManager.getCache(REGISTRY_CACHE_NAME);
      Set<String> uris = cache.getOrDefault(key, new HashSet<String>());
      uris.remove(uri);
      cache.put(key, uris);
    } else {
      log.warn(String.format("Unable to unregister service: {'%s': '%s'}. CacheManager is null.", key, uri));
    }
  }

  private String getServiceKey() {
    // Contents removed for brevity. Method forms and returns the cache key where we'll store our URIs.
  }

  private String getServiceURI(ServletContextEvent sce) {
    // Contents removed for brevity. Method grabs the current IP and port that the server is bound to and forms a URI.
  }
}
{% endcodeblock %}

So now when my `ServletContext` is started, my JAX-WS service URI will be registered. And when it is stopped, my URI will be removed. Since I configured my Infinispan cache the same on both the JBoss WildFly and Camel sides, the local cache instances are connected and will receive events and updates.

That's it! If you want to give it a go, check out the full source code at [https://github.com/joshdreagan/infinispan-discovery](https://github.com/joshdreagan/infinispan-discovery). Hopefully it's useful...
