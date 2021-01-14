---
title: Camel Aggregation Strategies
tags:
  - fuse
  - camel
cover: /2018/08/30/camel_aggregation_strategies/post-bg.jpg
thumbnail: /2018/08/30/camel_aggregation_strategies/post-bg.jpg
date: 2018-08-30 22:43:34
---


One of the many (many many many) extension points inside [Apache Camel](http://camel.apache.org/) is the `org.apache.camel.processor.aggregate.AggregationStrategy`. These are used in everything from [Content Enrichers](http://camel.apache.org/content-enricher.html) to [Splitters](http://camel.apache.org/splitter.html) to [Aggregators](http://camel.apache.org/aggregator2.html) and more. Since their use is so prevalent, I figured that I'd dedicate a whole blog post just for them. So here goes...<!-- more -->

So what are AggregationStrategy's anyway? Simple... they're implementations of the `org.apache.camel.processor.aggregate.AggregationStrategy` that allow you to specify exactly how two exchanges will be merged. This specification can be as simple or as complex as you require for your use case. Maybe you just want to take the first response and ignore all others. Maybe you want to combine the XML bodies into a list and then merge a select few headers. The limit really is your imagination. But what do I mean by "merging exchanges"? Let's take a look at a few concrete examples.

## Out of the Box

For starters, there are several implementations that are included out of the box. You can use them "as-is" without writing any custom code at all. Let's talk through a few of them with some potential use cases.

The first is the `org.apache.camel.processor.aggregate.UseLatestAggregationStrategy` implementation. It's the default strategy for most Camel EIPs that accept aggregation strategies. So if you don't specify any strategy, this is likely the one you're using. Basically, it takes the last exchange it receives and just uses that (ignoring any others that may have been aggregated prior). One example use case for this would be when doing an [Aggregator](http://camel.apache.org/aggregator2.html). Perhaps you're receiving many messages as input, but you want to buffer them (giving the user time to send in corrections/updates), and then only send the latest message to the backend after some period of inactivity. That might look like below:

{% codeblock lang:xml %}
<bean id="useLatest" class="org.apache.camel.processor.aggregate.UseLatestAggregationStrategy"/>
<camelContext xmlns="http://activemq.apache.org/camel/schema/spring">
  <route>
    <from uri="direct:acceptUpdateableRequest"/>
    <aggregator strategyRef="useLatest" completionTimeout="5000">
      <correlationExpression>
        <header>UniqueRequestID</header>
      </correlationExpression>
      <to uri="direct:bufferedSendToBackend"/>
    </aggregator>
  </route>
</camelContext>
{% endcodeblock %}

For the next use case, we'll cover the (very similar) `org.apache.camel.processor.aggregate.UseOriginalAggregationStrategy` implementation. As the name would suggest, it "merges" two exchanges together by completely ignoring the new exchange and just taking the original. One example of where this might be useful is when doing a [Multicast](http://camel.apache.org/multicast.html). Lets say I wanted to send a copy of a message off to multiple recipients, but really don't care about their response. After the multicast is completed, I want to perform some transformation on the original message, and then return the result. Instead of rolling my own implementation, I could simply use the one provided. Something like this:

{% codeblock lang:xml %}
<bean id="useOriginal" class="org.apache.camel.processor.aggregate.UseOriginalAggregationStrategy"/>
<camelContext xmlns="http://activemq.apache.org/camel/schema/spring">
  <route>
    <from uri="direct:acceptRequest"/>
    <multicast strategyRef="useOriginal">
      <to uri="direct:recipient1"/>
      <to uri="direct:recipient2"/>
    </multicast>
    <to uri="xslt:transformOriginal.xsl"/>
  </route>
</camelContext>
{% endcodeblock %}

The next set of implementations, I'll cover as a group. They are the `org.apache.camel.processor.aggregate.GroupedExchangeAggregationStrategy`, `org.apache.camel.processor.aggregate.GroupedMessageAggregationStrategy`, and `org.apache.camel.processor.aggregate.GroupedBodyAggregationStrategy` strategies. They will combine the exchanges into a list and then pass the list itself along to the next processor. They only differ by what they put in the list (ie, `List<Exchange>`, `List<Message>`, or `List<Object>`). So, for instance, if you wanted to split a message, process each individual part, and then combine the individual results back into a list, you could do so easily using a [Splitter](http://camel.apache.org/splitter.html) like below:

{% codeblock lang:xml %}
<bean id="listOfBody" class="org.apache.camel.processor.aggregate.GroupedBodyAggregationStrategy"/>
<camelContext xmlns="http://activemq.apache.org/camel/schema/spring">
  <route>
    <from uri="direct:acceptListRequestExpectingListResponse"/>
    <split strategyRef="listOfBody">
      <simple>${body}</simple>
      <to uri="direct:sendIndividualRequest"/>
    </split>
  </route>
</camelContext>
{% endcodeblock %}

The final implementation that I'll cover for this section is the `org.apache.camel.util.toolbox.XsltAggregationStrategy`. It allows you to provide an XSLT that will be used to merge the original and new exchanges together. A great use case for this is when you want to [Enrich](http://camel.apache.org/content-enricher.html) an XML request with some extra data retrieved from a backend.

{% codeblock lang:xml %}
<bean id="xsltEnrichmentStrategy" class="org.apache.camel.util.toolbox.XsltAggregationStrategy">
  <constructor-arg value="/META-INF/xslt/EnrichIndexHtml.xsl"/>
</bean>
<camelContext xmlns="http://activemq.apache.org/camel/schema/spring">
  <route>
    <from uri="direct:acceptRequest"/>
    <to uri="language:constant:classpath:/META-INF/html/index.html"/>
    <enrich strategyRef="xsltEnrichmentStrategy">
      <constant>direct:fetchCds</constant>
    </enrich>
  </route>
</camelContext>
{% endcodeblock %}

Since this example is a little more complex, it requires more than just a code snippet to explain. So I've put together an example application and thrown it up on GitHub. Take a look... [https://github.com/joshdreagan/camel-xslt-enricher](https://github.com/joshdreagan/camel-xslt-enricher)

It's amazing how many use cases these "canned" aggregation strategies cover. But what if I they're not quite exactly what you need?

## Semi-Custom

In this section, we'll discuss what I call "semi-custom" strategies. Basically, they're base/utility classes that make it easy for you to implement a custom strategy with very little Java code.

The first class we'll talk about is the `org.apache.camel.processor.aggregate.AbstractListAggregationStrategy`. Similar to the grouping implementations mentioned above, the end result of this strategy is a list of items. The difference is that you have total control over what data gets placed in said list as well as where you pull it from. Here's a very simple example implementation:

{% codeblock lang:java %}
package org.apache.camel.examples;

import org.apache.camel.Exchange;
import org.apache.camel.processor.aggregate.AbstractListAggregationStrategy;

public class SimpleListAggregationStrategy extends AbstractListAggregationStrategy<String> {

  @Override
  public String getValue(Exchange exchange) {
    return exchange.getIn().getHeader("MyAwesomeHeader", String.class);
  }
}
{% endcodeblock %}

If you need even more control over the aggregation, you can use the `org.apache.camel.util.toolbox.FlexibleAggregationStrategy`. The FlexibleAggregationStrategy is a fluent strategy builder that lets you define fairly complex aggregation strategy implementations using a very concise syntax. If you're using the Java DSL to define your Camel routes (or are using any Java based bean wiring mechanism), you can just use the fluent builder directly. However, if you're using it from the Spring DSL (using Spring's XML bean definitions) it might be easier to wrapper it in a simple Java implementation. See below for an example:

{% codeblock lang:java %}
package org.apache.camel.examples;

import org.apache.camel.Exchange;
import org.apache.camel.model.language.SimpleExpression;
import org.apache.camel.processor.aggregate.AggregationStrategy;
import org.apache.camel.util.toolbox.AggregationStrategies;

public class CorrelationIdAggregationStrategy implements AggregationStrategy {

  private final AggregationStrategy delegate;

  public FluentAggregationStrategy() {
    delegate = AggregationStrategies.flexible()
            .storeInHeader("MyCorrelationID")
            .pick(new SimpleExpression("${body}"))
    ;
  }

  @Override
  public Exchange aggregate(Exchange oldExchange, Exchange newExchange) {
    return delegate.aggregate(oldExchange, newExchange);
  }
}
{% endcodeblock %}

You could then use your implementation like this:

{% codeblock lang:xml %}
<bean id="uuidEnrichmentStrategy" class="org.apache.camel.examples.CorrelationIdAggregationStrategy"/>
<camelContext xmlns="http://activemq.apache.org/camel/schema/spring">
  <route>
    <from uri="direct:acceptRequest"/>
    <enrich strategyRef="uuidEnrichmentStrategy">
      <constant>direct:fetchUuid</constant>
    </enrich>
  </route>
  <route>
    <from uri="direct:fetchUuid"/>
    <bean beanType="java.util.UUID" method="randomUUID"/>
    <convertBodyTo type="java.lang.String"/>
  </route>
</camelContext>
{% endcodeblock %}

Pretty powerful stuff! But what if you're feeling even more imaginative?

## Custom

The last type of strategy that I'll talk about is a "completely custom" implementation. This basically just means that you will implement the `org.apache.camel.processor.aggregate.AggregationStrategy` interface directly without using any helper base classes (which might restrict you in some ways). Because of this direct implementation, you are free to do literally anything you want.

One example that I whipped up for a customer a while back is what I called the "semi-streaming aggregation strategy".

{% codeblock lang:java %}
package org.apache.camel.examples;

import java.util.Comparator;
import java.util.Objects;
import java.util.SortedSet;
import java.util.TreeSet;
import org.apache.camel.CamelContext;
import org.apache.camel.CamelContextAware;
import org.apache.camel.Exchange;
import org.apache.camel.Message;
import org.apache.camel.Processor;
import org.apache.camel.RuntimeCamelException;
import org.apache.camel.processor.aggregate.AggregateProcessor;
import org.apache.camel.processor.aggregate.AggregationStrategy;
import org.apache.camel.util.ExchangeHelper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.InitializingBean;

public class SemiStreamingAggregationStrategy implements AggregationStrategy, CamelContextAware, InitializingBean {

  private static final Logger log = LoggerFactory.getLogger(SemiStreamingAggregationStrategy.class);

  public static final String LAST_PROCESSED_INDEX = "CamelAggregatorLastProcessedIndex";

  private String aggregateProcessorId;
  private CamelContext camelContext;
  private String sequenceIdHeaderName;

  // Lazily initialized.
  private AggregateProcessor _aggregateProcessor;
  private Comparator<Message> _messageComparator;

  public String getAggregateProcessorId() {
    return aggregateProcessorId;
  }

  public void setAggregateProcessorId(String aggregateProcessorId) {
    this.aggregateProcessorId = aggregateProcessorId;
  }

  @Override
  public void setCamelContext(CamelContext camelContext) {
    this.camelContext = camelContext;
  }

  @Override
  public CamelContext getCamelContext() {
    return camelContext;
  }

  public String getSequenceIdHeaderName() {
    return sequenceIdHeaderName;
  }

  public void setSequenceIdHeaderName(String sequenceIdHeaderName) {
    this.sequenceIdHeaderName = sequenceIdHeaderName;
  }

  protected AggregateProcessor _aggregateProcessor() {
    if (_aggregateProcessor == null) {
      _aggregateProcessor = camelContext.getProcessor(aggregateProcessorId, AggregateProcessor.class);
    }
    return _aggregateProcessor;
  }

  protected Comparator<Message> _messageComparator() {
    if (_messageComparator == null) {
      _messageComparator = (Message t, Message t1) -> t.getHeader(sequenceIdHeaderName, Comparable.class).compareTo(t1.getHeader(sequenceIdHeaderName, Comparable.class));
    }
    return _messageComparator;
  }

  @Override
  public void afterPropertiesSet() throws Exception {
    Objects.requireNonNull(aggregateProcessorId, "The aggregateProcessorId property must not be null.");
    Objects.requireNonNull(camelContext, "The camelContext property must not be null.");
    Objects.requireNonNull(sequenceIdHeaderName, "The sequenceIdHeaderName property must not be null.");
  }

  @Override
  public Exchange aggregate(Exchange oldExchange, Exchange newExchange) {

    Exchange aggregateExchange = initializeAggregateExchange(oldExchange, newExchange);
    log.info(String.format("Pending messages: [%s] messages", aggregateExchange.getIn().getBody(SortedSet.class).size()));

    appendMessage(aggregateExchange, newExchange.getIn());

    findAndEmitSequencedMessages(aggregateExchange);

    return aggregateExchange;
  }

  protected Exchange initializeAggregateExchange(Exchange oldExchange, Exchange newExchange) {

    Exchange aggregateExchange;
    if (oldExchange == null) {
      aggregateExchange = ExchangeHelper.copyExchangeAndSetCamelContext(newExchange, camelContext);
      SortedSet<Message> pendingMessages = new TreeSet<>(_messageComparator());
      aggregateExchange.getIn().setBody(pendingMessages);
      aggregateExchange.setProperty(LAST_PROCESSED_INDEX, -1L);
    } else {
      aggregateExchange = oldExchange;
    }

    return aggregateExchange;
  }

  protected void appendMessage(Exchange aggregateExchange, Message message) {
    log.info(String.format("Adding message: index [%s], body [%s]", message.getHeader(sequenceIdHeaderName), message.getBody()));
    aggregateExchange.getIn().getBody(SortedSet.class).add(message);
  }

  protected void findAndEmitSequencedMessages(Exchange aggregateExchange) {

    SortedSet<Message> pendingMessages = aggregateExchange.getIn().getBody(SortedSet.class);
    Long lastProcessedIndex = aggregateExchange.getProperty(LAST_PROCESSED_INDEX, Long.class);

    Message currentMessage;
    Long currentMessageIndex;
    SortedSet<Message> messagesToBeEmitted = new TreeSet<>(_messageComparator());
    do {
      currentMessage = pendingMessages.first();
      currentMessageIndex = currentMessage.getHeader(sequenceIdHeaderName, Long.class);
      if (currentMessageIndex == lastProcessedIndex + 1) {
        messagesToBeEmitted.add(currentMessage);
        pendingMessages.remove(currentMessage);
        lastProcessedIndex = currentMessageIndex;
      } else {
        break;
      }
    } while (!pendingMessages.isEmpty());
    if (!messagesToBeEmitted.isEmpty()) {
      log.info(String.format("Messages to be emitted: [%s] messages", messagesToBeEmitted.size()));
      aggregateExchange.setProperty(LAST_PROCESSED_INDEX, lastProcessedIndex);
      aggregateExchange.getIn().setBody(pendingMessages);
      Exchange exchangeToBeEmitted = ExchangeHelper.copyExchangeAndSetCamelContext(aggregateExchange, camelContext);
      exchangeToBeEmitted.getIn().setBody(messagesToBeEmitted);
      try {
        for (Processor processor : _aggregateProcessor().next()) {
          processor.process(exchangeToBeEmitted);
        }
      } catch (Exception e) {
        throw new RuntimeCamelException(e);
      }
    }
  }
}
{% endcodeblock %}

Here's a link to the full source for your perusal: [[https://github.com/joshdreagan/camel-streaming-aggregation](https://github.com/joshdreagan/camel-streaming-aggregation)]. In this implementation, I was asked to do ordering aggregation of incoming messages. But as the messages came in, if the next sequential block was completed, the customer wanted those messages to be emitted at that time instead of waiting for the entire batch to complete. So, for example, if I got messages [1,3,5], those messages would be aggregated and stored in the aggregation repository. But then, when message [2] came in, messages [1,2,3] would be emitted/processed (while message [5] would remain in the repository). Finally, when message [4] came in, messages [4,5] would be emitted/processed. That's about as custom as they come!

Hopefully this helps highlight some of the power and flexibility of Camel. Like I said at the beginning of this post, your imagination is the limit (or rather your use case). Enjoy!
