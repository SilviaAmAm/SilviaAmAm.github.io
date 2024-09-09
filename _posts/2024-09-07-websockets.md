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
WSGI applications are synchronous callables that take a request forwarded by the WSGI server and return a response.

This is not suitable for long lived connections like WebSocket.

For this reason, the Asynchronous Server Gateway Interface (ASGI) standard was introduced. ASGI is a superset of WSGI designed to support forwarding requests to asynchronous capable Python applications. [https://en.wikipedia.org/wiki/Asynchronous_Server_Gateway_Interface]

An ASGI compatible application is an asynchronous program that takes a "scope" (details about the connection), a callable to send messages and a callable to receive messages.


## Django Channels

Django applications are WSGI compatible, but not ASGI compatible. Therefore, the Django Channels project was started. [https://test-channels.readthedocs.io/en/latest/] Django Channels aims to make Django ASGI compatible so that, among other things, it can handle WebSocket connections. It was designed to be compatible with "normal" Django, so that standard WSGI applications keep working.

Django works by taking an incoming request, routing it down to the right `View` which handles the request and produces a response.

In Django Channels the core concept is a _channel_. The channel is an "_ordered, first-in first-out queue with message expiry and at-most-once delivery to only one listener at a time_". [https://test-channels.readthedocs.io/en/latest/concepts.html#what-is-a-channel]

When an event occurs, "producers" load messages into channels. "Consumers" are listening for messages on particular channels. Based on the type of the message that comes in, Django Channels looks at a routing table and starts an instance of the right consumer. Each consumer is associated with a unique channel name that can be used to communicate with the client. When the consumer is instantiated, it receives a scope (information about the connection), a callable to send messages to the client and a callable to receive messages. [https://github.com/django/channels/blob/643d0832698d5306c01927add7b4aa34da1c457d/channels/consumer.py#L27]

When the consumer receives a message it can execute some code and/or send additional messages.  

When there are multiple instances of the same consumer at the same time, it must be possible to send messages to all the associated channels. For this, there is the concept of a "channel group". You can write your consumers so that whenever a connection is opened, the unique channel name is added to a group. Then, whenever an event happens, a messages is sent to all the channels that are in that group.


In the case of HTTP requests, there is then an `AsgiRequest` adapter class that transforms the message into a Django request, and then the request goes through the normal routing and arrives at a particular view.
For websockets, it is also possible to route the messages to the right consumer based on the path of the incoming request. 

Note:
When in the documentation they refer to "channel layers" I think that they refer to the Python code that handles receiving a message and then sending it "somewhere". For example, when using the RedisChannelLayer, this layer receives the messages and handles sending/retrieving them to the Redis backend.


In this new architecture, the HTTP request/response cycle looks as follows:

- A request comes in from a client.
- The request is transformed into a "message" and is loaded into the http channel (queue).
- The consumer listening on the channel for HTTP requests picks up the message.
- The consumer runs some code and at the end loads a new message into the queue (the equivalent of a response).
- This "response" message is picked up and sent to the client.

However, now it is possible to support other flows! Data can be sent to the client as a result of events. For example:
- A client opens a WebSocket connection with the server.
- Some process is doing some work and when it finishes it loads a message onto the queue.
- A consumer is listening for messages. It picks up the message and sends it to the client.
