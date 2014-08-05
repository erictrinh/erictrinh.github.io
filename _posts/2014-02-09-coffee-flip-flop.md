---
layout: post
title:  "Coffeescript Flip Flop"
---

I've been flip-flopping on using Coffeescript in my projects lately. On one hand, it's shiny, expressive, and it's Just Javascript&trade;. On the other hand, it's yet another compile-to-JS abstraction that doesn't necessarily play well with other tools in my stack (JSX, Node, a plethora of debug tools) and often introduces context switching (for example, when switching between using an editor and the browser REPL).

What I've decided for now is to do most of my work in Javascript. It's simply a matter of it being the most compatible with the current tools out there. However, I was recently working on an older project in which I had used Coffeescript, and I came upon this gem:

```coffeescript
toQueryString = (hash) ->
  encode = window.encodeURIComponent
  ("#{encode(key)}=#{encode(val)}" for key, val of hash).join('&')
```

It's a function that takes a hash like `{a: 1, b: "boom shaka laka"}` and turns it into a url-encoded query string, like `a=1&b=boom%20shaka%20laka`[^global]. The conciseness is a little bit terrifying, especially compared to the equivalent Javascript:

```javascript
var toQueryString = function(hash) {
  var encode = window.encodeURIComponent,
    result = [],
    key;

  for (key in hash) {
    var val = hash[key];
    result.push(encode(key) + '=' + encode(val));
  }

  return result.join('&');
};
```

The elegance of the Coffeescript solution comes down to two things: **string interpolation** and (for lack of a better term) **everything-as-an-expression**.

String interpolation isn't the sexiest Coffeescript feature, but I sorely miss it when I'm Javascripting. For example, I weep uncontrollably when I have to write this:

```javascript
"Hello, " + name + "! Your codename is '" + codeName + ".'"
```

instead of this:

```coffeescript
"Hello, #{name}! Your codename is '#{codeName}.'"
```

Of course, this isn't as big of a gamechanger as Coffeescript's loop comprehensions. The `for...of` loop is an expression that returns an array, which is `join`ed into a string. We can't do this in a functional way in Javascript, because there's no native method for `map`ing over an object[^arrays], which means we have to use Javascript's `for...in` to mutate temporary variables instead.

There are far more language features found in Coffeescript that make it a joy to use[^favs]. I'm pretty sure I'll be sticking with Javascript, but every time I sit down to write Coffeescript, it sways me a little more towards compile-to-JS[^es6].

[^global]: Of course, this could easily be expressed as a one-liner with no reference to `window` but I like to make it explicit that I'm "importing" a function from the global scope.

[^arrays]: ES5 added native `map`, `filter`, and `reduce` methods to Arrays, thus mostly eliminating the need for imperative `for` loops. Unfortunately, you can't yet `map` over an object. At least, not without using a Javascript library like [Underscore](http://underscorejs.org/) or [Lo-Dash](http://lodash.com/).

[^favs]: Destructuring assignment and the existential operator are my other favorites. But I don't want to turn this into the [Coffeescript docs](http://coffeescript.org/).

[^es6]: ECMAScript Harmony (aka ES6) will improve things, of course. Coffeescript features like destructuring assignment will be coming in the next ECMAScript, but that's still a ways away, and when do you think we'll be able to write pure ES6 without worrying about older browser support? That's the promise of compile-to-JS languages, of course, but tooling support is a little sparse at the moment.
