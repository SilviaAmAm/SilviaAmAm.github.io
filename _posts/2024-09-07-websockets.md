---
layout: post
title:  "Django and WebSocket"
date:   2024-09-07 08:52:11 +0100
categories: Software
mathjax: false
---

# Use case

If you have a webpage where you want information to be updated automatically, without having to refresh the page, there are a few options available:

* Short polling: the client makes a request to the server every couple of seconds to check if there is an update. This has the following disadvantages:

- It creates a lot of traffic to the server
- Depending on how long the interval is between the requests, there can be a delay between when the data changes and when it is updated on the webpage.

* Long polling: the client makes a request to the server and the server keeps this connection open until it has data to communicate back to the client. Once the server responds, the connection is closed. This has the following disadvantages:

- There is the overhead of opening a new connection for every data exchange.
- It can be resource intensive, since while the server is waiting to return data it is essentially "hanging".

* WebSocket: a 2-way persistent connection is opened between the client and the server. The server can send data to the client without needing a request first. This has the advantage of achieving real time updates while using less resources, but it has the following disadvantages:

- A more complex setup is needed.
- Old browsers don't support WebSocket.

# WebSocket

WebSocket is a communication protocol. It provides a 2-way simultaneous communication channel over a single TCP connection.

It is different from HTTP, but it was designed to work over HTTP ports 443 and 80 and it supports going through HTTP proxies and intermediaries. This means that it is compatible with HTTP: to start a WebSocket connection you make an HTTP request with the `HTTP Upgrade` header. [https://en.wikipedia.org/wiki/WebSocket]

ADD Diagram

# Django and WebSocket

## WSGI vs ASGI

Web Server Gateway Inteface (WSGI) is a standard for running Python web applications. [https://www.fullstackpython.com/wsgi-servers.html] 

The WSGI standard defines the interface between the web server and the web application. So it has two sides: the "server" side and the "application" side. [https://peps.python.org/pep-3333/#specification-overview] The server side invokes a callable object that is provided by the application side. 

This is an example of a WSGI application [https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface]:

```python
def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/plain')])
    yield b'Hello, World!\n'
```

When a WSGI server invokes this application, it will return in the response "Hello, World!".

WSGI applications are synchronous callables that take a request forwarded by the WSGI server and return a response.
This is not suitable for long lived connections like WebSocket.

For this reason, the Asynchronous Server Gateway Interface (ASGI) standard was introduced. ASGI is a superset of WSGI designed to support forwarding requests to asynchronous capable Python applications. [https://en.wikipedia.org/wiki/Asynchronous_Server_Gateway_Interface]

An ASGI compatible application is an asynchronous program that takes a "scope" (details about the connection), a callable to send messages and a callable to receive messages.

This is an example of an ASGI application [https://asgi.readthedocs.io/en/latest/introduction.html]:

```python 
async def application(scope, receive, send):
    event = await receive()
    await send({"type": "websocket.send", "text": "Hello world!"})
```


## Django Channels

Django applications are WSGI compatible, but not ASGI compatible. Therefore, the Django Channels project was started. [https://test-channels.readthedocs.io/en/latest/] Django Channels aims to make Django ASGI compatible so that, among other things, it can handle WebSocket connections. It was designed to be compatible with "normal" Django, so that standard WSGI applications keep working.

Django works by taking an incoming request, routing it down to the right `View` which handles the request and produces a response.

Django Channels works around the concept of events. When an event happens, a message is put into a queue. Consumers listen for these messages and can execute some code and/or send additional messages.

There are a few key concepts and vocabulary that is used in the Django Channles project:

- Channel: The channel is an "_ordered, first-in first-out queue with message expiry and at-most-once delivery to only one listener at a time_". [https://test-channels.readthedocs.io/en/latest/concepts.html#what-is-a-channel]. 
- Producers: This is anyone loading messages into a channel.
- Consumers: Consumers are the entities listening for messages on particular channels. Based on the type of message that comes in, Django Channels looks at a routing table and starts an instance of the right consumer. Each consumer is associated with a unique channel name that can be used to communicate with the client, since each consumer will live until the connection is closed. When the consumer is instantiated, it receives a scope (information about the connection), a callable to send messages and a callable to receive messages. [https://github.com/django/channels/blob/643d0832698d5306c01927add7b4aa34da1c457d/channels/consumer.py#L27]
- Channel groups: When there are multiple instances of the same consumer at the same time, it must be possible to send messages to all the associated channels. For this, there is the concept of a "channel group". You can write your consumers so that whenever a connection is opened, the unique channel name is added to a group. Then, whenever an event happens, a messages can be sent to all the channels that are in that group.
- Channel layer: Python code that handles loading messages into a queue and receiving messages from it. For example, when using the RedisChannelLayer, this layer receives the messages and handles sending/retrieving them to the Redis backend [https://github.com/django/channels_redis/blob/13cef4502a95fbb7f41bb97eaa6f4d02c1f7c91c/channels_redis/core.py#L98].

Note:
There are 2 major "types" of channels:
- Channel for dispatching messages to consumers. When a message gets added to a channel, a worker picks it up and spins up a consumer. 
- A "reply" channel. Only the interface server listens on the reply channel and then sends messages back to the client. [https://test-channels.readthedocs.io/en/latest/concepts.html#channel-types]

## Deploying Django Channels applications

There are more than one way to deploy Deploying Django Channels applications. Some options:

1. Exactly like normal Django apps. If no channel layer is specified, Django Channels apps work just like WSGI apps. However, it will only support HTTP traffic.
2. Only with an ASGI server. All traffic, including HTTP traffic, goes through the ASGI server.
3. With both an ASGI and a WSGI server. In this case, the WSGI server can handle the HTTP traffic, while WSGI server handles the WebSocket and/or HTTP/2 traffic.

For options 2. and 3., a few components are needed. In the Django Channels documentation, they mention [https://test-channels.readthedocs.io/en/latest/deploying.html#deploying]:

1. An "interface" server
2. A channel backend
3. Worker servers

### Interface servers

The interface server is an ASGI compatible web server. It has the role of receiving incoming requests and loading messages into the channels.

A common choice for the interface server is Daphne [https://github.com/django/daphne/].

According to the documentation, it may be preferable to route HTTP traffic to a WSGI web server while routing all WebSocket and HTTP/2 to the ASGI server. This is because the ASGI specification and Daphne are relatively new, so it is still less "battle tested" [https://test-channels.readthedocs.io/en/latest/deploying.html#running-asgi-alongside-wsgi]. In this case, a reverse proxy (like nginx) needs to be added. It takes incoming requests and routes the HTTP traffic to the WSGI server and other traffic to the ASGI server.

It is important to run the interface server inside some program that can take care of restarting the process when it exits for whatever reason.

### Channels backend

One has to choose a backend supported by one of the Channel Layers [https://test-channels.readthedocs.io/en/latest/backends.html]. The recommended backend is Redis. 

### Workers

These are mini-ASGI compatible applications that talk to the Channels backend. They listen on either specified channels or all channels and can perform some work, including spinning up consumers to handle requests. The work of running consumers is decoupled from the work of talking to clients, you can run  multiple "worker servers” to do additional processing of messages.

It is important to run workers inside some program that can take care of restarting the process when it exits, since the worker itself has no retry-on-exit logic.

