---
layout: post
title:  "Java Concurrency: Future vs Promise, Part III"
date: 2016-08-14 06:25:06 -0700
comments: true
---

Use Promise To Combine Async Requests
=====================================

In <a href="https://nzhong.github.io/blog/2016/08/java-concurrency-future-vs-promise-2">Part II</a> we talked about
Future vs CompletableFuture(Java's version of Promise). Now let's take a look at some examples of how 
we can actually use them.

Some of the most common real world scenarios that keeps popping up, have to deal with multiple async requests.

- One is <b>serial</b>: we need the result of "request 1" to feed into "request 2", and the result of "request 2"
into "request 3", and so on. So request 1,2,3 have to be run in that order, 
and one can not start before the prior one ended. 

- The other one is 
<b>parallel</b>: we need all the results from "request 1.1", "request 1.2", ... "request 1.N" before we could
proceed to request 2, while "1.1", "1.2", ... "1.N" are independent of each other, thus can be executed in parallel.

For each of these two challenges, we will first take a look at how we used to solve them without Promise, 
and then, with.

&nbsp;

Multiple Async Requests - Serial
================================

&nbsp;

#### Setup

For this post, let's use a simple setup: we have three slow requests, each one being slow by doing a 
`Thread.sleep(2000);` Also I have intentionally made the input & output of each request different and chain-able:

- request 1 output a String, 
- request 2 take a String and output a int (length of input), 
- and request 3 take a int, and output a String.

```
public class AsyncSerial {
  private static String slow_request_1() {
    try {
      Thread.sleep(2000);
    } catch (InterruptedException e) { e.printStackTrace(); }
    return "result from request 1";
  }

  private static int slow_request_2(final String input) {
    try {
      Thread.sleep(2000);
    } catch (InterruptedException e) { e.printStackTrace(); }
    return input==null ? 0 : input.length();
  }

  private static String slow_request_3(int input) {
    try {
      Thread.sleep(2000);
    } catch (InterruptedException e) { e.printStackTrace(); }
    return "the length of request 1 result ="+input;
  }

}
```

&nbsp;

#### Blocking version

The blocking version is the easiest:

```
public class AsyncSerial {
  . . . 

  public static void main(String[] args) {
    // // blocking version
    String result1 = slow_request_1();
    int result2 = slow_request_2(result1);
    String result3 = slow_request_3(result2);
    System.out.println( result3 );
  }
}
```

and after about 6 seconds, we see the output `the length of request 1 result =21`

&nbsp;

#### Callback version

It's at this stage that we realized that Future is not very helpful in this scenario. Because of the serial nature
of the call chain, we need one request to finish before calling the next. So if we want to do this in anything
that resemble some kind of async fashion, we could use 'callback-style': 
pass in the 'next request when this one is done' as an argument to the current request.

Unfortunately we need to modify our existing requests with the additional callback as a parameter. And
here is where it gets ugly:

```
  private static String slow_request_1_withCB(
        final BiFunction<String, Function<Integer, String>, String> cb1,
        Function<Integer, String> cb2) {
    String request1Output = slow_request_1();
    return cb1.apply( request1Output, cb2 );
  }

  private static String slow_request_2_withCB(final String input, final Function<Integer, String> callback) {
    int request2Output = slow_request_2(input);
    return callback.apply( request2Output );
  }

  private static String slow_request_3_withCB(int input) {
    String request3Output = slow_request_3(input);
    return request3Output;
  }

  public static void main(String[] args) {
    // callback (hell) version
    System.out.println(slow_request_1_withCB(AsyncSerial::slow_request_2_withCB, AsyncSerial::slow_request_3_withCB));
  }
```

As you can see, the problem is that the chaining of the calls have to be written into the logic. 
When we call 'request 1' not only do we have to pass in 'request 2' as a parameter, we also have to supply 'request 3'
as a parameter to 'request 1'! This is because at the end of request 1, when 1 calls 2, 
1 needs to tell 2 that when <i>it</i> finishes <i>it</i> needs to call 3. In other words 'request 1' needs to know 
not only its immediate successor, but the whole chain.

&nbsp;

#### Promise (CompletableFuture) version

The good news is, the CompletableFuture version is pretty much as simple as the blocking version, while remain async.

```
  public static void main(String[] args) {
    // promise version
    CompletableFuture.supplyAsync( AsyncSerial::slow_request_1 )
        .thenApply( AsyncSerial::slow_request_2 )
        .thenApply( AsyncSerial::slow_request_3 )
        .thenAccept( System.out::println );
    System.out.println("to prove it is async, this line should be printed before the result from above is printed");
  }
```

As we can see, the power & flexibility of CompletableFuture is in full display here. The `CompletableFuture.supplyAsync`
turns a regular blocking call into an Async Promise, and you can easily chain multiple requests by 
`.thenApply` and `.thenAccept`.

The difference between `.thenApply` and `.thenAccept` is that 

- `.thenApply` takes a <a href="https://docs.oracle.com/javase/8/docs/api/java/util/function/Function.html">Function< T,R ></a> which 
has both an input and an output. So it's usually meant for a <b>middle</b> transformation.
- `.thenAccept` takes a <a href="https://docs.oracle.com/javase/8/docs/api/java/util/function/Consumer.html">Consumer< T ></a> which just has 
an input. So it's usually meant for the <b>last</b> transformation.

A complete Java source code can be found 
<a href="https://github.com/nzhong/future-vs-promise/blob/master/src/main/java/com/learn/promise/AsyncSerial.java">here</a>.


&nbsp;

Multiple Async Requests - Parallel
==================================

&nbsp;

#### Setup

Let's again use a simple setup: we have three slow requests, this time each one being slow by doing a 
`Thread.sleep(n);` but sleep for a random 1 - 2 seconds. After the sleep, request 1, 2, 3 will return String "1", "2", "3".
This time we have to wait for all three requests to complete, before proceeding.

```
public class AsyncParallel {
  private static SecureRandom secureRandom = new SecureRandom();
  static {
    secureRandom.setSeed( System.currentTimeMillis() );
  }
  private static void waitForAWhile() {
    try {
      long sleepTimeInMilli = 1000L + 100L*secureRandom.nextInt(10);  // 1-2 seconds
      Thread.sleep(sleepTimeInMilli);
    } catch (InterruptedException ie) { ie.printStackTrace(); }
  }

  private static String slow_request_1() {
    waitForAWhile();
    return "1";
  }
  private static String slow_request_2() {
    waitForAWhile();
    return "2";
  }
  private static String slow_request_3() {
    waitForAWhile();
    return "3";
  }
}
```

&nbsp;

#### Blocking version

Again, the blocking version is simple:

```
public class AsyncParallel {
  . . . 

  public static void main(String[] args) {
    // // blocking version
    String s1 = slow_request_1();
    String s2 = slow_request_2();
    String s3 = slow_request_3();
    System.out.println( s1+s2+s3 )
  }
}
```

and after about 3-6 seconds, we see the output `123`. Of course though simple, this code is really bad.
We are wasting a lot of time waiting for each request to wake up from their sleep. In fact we are waiting for the <b>sum</b>
of their sleep time.


&nbsp;

#### CompletionService version

Since Java 7, Oracle introduced <a href="https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionService.html">CompletionService</a>,
where we can submit multiple requests, and these requests can run in parallel, and we can get the result back not in the order of submission but 
in the order of who finishes first.

```
  private static final ExecutorService THREAD_POOL = Executors.newCachedThreadPool();

  public static void main(String[] args) {
    // completionService version
    final ExecutorCompletionService<String> completionService = new ExecutorCompletionService<>(THREAD_POOL);
    completionService.submit( AsyncParallel::slow_request_1 );
    completionService.submit( AsyncParallel::slow_request_2 );
    completionService.submit( AsyncParallel::slow_request_3 );
    int numFetched = 0;
    StringBuilder buf = new StringBuilder();
    while (numFetched<3) {
      try {
        final Future<String> future = completionService.take();
        buf.append(future.get());
        ++numFetched;
      } catch (InterruptedException ie) {
        break;
      }
      catch (Throwable e) {
        ++numFetched;
      }
    }
    System.out.println( buf.toString() );
  }
```

If we run the above code several times, we will see that sometimes the printout is "123", sometimes it could be "132", "312", and so on.
This is because completionService.take() will return in the order of who finishes first.

Though the above code is longer than the blocking version, we now run requests in parallel, and the total wait time is the slowest request,
instead of the sum of all requests. Much more efficient!

&nbsp;

#### Promise version

As usual we saved the best for last. With CompletableFuture we can achieve the efficiency of the CompletionService version, while
keeping the blocking version's simplicity:

```
public static void main(String[] args) {
  // promise version
  CompletableFuture<String> cf1 = CompletableFuture.supplyAsync( AsyncParallel::slow_request_1 );
  CompletableFuture<String> cf2 = CompletableFuture.supplyAsync( AsyncParallel::slow_request_2 );
  CompletableFuture<String> cf3 = CompletableFuture.supplyAsync( AsyncParallel::slow_request_3 );
  CompletableFuture.allOf(cf1, cf2, cf3).thenApply(v -> {
    System.out.println( cf1.join()+cf2.join()+cf3.join() );
    return null;
  });
  System.out.println("to prove it is async, this line should be printed before the result from above is printed");
}
```

A complete Java source code can be found 
<a href="https://github.com/nzhong/future-vs-promise/blob/master/src/main/java/com/learn/promise/AsyncParallel.java">here</a>.

***

- <a href="https://nzhong.github.io/blog/2016/07/java-concurrency-future-vs-promise-1">Part I: Future explained</a>
- <a href="https://nzhong.github.io/blog/2016/08/java-concurrency-future-vs-promise-2">Part II: Future vs Promise</a>
- <b>Part III (this article): Use Promise To Combine Async Requests</b>


<br/><br/><br/>
<br/><br/><br/>