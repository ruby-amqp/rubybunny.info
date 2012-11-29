---
title: "Getting Started with Ruby and RabbitMQ"
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
 * Learning how the amqp gem can be integrated with Ruby objects in a way that makes unit testing easy.

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a> (including images and stylesheets). The source is available [on GitHub](https://github.com/ruby-amqp/rubybunny.info).


## Which versions of Bunny does this guide cover?

This guide covers Bunny 0.9.0 and later.

## Installing RabbitMQ

The [RabbitMQ site](http://rabbitmq.com) has a good [installation guide](http://www.rabbitmq.com/install.html) that addresses many operating systems.
On Mac OS X, the fastest way to install RabbitMQ is with [Homebrew](http://mxcl.github.com/homebrew/):

    brew install rabbitmq

then run it:

    rabbitmq-server

On Debian and Ubuntu, you can either [download the RabbitMQ .deb package](http://www.rabbitmq.com/server.html) and install it with [dpkg](http://www.debian.org/doc/FAQ/ch-pkgtools.en.html) or make use of the [apt repository](http://www.rabbitmq.com/debian.html#apt_ that the RabbitMQ team provides.

For RPM-based distributions like RedHat or CentOS, the RabbitMQ team provides an [RPM package](http://www.rabbitmq.com/install.html#rpm).

<div class="alert alert-error"><strong>Note:</strong> The RabbitMQ package that ships with recent Ubuntu versions (for example, 11.04)
is outdated and *will not work with Bunny 0.8.0 and later versions* (you will need at least RabbitMQ v2.0 for use with this guide).</div>

## Installing Bunny

### Make sure that you have Ruby and [Rubygems](http://docs.rubygems.org/read/chapter/3) installed

This guide assumes that you have installed one of the following supported Ruby implementations:

 * Ruby v1.9.3
 * Ruby v1.9.2
 * JRuby v1.7 or higher
 * Rubinius v2.0 or higher
 * Ruby Enterprise Edition
 * Ruby v1.8.7

### You can use Rubygems to install Bunny

    gem install amqp

### Adding Bunny as a dependency with Bundler

``` ruby
source :rubygems

gem "bunny", "~> 0.9.0.pre1"
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

This example demonstrates a very common communication scenario: *application A* wants to publish a message that will end up in a queue that *application B* listens on. In this case, the queue name is "amqpgem.examples.hello". Let us go through the code step by step:

``` ruby
require "rubygems"
require "bunny"
```

is the simplest way to load the amqp gem if you have installed it with RubyGems, but remember that you can omit the rubygems line if your environment does not need it. The following piece of code

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
```

Routing key is one of the **message properties**. The default exchange will route the message to a queue that has the same name as the message's routing key.
This is how our message ends up in the "bunny.examples.hello_world" queue.

This diagram demonstrates the "Hello, world" example data flow:

![Hello, World AMQP example data flow](http://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/001_hello_world_example_routing.png)

For the sake of simplicity, both the message producer (App I) and the consumer (App II) are running in the same Ruby process.
Now let us move on to a little bit more sophisticated example.
