---
layout: post
title:  "Why are my Backbone sub-view events gone?"
---

I stumbled into a head-scratching bug the other day while writing a Backbone view. The view in question had a sub-view, something like this:

```javascript
var ChildView = Backbone.View.extend({
  events: {
    'click .button': 'log'
  },

  log: function(e) {
    e.preventDefault();
    console.log('I got clicked, ma!');
  },

  // ..
});

var ParentView = Backbone.View.extend({
  initialize: function() {
    this.child = new ChildView();
  },

  // ..
  render: function() {
    this.$el.html(this.template(this));
    this.$('.container').html(this.child.render().el);
    return this;
  }
});
```

Whenever the parent renders, it also re-renders its child view. This seems like perfectly correct code, and if you run this, it seems to work. If you click `.button`, it'll log to the console, as expected.

The problem is if your parent view needs to re-render for any reason, you'll be in for a surprise. See, when the parent view re-renders, all of the child view's events will be wiped out. You'll still see all of the child's HTML rendered, but now, clicking `.button` won't do anything. This has to do with jQuery's `html` function, which wipes out all events inside the target element, like a pompous jerk.

An easy fix is to use jQuery's `detach` method, a fun easter egg that allows you to detach document fragments from the DOM without wiping out their events. Let's re-write the parent view's `render` method using `detach`:

```javascript
render: function() {
  this.child.$el.detach();

  this.$el.html(this.template(this));
  this.$('.container').html(this.child.render().el);
  return this;
}
```

Now it works like a charm!
