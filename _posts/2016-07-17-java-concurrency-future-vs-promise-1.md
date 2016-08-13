---
layout: post
title:  "Java Concurrency: Future vs Promise, Part I"
date: 2016-07-17 16:25:06 -0700
tags:  Java Concurrency Async
comments: true
---

Future
======

<a href="https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html">Future</a> was introduced in Java 7. It represents the result of an asynchronous computation that may or may not have completed. One way to think of it is like this:

```
  Future<Double> futureResult = some_very_slow_calculation( ... );
  // since the some_very_slow_calculation( ... ) method is async, we get the
  // Future<Double> (thus reach here) immediately, usually before the calculation is done.

  /* why wait? I can go work on something else! */
  do_some_other_work();

  // now maybe the slow calculation is done?
  Double result = futureResult.get();  // this is a BLOCKING call
```

Of course we rarely use Future in the above manner, since it is too wishful thinking. What if do_some_other_work() needs the result of the calculation? What if do_some_other_work() finishes too quickly, and we are back at

```
  Double result = futureResult.get();
```

which is blocking?

Fortunately Future also offers get() with timeout, isDone(), and cancel(). One of the more common ways to use Future, is actually to use <a href="https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/FutureTask.html">FutureTask</a>, which implements Future. We could do something like this:

```
  // private static final ExecutorService THREAD_POOL = Executors.newFixedThreadPool(8);

  FutureTask<Double> ft = new FutureTask<Double>( ()->very_slow_blocking_calculation( ... ) );
  try {
      THREAD_POOL.execute(ft);
      Double result = ft.get(1000, TimeUnit.MILLISECONDS);  // 1 second timeout
      // continue to do other work, with result
      . . .
  } catch (Exception e) {
      e.printStackTrace();
      ft.cancel(true);
  }
```

Or, if we don't want to give up so easily (let's say the very_slow_blocking_calculation is important), we could

```
  // private static final ExecutorService THREAD_POOL = Executors.newFixedThreadPool(8);

  FutureTask<Double> ft = new FutureTask<Double>( ()->very_slow_blocking_calculation( ... ) );
  try {
      THREAD_POOL.execute(ft);
      while( !ft.isDone() ) {
          // while we wait, we can go work on something else
          do_some_other_work();
      }
      Double result = ft.get();  // this will return immediately
      . . .
  } catch (Exception e) {
      e.printStackTrace();
      ft.cancel(true);
  }
```

A complete Java source code can be found 
<a href="https://github.com/nzhong/future-vs-promise/blob/master/src/main/java/com/learn/promise/FutureSimpleGet.java">here</a>.

***

- <b>Part I (this article): Future explained</b>
- <a href="https://nzhong.github.io/blog/2016/08/java-concurrency-future-vs-promise-2">Part II: Future vs Promise</a>
- <a href="https://nzhong.github.io/blog/2016/08/java-concurrency-future-vs-promise-3">Part III: Use Promise to combine multiple async requests, serial or parallel</a>

<br/><br/><br/>
<br/><br/><br/>