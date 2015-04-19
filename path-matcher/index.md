# Simple Request Path Matcher

In this lesson we will implement a way to limit some middlewares to run only if they match specified paths.

# Matching A Path

By default a middleware would run for any request path:

```js
app.use(function(req,res) {
  res.end("ok");
});
// GET / => ok
// GET /foo => ok
// GET /bar => ok
```

We can limit it to run only if the request path matches a path prefix:

```js
app.use("/foo",function(req,res) {
  res.end("ok");
});
// GET / => 404
// GET /foo => ok
// GET /foobar => 404
// GET /foo/bar => ok
// GET /bar => 404
```

# Running Lesson Tests

The test repo is at `https://github.com/hayeah/fork2-myexpress-verify`

You should `git pull` to get new tests.

# Implement Request Path Matcher

Our goal is to extend `app.use` so it takes an optional path argument:

```js
// use this middleware only for "/foo/*"
app.use("/foo",function(req,res) {
  res.end("foo!");
});
```

To help us to implement this functionality, we will create a new class called `Layer`. When `app.use` adds a middleware, it creates a Layer and adds it to the `app.stack`. In `next` a middleware should be called only if the request path matches the layer path.

The Layer class works like this:

```js
// create a new layer
layer = new Layer("/foo",middleware);

// layer's handle property is the middleware for the layer
layer.handle === middleware

// layer's match method returns undefined is request path doesn't match
layer.match("/bar") // => undefined

// layer's match method returns an object if the request path matches
layer.match("/foo") // => {path: "/foo"}

// layer's matches if it is the prefix of the request path
layer.match("/foo/bar") // => {path: "/foo"}
```

**Implement**: Layer class and the `match` method.

Create the Layer class in `lib/layer.js`.

Example:

```
layer = new Layer("/foo",middleware);
layer.handle             // => middleware

// path matching
layer.match("/bar");     // => undefined
layer.match("/foo");     // => {path: "/foo"}
layer.match("/foo/bar"); // => {path: "/foo"}
```

Pass: `mocha verify -R spec -g 'Layer class and the match method'`

```
Layer class and the match method
  ✓ sets layer.handle to be the middleware
  ✓ returns undefined if path doesn't match
  ✓ returns matched path if layer matches the request path exactly
  ✓ returns matched prefix if the layer matches the prefix of the request path
```

**Implement**: `app.use` should create a Layer and add it to `app.stack`.

```js
// should create a layer with path "/"
app.use(middleware);
// should create a layer with path "/foo"
app.use("/foo",middleware);
```

Pass: `mocha verify -R spec -g 'app.use should add a Layer to stack'`

```
app.use should add a Layer to stack
  ✓ first layer's path should be /
  ✓ second layer's path should be /foo
```

**Implement**: Should call a middleware if the layer matches the request path.

Examples:

1 The middlewares called should match request path

```js
app.use("/foo",function(req,res,next) {
  res.end("foo");
});

app.use("/",function(req,res) {
  res.end("root");
});
// GET / => root
// GET /foo => foo
// GET /foo/bar => foo
```

Hint: use `request.url` to get the current request's path.

Pass: `mocha verify -R spec -g 'The middlewares called should match request path'`

```
The middlewares called should match request path:
  ✓ returns root for GET /
  ✓ returns foo for GET /foo
  ✓ returns foo for GET /foo/bar
```

2 The error handlers called should match request path

```js
app.use("/foo",function(req,res,next) {
  throw "boom!"
});

app.use("/foo/a",function(err,req,res,next) {
  res.end("error handled /foo/a");
});

app.use("/foo/b",function(err,req,res,next) {
  res.end("error handled /foo/b");
});

// GET /foo/a => error handled /foo/a
// GET /foo/b => error handled /foo/b
// GET /foo   => 500
```

**Commit**

```
git commit
```

