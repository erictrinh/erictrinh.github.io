---
layout: post
title:  "Yummy, Bacony Adventures"
---

I've been trying to come up with an excuse to use [Bacon.js](https://github.com/baconjs/bacon.js) somewhere in my projects since I learned about functional reactive programming. Bacon allows you to perform transforms on immutable event streams, an abstraction that is independent of *time itself*. Chew on that.

One of the things I've been playing with on and off is building a window manager using the excellent [Zephyros](https://github.com/sdegutis/zephyros) project, an app that allows you to tap into global shortcuts to resize windows, run shell scripts, etc. using whatever language you prefer (like Javascript via [Node](https://github.com/danielepolencic/node-zephyros)).

One of the things I wanted to implement was dual-key shortcuts. For example, pressing the up and right arrow keys in quick succession would result in a window being resized to the top-right quarter of the screen. This could be implemented via a "dumb" `KeyEmitter`, an `EventEmitter` that simply emits whatever key was pressed. For example, when I press the `q` key, the `KeyEmitter` emits the string `"q"`. A layer sits on top of the `KeyEmitter`, listening for key presses and transforming those events into more complex shortcut combinations.

Here's a preliminary implementation (`vent` is the more complex `EventEmitter` we'll eventually export):

{% highlight javascript %}
var lastKey = '';
var lastTime = Date.now();

keyEmitter.on('key', function(key) {
  var now = Date.now();

  var dualKey = lastKey + ' ' + key;

  if (now - lastTime < 300) {
    vent.emit('shortcut', dualKey);
  } else {
    vent.emit('shortcut', key);
  }

  lastKey = key;
  lastTime = Date.now();
});
{% endhighlight %}

We're keeping track of the last key pressed, as well as when the last key was pressed. Whenever a new key is pressed, we diff the time between this key press and the last key press, emitting a dual-key string if the time between key presses is less than 300 milliseconds. For example, pressing `k` then `b` in succession will result in the string `"k b"` being emitted. This works, but check out the equivalent code using Bacon:

{% highlight javascript %}
var keys = Bacon.fromEventTarget(keyEmitter, 'key'),
  times = keys.map(function() { return Date.now(); }),
  dualKeys = keys.diff('', function(a, b) { return a + ' ' + b; }),
  timeDiffs = times.diff(Date.now(), function(a, b) { return b - a; });

Bacon.onValues(keys, dualKeys, timeDiffs, function(key, dualKey, diff) {
  vent.emit('shortcut', diff < 300 ? dualKey : key);
});
{% endhighlight %}

Dense, but shorter and more expressive. We're creating a few new event streams here. The first is `times`, which emits the current time for each key press. We're only interested in `timeDiffs` though, the time between key presses. We also create the event stream `dualKeys`, which is just a stream consisting of the last key and the current key pressed, concatenated together with a space in between[^diff].

`Bacon.onValues` essentially zips together `keys`, `dualKeys`, and `timeDiffs`, so that when there are values available from each stream (which occurs every time there's a key press), it passes the stream values into the given function. There, we do our time diff logic to check if the keys were pressed within 300 milliseconds of each other.

So what's the difference here? In the first example, we're keeping track of *state*. We create variables to keep track of what was last pressed and when. Those variables change as the program runs, and their values are highly dependent on when we observe them. Bacon's event stream transforms are declarative, so we don't need to think about state. This isn't such a big deal now since our program is so small, but eliminating state is generally good because it means our code is more maintainable in the long run. And as you saw, working with Bacon's event stream abstraction also allows for better expressiveness because it allows us to think about what we want instead of how we get it.

[^diff]: You might be wondering what the first argument to Bacons `diff` function is. That's the seed value for our diff stream. Which means that the first time a key is pressed (say, `f`), the string emitted from `dualKeys` will be `" f"` (concatenating an empty string, a space, and "f"), which is nonsensical as a key combo. We could transform the stream to skip the first combo emitted so we don't have this problem (using Bacon's `skip` function), but that introduces other problems with syncing the 3 streams we need to read from in the `onValues` callback. Fortunately, for our purposes, we don't really care about this edge case.
