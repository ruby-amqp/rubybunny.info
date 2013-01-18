---
title: "Durability and related matters"
layout: article
---

## About This Guide

This guide covers queue, exchange and message durability, as well as other topics related to durability, for example, durability in clustered environments.

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a>
(including images and stylesheets). The source is available [on Github](https://github.com/ruby-amqp/rubybunny.info).


## What version of Bunny does this guide cover?

This guide covers Bunny 0.9.0.

## Entity durability and message persistence

### Exchange durability

AMQP separates the concept of entity durability (queues, exchanges) from message persistence. Exchanges can be durable or transient. Durable exchanges survive broker restart, transient exchanges do not (they have to be redeclared when the broker comes back online), however, not all scenarios and use cases mandate exchanges to be durable.

To create a durable exchange, declare it with the `:durable => true` argument:

``` ruby
ch = conn.create_channel
exch = ch.direct("my_direct_exchange", :durable => true)
```

### Queue durability

Queues can be durable or transient. Durable queues survive broker restart, transient queues do not (they have to be redeclared when the broker comes back online), however, not all scenarios and use cases mandate queues to be durable.

To create a durable queue, declare it with the `:durable => true` argument:

``` ruby
ch = conn.create_channel
q = ch.queue("my_queue", :durable => true)
```

Durability of a queue does not make _messages_ that are routed to that queue durable. If a broker is taken down and then brought back up, durable queues will be re-declared during broker startup, however, only _persistent_ messages will be recovered.

### Binding durability

Bindings of durable queues to durable exchanges are automatically durable and are restored after a broker restart. The AMQP 0.9.1 specification states that the binding of durable queues to transient exchanges must be allowed. In this case, since the exchange would not survive a broker restart, neither would any bindings to such and exchange.

### Message persistence

Messages may be published as persistent and this, in conjunction with queue durability, is what makes an AMQP broker persist them to disk. If the server is restarted, the system ensures that received persistent messages in durable queues are not lost. Simply publishing a message to a durable exchange or the fact that a queue to which a message is routed is durable does not make that message persistent. Message persistence depends on the persistence mode of the message itself.

**Note** that publishing persistent messages affects performance (just like with data stores, durability comes at a certain cost to performance).

Pass the `:persistent => true` argument to the `Bunny::Exchange#publish` method to publish your message as persistent:

``` ruby
exch.publish("My message", :persistent => true)
```

### Clustering

To achieve the degree of durability that critical applications need, it is necessary but not enough to use durable queues, exchanges and persistent messages. You need to use a cluster of brokers because otherwise, a single hardware problem may bring a broker down completely.

See the [RabbitMQ clustering guide](http://www.rabbitmq.com/clustering.html) for in-depth discussion of this topic.

### Highly available queues

Whilst the use of clustering provides for greater durability of critical systems, in order to achieve the highest level of resilience for queues and messages, high availability configuration should be used. This is because although exchanges and bindings survive the loss of individual nodes by using clustering, queues and their messages do not. A queue and its contents reside on exactly one node, thus the loss of a node will render its queues unavailable.

See the [RabbitMQ high availability guide](http://www.rabbitmq.com/ha.html) for an explanation of this topic.

## What to Read Next

The documentation is organized as [a number of guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in this order:

 * [Queues and Consumers](/articles/queues.html)
 * [Exchanges and Publishing](/articles/exchanges.html)
 * [Bindings](/articles/bindings.html)
 * [RabbitMQ Extensions to AMQP 0.9.1](/articles/rabbitmq_extensions.html)

## Tell Us What You Think!

Please take a moment to tell us what you think about this guide [on Twitter](http://twitter.com/rubyamqp) or the [Bunny mailing list](https://groups.google.com/forum/#!forum/ruby-amqp)

Let us know what was unclear or what has not been covered. Maybe you do not like the guide style or grammar or discover spelling mistakes. Reader feedback is key to making the documentation better.
