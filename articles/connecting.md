---
title: "Connecting to RabbitMQ from Ruby with Bunny"
layout: article
---

## About this guide

This guide covers connection to RabbitMQ with Bunny, connection error handling, authentication failure handling and related issues.

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a>
(including images and stylesheets). The source is available [on Github](https://github.com/ruby-amqp/rubybunny.info).


## What version of Bunny does this guide cover?

This guide covers Bunny 0.9.0.



## Two ways to specify connection parameters

With Bunny, connection parameters (host, port, username, vhost and so on) can be passed in two forms:

 * As a map of attributes
 * As a connection URI string (à la JDBC)


### Using a Map of Parameters

Map options that Bunny will recognize are

 * `:host`
 * `:port`
 * `:user` or `:username`
 * `:pass` or `:password`
 * `:vhost` or `virtual_host`
 * `:heartbeat` or `:heartbeat_interval`, in seconds, default is 0 (no heartbeats). `:server` means "use the value from RabbitMQ config"
 * `:log_level` (symbol or integer, default: `Logger::WARN`): log level to use.
 * `:log_file` (string or `IO`): log file or `IO` object to use. Defaults to `STDOUT`.
 * `:automatically_recover` (boolean, default: `true`): when `false`, will disable automatic network failure recovery
 * `:network_recovery_interval` (number, default: ``): interval between reconnection attempts
 * `:threaded` (boolean): switches to single-threaded connections when set to `false`. Only recommended for apps that only publish messages.

To connect to RabbitMQ with a map of parameters, pass them to `Bunny.new`. The connection
will be established when `Bunny::Session#start` is called:

``` ruby
conn = Bunny.new(:host => "localhost", :vhost => "myapp.production", :user => "bunny", :password => "t0ps3kret")
conn.start
```

`Bunny.new` returns a connection instance that is used to open channels. More about channels later in this guide.

#### Default parameters

Default connection parameters are

``` ruby
{
  :host      => "127.0.0.1",
  :port      => 5672,
  :ssl       => false,
  :vhost     => "/",
  :user      => "guest",
  :pass      => "guest",
  :heartbeat => :server, # will use RabbitMQ setting
  :frame_max => 131072
}
```

### Single-Threaded Mode

Bunny 0.9+ uses a separate thread for network (I/O) activity. In some cases, developers may want to disable it and
handle network failure issues manually. This is possible by passing the `:threaded` option as `false`.


### Reconnection Interval

When Bunny detects a network issue, it will by default try to reconnect. This behavior can be tweaked using two options:

 * `:network_recovery_interval` controls for how long Bunny will wait between reconnection attempts (including the first one). The value is in seconds, `5.0` is the default.
 * `:automatically_recover` can be set to `false` to completely disable network recovery. In such case, Bunny will raise
    exceptions on the thread `Bunny::Session` was instantiated on.


### Using Connection Strings

It is also possible to specify connection parameters as a URI string:

``` ruby
b = Bunny.new("amqp://guest:guest@vm188.dev.megacorp.com/profitd.qa")
b.start
```

Unfortunately, there is no URI standard for AMQP URIs, so while several schemes used in the wild share the same basic idea, they differ in some details.
The implementation used by Bunny aims to encourage URIs that work as widely as possible.

Here are some examples of valid AMQP URIs:

 * amqp://dev.rabbitmq.com
 * amqp://dev.rabbitmq.com:5672
 * amqp://guest:guest@dev.rabbitmq.com:5672
 * amqp://hedgehog:t0ps3kr3t@hub.megacorp.internal/production
 * amqps://hub.megacorp.internal/%2Fvault

The URI scheme should be "amqp", or "amqps" if SSL is required.

The host, port, username and password are represented in the authority component of the URI in the same way as in HTTP URIs.

The vhost is obtained from the first segment of the path, with the leading slash removed.  The path should contain only a single segment (i.e, the only slash in it should be the leading one). If the vhost is to include slashes or other reserved URI characters, these should be percent-escaped.

Here are some examples that demonstrate how `AMQ::Settings.parse_amqp_url` parses out the vhost from connection URIs:

``` ruby
AMQ::Settings.parse_amqp_url("amqp://dev.rabbitmq.com")            # => vhost is nil, so default ("/") will be used
AMQ::Settings.parse_amqp_url("amqp://dev.rabbitmq.com/")           # => vhost is an empty string
AMQ::Settings.parse_amqp_url("amqp://dev.rabbitmq.com/%2Fvault")   # => vhost is "/vault"
AMQ::Settings.parse_amqp_url("amqp://dev.rabbitmq.com/production") # => vhost is "production"
AMQ::Settings.parse_amqp_url("amqp://dev.rabbitmq.com/a.b.c")      # => vhost is "a.b.c"
AMQ::Settings.parse_amqp_url("amqp://dev.rabbitmq.com/foo/bar")    # => ArgumentError
```

### Connection Failures

If a connection does not succeed, Bunny will raise one of the following exceptions:

 * `Bunny::PossibleAuthenticationFailureException` indicates an authentication issue or that connection to RabbitMQ was closed before successfully finishing connection negotiation
 * `Bunny::TCPConnectionFailed` indicates that connection to the host has failed. Either the address is not reachable or DNS entry does not exist. Often may suggest a misconfiguration.


## PaaS Environments

### The RABBITMQ_URL Environment Variable

If no arguments are passed to `Bunny.new` but the `RABBITMQ_URL` environment variable is set, Bunny will use it as connection
URI.


## Opening a Channel

Some applications need multiple connections to RabbitMQ. However, it is undesirable to keep many TCP connections open at the same time because
doing so consumes system resources and makes it more difficult to configure firewalls. AMQP 0-9-1 connections are multiplexed with channels that can
be thought of as "lightweight connections that share a single TCP connection".

To open a channel, use the `Bunny::Session#create_channel` method:

``` ruby
conn = Bunny.new
conn.start

ch   = conn.create_channel
```

Channels are typically long lived: you open one or more of them and use them for a period of time, as opposed to opening
a new channel for each published message, for example.


## Closing Channels

To close a channel, use the `Bunny::Channel#close` method. A closed channel
can no longer be used.

``` ruby
conn = Bunny.new
conn.start

ch   = conn.create_channel
ch.close
```


## Connecting in Web applications (Ruby on Rails, Sinatra, etc)

When connecting in Web apps, the rule of thumb is: do it in an initializer, not controller
actions or request handlers.

### Ruby on Rails

Currently Bunny does not have integration points for Rails (e.g. a rail tie).


## Disconnecting

To close a connection, use the `Bunny::Session#close` function. This will automatically
close all channels of that connection first:

``` ruby
conn = Bunny.new
conn.start

conn.close
```


## Troubleshooting

If you have read this guide and still have issues with connecting, check our [Troubleshooting guide](/articles/troubleshooting.html)
and feel free to ask [on the mailing list](https://groups.google.com/forum/#!forum/ruby-amqp).


## Wrapping Up

There are two ways to specify connection parameters with Bunny: with a map of parameters or via URI string.
Connection issues are indicated by various exceptions. If the `RABBITMQ_URL` env variable is set, Bunny
will use its value as RabbitMQ connection URI.


## What to Read Next

The documentation is organized as [a number of guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in this order:

 * [Queues and Consumers](/articles/queues.html)
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
