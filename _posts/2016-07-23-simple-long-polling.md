---
layout: post
title:  "Simple Long Polling"
date: 2016-07-23 16:25:06 -0700
comments: false
---

Source Material:

* <a href="https://www.pubnub.com/blog/2014-12-01-http-long-polling/">What is HTTP Long Polling?</a>
* <a href="https://techoctave.com/c7/posts/60-simple-long-polling-example-with-javascript-and-jquery">Simple Long Polling Example with JavaScript and jQuery</a>
* <a href="https://www.javacodegeeks.com/2013/08/async-servlet-feature-of-servlet-3.html">Async Servlet Feature of Servlet 3</a>
* <a href="http://stackoverflow.com/questions/12555043/my-understanding-of-http-polling-long-polling-http-streaming-and-websockets">My Understanding of HTTP Polling, Long Polling, HTTP Streaming and WebSockets</a>

Server Push
===========

Our goal really is to implement <b>server push</b> over HTTP. Long Polling is just one method to achieve this, together with (regular) polling, HTTP streaming, and WebSockets:

(In the order of slowest to fastest)

* <b>(regular) Polling</b>: client sends AJAX request at regular interval (say every 1 sec), and server responds immediately. Thus  the client will be notified with at most (1 second) delay.
* <b>Long Polling</b>: client sends AJAX request, and server holds on to the request/response until it has an update (push event). When the client receives the push event it sends another AJAX request.
* <b>HTTP streaming</b>: Server responds with a header with "Transfer Encoding: chunked" and hence the client do not need to initiate a new request immediately.
* <b>WebSockets</b>: <a href="https://www.websocket.org/aboutwebsocket.html">About HTML5 WebSocket</a>

This post only focuses on Long Polling.

Long Polling: Client Side Javascript
====================================

The traditional client side javascript code that does long polling is as follows:

```
(function poll(){
    $.ajax({ 
        url: "server-url", 
        success: function(data){
            // do something with the data
        },
        dataType: "json",
        complete: poll, 
        timeout: 30000 
    });
})();
```






