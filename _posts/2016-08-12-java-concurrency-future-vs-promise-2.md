---
layout: post
title:  "Java Concurrency: Future vs Promise, Part II"
date: 2016-08-12 06:25:06 -0700
comments: true
---

Future vs Promise
=================

In <a href="https://nzhong.github.io/blog/2016/07/java-concurrency-future-vs-promise-1">Part I</a> 
we briefly talked about the <a href="https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html">Future</a> 
interface in Java. In other programming languages such as Javascript, there is also 
<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise">Promise</a>, which is quite similar 
to Future, but are more powerful in the sense that you can chain them. Now Java 8 introduces 
<a href="https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html">CompletableFuture</a> which 
can be considered as Java's Promise. In this post I will use CompletableFuture and Promise interchangeably.

Now before diving into the features and usages of CompletableFuture, let us first consider the question: what's the relationship &
differences between Future and Promise? Other than that Future is an interface, and CompletableFuture is a class that implements it?

Both <i>Future</i> and <i>Promise</i> represents the result of an asynchronous computation result. The difference is that a Future 
is a <b>read-only</b> container of that result, while a Promise <b>can be updated</b> (completable). 

***

One way to imagine this difference, is we have a <b>Caller thread</b>, the Consumer of a (potentially) slow routine, and a 
<b>Callee thread</b>, which is the (async) Implementor of the slow routine:

```
Caller/Consumer  --- calls (async) --->  Callee/Implementor 
```

The <b>Caller/Consumer</b> calls the async routine, and much like in 
<a href="https://nzhong.github.io/blog/2016/07/java-concurrency-future-vs-promise-1">Part I</a>,
gets a <b>Future</b> back, and wait on the Future object till the result is in. From this side of the view, the Future object is read-only: 
all we can do is to wait. We could decide to wait for a limited time only (`.get(long timeout, TimeUnit unit)`), or even cancel it 
(`.cancel(boolean mayInterruptIfRunning)`), but we can't alter the result. Afterall, we are the caller, the uninfluential outsider.

```
  // Caller / Consumer
  public void someMethod() {
    Future<String> f = asyncSlowFetch();
    String result = f.get();
    System.out.println( "caller thread got result back: " + result );
  }
```

***

The <b>Callee/Implementor</b> usually construct a <b>CompletableFuture</b> instance, and returns it immediately, before starting on its 
slow and steady routine. When the slow routine is finished, the Callee/Implementor finishes the CompletableFuture instance with the 
`.complete(T value)` method. (and it is at this point in time, that the caller's .get() can return).
From this side of the view, we are dealing with a CompletableFuture object that the implementor can manipulate.

```
  private static final ExecutorService THREAD_POOL = Executors.newCachedThreadPool();

  // Callee / Implementor
  public Future<String> asyncSlowFetch() {
    CompletableFuture<String> promise = new CompletableFuture<>();
    THREAD_POOL.execute( ()-> {
      String result = slow_calculattion(...);
      promise.complete(result);
    } );
    return promise;
  }
```

A complete Java source code can be found 
<a href="https://github.com/nzhong/future-vs-promise/blob/master/src/main/java/com/learn/promise/CallerCallee.java">here</a>.

***

To hammer this point home, let's try another example. Let's say we have many threads all waiting for a single thread to finish, 
before they could continue. In the old days we could do something like this:

```
  private static final String LOCK = "LOCK";

  // in the N (waiting) threads
  synchronized (LOCK) {
    LOCK.wait();
  }

  // in the Single (holding) thread, to wake up everyone
  synchronized (LOCK) {
    LOCK.notifyAll();
  }
```

A complete Java source code can be found 
<a href="https://github.com/nzhong/future-vs-promise/blob/master/src/main/java/com/learn/promise/NThreadWaitLock.java">here</a> (notify/wait version).

Now with Future & Promise we could achieve similar behavior like this:

```
  private static final CompletableFuture<String> PROMISE = new CompletableFuture<>();

  // in the N (waiting) threads
  PROMISE.get();

  // in the Single (holding) thread, to wake up everyone
  PROMISE.complete("WAKE UP!!!");
```

A complete Java source code can be found 
<a href="https://github.com/nzhong/future-vs-promise/blob/master/src/main/java/com/learn/promise/NThreadWaitPromise.java">here</a> (future/promise version).

***

- <a href="https://nzhong.github.io/blog/2016/07/java-concurrency-future-vs-promise-1">Part I: Future explained</a>
- <b>Part II (this article): Future vs Promise</b>
- <a href="https://nzhong.github.io/blog/2016/08/java-concurrency-future-vs-promise-3">Part III: Use Promise to combine multiple async requests, serial or parallel</a>


<br/><br/><br/>
<br/><br/><br/>