---
title: "Working with RabbitMQ queues and consumers from Ruby with Bunny"
layout: article
---

## About this guide

This guide covers everything related to queues in the AMQP 0.9.1 specification, common usage scenarios and how to accomplish
typical operations using Bunny.

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a>
(including images and stylesheets). The source is available [on Github](https://github.com/ruby-amqp/rubybunny.info).

## What version of Bunny does this guide cover?

This guide covers Bunny 0.9.0


## Queues in AMQP 0.9.1: Overview

### What are AMQP Queues?

*Queues* store and forward messages to consumers. They are similar to mailboxes in SMTP. Messages flow from producing applications to [exchanges](/articles/exchanges.html)
that route them to queues and finally queues deliver the messages to consumer applications (or consumer applications fetch messages as needed).

Note that unlike some other messaging protocols/systems, messages are not delivered directly to queues. They are delivered to exchanges that route
messages to queues using rules known as *bindings*.

AMQP is a programmable protocol, so queues and bindings alike are declared by applications.

### Concept of Bindings

A *binding* is an association between a queue and an exchange. Queues must be bound to at least one exchange in order to receive messages from publishers. Learn more
about bindings in the [Bindings guide](/articles/bindings.html).

### Queue Attributes

Queues have several attributes associated with them:

 * Name
 * Exclusivity
 * Durability
 * Whether the queue is auto-deleted when no longer used
 * Other metadata (sometimes called *X-arguments*)

These attributes define how queues can be used, what their life-cycle is like and other aspects of queue behavior.

## Queue Names and Declaring Queues

Every AMQP queue has a name that identifies it. Queue names often contain several segments separated by a dot ".", in a similar fashion to URI path segments being separated by a slash "/", although almost any string can represent a segment (with some limitations - see below).

Before a queue can be used, it has to be *declared*. Declaring a queue will cause it to be created if it does not already exist. The declaration will have no effect if the queue does already exist and its attributes are the *same as those in the declaration*. When the existing queue attributes are not the same as those in the declaration a channel-level exception is raised. This case is explained later in this guide.

### Explicitly Named Queues

Applications may pick queue names or ask the broker to generate a name for them.

To declare a queue with a particular name, for example, "images.resize", use the `Bunny::Channel#queue` method:

``` ruby
ch.queue("images.resize", :exclusive => false, :auto_delete => true)
```

The same example in context:

``` ruby
require "bunny"

conn = Bunny.new
conn.start

ch   = conn.create_channel
q    = ch.queue("images.resize", :exclusive => false, :auto_delete => true)
```


### Server-named queues

To ask an AMQP broker to generate a unique queue name for you, pass an *empty string* as the queue name argument. A generated queue name
will be assigned to the `Bunny::Queue` instance the method returns:

``` ruby
ch.queue("", :exclusive => true)
```

The same example in context:

``` ruby
require "bunny"

conn = Bunny.new
conn.start

ch   = conn.create_channel
q    = ch.queue("", :exclusive => true)
```

Note that, while it is common to declare server-named queues as `:exclusive`, it is not necessary.


### Reserved Queue Name Prefix

Queue names starting with "amq." are reserved for internal use by the broker. Attempts to declare a queue with a name that violates this rule will
result in a channel-level exception with reply code `403 (ACCESS_REFUSED)` and a reply message similar to this:

    ACCESS_REFUSED - queue name 'amq.queue' contains reserved prefix 'amq.*'
    
This error results in the channel that was used for the declaration being forcibly closed by RabbitMQ. If the program subsequently tries to communicate with RabbitMQ using the same channel then Bunny will raise a `Bunny::ChannelAlreadyClosed` error. In order to continue communications in the same program after such an error, a different channel would have to be used.

### Queue Re-Declaration With Different Attributes

When queue declaration attributes are different from those that the queue already has, a channel-level exception with code `406 (PRECONDITION_FAILED)`
will be raised. The reply text will be similar to this:

    PRECONDITION_FAILED - parameters for queue 'bunny.examples.channel_exception' in vhost '/' not equivalent

This error results in the channel that was used for the declaration being forcibly closed by RabbitMQ. If the program subsequently tries to communicate with RabbitMQ using the same channel then Bunny will raise a `Bunny::ChannelAlreadyClosed` error. In order to continue communications in the same program after such an error, a different channel would have to be used.

## Queue Life-cycle Patterns

According to the AMQP 0.9.1 specification, there are two common message queue life-cycle patterns:

 * Durable message queues that are shared by many consumers and have an independent existence: i.e. they will continue to exist and collect messages whether or not there are consumers to receive them.
 * Temporary message queues that are private to one consumer and are tied to that consumer. When the consumer disconnects, the message queue is deleted.

There are some variations of these, such as shared message queues that are deleted when the last of many consumers disconnects.

Let us examine the example of a well-known service like an event collector (event logger). A logger is usually up and running regardless of the existence of services
that want to log anything at a particular point in time. Other applications know which queues to use in order to communicate with the logger and can rely on those queues
being available and able to survive broker restarts. In this case, explicitly named durable queues are optimal and the coupling that is created between
applications is not an issue.

Another example of a well-known long-lived service is a distributed metadata/directory/locking server like [Apache Zookeeper](http://zookeeper.apache.org),
[Google's Chubby](http://labs.google.com/papers/chubby.html) or DNS. Services like this benefit from using well-known, not server-generated,
queue names and so do any other applications that use them.

A different sort of scenario is in "a cloud setting" when some kind of worker/instance might start and stop at any time so that other applications cannot
rely on it being available. In this case, it is possible to use well-known queue names, but a much better solution is to use server-generated, short-lived queues
that are bound to topic or fanout exchanges in order to receive relevant messages.

Imagine a service that processes an endless stream of events — Twitter is one example. When traffic increases, development operations may start additional application
instances in the cloud to handle the load. Those new instances want to subscribe to receive messages to process, but the rest of the system does not know anything about
them and cannot rely on them being online or try to address them directly. The new instances process events from a shared stream and are the same as their peers. In a case
like this, there is no reason for message consumers not to use queue names generated by the broker.

In general, use of explicitly named or server-named queues depends on the messaging pattern that your application needs. [Enterprise Integration Patterns](http://www.eaipatterns.com/)
discusses many messaging patterns in depth and the RabbitMQ FAQ also has a section on [use cases](http://www.rabbitmq.com/faq.html#scenarios).

## Declaring a Durable Shared Queue

To declare a durable shared queue, you pass a queue name that is a non-blank string and use the `:durable` option:

``` ruby
ch.queue("images.resize", :durable => true, :auto_delete => false)
```

The same example in context:

``` ruby
require "bunny"

conn = Bunny.new
conn.start

ch   = conn.create_channel
q    = ch.queue("images.resize", :durable => true, :auto_delete => false)
```


## Declaring a Temporary Exclusive Queue

To declare a server-named, exclusive, auto-deleted queue, pass "" (an empty string) as the queue name and use the `:exclusive` option:

``` ruby
ch.queue("", :exclusive => true)
```

The same example in context:

``` ruby
require "bunny"

conn = Bunny.new
conn.start

ch   = conn.create_channel
q    = ch.queue("", :exclusive => true)
```

Exclusive queues may only be accessed by the current connection and are deleted when that connection closes. The declaration of an exclusive queue by other
connections is not allowed and will result in a channel-level exception with the code `405 (RESOURCE_LOCKED)`

Exclusive queues will be deleted when the connection they were declared on is closed.


## Binding Queues to Exchanges

In order to receive messages, a queue needs to be bound to at least one exchange. Most of the time binding is explcit (done by applications). **Please note:** All queues are automatically bound to the default unnamed RabbitMQ direct exchange with a routing key that is the same as the queue name.

To bind a queue to an exchange,
use the `Bunny::Queue#bind` method:

``` ruby
q = ch.queue("", :exclusive => true)
x = ch.fanout("logging.events")

q.bind(x)
```

The same example in context:

``` ruby
require "bunny"

conn = Bunny.new
conn.start

ch   = conn.create_channel
q = ch.queue("", :exclusive => true)
x = ch.fanout("logging.events")

q.bind(x)
```


## Subscribing to receive messages ("push API")

To set up a queue subscription to enable an application to receive messages as they arrive in a queue, one uses the `Bunny::Queue#subscribe` method
to request that the server starts a *consumer*. Consumer is the name for subscription that the AMQP 0.9.1 specification uses. Consumers last as long as the channel that they were declared on,
or until the client cancels them (unsubscribes).

Consumers are identified by *consumer tags* and have a number of events that they can react to:

 * Message delivery
 * Consumer registration confirmation
 * Consumer cancellation

### Handling Messages With a Block

A message handler will process messages that RabbitMQ pushes to the consumer.
One way to define a handler is:

``` ruby
q = ch.queue("", :exclusive => true)
q.subscribe(:block => true, :ack => true) do |delivery_info, properties, payload|
  puts "Received #{payload}, message properties are #{properties.inspect}"
end
```

The same example in context:

``` ruby
require "bunny"

conn = Bunny.new
conn.start

ch   = conn.create_channel
q = ch.queue("", :exclusive => true)
q.subscribe(:block => true, :ack => true) do |delivery_info, properties, payload|
  puts "Received #{payload}, message properties are #{properties.inspect}"
end
```

The block should accept three arguments:

 * Delivery information (can be used to acknowledge messages, for example; will be covered in more detail later)
 * Message properties (metadata)
 * Message payload (body)

Both delivery information and message properties can be treated as Hash-like objects or structures.
For example, to get delivery tag, you can use either

``` ruby
delivery_info[:delivery_tag]
```

or

``` ruby
delivery_info.delivery_tag
```


#### Consumer Tags

`Bunny::Queue#subscribe` can take a `:consumer_tag` argument (any unique string) or let RabbitMQ generate one.

#### Blocking or Non-Blocking Behavior

The subscribe method will not block the calling thread by default. If you want to block, pass `:block => true` to
`Bunny::Queue#subscribe`. In Bunny 0.9.0 and later, network activity and dispatch of delivered messages
to consumers happens in separate threads that Bunny maintains internally, so it does not have to
block the thread that calls `Bunny::Queue#subscribe`. However, it may be convenient to do so
in long-running consumer applications.


### Accessing Message Properties (Metadata)

The *properties* parameter in the example above provides access to message metadata and delivery information:

 * Message content type
 * Message content encoding
 * Message routing key
 * Message delivery mode (persistent or not)
 * Consumer tag this delivery is for
 * Delivery tag
 * Message priority
 * Whether or not message is redelivered
 * Producer application id

Message properties can be treated as Hash-like objects or structures.
For example, to get message type, you can use either

``` ruby
properties[:type]
```

or

``` ruby
properties.type
```

and so on. An example to demonstrate how to access some of those attributes:

``` ruby
require 'bunny'

connection = Bunny.new
connection.start

ch = connection.create_channel
q = connection.queue('', :exclusive => true)
x  = ch.default_exchange

# set up the consumer
q.subscribe(:exclusive => true, :ack => false) do |delivery_info, properties, payload|
  puts properties.content_type # => "application/octet-stream"
  puts properties.priority     # => 8

  puts properties.headers["time"] # => a Time instance

  puts properties.headers["coordinates"]["latitude"] # => 59.35
  puts properties.headers["participants"]            # => 11
  puts properties.headers["venue"]                   # => "Stockholm"
  puts properties.headers["true_field"]              # => true
  puts properties.headers["false_field"]             # => false
  puts properties.headers["nil_field"]               # => nil
  puts properties.headers["ary_field"].inspect       # => ["one", 2.0, 3, [{ "abc" => 123}]]

  puts properties.timestamp      # => a Time instance
  puts properties.type           # => "kinda.checkin"
  puts properties.reply_to       # => "a.sender"
  puts properties.correlation_id # => "r-1"
  puts properties.message_id     # => "m-1"
  puts properties.app_id         # => "bunny.example"

  puts delivery_info.consumer_tag # => a string
  puts delivery_info.redelivered? # => false
  puts delivery_info.delivery_tag # => 1
  puts delivery_info.routing_key  # => server generated queue name prefixed with "amq.gen-"
  puts delivery_info.exchange     # => ""
end

# publishing
x.publish("hello",
          :routing_key => "#{q.name}",
          :app_id      => "bunny.example",
          :priority    => 8,
          :type        => "kinda.checkin",
          # headers table keys can be anything
          :headers     => {
            :coordinates => {
              :latitude  => 59.35,
              :longitude => 18.066667
            },
            :time         => Time.now,
            :participants => 11,
            :venue        => "Stockholm",
            :true_field   => true,
            :false_field  => false,
            :nil_field    => nil,
            :ary_field    => ["one", 2.0, 3, [{"abc" => 123}]]
          },
          :timestamp      => Time.now.to_i,
          :reply_to       => "a.sender",
          :correlation_id => "r-1",
          :message_id     => "m-1")

sleep 1.0
connection.close
```

The full list of properties (note that most of them are optional and may not be present) is:

 * `:delivery_tag`
 * `:redelivered`
 * `:exchange`
 * `:routing_key`
 * `:content_type`
 * `:content_encoding`
 * `:headers`
 * `:delivery_mode`
 * `:priority`
 * `:correlation_id`
 * `:reply_to`
 * `:expiration`
 * `:message_id`
 * `:timestamp`
 * `:type`
 * `:user_id`
 * `:app_id`
 * `:cluster_id`



### Consumer Instances

Bunny 0.9.0 introduces a new `Bunny::Consumer` class which takes the following positional arguments when instantiated:

 * `channel`      *(mandatory)*
 * `queue`        *(mandatory)*
 * `consumer_tag` *(default = "")*
 * `no_ack`       *(default = false)*
 * `exclusive`    *(default = false)*
 * `arguments`    *(default = {})*
 
To create a consumer object:

```ruby
class ExampleConsumer < Bunny::Consumer
  attr_reader :cancelled
  
  def cancelled?
    @cancelled
  end

  def handle_cancellation(_)
    @cancelled = true
  end
end

connection = Bunny.new
connection.start

ch = connection.create_channel
q = connection.queue("testq")

consumer = ExampleConsumer.new(ch, q, "my_example_consumer", false, false, {:test_arg => 'test'})
```

or

```ruby
consumer = ExampleConsumer.new(ch, q)
```

If the `consumer_tag` is empty then RabbitMQ will generate one, but it can also be set in the code:

```ruby
consumer.consumer_tag = "another_example_consumer"
```

`Bunny::Consumer` contains a *delivery handler* and when the consumer consumes a message then the delivery information, message properties (metadata) and body (payload) are
passed to it. In order to process consumed messages a block is passed to the consumer:

```ruby
consumer.on_delivery() do |delivery_info, metadata, payload|
  puts payload
end
```

Consumers may need to react to events other than message delivery.
For example, consumers can be cancelled by RabbitMQ in some situations:

 * When a consumer is cancelled via the RabbitMQ Management UI
 * When the queue from which messages are consumed is deleted

To handle these *consumer cancellation notification* events, consumers have a *cancellation handler* (see the `handle_cancellation` method in the example below).


### Registering Consumer Instances

To register a consumer and start consuming messages, pass a consumer object to the `Bunny::Queue#subscribe_with` method. Here is an example:

```ruby
require 'bunny'

# Define consumer subclass
class ExampleConsumer < Bunny::Consumer
  attr_reader :cancelled
  
  def cancelled?
    @cancelled
  end

  def handle_cancellation(_)
    @cancelled = true
  end
end

connection = Bunny.new
connection.start

consumer = nil

ch1 = connection.create_channel

t = Thread.new do
  ch2 = connection.create_channel
  q = ch2.queue("testq")

  consumer = ExampleConsumer.new(ch2, q)
  
  # Pass block to consumer delivery handler
  consumer.on_delivery() do |delivery_info, metadata, payload|
    puts payload
  end
  
  # Register the consumer
  q.subscribe_with(consumer)
end
t.abort_on_exception = true

sleep 0.5

x = ch1.default_exchange

# Publish messages
x.publish('Hello', :routing_key => "testq")
x.publish('World', :routing_key => "testq")

sleep 0.5

# Delete the queue triggering the consumer cancellation handler
ch1.queue("testq").delete

sleep 0.5

puts 'Consumer has been cancelled' if consumer.cancelled

sleep 2
connection.close
```

As with the `Bunny::Queue#subscribe` method, the `Bunny::Queue#subscribe_with` method can take a `:block` argument to block the current thread:

```ruby
q.subscribe_with(consumer, :block => true)
```

### Exclusive Consumers

Consumers can request exclusive access to the queue (meaning only this consumer can access the queue). This is useful when you want a long-lived shared queue
to be temporarily accessible by just one application (or thread, or process). If the application employing the exclusive consumer crashes or loses the
TCP connection to the broker, then the channel is closed and the exclusive consumer is cancelled.

To exclusively receive messages from the queue, pass the `:exclusive` option to `Bunny::Queue#subscribe`:

``` ruby
q = ch.queue("", :exclusive => true)
q.subscribe(:block => true, :ack => true) do |delivery_info, properties, payload|
  # ...
end
```

or the positional `exclusive` parameter to your `Bunny::Consumer` subclass:

``` ruby
class ExampleConsumer < Bunny::Consumer
  def cancelled?
    @cancelled
  end

  def handle_cancellation(_)
    @cancelled = true
  end
end

# channel, queue, consumer tag, no_ack, exclusive
consumer = ExampleConsumer.new(ch, q, "", false, true)
q.subscribe_with(consumer)
```

If a queue has an exclusive consumer, attempts to register another consumer will fail with an [access refused](http://www.rabbitmq.com/amqp-0-9-1-reference.html#constant.access-refused)
channel-level exception (code: 403).

It is not possible to register an exclusive consumer on a queue that already has consumers.


### Using Multiple Consumers Per Queue

It is possible to have multiple non-exclusive consumers on queues. In that case, messages will be
distributed between them according to prefetch levels of their channels (more on this later in this
guide). If prefetch values are equal for all consumers, each consumer will get about the same # of messages.


### Cancelling a Consumer

TBD


### Message Acknowledgements

Consumer applications — applications that receive and process messages ‚ may occasionally fail to process individual messages, or will just crash.
There is also the possibility of network issues causing problems. This raises a question — "When should the AMQP broker remove messages from queues?"

The AMQP 0.9.1 specification proposes two choices:

 * After broker sends a message to an application (using either basic.deliver or basic.get-ok methods).
 * After the application sends back an acknowledgement (using basic.ack AMQP method).

The former choice is called the *automatic acknowledgement model*, while the latter is called the *explicit acknowledgement model*.
With the explicit model, the application chooses when it is time to send an acknowledgement. It can be right after receiving a message,
or after persisting it to a data store before processing, or after fully processing the message (for example, successfully fetching a Web page,
processing and storing it into some persistent data store).

![Message Acknowledgements](https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/006_amqp_091_message_acknowledgements.png)

If a consumer dies without sending an acknowledgement, the AMQP broker will redeliver it to another consumer, or, if none are available at the time,
the broker will wait until at least one consumer is registered for the same queue before attempting redelivery.

The acknowledgement model is chosen when a new consumer is registered for a queue. By default, `Bunny::Queue#subscribe` will use the *automatic* model.
To switch to the *explicit* model, the `:ack` option should be used:

``` rub
q = ch.queue("", :exclusive => true).subscribe(:ack => true) do |delivery_info, properties, payload|
  # ...
end
```

To demonstrate how redelivery works, let us have a look at the following code example:

``` ruby
#!/usr/bin/env ruby
# encoding: utf-8

require "rubygems"
require "bunny"

puts "=> Subscribing for messages using explicit acknowledgements model"
puts

connection1 = Bunny.new
connection1.start

connection2 = Bunny.new
connection2.start

connection3 = Bunny.new
connection3.start

ch1 = connection1.create_channel
ch1.prefetch(1)

ch2 = connection2.create_channel
ch2.prefetch(1)

ch3 = connection3.create_channel
ch3.prefetch(1)

x   = ch3.direct("amq.direct")
q1  = ch1.queue("bunny.examples.acknowledgements.explicit", :auto_delete => false)
q1.purge

q1.bind(x).subscribe(:ack => true, :block => false) do |delivery_info, properties, payload|
  # do some work
  sleep(0.2)

  # acknowledge some messages, they will be removed from the queue
  if rand > 0.5
    # FYI: there is a shortcut, Bunny::Channel.ack
    ch1.acknowledge(delivery_info.delivery_tag, false)
    puts "[consumer1] Got message ##{properties.headers['i']}, redelivered?: #{delivery_info.redelivered?}, ack-ed"
  else
    # some messages are not ack-ed and will remain in the queue for redelivery
    # when app #1 connection is closed (either properly or due to a crash)
    puts "[consumer1] Got message ##{properties.headers['i']}, SKIPPPED"
  end
end

q2   = ch2.queue("bunny.examples.acknowledgements.explicit", :auto_delete => false)
q2.bind(x).subscribe(:ack => true, :block => false) do |delivery_info, properties, payload|
  # do some work
  sleep(0.2)

  ch2.acknowledge(delivery_info.delivery_tag, false)
  puts "[consumer2] Got message ##{properties.headers['i']}, redelivered?: #{delivery_info.redelivered?}, ack-ed"
end

t1 = Thread.new do
  i = 0
  loop do
    sleep 0.5

    x.publish("Message ##{i}", :headers => { :i => i })
    i += 1
  end
end
t1.abort_on_exception = true

t2 = Thread.new do
  sleep 4.0

  connection1.close
  puts "----- Connection 1 is now closed (we pretend that it has crashed) -----"
end
t2.abort_on_exception = true


sleep 7.0
connection2.close
connection3.close
```

So what is going on here? This example uses three AMQP connections to imitate three applications, one producer and two consumers.
Each AMQP connection opens a single channel. The consumers share a queue and the producer publishes messages to the queue periodically using an `amq.direct` exchange.

Both "applications" subscribe to receive messages using the explicit acknowledgement model. The AMQP broker by default will send each message to
the next consumer in sequence (this kind of load balancing is known as *round-robin*). This means that some messages will be delivered
to consumer #1 and some to consumer #2.

To demonstrate message redelivery we make consumer #1 randomly select which messages to acknowledge. After 4 seconds we disconnect it (to imitate a crash).
When that happens, the AMQP broker redelivers unacknowledged messages to consumer #2 which acknowledges them unconditionally. After 10 seconds, this example
closes all outstanding connections and exits.

An extract of output produced by this example:

```
=> Subscribing for messages using explicit acknowledgements model

[consumer1] Got message #0, redelivered?: false, ack-ed
[consumer2] Got message #1, redelivered?: false, ack-ed
[consumer1] Got message #2, redelivered?: false, ack-ed
[consumer2] Got message #3, redelivered?: false, ack-ed
[consumer1] Got message #4, SKIPPPED
[consumer2] Got message #5, redelivered?: false, ack-ed
[consumer1] Got message #6, SKIPPPED
----- Connection 1 is now closed (we pretend that it has crashed) -----
[consumer2] Got message #4, redelivered?: true, ack-ed
[consumer2] Got message #6, redelivered?: true, ack-ed
[consumer2] Got message #7, redelivered?: false, ack-ed
[consumer2] Got message #8, redelivered?: false, ack-ed
[consumer2] Got message #9, redelivered?: false, ack-ed
[consumer2] Got message #10, redelivered?: false, ack-ed
[consumer2] Got message #11, redelivered?: false, ack-ed
[consumer2] Got message #12, redelivered?: false, ack-ed
```

As we can see, consumer #1 did not acknowledge two messages (labelled 4 and 6):

```
[consumer1] Got message #4, SKIPPPED
...
[consumer1] Got message #6, SKIPPPED
```

and then, once consumer #1 had "crashed", those messages were immediately redelivered to the consumer #2:

```
----- Connection 1 is now closed (we pretend that it has crashed) -----
[consumer2] Got message #4, redelivered?: true, ack-ed
[consumer2] Got message #6, redelivered?: true, ack-ed
```

To acknowledge a message use `Bunny::Channel#ack`:

```
# FYI: there is a shortcut, Bunny::Channel.ack
ch1.acknowledge(delivery_info.delivery_tag, false)
```

`Bunny::Channel#ack` takes two arguments: a message *delivery tag* and a flag that indicates whether or not we want to acknowledge multiple messages at once.
Delivery tag is simply a channel-specific increasing number that the server uses to identify deliveries.

When acknowledging multiple messages at once, the delivery tag is treated as "up to and including". For example, if delivery tag = 5 that would mean "acknowledge messages 1, 2, 3, 4 and 5".

<p class="alert alert-error">
Acknowledgements are channel-specific. Applications must not receive messages on one channel and acknowledge them on another.
</p>

<p class="alert alert-error">
A message MUST not be acknowledged more than once. Doing so will result in a channel-level exception (PRECONDITION_FAILED) with an error message like this: "PRECONDITION_FAILED - unknown delivery tag"
</p>

### Rejecting messages

When a consumer application receives a message, processing of that message may or may not succeed. An application can indicate to the broker that message
processing has failed (or cannot be accomplished at the time) by rejecting a message. When rejecting a message, an application can ask the broker to discard or requeue it.

To reject a message use the `Bunny::Channel#reject` method:

``` ruby
ch1.reject(delivery_info.delivery_tag)
```

in the example above, messages are rejected without requeueing (broker will simply discard them). To requeue a rejected message, use the second argument
that `Bunny::Queue#reject` takes:

``` ruby
ch1.reject(delivery_info.delivery_tag)
```

### Negative acknowledgements

Messages are rejected with the `basic.reject` AMQP method. There is one limitation that `basic.reject` has:
there is no way to reject multiple messages, as you can do with acknowledgements. However, if you are using [RabbitMQ](http://rabbitmq.com), then there is a solution.
RabbitMQ provides an AMQP 0.9.1 extension known as [negative acknowledgements](http://www.rabbitmq.com/extensions.html#negative-acknowledgements) (nacks) and
Bunny supports this extension. For more information, please refer to the [RabbitMQ Extensions guide](/articles/rabbitmq_extensions.html).

### QoS — Prefetching messages

For cases when multiple consumers share a queue, it is useful to be able to specify how many messages each consumer can be sent at once before sending the next acknowledgement.
This can be used as a simple load balancing technique  to improve throughput if messages tend to be published in batches. For example, if a producing application
sends messages every minute because of the nature of the work it is doing.

Imagine a website that takes data from social media sources like Twitter or Facebook during the Champions League final (or the Superbowl),
and then calculates how many tweets mention a particular team during the last minute. The site could be structured as 3 applications:

 * A crawler that uses streaming APIs to fetch tweets/statuses, normalizes them and sends them in JSON for processing by other applications ("app A").
 * A calculator that detects what team is mentioned in a message, updates statistics and pushes an update to the Web UI once a minute ("app B").
 * A Web UI that fans visit to see the stats ("app C").

In this imaginary example, the "tweets per second" rate will vary, but to improve the throughput of the system and to decrease the maximum number of messages
that the AMQP broker has to hold in memory at once, applications can be designed in such a way that application "app B", the "calculator",
receives 5000 messages and then acknowledges them all at once. The broker will not send message 5001 unless it receives an acknowledgement.

In AMQP parlance this is know as *QoS* or *message prefetching*. Prefetching is configured on a per-channel (typically) or per-connection (rarely used) basis.
To configure prefetching per channel, use the `Bunny::Channel#prefetch` method. Let us return to the example we used in the earlier example:

``` ruby
ch1 = connection1.create_channel
ch1.prefetch(1)
```

In that example, one consumer prefetches three messages and another consumer prefetches just one. If we take a look at the output that the example produces,
we will see that `consumer1` fetched four messages and acknowledged one. After that, all subsequent messages were delivered to `consumer2`:

```
[consumer1] Got message #4, SKIPPPED
[consumer2] Got message #5, redelivered?: false, ack-ed
[consumer1] Got message #6, SKIPPPED
----- Connection 1 is now closed (we pretend that it has crashed) -----
[consumer2] Got message #4, redelivered?: true, ack-ed
[consumer2] Got message #6, redelivered?: true, ack-ed
```

<span class="alert alert-error">The prefetching setting is ignored for consumers that do not use explicit acknowledgements.</span>


## How Message Acknowledgements Relate to Transactions and Publisher Confirms

In cases where you cannot afford to lose a single message, AMQP 0.9.1 applications can use one or a combination of the following protocol features:

 * Publisher confirms (a RabbitMQ-specific extension to AMQP 0.9.1)
 * Publishing messages as immediate
 * Transactions (noticeable overhead)

This topic is covered in depth in the [Working With Exchanges](/articles/exchanges.html) guide. In this guide, we will only mention how
message acknowledgements are related to AMQP transactions and the Publisher Confirms extension.

Let us consider a publisher application (P) that communications with a consumer (C) using AMQP 0.9.1. Their communication can be graphically represented like this:

<pre>
-----       -----       -----
|   |   S1  |   |   S2  |   |
| P | ====> | B | ====> | C |
|   |       |   |       |   |
-----       -----       -----
</pre>

We have two network segments, S1 and S2. Each of them may fail. P is concerned with making sure that messages cross S1, while broker (B) and C are concerned with ensuring
that messages cross S2 and are only removed from the queue when they are processed successfully.

Message acknowledgements cover reliable delivery over S2 as well as successful processing. For S1, P has to use transactions (a heavyweight solution) or the more lightweight
Publisher Confirms RabbitMQ extension.


## Fetching messages when needed ("pull API")

The AMQP 0.9.1 specification also provides a way for applications to fetch (pull) messages from the queue only when necessary.
For that, use the `Bunny::Queue#pop` function which returns a triple of `[delivery_info, properties, payload]`:

``` ruby
delivery_info, properties, payload = q.pop
```

The same example in context:

``` ruby
# TBD
```

The metadata map has the same keys as for delivery handlers (see the "Push API" section above).

If the queue is empty, then `nil` will be returned.

## Unsubscribing From Messages

Sometimes it is necessary to unsubscribe from messages without deleting a queue. To do so, use the `Bunny::Consumer#cancel` method:

``` ruby
# TBD
```

The consumer tag is either known to your application ahead of time or generated by the broker and can be obtained from
the consumer returned by `Bunny::Queue#subscribe` (only makes sense with the non-blocking version, that is, when `:block => false` is used).

In AMQP 0.9.1 parlance, unsubscribing from messages is often referred to as "cancelling a consumer". Once a consumer is cancelled, messages will
no longer be delivered to it, however, due to the asynchronous nature of the protocol, it is possible for "in flight" messages to be received
after this call completes.

Fetching messages with `Bunny::Queue#get` is still possible even after a consumer is cancelled.


## Unbinding Queues From Exchanges

To unbind a queue from an exchange use the `Bunny::Queue#unbind` function:

``` ruby
q.unbind(x)
```

Note that trying to unbind a queue from an exchange that the queue was never bound to will
result in a channel-level exception.

## Querying the Number of Messages in a Queue

It is possible to query the number of messages sitting in the queue by declaring the queue
with the `:passive` attribute set.
The response (`queue.declare-ok` AMQP method) will include the number of messages along with
other attributes. However, Bunny provides a convenience function `langohr.queue/status`:

{% gist 7703ef627a4234b2cabe %}

## Querying the Number of Consumers On a Queue

It is possible to query the number of consumers on a queue by declaring the queue with the ":passive" attribute set. The response (`queue.declare-ok` AMQP method)
will include the number of consumers along with other attributes. However, Bunny provides a convenience function `langohr.queue/status`:

{% gist adb53fc1201630a5199c %}

## Purging queues

It is possible to purge a queue (remove all of the messages from it) using the `langohr.queues/purge` function:

{% gist be4b2501a72874e6d214 %}

Note that this example purges a newly declared queue with a unique server-generated name. When a queue is declared,
it is empty, so for server-named queues, there is no need to purge them before they are used.

## Deleting Queues

To delete a queue, use the `langohr.queue/delete` function:

{% gist dd6a06ac1e50e4c142eb %}

When a queue is deleted, all of the messages in it are deleted as well.


## Queue Durability vs Message Durability

See [Durability guide](/articles/durability.html)


## RabbitMQ Extensions Related to Queues

See [RabbitMQ Extensions guide](/articles/rabbitmq_extensions.html)



## Wrapping Up

AMQP queues can be client-named or server-named. It is possible to either subscribe for
messages to be pushed to consumers (register a consumer) or pull messages from the client
as needed. Consumers are identified by consumer tags.

For messages to be routed to queues, queues need to be bound to exchanges.

Most functions related to queues are found in two Bunny namespaces:

 * `langohr.queue`
 * `langohr.basic`


## What to Read Next

The documentation is organized as [a number of guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in this order:

 * [Exchanges and Publishing](/articles/exchanges.html)
 * [Bindings](/articles/bindings.html)
 * [RabbitMQ Extensions to AMQP 0.9.1](/articles/rabbitmq_extensions.html)
 * [Durability and Related Matters](/articles/durability.html)
 * [Error Handling and Recovery](/articles/error_handling.html)
 * [Troubleshooting](/articles/troubleshooting.html)
 * [Using TLS (SSL) Connections](/articles/tls.html)


## Tell Us What You Think!

Please take a moment to tell us what you think about this guide [on Twitter](http://twitter.com/rubyamqp) or the [Bunny mailing list](https://groups.google.com/forum/#!forum/ruby-amqp)

Let us know what was unclear or what has not been covered. Maybe you do not like the guide style or grammar or discover spelling mistakes. Reader feedback is key to making the documentation better.
