---
layout: post
title:  "React + MongoDB + Socket.io = Magic"
---

In React, you can model your application state as one Javascript object[^flux]. As it turns out, this is *absolutely fantastic* for easy server-side persistence and real-time syncing via websockets. In this post, I'll go over a strategy to easily do just that with [MongoDB](http://www.mongodb.org/) and [Socket.io](http://socket.io/).

This assumes a server that can
1. Handle RESTful commands (GET, POST, PUT) at the `/message` endpoint
2. Broadcasts the socket event `new state` when it's `emit`ed by the client

Let's start off with a stupidly simple React app.

```javascript
var App = React.createClass({
  getInitialState: function() {
    return { message: '' };
  },

  handleChangeMessage: function(e) {
    this.setState({ message: e.target.value });
  },

  render: function() {
    <input type="text" value={this.state.message} onChange={this.handleChangeMessage} />
  }
});
```

Our app consists of an exciting text box where you can type things. Cool, right? Now let's get fancier by adding/modifying a few methods in our React component.

```javascript
// ...
componentDidMount: function() {
  // grab state from the server
  $.ajax({ url: '/message' })
    .then(function(data) {
      this.setState(data);
    }.bind(this))
  // listen for state changes on the socket
  socket.on('new state', function(newState) {
    this.setState(newState);
  }.bind(this));
},

networkSetState: function(newStateDiff) {
  // do some awesome network things here
  // 1. put the entire state into the database
  this.saveStateToDB();
  // 2. put diffs onto the websocket
  this.postToSocket(newStateDiff);
  // 3. set state as per usual
  this.setState(newStateDiff);
},

postToSocket: function(newStateDiff) {
  socket.emit('new state', newStateDiff);
},

saveStateToDB: _.debounce(function() {
  $.ajax({ url: '/message', type: 'PUT', data: this.state });
}, 5000),

handleChangeMessage: function(e) {
  this.networkSetState({ message: e.target.value });
},
// ...
```

This seems like a lot of work, but believe it or not, you will never[^never] have to write more code for sync or server-side persistence as your app gets more complex.

You can use this as a normal React component, except you have to remember to call `networkSetState` instead of plain ol' `setState`. `networkSetState` is just a wrapper around `setState` that happens to also notify the server and websockets of any state changes.

Now if you open two browser windows side by side and type a message in one window, you'll see the other window updating in real-time. When you stop typing, the state will be saved to the server. 

This is cool because it scales to more sophisticated apps. If you use this as a top-level component in your React app, you can get incredibly complex apps that auto-sync and auto-save like pure unicorn magic. Just worry about client-side state changes, and those changes are persisted and updated everywhere.

[^flux]: Whether this is a scalable approach is up for contention. Facebook recommends you use [Flux](http://facebook.github.io/flux/docs/overview.html) for larger applications.
[^never]: Never say never, right? This example is a very simplistic persistence strategy, and you might want to update it if you want to do something more complex.
