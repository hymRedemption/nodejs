# Route Chaining

In this lesson we'll modify `makeRoute` to support advanced route chaining.

# Express app.route Chaining API

In the previous lesson `makeRoute` accepts only one request handler. Express' route object can accept many handlers.

We can add multiple HTTP verb handlers to a route:

```js
route = app.route("/foo");
// add a get handler to the /foo route
route.get(function(req,res) {
  res.end("got foo");
});
// add a post handler to the /foo route
route.post(function(req,res) {
  res.end("posted to foo");
});

// GET /foo => got foo
// POST /foo => posted to foo
```

The above can be written using the **route chaining API**:

```js
app.route("/foo")
  .get(function(req,res) {
    res.end("got foo");
  })
  .post(function(req,res) {
    res.end("posted to foo");
  });
```

In fact, `app.get(path,handler)` is the same as `app.route(path).get(handler)`.

# Skipping Handlers In A Route

Within a route a handler can call `next("route")` to skip all the remaining handlers. For example:

```
// the first route
app.route("/")
  .get(function(req,res,next) {
  // go to the next handler
    next();
  })
  .get(function(req,res,next) {
    // skip the remaining handlers
    next("route");
  })
  .get(function(req,res,next) {
    // this is skipped
  })

// the second route
app.route("/").get(function(req,res) {
  res.end("second route's handler");
})
// GET / => second route's handler
```

# Same Same But Different

You can add many handlers to a route, and you can also add many handlers to an app as middleware. How are routes different from apps?

1. Handlers in a route are called depending on whether the `request.method` matches the HTTP verb.
2. Handlers (aka middlewares) in an app are called depending on whether the `request.path` matches the Layer path.

Also, `next('route')` only works within a route. Let's see how `next` is different between an app and a route.

For a route:

```js
// Adding two handlers a route
var route = app.route("/foo");
route.get(handler1);
route.post(handler2);
app.use(handler3);
app.use(error_handler);
```

+ calling `next()` in handler1 goes to handler2
+ calling `next('route')` in handler1 will skip handler2 and go to handler3

And for two different routes:

```js
// Create two different routes
app.use("/foo",handler1);
app.use("/foo",handler2);
app.use(handler3);
app.use(error_handler);
```

+ calling `next()` in handler1 goes to handler2
+ calling `next('route')` in handler1 will go to error_handler

# How Is Route Chaining Useful?

In this example, we create a route that loads the forum and thread data before serving any HTTP action:

```js
// load forum data
function loadForum(req,res,next) {
  // in a real app, this would be async
  res.locals.forum = getForum();
  next();
}

// load thread data
function loadThread(req,res,next) {
  res.locals.thread = getThread();
  next();
}

app.route('/forum/:fid/thread/:tid')
  .all(loadForum)
  .all(loadThread)
  .get(function() { ... })
  .post(function() { ... });
```

`route.all` adds a handler that runs for any HTTP verb, so `get` and `post` are able to use the data loaded by the `all` handlers.

# How Is next('route') Useful?

Suppose we want to authenticate a user before serving a request, we can do it like this:

```js
function authUser() {
  if(isUserLogin() == true) {
    next();
  } else {
    next("route");
  }
}

app.route('/forum/:fid/thread/:tid')
  .all(authUser)
  .all(loadForum)
  .all(loadThread)
  .get(function() { ... })
  .post(function() { ... });
app.use(signinPage);
```

If `isUserLogin()` is not true then all the remaining handlers in the route are skipped, and go to `signinPage`.

# Running Lesson Tests

The test repo is at `https://github.com/hayeah/fork2-myexpress-verify`

You should `git pull` to get new tests

# Implmentation Plan

A route is a function (like app):

```
function(req,res,next) {
  ...
}
```

And like an app, we can add multiple handlers to it:

```
route.all(handler0);
route.get(handler1);
route.post(handler2);
```

Since a route is a request handler (i.e. `function(req,res,next)`), we can use it as a middleware:

```
app.use(route);
```

In the first part of this lesson, we'll focus on implementing route.

In the second part, we'll work on `app.route` and `app.VERB`.


# Implement Multiple Handlers

A route, like an app, also has a stack of handlers. In this section, we'll implement support for adding multiple handlers to a route.

First, we'll need to change `makeRoute(verb,handler)` from previous lesson to `makeRoute()`.

**Implement**: `route.use(verb,handler)` adds a handler to the stack

Example:

```
route = makeRoute();
route.use("get",handler1);
route.use("post",handler2);
route.stack // =>
[{verb: "get", handler: handler1},
 {verb: "post", handler: handler2}]
```

Pass: `mocha verify/route_spec.js -R spec -g "Add handlers to a route"`

```
Add handlers to a route:
  ✓ adds multiple handlers to route
  ✓ pushes action object to the stack
```

# Implement Route Handlers Invokation

Calling a route (`route(req,res,next)`) should invoke handlers added to it, in a similar way as how an app would invoke its middlewares.

**Implement**: Calling the route should invoke the middlewares in the stack

**1 be able to go to the next handler by calling next**

```js
route.use("get",function(req,res,next) {
  next();
});
route.use("get",function(req,res) {
  res.end("handler2");
});
app.use(route);

// GET / => handler2
```

Pass: `mocha verify/route_spec.js -R spec -g "calling next"`

```
Implement Route Handlers Invokation:
  calling next():
    ✓ goes to the next handler
    ✓ exits the route if there's no more handler
```

**2 error handling**

To make the implementation simpler, we won't support error handlers in route.

+ A thrown error would be handled by the app.
+ A error passed into next (`next(err)`) should immediately exit the app.

```js
route.use("get",function(req,res,next) {
  next(new Error("boom!")); // will exit the route
});
route.use("get",function(req,res) {
  // won't get here
});
app.use(route);
// GET / => 500
```

Pass: `mocha verify/route_spec.js -R spec -g "error handling"`

```
Implement Route Handlers Invokation:
  error handling:
    ✓ exits the route if there's an error
```

**3 should match HTTP verbs**

```js
route.use("get",function(req,res) {
  res.end("got");
});
route.use("post",function(req,res) {
  res.end("posted");
});

app.use(route);

// GET   /  => got
// POST  /  => posted
// PUT   /  => 404
```

Pass: `mocha verify/route_spec.js -R spec -g "verb matching"`

```
Implement Route Handlers Invokation:
  verb matching:
    ✓ matches GET to the get handler
    ✓ matches POST to the post handler
```

**4 the "all" verb should match any VERB**

```js
route.use("all",function(req,res) {
  res.end("all");
});
app.use(route);

// GET   /  => all
// POST  /  => all
```

Pass: `mocha verify/route_spec.js -R spec -g "match any verb"`

```
Implement Route Handlers Invokation:
  match any verb:
    ✓ matches POST to the all handler
    ✓ matches GET to the all handler
```

**5 calling next('route') should skip the remaining handlers**

```js
route.use("get",function(req,res,next) {
  next('route');
});
route.use("get",function() {
  // won't get here
});
app.use(route);
app.use(function(req,res) {
  res.end("middleware");
});
// GET / => middleware
```

Pass: `mocha verify/route_spec.js -R spec -g "calling next\(route\)"`

```
Implement Route Handlers Invokation:
  calling next(route):
    ✓ skip remaining handlers
```

# Implement Verbs For Route

Let's generate all the http verbs for route. For example:

```js
route.get(handler);
// is the same as
route.use("get",handler);
```

Also, it should be possible to chain the http verbs:

```js
route
  .get(handler1)
  .post(handler2)
  .put(handler3);
```

Which is the same as:

```js
route.get(handler1);
route.post(handler2);
route.put(handler3);
```

**Implement**: route.VERB

Hint: aside from the verbs in the `methods` package, you should also add "all"

Pass: `mocha verify/route_spec.js -R spec -g "Implement Verbs For Route"`

```
Implement Verbs For Route
  ✓ should respond to get
  ✓ should respond to post
  ✓ should respond to put
  ✓ should respond to head
  ✓ should respond to delete
  ✓ should respond to options
  ✓ should respond to trace
  ✓ should respond to copy
  ✓ should respond to lock
  ✓ should respond to mkcol
  ✓ should respond to move
  ✓ should respond to propfind
  ✓ should respond to proppatch
  ✓ should respond to unlock
  ✓ should respond to report
  ✓ should respond to mkactivity
  ✓ should respond to checkout
  ✓ should respond to merge
  ✓ should respond to m-search
  ✓ should respond to notify
  ✓ should respond to subscribe
  ✓ should respond to unsubscribe
  ✓ should respond to patch
  ✓ should respond to search
  ✓ should respond to all
  ✓ should be able to chain verbs
```

# Implement app.route

Each time `app.route(path)` is called a new Layer is created and added to the `app.stack`.

**Implement**: app.route creates a new route

Example:

```
app.route("/foo")
  .get(function(req,res) {
    res.end("got foo");
  })
  .post(function(req,res) {
    res.end("posted foo");
  });
```

Hint: Remember, only one route is created, and `.VERB` adds handler to the route.

Pass: `mocha verify/route_spec.js -R spec -g "Implement app.route"`

```
Implement app.route
  ✓ can create a new route
```

# Implement Verbs For App

`app.VERB(path,handler)` is just a shortcut for `app.route(path).VERB(handler)`. We can create two different routes by chaining:

```js
app.get("/foo",fooHandler)
  .get("/bar",barHandler);
```

Which is equivalent to:

```js
app.route("/foo").get(fooHandler);
app.route("/bar").get(barHandler);
```

**Implement**: app.VERB creates a new route and add one single handler to it.

Hint: use `app.route` to implement this

Hint: aside from the verbs in the `methods` package, you should also add "all"

Pass: `mocha verify/route_spec.js -R spec -g "Implement Verbs For App"`

```
Implement Verbs For App
  ✓ creates a new route for get
  ✓ creates a new route for post
  ✓ creates a new route for put
  ✓ creates a new route for head
  ✓ creates a new route for delete
  ✓ creates a new route for options
  ✓ creates a new route for trace
  ✓ creates a new route for copy
  ✓ creates a new route for lock
  ✓ creates a new route for mkcol
  ✓ creates a new route for move
  ✓ creates a new route for propfind
  ✓ creates a new route for proppatch
  ✓ creates a new route for unlock
  ✓ creates a new route for report
  ✓ creates a new route for mkactivity
  ✓ creates a new route for checkout
  ✓ creates a new route for merge
  ✓ creates a new route for m-search
  ✓ creates a new route for notify
  ✓ creates a new route for subscribe
  ✓ creates a new route for unsubscribe
  ✓ creates a new route for patch
  ✓ creates a new route for search
  ✓ creates a new route for all
  ✓ can chain VERBS
```