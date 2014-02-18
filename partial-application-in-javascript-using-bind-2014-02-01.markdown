title: Partial Application in JavaScript using bind()

There is a pattern in JavaScript that I consider highly underused, because it leads to more concise code that is both easier to write and read. You probably all know of [Function.prototype.bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind). It's used most of the time to get rid of all these `var that = this` or `var self = this` assignments you used to see everywhere. A common example would be:

```javascript
this.setup = function () {
   this.on('event', this.handleEvent.bind(this));
};
```

The first argument passed to `bind` will serve as `this` within the scope of the function it returns. A lesser known property of `bind` is that it accepts more than one parameter. *Every parameter to `bind` after the first will be prepended to the list of parameters when invoking the bound function.*

That means we can create partially applied functions like this:

```javascript
var add = function (a, b) {
  return a + b;
};
var add2 = add.bind(null, 2);

add2(10) === 12;
```

Exciting, eh? It's more obvious when this can get helpful when we extend the initial event handling example. Another common pattern when handling events is that you want to provide some content when calling the handler:

```javascript
this.setup = function () {
  this.on('tweet', function (e, data) {
    this.handleStreamEvent('tweet', e, data);
  }.bind(this));
  this.on('retweet', function (e, data) {
    this.handleStreamEvent('retweet', e, data);
  }.bind(this));
};
```

If the event handlers for `tweet` and `retweet` share much of their logic, it's a good idea to structure your code like this. The downside, however, is obvious. You have a lot of boilerplate code. In both cases, you need to create an anonymous function, call the event handler in there, pass on the parameters and remember to bind the function so the `this` context is properly set up.

Can't we make this simpler? Indeed we can!

```javascript
this.setup = function () {
  this.on('tweet', this.handleStreamEvent.bind(this, 'tweet'));
  this.on('retweet', this.handleStreamEvent.bind(this, 'retweet'));
};
```

Beautiful, isn't it? So what happened here? Instead of calling the function within an anonymous wrapper, we create two partially applied functions that take the `this` context and two different first parameters for both of them. `e` and `data` will be passed on without us having to worry about it.

If you are like me a few months ago, this is the point where you raise your eyebrows in shock and go through your code to clean up all these occurrences. When you're done, please tell your friends.
