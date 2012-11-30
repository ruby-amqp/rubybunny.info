---
title: "Getting Started with Ruby and RabbitMQ with Bunny"
layout: article
---

## About this guide

This guide is a quick tutorial that helps you to get started with the AMQP 0.9.1 specification in general and [Bunny](http://github.com/ruby-amqp/bunny) in particular.
It should take about 20 minutes to read and study the provided code examples. This guide covers:

 * Installing RabbitMQ, a mature popular messaging broker server.
 * Installing Bunny via [Rubygems](http://rubygems.org) and [Bundler](http://gembundler.com).
 * Running a "Hello, world" messaging example that is a simple demonstration of 1:1 communication.
 * Creating a "Twitter-like" publish/subscribe example with one publisher and four subscribers that demonstrates 1:n communication.
 * Creating a topic routing example with two publishers and eight subscribers showcasing n:m communication when subscribers only receive messages that they are interested in.

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a> (including images and stylesheets).
The source is available [on GitHub](https://github.com/ruby-amqp/rubybunny.info).


## Which versions of Bunny does this guide cover?

This guide covers Bunny 0.9.0 and later.

## Installing RabbitMQ

The [RabbitMQ site](http://rabbitmq.com) has a good [installation guide](http://www.rabbitmq.com/install.html) that addresses many operating systems.
On Mac OS X, the fastest way to install RabbitMQ is with [Homebrew](http://mxcl.github.com/homebrew/):

    brew install rabbitmq

then run it:

    rabbitmq-server

On Debian and Ubuntu, you can either [download the RabbitMQ .deb package](http://www.rabbitmq.com/server.html) and install it with [dpkg](http://www.debian.org/doc/FAQ/ch-pkgtools.en.html) or make use of the [apt repository](http://www.rabbitmq.com/debian.html#apt_) that the RabbitMQ team provides.

For RPM-based distributions like RedHat or CentOS, the RabbitMQ team provides an [RPM package](http://www.rabbitmq.com/install.html#rpm).

<div class="alert alert-error"><strong>Note:</strong> The RabbitMQ packages that ship with Ubuntu versions earlier than 11.10
are outdated and <strong>will not work with Bunny 0.9.0 and later versions</strong> (you will need at least RabbitMQ v2.0 for use with this guide).</div>

## Installing Bunny

### Make sure that you have Ruby and [Rubygems](http://docs.rubygems.org/read/chapter/3) installed

This guide assumes that you have installed one of the following supported Ruby implementations:

 * Ruby v1.9.3
 * Ruby v1.9.2
 * JRuby v1.7.0 or higher
 * Rubinius v2.0 or higher
 * Ruby Enterprise Edition
 * Ruby v1.8.7

### You can use Rubygems to install Bunny

    gem install bunny --pre

### Adding Bunny as a dependency with Bundler

``` ruby
source :rubygems

gem "bunny", "~> 0.9.0.pre2"
```

### Verifying your installation

Verify your installation with a quick irb session:

```
irb -rubygems
:001 > require "bunny"
=> true
:002 > Bunny::VERSION
=> "0.9.0"
```

## "Hello, world" example

Let us begin with the classic "Hello, world" example. First, here is the code:

``` ruby
#!/usr/bin/env ruby
# encoding: utf-8

require "rubygems"
require "bunny"

conn = Bunny.new
conn.start

ch = conn.create_channel
q  = ch.queue("bunny.examples.hello_world", :auto_delete => true)
x  = ch.default_exchange

q.subscribe do |metadata, payload|
  puts "Received #{payload}"
end

x.publish("Hello!", :routing_key => q.name)

sleep 1.0
conn.close
```

This example demonstrates a very common communication scenario: *application A* wants to publish a message that will end up in a queue that *application B* listens on. In this case, the queue name is "bunny.examples.hello_world". Let us go through the code step by step:

``` ruby
require "rubygems"
require "bunny"
```

is the simplest way to load Bunny if you have installed it with RubyGems, but remember that you can omit the rubygems line if your environment does not need it. The following piece of code

``` ruby
conn = Bunny.new
conn.start
```

connects to RabbitMQ running on localhost, with the default port (5672), username (guest), password (guest) and virtual host ('/').

The next line

``` ruby
ch = conn.create_channel
```

opens a new _channel_. AMQP 0.9.1 is a multi-channeled protocol that uses channels to multiplex a TCP connection.

Channels are opened on a connection. `Bunny::Session#create_channel` will return only when Bunny
receives a confirmation that the channel is open from RabbitMQ.

This line

``` ruby
q  = ch.queue("bunny.examples.hello_world", :auto_delete => true)
```

declares a **queue** on the channel that we have just opened. Consumer applications get messages from queues. We declared this queue with
the "auto-delete" parameter. Basically, this means that the queue will be deleted when there are no more processes consuming messages from it.

The next line

``` ruby
x  = ch.default_exchange
```

instantiates an **exchange**. Exchanges receive messages that are sent by producers. Exchanges route messages to queues according to rules called **bindings**.
In this particular example, there are no explicitly defined bindings. The exchange that we use is known as the **default exchange** and it has implied
bindings to all queues. Before we get into that, let us see how we define a handler for incoming messages

``` ruby
q.subscribe do |metadata, payload|
  puts "Received #{payload}"
end
```

`Bunny::Queue#subscribe` takes a block that will be called every time a message arrives. This will happen in a thread pool, so `Bunny::Queue#subscribe`
does not block the thread that invokes it.

Finally, we publish our message

``` ruby
x.publish("Hello!", :routing_key => q.name)
```

Routing key is one of the **message properties**. The default exchange will route the message to a queue that has the same name as the message's routing key.
This is how our message ends up in the "bunny.examples.hello_world" queue.

This diagram demonstrates the "Hello, world" example data flow:

![Hello, World AMQP example data flow](http://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/001_hello_world_example_routing.png)

For the sake of simplicity, both the message producer (publisher) and the consumer are running in the same Ruby process.
Now let us move on to a little bit more sophisticated example.

## Blabblr: one-to-many publish/subscribe (pubsub) example

The previous example demonstrated how a connection to a broker is made and how to do 1:1 communication using the default exchange. Now let us take a look
at another common scenario: broadcast, or multiple consumers and one producer.

A very well-known broadcast example is Twitter: every time a person tweets, followers receive a notification. Blabbr, our imaginary information network,
models this scenario: every network member has a separate queue and publishes blabs to a separate exchange. Three Blabbr members, Joe, Aaron and Bob,
follow the official NBA account on Blabbr to get updates about what is happening in the world of basketball. Here is the code:

``` ruby
require "rubygems"
require "bunny"

conn = Bunny.new("amqp://guest:guest@localhost:55672")
conn.start

ch  = conn.create_channel
x   = ch.fanout("nba.scores")

ch.queue("joe",   :auto_delete => true).bind(x).subscribe do |properties, payload|
  puts "#{payload} => joe"
end

ch.queue("aaron", :auto_delete => true).bind(x).subscribe do |properties, payload|
  puts "#{payload} => aaron"
end

ch.queue("bob",   :auto_delete => true).bind(x).subscribe do |properties, payload|
  puts "#{payload} => bob"
end

x.publish("BOS 101, NYK 89").publish("ORL 85, ALT 88")

conn.close
```

Unlike the "Hello, world" example above, here we use a connection URI instead of
the default arguments.

In this example, opening a channel is no different to opening a channel in the previous example, however, the exchange is declared differently:

``` ruby
x   = ch.fanout("nba.scores")
```

The exchange that we declare above using `Bunny::Channel#fanout` is a **fanout exchange**.
A fanout exchange delivers messages to all of the queues that are bound to it: exactly what we want in the case of Blabbr!

This piece of code

``` ruby
ch.queue("joe",   :auto_delete => true).bind(x).subscribe do |properties, payload|
  puts "#{payload} => joe"
end
```

is similar to the subscription code that we used for message delivery previously,
but what does that `Bunny::Queue#bind` method do? It sets up a binding between the queue and
the exchange that you pass to it. We need to do this to make sure that our fanout exchange
routes messages to the queues of any subscribed followers.

``` ruby
x.publish("BOS 101, NYK 89").publish("ORL 85, ALT 88")
```

demonstrates `Bunny::Exchange#publish` call chaining. Blabbr members use a fanout exchange for
publishing, so there is no need to specify a message routing key because every queue that is
bound to the exchange will get its own copy of all messages, regardless of the queue name and
routing key used.

A diagram for Blabbr looks like this:

![Blabbr Data Flow](https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/002_blabbr_example_routing.png)

Blabbr is pretty unlikely to secure hundreds of millions of dollars in funding,
but it does a pretty good job of demonstrating how one can use RabbitMQ fanout exchanges to do broadcasting.


## Weathr: many-to-many topic routing example

So far, we have seen point-to-point communication and broadcasting. Those two communication styles are possible with many protocols, for instance, HTTP handles these scenarios just fine.
You may ask "what differentiates RabbitMQ?" Well, next we are going to introduce you to **topic exchanges** and routing with patterns, one of the features that makes RabbitMQ very powerful.

Our third example involves weather condition updates. What makes it different from the previous two examples is that not all of the consumers are interested in all of the messages.
People who live in Portland usually do not care about the weather in Hong Kong (unless they are visiting soon). They are much more interested in weather conditions around Portland,
possibly all of Oregon and sometimes a few neighbouring states.

Our example features multiple consumer applications monitoring updates for different regions.
Some are interested in updates for a specific city, others for a specific state and so on,
all the way up to continents. Updates may overlap so that an update for San Diego, CA
appears as an update for California, but also should show up on the North America updates list.

Here is the code:

``` ruby
#!/usr/bin/env ruby
# encoding: utf-8

require "rubygems"
require "bunny"

connection = Bunny.new
connection.start

channel  = connection.create_channel
# topic exchange name can be any string
exchange = channel.topic("weathr", :auto_delete => true)

# Subscribers.
channel.queue("", :exclusive => true).bind(exchange, :routing_key => "americas.north.#").subscribe do |properties, payload|
  puts "An update for North America: #{payload}, routing key is #{properties.routing_key}"
end
channel.queue("americas.south").bind(exchange, :routing_key => "americas.south.#").subscribe do |properties, payload|
  puts "An update for South America: #{payload}, routing key is #{properties.routing_key}"
end
channel.queue("us.california").bind(exchange, :routing_key => "americas.north.us.ca.*").subscribe do |properties, payload|
  puts "An update for US/California: #{payload}, routing key is #{properties.routing_key}"
end
channel.queue("us.tx.austin").bind(exchange, :routing_key => "#.tx.austin").subscribe do |properties, payload|
  puts "An update for Austin, TX: #{payload}, routing key is #{properties.routing_key}"
end
channel.queue("it.rome").bind(exchange, :routing_key => "europe.italy.rome").subscribe do |properties, payload|
  puts "An update for Rome, Italy: #{payload}, routing key is #{properties.routing_key}"
end
channel.queue("asia.hk").bind(exchange, :routing_key => "asia.southeast.hk.#").subscribe do |properties, payload|
  puts "An update for Hong Kong: #{payload}, routing key is #{properties.routing_key}"
end

exchange.publish("San Diego update", :routing_key => "americas.north.us.ca.sandiego").
  publish("Berkeley update",         :routing_key => "americas.north.us.ca.berkeley").
  publish("San Francisco update",    :routing_key => "americas.north.us.ca.sanfrancisco").
  publish("New York update",         :routing_key => "americas.north.us.ny.newyork").
  publish("São Paolo update",        :routing_key => "americas.south.brazil.saopaolo").
  publish("Hong Kong update",        :routing_key => "asia.southeast.hk.hongkong").
  publish("Kyoto update",            :routing_key => "asia.southeast.japan.kyoto").
  publish("Shanghai update",         :routing_key => "asia.southeast.prc.shanghai").
  publish("Rome update",             :routing_key => "europe.italy.roma").
  publish("Paris update",            :routing_key => "europe.france.paris")

sleep 1.0

connection.close
```

The first line that is different from the Blabbr example is

``` ruby
exchange = channel.topic("weathr", :auto_delete => true)
```

We use a topic exchange here. Topic exchanges are used for [multicast](http://en.wikipedia.org/wiki/Multicast) messaging
where consumers indicate which topics they are interested in (think of it as subscribing to a feed for an individual tag in
your favourite blog as opposed to the full feed). Routing with a topic exchange is done by specifying a **routing pattern**
on binding, for example:

``` ruby
channel.queue("americas.south").bind(exchange, :routing_key => "americas.south.#").subscribe do |properties, payload|
  puts "An update for South America: #{payload}, routing key is #{properties.routing_key}"
end
```
Here we bind a queue with the name of "americas.south" to the topic exchange declared earlier using the {% yard_link Bunny::Queue#bind %} method.  This means that only messages with a routing key matching "americas.south.#" will be routed to that queue. A routing pattern consists of several words separated by dots, in a similar way to URI path segments joined by slashes. Here are a few examples:

 * asia.southeast.thailand.bangkok
 * sports.basketball
 * usa.nasdaq.aapl
 * tasks.search.indexing.accounts

Now let us take a look at a few routing keys that match the "americas.south.#" pattern:

 * americas.south
 * americas.south.*brazil*
 * americas.south.*brazil.saopaolo*
 * americas.south.*chile.santiago*

In other words, the "#" part of the pattern matches 0 or more words.

For a pattern like "americas.south.*", some matching routing keys would be:

 * americas.south.*brazil*
 * americas.south.*chile*
 * americas.south.*peru*

but not

 * americas.south
 * americas.south.chile.santiago

so "*" only matches a single word. The AMQP 0.9.1 specification says that topic segments (words) may contain the letters A-Z and a-z and digits 0-9.

A (very simplistic) diagram to demonstrate topic exchange in action:

![Weathr Data Flow](https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/003_weathr_example_routing.png)


As in the previous examples, the block we pass to `Bunny::Queue#subscribe` takes multiple arguments:
**message metadata** (properties) and **message body** (often called the **payload**).
Long story short, the properties parameter lets you access metadata associated with the message. Some examples of message metadata
attributes are:

 * message content type
 * message content encoding
 * message priority
 * message expiration time
 * message identifier
 * reply to (specifies which message this is a reply to)
 * application id (identifier of the application that produced the message)

and so on.

As the following binding demonstrates, `"#"` and `"*"` can also appear at the beginning of routing patterns:

``` ruby
channel.queue("us.tx.austin").bind(exchange, :routing_key => "#.tx.austin").subscribe do |properties, payload|
  puts "An update for Austin, TX: #{payload}, routing key is #{properties.routing_key}"
end
```

For this example the publishing of messages is no different from that of previous examples. If we were to run the program,
a message published with a routing key of `"americas.north.us.ca.berkeley"` would be routed to two queues: `"us.california"` and the **server-named queue**
that we declared by passing a blank string as the name:

``` ruby
channel.queue("", :exclusive => true).bind(exchange, :routing_key => "americas.north.#").subscribe do |properties, payload|
  puts "An update for North America: #{payload}, routing key is #{properties.routing_key}"
end
```

The name of the server-named queue is generated by the broker and sent back to the client with a queue declaration confirmation.



## Wrapping up

This is the end of the tutorial. Congratulations! You have learned quite a bit about both AMQP 0.9.1 and Bunny. This is only the tip of the iceberg.
RabbitMQ has many more features to offer:

 * Reliable delivery of messages
 * Message confirmations (a way to tell broker that a message was or was not processed successfully)
 * Message redelivery when consumer applications fail or crash
 * Load balancing of messages between multiple consumers
 * Message metadata attributes
 * High Availability features

and so on. Other guides explain these features in depth, as well as use cases for them. To stay up to date with Bunny development, [follow @rubyamqp on Twitter](http://twitter.com/rubyamqp)
and [join our mailing list](http://groups.google.com/group/ruby-amqp).

## What to read next

Documentation is organized as a number of <a href="/articles/guides.html">documentation guides</a>, covering all kinds of topics including use cases for various exchange types,
fault-tolerant message processing with acknowledgements and error handling.

We recommend that you read the following guides next, if possible, in this order:

 * [AMQP 0.9.1 Model Explained](http://www.rabbitmq.com/tutorials/amqp-concepts.html). A simple two page introduction to the AMQP Model concepts and features. Understanding the AMQP 0.9.1 Model
   will make a lot of other documentation, both for Bunny and RabbitMQ itself, easier to follow. With this guide, you don't have to spend hours reading the whole specification.
 * [Connecting to the broker](/articles/connecting_to_broker/). This guide explains how to connect to RabbitMQ and how to integrate Bunny into standalone and Web applications.
 * [Queues and Consumers](/articles/queues/). This guide focuses on features that consumer applications use heavily.
 * [Exchanges and Publishing](/articles/exchanges/). This guide focuses on features that producer applications use heavily.
 * [Patterns and Use Cases](/articles/patterns_and_use_cases/). This guide focuses on the implementation of [common messaging patterns](http://www.eaipatterns.com/) using AMQP 0.9.1 Model features as building blocks.
 * [Error Handling and Recovery](/articles/error_handling/). This guide explains how to handle protocol errors, network failures and other things that may go wrong in real world projects.

If you are migrating your application from earlier versions of Bunny (0.7.x and 0.8.x), to 0.9.x and later, there is the [Bunny 0.9 migration guide](/articles/09_migration) to help you.
