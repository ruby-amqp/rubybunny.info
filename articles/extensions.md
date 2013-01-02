---
title: "Working with RabbitMQ extensions from Ruby with Bunny"
layout: article
---

## About this guide

Bunny 0.9 supports all [RabbitMQ extensions to AMQP 0.9.1](http://www.rabbitmq.com/extensions.html):

  * [Publisher confirmations](http://www.rabbitmq.com/confirms.html)
  * [Negative acknowledgements](http://www.rabbitmq.com/nack.html) (basic.nack)
  * [Exchange-to-Exchange Bindings](http://www.rabbitmq.com/e2e.html)
  * [Alternate Exchanges](http://www.rabbitmq.com/ae.html)
  * [Per-queue Message Time-to-Live](http://www.rabbitmq.com/ttl.html#per-queue-message-ttl)
  * [Per-message Time-to-Live](http://www.rabbitmq.com/ttl.html#per-message-ttl)
  * [Queue Leases](http://www.rabbitmq.com/ttl.html#queue-ttl)
  * [Consumer Cancellation Notifications](http://www.rabbitmq.com/consumer-cancel.html)
  * [Sender-selected Distribution](http://www.rabbitmq.com/sender-selected.html)
  * [Dead Letter Exchanges](http://www.rabbitmq.com/dlx.html)
  * [Validated user_id](http://www.rabbitmq.com/validated-user-id.html)


## What version of Bunny does this guide cover?

This guide covers Bunny 0.9.


## Enabling RabbitMQ Extensions

You don't need to require any additional files to make Bunny 0.9 support RabbitMQ extensions.
The support is built into the core.


## Per-queue Message Time-to-Live

Per-queue Message Time-to-Live (TTL) is a RabbitMQ extension to AMQP 0.9.1 that allows developers to control how long
a message published to a queue can live before it is discarded.
A message that has been in the queue for longer than the configured TTL is said to be dead. Dead messages will not be delivered
to consumers and cannot be fetched using the *basic.get* operation (`Bunny::Queue#pop`).

Message TTL is specified using the *x-message-ttl* argument on declaration. With Bunny, you pass it to `Bunny::Queue#initialize` or `Bunny::Channel#queue`:

``` ruby
# 1000 milliseconds
channel.queue("", :arguments => { "x-message-ttl" => 1000 })
```

When a published message is routed to multiple queues, each of the queues gets a _copy of the message_. If the message subsequently dies in one of the queues,
it has no effect on copies of the message in other queues.

### Example

The example below sets the message TTL for a new server-named queue to be 1000 milliseconds. It then publishes several messages that are routed to the queue and tries
to fetch messages using the *basic.get* AMQP 0.9.1 method ({% yard_link Bunny::Queue#pop %} after 0.7 and 1.5 seconds:

``` ruby
#!/usr/bin/env ruby
# encoding: utf-8

require "rubygems"
require "bunny"

puts "=> Using per-queue message TTL"
puts

conn = Bunny.new
conn.start

ch   = conn.create_channel
x    = ch.fanout("amq.fanout")
q    = ch.queue("", :exclusive => true, :arguments => {"x-message-ttl" => 1000}).bind(x)

10.times do |i|
  x.publish("Message #{i}")
end

sleep 0.7
_, _, content1 = q.pop
puts "Fetched #{content1.inspect} after 0.7 second"

sleep 0.8
_, _, content2 = q.pop
msg = if content2
        content2.inspect
      else
        "nothing"
      end
puts "Fetched #{msg} after 1.5 second"

sleep 0.7
puts "Closing..."
conn.close
```

### Learn More

See also rabbitmq.com section on [Per-queue Message TTL](http://www.rabbitmq.com/ttl.html#per-queue-message-ttl)



## Publisher Confirms (Publisher Acknowledgements)

In some situations it is essential that no messages are lost. The only reliable way of ensuring this is by using confirmations.
The [Publisher Confirms AMQP extension](http://www.rabbitmq.com/blog/2011/02/10/introducing-publisher-confirms/) was designed to solve the reliable publishing problem.

Publisher confirms are similar to message acknowledgements documented in the [Queues and Consumers](/articles/queues/) guide but involve
a publisher and a RabbitMQ node instead of a consumer and a RabbitMQ node.

![RabbitMQ Message Acknowledgements](https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/006_amqp_091_message_acknowledgements.png)

![RabbitMQ Publisher Confirms](https://github.com/ruby-amqp/amqp/raw/master/docs/diagrams/007_rabbitmq_publisher_confirms.png)

### Public API

To use publisher confirmations, first put the channel into confirmation mode using `Bunny::Channel#confirm_select`:

```
channel.confirm_select
```

From this moment on, every message published on this channel will cause the channel's _publisher index_ (message counter) to be incremented.
It is possible to access the index using `Bunny::Channel#next_publish_seq_no` method. To check whether the channel is in confirmation mode,
use the `Bunny::Channel#using_publisher_confirmations?` predicate.

``` ruby
ch.using_publisher_confirmations? # => false
ch.confirm_select
ch.using_publisher_confirmations? # => true
```


### Example

``` ruby
#!/usr/bin/env ruby
# encoding: utf-8

require "rubygems"
require "bunny"

puts "=> Using publisher confirms"
puts

conn = Bunny.new
conn.start

ch   = conn.create_channel
x    = ch.fanout("amq.fanout")
q    = ch.queue("", :exclusive => true).bind(x)

ch.confirm_select
1000.times do
  x.publish("")
end
ch.wait_for_confirms

sleep 0.2
puts "Received acks for all published messages. #{q.name} now has #{q.message_count} messages."

sleep 0.7
puts "Closing..."
conn.close
```

### Learn More

See also rabbitmq.com section on [Publisher Confirms](http://www.rabbitmq.com/confirms.html)



## basic.nack

The AMQP 0.9.1 specification defines the basic.reject method that allows clients to reject individual, delivered messages, instructing the broker to either
discard them or requeue them. Unfortunately, basic.reject provides no support for negatively acknowledging messages in bulk.

To solve this, RabbitMQ supports the basic.nack method that provides all of the functionality of basic.reject whilst also allowing for bulk processing of messages.

### Public API

Bunny exposes `basic.nack` via the `Bunny::Channel#nack` method, similar to `Bunny::Channel#ack` and `Bunny::Channel#reject`:

``` ruby
# nack multiple messages at once
subject.nack(delivery_info.delivery_tag, false, true)

# nack a single message at once, the same as ch.reject(delivery_info.delivery_tag, false)
subject.nack(delivery_info.delivery_tag, false)
```

### Example

``` ruby
#!/usr/bin/env ruby
# encoding: utf-8

require "rubygems"
require "bunny"

puts "=> Using publisher confirms"
puts

conn = Bunny.new
conn.start

ch   = conn.create_channel
q    = ch.queue("", :exclusive => true)

20.times do
  q.publish("")
end

20.times do
  delivery_info, _, _ = q.pop(:ack => true)

  if delivery_info.delivery_tag == 20
    # requeue them all at once with basic.nack
    ch.nack(delivery_info.delivery_tag, true, true)
  end
end

puts "Queue #{q.name} still has #{q.message_count} messages in it"

sleep 0.7
puts "Disconnecting..."
conn.close
```

### Learn More

See also rabbitmq.com section on [basic.nack](http://www.rabbitmq.com/nack.html)




## Alternate Exchanges

Alternate Exchanges is a RabbitMQ extension to AMQP 0.9.1 that allows developers to define "fallback" exchanges
where unroutable messages will be sent.

### Public API

To specify exchange A as an alternate exchange to exchange B, specify the 'alternate-exchange' argument on declaration of B:

``` ruby
ch   = conn.create_channel
x1   = ch.fanout("bunny.examples.ae.exchange1", :auto_delete => true, :durable => false)
x2   = ch.fanout("bunny.examples.ae.exchange2", :auto_delete => true, :durable => false, :arguments => {
                   "alternate-exchange" => x1.name
                 })
```

### Example

``` ruby
#!/usr/bin/env ruby
# encoding: utf-8

require "rubygems"
require "bunny"

puts "=> Using alternate exchanges"
puts

conn = Bunny.new
conn.start

ch   = conn.create_channel
x1   = ch.fanout("bunny.examples.ae.exchange1", :auto_delete => true, :durable => false)
x2   = ch.fanout("bunny.examples.ae.exchange2", :auto_delete => true, :durable => false, :arguments => {
                   "alternate-exchange" => x1.name
                 })
q    = ch.queue("", :exclusive => true)
q.bind(x1)

x2.publish("")

sleep 0.2
puts "Queue #{q.name} now has #{q.message_count} message in it"

sleep 0.7
puts "Disconnecting..."
conn.close
```

### Learn More

See also rabbitmq.com section on [Alternate Exchanges](http://www.rabbitmq.com/ae.html)



## Exchange-To-Exchange Bindings

RabbitMQ supports [exchange-to-exchange bindings](http://www.rabbitmq.com/e2e.html) to allow even richer routing topologies as well as a backbone for
some other features (e.g. tracing).


### Public API

Bunny 0.9 exposes it via `Bunny::Exchange#bind` which is semantically the same as `Bunny::Queue#bind` but binds
two exchanges:

``` ruby
# x2 will be the source
x1.bind(x2, :routing_key => "americas.north.us.ca.*")
```

### Example

``` ruby
#!/usr/bin/env ruby
# encoding: utf-8

require "rubygems"
require "bunny"

puts "=> Using exchange-to-exchange bindings"
puts

conn = Bunny.new
conn.start

ch   = conn.create_channel
x1   = ch.fanout("bunny.examples.e2e.exchange1", :auto_delete => true, :durable => false)
x2   = ch.fanout("bunny.examples.e2e.exchange2", :auto_delete => true, :durable => false)
# x1 will be the source
x2.bind(x1)

q    = ch.queue("", :exclusive => true)
q.bind(x2)

x1.publish("")

sleep 0.2
puts "Queue #{q.name} now has #{q.message_count} message in it"

sleep 0.7
puts "Disconnecting..."
conn.close
```

### Learn More

See also rabbitmq.com section on [Exchange-to-Exchange Bindings](http://www.rabbitmq.com/e2e.html)



## Consumer Cancellation Notifications

TBD



## Queue Leases

TBD


## Per-Message Time-to-Live

TBD


## Sender-Selected Routing

TBD


## Dead Letter Exchange (DLX)

TBD


## Wrapping Up

TBD

## What to Read Next

The documentation is organized as [a number of guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in this order:

 * [Durability and Related Matters](/articles/durability.html)
 * [Error Handling and Recovery](/articles/error_handling.html)
 * [Troubleshooting](/articles/troubleshooting.html)
 * [Using TLS (SSL) Connections](/articles/tls.html)


## Tell Us What You Think!

Please take a moment to tell us what you think about this guide [on Twitter](http://twitter.com/rubyamqp) or the [Bunny mailing list](https://groups.google.com/forum/#!forum/ruby-amqp)

Let us know what was unclear or what has not been covered. Maybe you do not like the guide style or grammar or discover spelling mistakes. Reader feedback is key to making the documentation better.
