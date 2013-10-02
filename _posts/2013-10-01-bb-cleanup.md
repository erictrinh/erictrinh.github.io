---
layout: post
title:  "Cleaning Up Global Views in Backbone"
---

In Javascript web apps, there's a fun thing called a memory leak that happens to the best of us from time to time. These bundles of joy usually occur when we've forgotten to clean up dead views, unbinding the necessary event handlers.

In Backbone, this amounts to calling `.remove()` on a view after we're done with it, thus annihilating it from the DOM and causing it to `stopListening` on any models it may have been listening to. Simple enough. The tricky bit is knowing when to actually call `remove`. In this post, I'll show you how I deal with cleaning up global views[^gviews].

For my own apps, I've often relied on a global event dispatcher to signal when my views should be cleaned up. That method would work something like this:

{% highlight javascript %}
window.vent = _.clone(Backbone.Events);

window.MyRouter = Backbone.Router.extend({
  // ..

  edit: function() {
    vent.trigger('pageunload');
    var editView = new EditView();

    // ..

    vent.once('pageunload', function() {
      editView.remove();
    });
  }

});
{% endhighlight %}

At the end of every routing function, I bind my cleanup function to the "pageunload" event. This is triggered when the next routing function is called (via `vent.trigger`), thus calling my cleanup function.

If you've been paying attention, this means we have some boilerplate at the top *and* bottom of all of our routing functions, which is a little annoying to remember to do for every single route function. Maybe we can DRY this up?

Well, check this out:

{% highlight javascript %}
window.MyRouter = Backbone.Router.extend({

  // we're overwriting the route method here
  // so we can get a "beforeroute" event
  route: function(route, name, callback) {
    if (!callback) callback = this[name];
    callback = _.wrap(callback, function(fun) {
      this.trigger('beforeroute');
      fun.apply(this, arguments);
    });

    Backbone.Router.prototype.call(this, route, name, callback);
  },

  // this is simply a wrapper around this.once('beforeroute', ...)
  cleanup: function(fun) {
    this.once('beforeroute', _.bind(fun, this));
  }

  // ..the rest of your router functions
});
{% endhighlight %}

It's short, but dense. What we've done is create a "beforeroute" event[^bfroute] that is triggered before any route function is executed. Our `cleanup` function leverages this to run our cleanup code right before the next route is matched.

This allows us to write our routing functions like this instead:

{% highlight javascript %}
edit: function() {
  var editView = new EditView();

  // ..

  this.cleanup(function() {
    editView.remove();
  });

}
{% endhighlight %}

In this version, half of our boilerplate is gone, and the actual cleanup phase is way more semantic[^bonus].

I've found that this pattern greatly increases the readability of my Backbone router code. The code practically reads like a book! It solves the problem of view cleanup in a flexible way, and you don't have to rely on any global event dispatchers, which is a nice bonus.

[^gviews]: What's a global view? I define a global view as a view that is directly created in your route function. These views tend to be larger in scope and may contain a variety of sub-views. Cleaning up sub-views, however, is a whole different animal and is out of scope for this post.

[^bfroute]: It's surprising that the "beforeroute" event isn't built into Backbone. At first glance you might think you could implement cleanup using the "route" event, which *is* built in. Unfortunately, that event is triggered *after* each route function, which means that your cleanup function will run right after you bind it to the "route" event. Any view you set up in your route function will immediately be removed, which is an excellent exercise in futility.

[^bonus]: Here's an extra idea for cleanup. My cleanup functions almost always consist of `someView.remove()`. For bonus points, try implementing something that will allow you to pass view instances into `this.cleanup` that will automatically call `.remove()` on the view. Something like `this.cleanup(someView1, someView2)` should remove the respective views from the DOM. It's a cool idea I may re-visit in a subsequent blog post.
