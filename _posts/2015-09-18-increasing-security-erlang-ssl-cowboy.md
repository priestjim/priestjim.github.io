---
layout: post
title: Increasing security in Erlang and Elixir SSL applications
excerpt: "Increase the security of your Erlang and Elixir SSL applications."
tags: [ssl, erlang, security, cowboy]
modified: 2015-09-18
date: 2015-09-18
comments: true
---

With the recent outbreak of major security flaws in the OpenSSL library (**CRIME, FREAK, POODLE, logjam, BEAST** etc), the need to properly configure our application's SSL layer (whichever purpose that might be serving) is bigger than ever.

SSL in Erlang is a very particular kind of beast. An SSL socket will behave almost exactly the same way as a `gen_tcp` socket. _Almost_. But we're not here to talk about that.

Conventional operation of an `ssl`-based application in Erlang is relatively simple. You can start different SSL sockets, each with its own set of certificates, ciphers and a battery of other SSL-specific options. Naturally, these options can only be set at either the listen leven (`ssl:listen/2`) or the accept level (`ssl:ssl_accept/4`), are usually exposed by libraries that use the `ssl` application through an initialization option and essentially control, among other things, the **security guarantees** that this specicic library/socket will be providing.

Since the part of code where you define this configuration is different for every application that uses SSL, I'll present the options in a generic manner and then provide 2 application-specific examples (Cowboy and RabbitMQ). In general, SSL needs can be stricter or looser depending on what kind clients the application expects (users or machines). The options below will guarantee you an SSL setup that's **balanced** between security and compatibility.

## Generic configuration

{% highlight erlang %}
{versions, ['tlsv1.2', 'tlsv1.1', 'tlsv1']}
{% endhighlight erlang %}

The `versions` tuple defines the SSL/TLS versions that the server supports. Since the [POODLE](https://isc.sans.edu/forums/diary/SSLv3+POODLE+Vulnerability+Official+Release/18827/){:target="_blank"} attack, `SSLv3` has mostly been deprecated in favor of `TLSv1` and newer revisions. By strictly defining TLS in your configuration, you completely avoid SSLv3 exposure.

{% highlight erlang %}
{dhfile, "dh-params.pem"}
{% endhighlight erlang %}

The Ephemeral [Diffie-Helman](http://www.wikiwand.com/en/Diffie%E2%80%93Hellman_key_exchange){:target="_blank"} key exchange is a very effective way of ensuring Forward Secrecy by exchanging a set of keys that never hit the wire. Since the DH key is effectively signed by the private key, it needs to be at least as strong as the private key. In addition, the default DH groups that most of the OpenSSL installations have are only a handful (since they are distributed with the OpenSSL package that has been built for the operating system it's running on) and hence predictable (not to mention, 1024 bits only).

In order to escape this situation, first we need to generate a fresh, strong DH group, store it in a file and then use the option above, to force our SSL application to use the new DH group. Fortunately, OpenSSL provides us with a tool to do that. Simply run

    openssl dhparam -out dh-params.pem 2048

to generate a new DH group file of 2048 bits and store it in `dh-params.pem`.

{% highlight erlang %}
{secure_renegotiate, true}
{% endhighlight erlang %}

SSL parameter renegotiation is a feature that allows a client and a server to renegotiate the parameters of the SSL connection on the fly. [RFC 5746](http://www.ietf.org/rfc/rfc5746.txt){:target="_blank"} defines a more secure way of doing this. By enabling secure renegotiation, you drop support for the insecure renegotiation, prone to MitM attacks.

{% highlight erlang %}
{reuse_sessions, true}
{% endhighlight erlang %}

A performance optimization setting, it allows clients to reuse pre-existing sessions, instead of initializing new ones. Read more about it [here](http://vincent.bernat.im/en/blog/2011-ssl-session-reuse-rfc5077.html){:target="_blank"}.

{% highlight erlang %}
{honor_cipher_order, true}
{% endhighlight erlang %}

An important security setting, it forces the cipher to be set based on the server-specified order instead of the client-specified order, hence enforcing the (usually more properly configured) security ordering of the server administrator.

{% highlight erlang %}
{ciphers, ["ECDHE-ECDSA-AES256-GCM-SHA384","ECDHE-RSA-AES256-GCM-SHA384",
        "ECDHE-ECDSA-AES256-SHA384","ECDHE-RSA-AES256-SHA384", "ECDHE-ECDSA-DES-CBC3-SHA",
        "ECDH-ECDSA-AES256-GCM-SHA384","ECDH-RSA-AES256-GCM-SHA384","ECDH-ECDSA-AES256-SHA384",
        "ECDH-RSA-AES256-SHA384","DHE-DSS-AES256-GCM-SHA384","DHE-DSS-AES256-SHA256",
        "AES256-GCM-SHA384","AES256-SHA256","ECDHE-ECDSA-AES128-GCM-SHA256",
        "ECDHE-RSA-AES128-GCM-SHA256","ECDHE-ECDSA-AES128-SHA256","ECDHE-RSA-AES128-SHA256",
        "ECDH-ECDSA-AES128-GCM-SHA256","ECDH-RSA-AES128-GCM-SHA256","ECDH-ECDSA-AES128-SHA256",
        "ECDH-RSA-AES128-SHA256","DHE-DSS-AES128-GCM-SHA256","DHE-DSS-AES128-SHA256",
        "AES128-GCM-SHA256","AES128-SHA256","ECDHE-ECDSA-AES256-SHA",
        "ECDHE-RSA-AES256-SHA","DHE-DSS-AES256-SHA","ECDH-ECDSA-AES256-SHA",
        "ECDH-RSA-AES256-SHA","AES256-SHA","ECDHE-ECDSA-AES128-SHA",
        "ECDHE-RSA-AES128-SHA","DHE-DSS-AES128-SHA","ECDH-ECDSA-AES128-SHA",
        "ECDH-RSA-AES128-SHA","AES128-SHA"]}
{% endhighlight erlang %}

This is the single most important configuration option of an Erlang SSL application. Ciphers (and their ordering) define the way the client and server encrypt information over the wire, from the initial Diffie-Helman key exchange, the session key encryption algorithm and the message digest algorithm. Selecting a good cipher suite is critical for the application's data security, confidentiality and performance. The cipher list above offers:

* A good balance between compatibility with **older browsers**. It can get stricter for Machine-To-Machine scenarios.
* **Perfect Forward Secrecy**.
* No old/insecure encryption and HMAC algorithms

Most of it was copied from [Mozilla's Server Side TLS](https://wiki.mozilla.org/Security/Server_Side_TLS){:target="_blank"} article and then modified and it has been tested in production with **Erlang 18.1** but should work on the 17.x series as well, given a relatively modern OpenSSL installation (1.0.2d or newer).

All of the above options result in getting an **A** rating from [SSLLabs](https://globalsign.ssllabs.com/){:target="_blank"}, which is a good compromise between security and compatibility.

## Cowboy configuration

In Cowboy, you can enable these options as part of the https listener initialization routine:

{% highlight erlang %}
{ok, CowboyPid} = cowboy:start_https(
    cowboy_https_receiver,
    100,
    [
        {port, 443},
        {cacertfile,"/path/to/testca/cacert.pem"},
        {certfile,"/path/to/server/cert.pem"},
        {keyfile,"/path/to/server/key.pem"},
        {versions, ['tlsv1.2', 'tlsv1.1', 'tlsv1']},
        {dhfile, "/path/to/testca/dh-params.pem"},
        {ciphers, ["ECDHE-ECDSA-AES256-GCM-SHA384","ECDHE-RSA-AES256-GCM-SHA384",
            "ECDHE-ECDSA-AES256-SHA384","ECDHE-RSA-AES256-SHA384", "ECDHE-ECDSA-DES-CBC3-SHA",
            "ECDH-ECDSA-AES256-GCM-SHA384","ECDH-RSA-AES256-GCM-SHA384","ECDH-ECDSA-AES256-SHA384",
            "ECDH-RSA-AES256-SHA384","DHE-DSS-AES256-GCM-SHA384","DHE-DSS-AES256-SHA256",
            "AES256-GCM-SHA384","AES256-SHA256","ECDHE-ECDSA-AES128-GCM-SHA256",
            "ECDHE-RSA-AES128-GCM-SHA256","ECDHE-ECDSA-AES128-SHA256","ECDHE-RSA-AES128-SHA256",
            "ECDH-ECDSA-AES128-GCM-SHA256","ECDH-RSA-AES128-GCM-SHA256","ECDH-ECDSA-AES128-SHA256",
            "ECDH-RSA-AES128-SHA256","DHE-DSS-AES128-GCM-SHA256","DHE-DSS-AES128-SHA256",
            "AES128-GCM-SHA256","AES128-SHA256","ECDHE-ECDSA-AES256-SHA",
            "ECDHE-RSA-AES256-SHA","DHE-DSS-AES256-SHA","ECDH-ECDSA-AES256-SHA",
            "ECDH-RSA-AES256-SHA","AES256-SHA","ECDHE-ECDSA-AES128-SHA",
            "ECDHE-RSA-AES128-SHA","DHE-DSS-AES128-SHA","ECDH-ECDSA-AES128-SHA",
            "ECDH-RSA-AES128-SHA","AES128-SHA"]},
        {secure_renegotiate, true},
        {reuse_sessions, true},
        {honor_cipher_order, true},
        {max_connections, infinity}
    ],
    [{max_keepalive, 1024}, {env, [{dispatch, Dispatch}]}]).

{% endhighlight erlang %}

## RabbitMQ configuration

In RabitMQ, you can enable these options in the rabbitmq.config file:

{% highlight erlang %}
[
  {rabbit, [
     {ssl_listeners, [5671]},
     {ssl_options, [{cacertfile,"/path/to/testca/cacert.pem"},
                    {certfile,"/path/to/server/cert.pem"},
                    {keyfile,"/path/to/server/key.pem"},
                    {verify,verify_peer},
                    {fail_if_no_peer_cert,false},
                    {versions, ['tlsv1.2', 'tlsv1.1', 'tlsv1']},
                    {dhfile, "/path/to/testca/dh-params.pem"},
                    {ciphers, ["ECDHE-ECDSA-AES256-GCM-SHA384","ECDHE-RSA-AES256-GCM-SHA384",
                        "ECDHE-ECDSA-AES256-SHA384","ECDHE-RSA-AES256-SHA384", "ECDHE-ECDSA-DES-CBC3-SHA",
                        "ECDH-ECDSA-AES256-GCM-SHA384","ECDH-RSA-AES256-GCM-SHA384","ECDH-ECDSA-AES256-SHA384",
                        "ECDH-RSA-AES256-SHA384","DHE-DSS-AES256-GCM-SHA384","DHE-DSS-AES256-SHA256",
                        "AES256-GCM-SHA384","AES256-SHA256","ECDHE-ECDSA-AES128-GCM-SHA256",
                        "ECDHE-RSA-AES128-GCM-SHA256","ECDHE-ECDSA-AES128-SHA256","ECDHE-RSA-AES128-SHA256",
                        "ECDH-ECDSA-AES128-GCM-SHA256","ECDH-RSA-AES128-GCM-SHA256","ECDH-ECDSA-AES128-SHA256",
                        "ECDH-RSA-AES128-SHA256","DHE-DSS-AES128-GCM-SHA256","DHE-DSS-AES128-SHA256",
                        "AES128-GCM-SHA256","AES128-SHA256","ECDHE-ECDSA-AES256-SHA",
                        "ECDHE-RSA-AES256-SHA","DHE-DSS-AES256-SHA","ECDH-ECDSA-AES256-SHA",
                        "ECDH-RSA-AES256-SHA","AES256-SHA","ECDHE-ECDSA-AES128-SHA",
                        "ECDHE-RSA-AES128-SHA","DHE-DSS-AES128-SHA","ECDH-ECDSA-AES128-SHA",
                        "ECDH-RSA-AES128-SHA","AES128-SHA"]},
                    {secure_renegotiate, true},
                    {reuse_sessions, true},
                    {honor_cipher_order, true},
                    {max_connections, infinity}]}
   ]}
].
{% endhighlight erlang %}

# Phoenix configuration

The popular Elixir web framework [Phoenix](http://www.phoenixframework.org){:target="_blank"} uses Cowboy under the scenes.
In order to configure it to use the above options, we simply have to specify them in the appropriate configuration file (such as `config/prod.exs`):

{% highlight elixir %}
use Mix.Config

config :phoenix, SSLApp.Endpoint,
  https: [port: 443,
          host: 'sslapp.com',
          cacertfile: '/path/to/testca/cacert.pem',
          certfile: '/path/to/server/cert.pem',
          keyfile: '/path/to/server/key.pem',
          versions: [:'tlsv1.2', :'tlsv1.1', :'tlsv1'],
          dhfile: '/path/to/testca/dh-params.pem',
          ciphers: ['ECDHE-ECDSA-AES256-GCM-SHA384','ECDHE-RSA-AES256-GCM-SHA384',
                        'ECDHE-ECDSA-AES256-SHA384','ECDHE-RSA-AES256-SHA384', 'ECDHE-ECDSA-DES-CBC3-SHA',
                        'ECDH-ECDSA-AES256-GCM-SHA384','ECDH-RSA-AES256-GCM-SHA384','ECDH-ECDSA-AES256-SHA384',
                        'ECDH-RSA-AES256-SHA384','DHE-DSS-AES256-GCM-SHA384','DHE-DSS-AES256-SHA256',
                        'AES256-GCM-SHA384','AES256-SHA256','ECDHE-ECDSA-AES128-GCM-SHA256',
                        'ECDHE-RSA-AES128-GCM-SHA256','ECDHE-ECDSA-AES128-SHA256','ECDHE-RSA-AES128-SHA256',
                        'ECDH-ECDSA-AES128-GCM-SHA256','ECDH-RSA-AES128-GCM-SHA256','ECDH-ECDSA-AES128-SHA256',
                        'ECDH-RSA-AES128-SHA256','DHE-DSS-AES128-GCM-SHA256','DHE-DSS-AES128-SHA256',
                        'AES128-GCM-SHA256','AES128-SHA256','ECDHE-ECDSA-AES256-SHA',
                        'ECDHE-RSA-AES256-SHA','DHE-DSS-AES256-SHA','ECDH-ECDSA-AES256-SHA',
                        'ECDH-RSA-AES256-SHA','AES256-SHA','ECDHE-ECDSA-AES128-SHA',
                        'ECDHE-RSA-AES128-SHA','DHE-DSS-AES128-SHA','ECDH-ECDSA-AES128-SHA',
                        'ECDH-RSA-AES128-SHA','AES128-SHA'],
          secure_renegotiate: true,
          reuse_sessions: true,
          honor_cipher_order: true,
          max_connections: :infinity]
{% endhighlight elixir %}
