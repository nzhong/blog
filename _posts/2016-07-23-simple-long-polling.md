---
layout: post
title:  "Simple Long Polling"
date: 2016-07-23 16:25:06 -0700
comments: true
---


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
        timeout: 60000 
    });
})();
```

The above code sends an AJAX request, and when the call is completed (either success, or error, which includes timeout), will fire off another request by recursively calling itself.

In the real world, we may want to differentiate the success and failure cases however. If the server is down for example, the above client code will fire off a million requests non-stop, one after another, since each request will be completed immediately with `net::ERR_EMPTY_RESPONSE` or `net::ERR_CONNECTION_REFUSED`. One simple solution is something like the following:

* If the request is completed successfully, fire off another request immediately.
* If the request fails with timeout (the timeout is set by client JS), it means there is no push event from server. fire off another request immediately.
* If the request fails with something other than timeout, wait for a while (for the server to recover), and then fire off another request.

The above logic looks something like this:

```
(function poll(){
    $.ajax({ 
        url: "server-url",
        success: function(data, status, jqXHR) {
            // do something with the data
            setTimeout( poll, 10 );
        },
        error: function(jqXHR, status, errorThrown) {
            if (status=='timeout') {
                console.log( 'request timed out.' );
                setTimeout( poll, 10 );
            }
            else {
                console.log(status);
                setTimeout( poll, 60000 );
            }
        },
        dataType: "json",
        timeout: 60000
    });
})();

```



Long Polling: Server Side Java
==============================

For this post, for simplicity, we will use embedded Jetty. Before we start, here is a simple routine that I use to simulate a long running process:

```
    private static SecureRandom secureRandom = new SecureRandom();
    static {
        secureRandom.setSeed( System.currentTimeMillis() );
    }

    private static String waitFor5To15Seconds() {
        long start = System.currentTimeMillis();
        try {
            long sleepTimeInMilli = 5000L + 1000L*secureRandom.nextInt(10);  // 5 - 15 seconds
            Thread.sleep(sleepTimeInMilli);
        } catch (InterruptedException ie) {
            ie.printStackTrace();
        }
        long end = System.currentTimeMillis();
        return "After "+(end-start)+"ms the server responds with ECHO.";
    }
```

Here we simply take a random number between 5 to 15 seconds, Thread.sleep, and return how long we spent. Please note that this waitFor5To15Seconds() rountine is a BLOCKING call.

Now the simplest embedded Jetty code for a <b>traditional</b> non-async servlet looks like this:

```
public class TestServer {
    . . . 
    private static String waitFor5To15Seconds() {
        // as above
    }

    public static class BlockingServlet extends HttpServlet
    {
        @Override
        protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException
        {
            try {
                resp.getWriter().write( waitFor5To15Seconds() );
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws Exception
    {
        Server server = new Server(9000);
        ServletContextHandler context = new ServletContextHandler();
        context.setContextPath("/");
        ServletHolder blockingHolder = context.addServlet(BlockingServlet.class,"/block");
        server.setHandler(context);
        server.start();
        server.join();
    }
}
```

If you run the above Java code, and in your browser load the URL "http://127.0.0.1:9000/block", after 5 - 15 seconds the browser will display something like "After 9004ms the server responds with ECHO."

But this style of servlet:

```
        @Override
        protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException
        {
            try {
                resp.getWriter().write( waitFor5To15Seconds() );
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
```

will not work well with Long Polling. Since each servlet handler will make a blocking call to wait for the next push event, and most servlet engines uses a limited thread pool to handle incoming requests, we will face <b>Thread Starvation</b> very soon, with no available threads to handle new incoming requests.

Fortunately <b>Asynchronous Servlets</b> has been introduced since servlet spec 3.0. Instead of waiting for the processing to be finished, the servlet handler simply start another new worker thread, pass along an instance of <b>AsyncContext</b> to the worker thread, and be done. The servlet handler thread is returned to the pool to handle other incoming requests, and with the <i>AsyncContext</i> instance, the new worker thread has everything it needs to complete the request.

The Async version of the above code looks like this:

```
    public static class EmbeddedAsyncServlet extends HttpServlet
    {
        @Override
        protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException
        {
            final AsyncContext asyncCtx = req.startAsync();
            asyncCtx.start( new Runnable(){
                public void run()
                {
                    ServletResponse response = asyncCtx.getResponse();
                    try {
                        response.getWriter().write( waitFor5To15Seconds() );
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                    asyncCtx.complete();
                }
            });
        }
    }

    public static void main(String[] args) throws Exception
    {
        . . .
        ServletHolder blockingHolder = context.addServlet(BlockingServlet.class,"/block");

        ServletHolder asyncHolder = context.addServlet(EmbeddedAsyncServlet.class,"/async");
        asyncHolder.setAsyncSupported(true);
        . . .
    }
```

If you want to test this, similar to the blocking version, try loading the URL "http://127.0.0.1:9000/async" in your browser. After 5 - 15 seconds the browser should display something like "After 7003ms the server responds with ECHO."



GitHub Repo
===========

I have put the above code into a simple github repo: <a href="https://github.com/nzhong/simple-long-polling">https://github.com/nzhong/simple-long-polling</a>. There are really just two files

* <a href="https://github.com/nzhong/simple-long-polling/blob/master/webapps/HTML/index.html">webapps/HTML/index.html</a> contains the client side javascript code
* <a href="https://github.com/nzhong/simple-long-polling/blob/master/src/main/java/com/learn/jetty/AsyncServer.java">src/main/java/com/learn/jetty/AsyncServer.java</a> contains the server side java code.

To try it out, you can do the following:

* git clone https://github.com/nzhong/simple-long-polling
* cd simple-long-polling
* mvn clean package
* java -jar target/simple-long-polling-1.0-SNAPSHOT.jar
* Open up your chrome browser, and load "http://127.0.0.1:9000/static/index.html"
* Open your Chrome's Developer Tools -> Console, and see the messages.


I tweaked the client side timeout a bit, just to add some varieties. The server side will wait randomly between 5-15 seconds, and the client side make its AJAX requests timeout = 12 seconds. So in the Chrome console you should see something like

* After 7002ms the server responds with ECHO.
* After 6004ms the server responds with ECHO.
* 2 request timed out.
* After 9003ms the server responds with ECHO.
* ...

The server side AsyncServer.java was also tweaked a bit, just so that the program can listen to three URLs:

* http://127.0.0.1:9000/block
* http://127.0.0.1:9000/async
* http://127.0.0.1:9000/static/*.html



Source Material
===============

The following links, among others, are what educated me about long polling:

* <a href="https://www.pubnub.com/blog/2014-12-01-http-long-polling/">What is HTTP Long Polling?</a>
* <a href="https://techoctave.com/c7/posts/60-simple-long-polling-example-with-javascript-and-jquery">Simple Long Polling Example with JavaScript and jQuery</a>
* <a href="https://www.javacodegeeks.com/2013/08/async-servlet-feature-of-servlet-3.html">Async Servlet Feature of Servlet 3</a>
* <a href="http://stackoverflow.com/questions/12555043/my-understanding-of-http-polling-long-polling-http-streaming-and-websockets">My Understanding of HTTP Polling, Long Polling, HTTP Streaming and WebSockets</a>



<br/><br/><br/>
<br/><br/><br/>