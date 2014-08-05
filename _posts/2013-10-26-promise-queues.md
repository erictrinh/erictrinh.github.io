---
layout: post
title:  "Promise Queues"
---

A [lot](http://blog.jcoglan.com/2013/03/30/callbacks-are-imperative-promises-are-functional-nodes-biggest-missed-opportunity/) has [already](http://domenic.me/2012/10/14/youre-missing-the-point-of-promises/) been [written](http://modernjavascript.blogspot.co.uk/2013/09/promise-patterns.html) about using promises for async flow control[^async]. Here, I want to share a strategy for running async functions sequentially when they could be called from anywhere in your app.

Imagine you're creating an app where you're working purely with async functions. How do you ensure operation A completes before operation B?

```javascript
asyncA(); asyncB(); asyncC();
```

Since all these functions are asynchronous, there's no guarantee they'll run in the right order. If your functions return promises, however, you can chain them using `then`, like this:

```javascript
asyncA().then(asyncB).then(asyncC);
```

In this case, `asyncA` must **complete** before `asyncB` even begins, ensuring they are run in the correct order. Okay, but what if you've got a more complex app, where these async functions can be called from anywhere in your program?

Enter the promise queue. Surprisingly, once you've got (jQuery-style) promises in place, a basic promise queue implementation is pretty simple:

```javascript
var PromiseQueue = function() {
  var promise = $.Deferred().resolve();

  return {
    push: function(fn) {
      promise = promise.then(fn, fn);
      return this;
    }
  };
};
```

Use it like this:

```javascript
var prom = new PromiseQueue();
prom.push(asyncA);
prom.push(asyncB);
prom.push(asyncC);
```

Of course, we can `prom.push(asyncThing)` from anywhere in our app, which is the point. This guarantees that our functions will run in the order we `push` them. There is one caveat though: our queue moves onto the next async function even if a previous async function fails. We can make our queue smarter by automatically retrying failed async operations:

```javascript
var RetryingPromiseQueue = function() {
  // this always returns a resolved promise
  var resolved = function() {
    return $.Deferred().resolve();
  };
  // this always returns a rejected promise
  var rejected = function() {
    return $.Deferred().reject();
  };
  var promise = resolved();

  return {
    push: function push(fn) {
      promise = promise
        .then(fn)
        .then(resolved, fn)
        .then(resolved, resolved);
      return this;
    }
  };
};
```

This is definitely a little hairier, but still relatively easy to follow. Now let's make up some contrived functions to test it!

```javascript
var wait = function(time) {
  return function() {
    var def = $.Deferred();

    setTimeout(function() {
      console.log(time + ' is up.');
      def.resolve();
    }, time);

    return def.promise();
  };
};

var waitThenFail = function(time) {
  // same as wait except reject() the def
  // and print "time is up. fail."
};
```

These two higher-order functions simply return an async function that waits a certain amount of time, then resolves or rejects, printing to the console. Let's try using them:

```javascript
var fancyThang = new RetryingPromiseQueue();

fancyThang.push(wait(2000));
fancyThang.push(waitThenFail(1000));
fancyThang.push(waitThenFail(2000));
fancyThang.push(wait(1000));

// Console output:
// 2000 is up.
// 1000 is up. fail.
// 1000 is up. fail.
// 2000 is up. fail.
// 2000 is up. fail.
// 1000 is up.
```

As you can see, functions that fail are retried once before the queue moves on. This is really cool! If you've ever implemented retry using callbacks, you know it can get messy quickly and usually results in a one-off solution. Here, we get global intelligent retry for free, with **any** async function (as long as it returns a promise). Sure, you could implement your own promise-less async queue, but our super-fancy auto-retrying queue is 21 lines long with generous(ish) comments and whitespace.

Now go have some promise fun[^fun]!

[^async]: I first got my introduction to promises in Trevor Burnham's excellent book, [Async Javascript](http://pragprog.com/book/tbajs/async-javascript). If you've ever wondered about Javascript's event loop/single-threaded model or just want an introduction to organizing async code with promises, check it out.

[^fun]: A few follow-up ideas: extending this idea even further, we could implement a queue that retries failed ajax requests every `x` minutes, or perhaps even using an exponential back-off algorithm. Another weakness of our promise queue is that if one async operation is super slow (let's say it takes 20 seconds), it will block the rest of the tasks we have queued up. With a little work, you could implement a queue which would provide some kind of timeout, so that if a promise takes too long to resolve, it's rejected automatically (hint: nested promises).
