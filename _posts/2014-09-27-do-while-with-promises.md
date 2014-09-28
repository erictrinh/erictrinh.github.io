---
layout: post
title:  "While Loops with Promises"
---

[If you couldn't tell before]({% post_url 2013-10-26-promise-queues %}), I think promises are a great abstraction for doing complex asynchronous work in Javascript.

On a recent project, I needed to poll an endpoint using AJAX repeatedly (until I got the response I wanted). This can be useful as a poor man's long polling[^longpoll], for processes that take a long time for the server to perform. In this case, the server would respond with `{status: "processing"}` until it finished performing a task. Then it would respond with `{status: "success"}` or `{status: "failure"}` when the task was done. So, we'd need to poll every 10 seconds until it returned "success" or "failure." This is essentially an asynchronous do-while loop, and the cool thing is, it can be completely encapsulated into a single promise:

```javascript
function asyncDoWhile(delay, fns, promise) {
  var prom = promise || $.Deferred(),
    doFn = fns.evalFn,
    successPredicateFn = fns.isSuccess || function() { return true; },
    failurePredicateFn = fns.isFailure || function() { return false; },
    timeout;

  doFn()
    .done(function() {
      if (successPredicateFn.apply(this, arguments)) {
        prom.resolve.apply(prom, arguments);
      } else if (failurePredicateFn.apply(this, arguments)) {
        prom.reject.apply(prom, arguments);
      } else {
        timeout = setTimeout(function() {
          asyncDoWhile(delay, fns, prom);
        }, delay);
      }
    })
    .fail(function() { prom.reject.apply(prom, arguments); });

  var result = prom.promise();
  result.abort = function() {
    clearTimeout(timeout);
  };

  return promise;
};
```

Use it like this:

```javascript
asyncDoWhile(10000, {
  // a function that returns a promise
  evalFn: function() {
    return $.get('/some/api_endpoint');
  },
  // the results of evalFn are passed through these functions
  isSuccess: function(data) {
    return data.status === 'success';
  },
  isFailure: function(data) {
    return data.status === 'failed';
  }
}).then(function(data) {
  // do something with the data
});
```

This will poll `/some/api_endpoint` every 10 seconds as long as neither `isSuccess` nor `isFailure` return true. `asyncDoWhile` returns a promise, too, which means you can chain it with `then`, for when you are done polling and want to show a success message, for example.

[^longpoll]: Of course, if you're depending a lot on this type of behavior in production, it might be worth it to just implement long polling or use websockets or something.
