---
title: "TLS (SSL) connections to RabbitMQ from Ruby with Bunny"
layout: article
---

## About This Guide

This guide covers TLS (SSL) connections to RabbitMQ with Bunny.

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a>
(including images and stylesheets). The source is available [on Github](https://github.com/ruby-amqp/rubybunny.info).


## What version of Bunny does this guide cover?

This guide covers Bunny 0.10.x and later versions.


## TLS Support in RabbitMQ

RabbitMQ version 2.x and 3.x support TLS/SSL on Erlang R13B or later. Using the most
recent version (e.g. R16B) is recommended.

To use TLS with RabbitMQ, you need a few things:

 * Server certificate and private key
 * [Configure RabbitMQ to use TLS](http://www.rabbitmq.com/ssl.html)
 * CA certificate that signs the server certificate
 * Client certificate and private (optional if peer verification is disabled)


## Generating Certificates For Development

The easiest way to generate a CA, server and client keys and certificates is by using
[tls-gen](https://github.com/ruby-amqp/tls-gen/). It requires `openssl` and `make` to be
available.

See [RabbitMQ TLS/SSL guide](http://www.rabbitmq.com/ssl.html) for more information
about TLS support on various platforms.


## Enabling TLS/SSL Support in RabbitMQ

TLS/SSL support is enabled using two arguments:

 * `ssl_listeners` (a list of ports TLS connections will use)
 * `ssl_options` (a proplist of options such as CA certificate file location, server key file location, and so on)

 An example:

``` erlang
[
  {rabbit, [
     {ssl_listeners, [5671]},
     {ssl_options, [{cacertfile,"/path/to/testca/cacert.pem"},
                    {certfile,"/path/to/server/cert.pem"},
                    {keyfile,"/path/to/server/key.pem"},
                    {verify,verify_peer},
                    {fail_if_no_peer_cert,false}]}
   ]}
].
```

Note that all paths must be absolute (no `~` and other shell-isms) and be readable
by the OS user RabbitMQ uses.

Learn more in the [RabbitMQ TLS/SSL guide](http://www.rabbitmq.com/ssl.html).

## Connecting to RabbitMQ from Bunny Using TLS/SSL

There are 3 options `Bunny.new` takes:

 * `:tls` which, when set to `true`, will set SSL context up and switch to TLS port (5671)
 * `:tls_cert` which is a string path to the client certificate (public key) in PEM format
 * `:tls_key` which is a string path to the client key (private key) in PEM format
 * `:tls_ca_certificates` which is an array of string paths to CA certificates in PEM format

An example:

``` ruby
conn = Bunny.new(:tls                   => true,
                 :tls_cert              => "examples/tls/client_cert.pem",
                 :tls_key               => "examples/tls/client_key.pem",
                 :tls_ca_certificates   => ["./examples/tls/cacert.pem"])
```

If you configure RabbitMQ to accept TLS connections on a separate port, you need to
specify both `:tls` and `:port` options.

Paths can be relative but it's recommended to use absolute paths and no symlinks
to avoid issues with path expansion.


### Providing Certificates & Keys When Using amqps:// URIs

It is possible to use `amqps://` URIs in combination with additional
options, e.g. to provide TLS certificate and key paths:

``` ruby
c = Bunny.new("amqps://bunny_gem:bunny_password@127.0.0.1/bunny_testbed",
        :tls_cert              => "spec/tls/client_cert.pem",
        :tls_key               => "spec/tls/client_key.pem",
        :tls_ca_certificates   => ["./spec/tls/cacert.pem"])
c.start
```

## Default Paths for TLS/SSL CA's on Linux

Bunny now will use the following TLS/SSL CA's paths on Linux by default:

 * `/etc/ssl/certs/ca-certificates.crt` on Ubuntu/Debian
 * `/etc/ssl/certs/ca-bundle.crt` on Amazon Linux
 * `/etc/ssl/ca-bundle.pem` on OpenSUSE
 * `/etc/pki/tls/certs/ca-bundle.crt` on Fedora/RHEL

and will log a warning if no CA files are available via default paths
or `:tls_ca_certificates`.


## What to Read Next

The documentation is organized as [a number of
guides](/articles/guides.html), covering various topics.


## Tell Us What You Think!

Please take a moment to tell us what you think about this guide [on
Twitter](http://twitter.com/rubyamqp) or the [Bunny mailing
list](https://groups.google.com/forum/#!forum/ruby-amqp)

Let us know what was unclear or what has not been covered. Maybe you
do not like the guide style or grammar or discover spelling
mistakes. Reader feedback is key to making the documentation better.
