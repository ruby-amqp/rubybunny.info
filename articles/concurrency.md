---
title: "Concurrency in Bunny and Applications That Use It"
layout: article
---

## About this guide

This guide covers concurrency in Bunny, concurrency safety
of key public API parts, potential for parallelism on Ruby runtimes
that support it, and related issues.

This work is licensed under a <a rel="license"
href="http://creativecommons.org/licenses/by/3.0/">Creative Commons
Attribution 3.0 Unported License</a> (including images and
stylesheets). The source is available [on
Github](https://github.com/ruby-amqp/rubybunny.info).


## What version of Bunny does this guide cover?

This guide covers Bunny 0.10.x and later versions.


## Concurrency in Bunny Design

Starting with Bunny 0.9, Bunny is developed with concurrency in mind.
This means several things:

 * Bunny avoids some well known concurrency problems in [amqp gem](http://rubyamqp.info),
   most notably long running operations in message handlers blocking event loop and thus all
   I/O activity in the library.

 * Connection (`Bunny::Session`) assumes there will be concurrent publishers
   and consumers.

 * Parts of the library can take advantage of parallelism on runtimes that
   provide it.

 * Parts of the library that previously were not concurrent now provide
   concurrency controls.

### I/O Activity Loop

TBD



## Consumer Work Pools

TBD


## Sharing Channels Between Threads

TBD


## Wrapping Up

TBD


## What to Read Next

The documentation is organized as [a number of
guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in
this order:

 * [RabbitMQ Extensions to AMQP 0.9.1](/articles/rabbitmq_extensions.html)
 * [Error Handling and Recovery](/articles/error_handling.html)
 * [Troubleshooting](/articles/troubleshooting.html)
 * [Using TLS (SSL) Connections](/articles/tls.html)



## Tell Us What You Think!

Please take a moment to tell us what you think about this guide [on
Twitter](http://twitter.com/rubyamqp) or the [Bunny mailing
list](https://groups.google.com/forum/#!forum/ruby-amqp).

Let us know what was unclear or what has not been covered. Maybe you
do not like the guide style or grammar or discover spelling
mistakes. Reader feedback is key to making the documentation better.
