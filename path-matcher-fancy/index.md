# Fancy Request Path Matcher

In this lesson we will extend the path matching functionality so it can:

+ extract parameters from path
+ trim matched prefix before processing a nested app

# Parameter Matching

With express, you can specify parameters in the path:

```js
app.use("/foo/:a/:b",function(req,res) {
  res.end(req.params + ":" + JSON.stringify(req.params));
});
// GET / => 404
// GET /foo/apple => 404
// GET /foo/apple/samsung => ok:{"a":"apple","b":"samsung"}
// GET /foo/google/facebook => ok:{"a":"google","b":"facebook"}
// GET /foo/google/facebook/amazon => ok:{"a":"google","b":"facebook"}
```

`:a` and `:b` can be anything, and a request path would match. Path parameters are available to the handlers as `req.params`.

# Nested App

Nested app is used to combine two apps into a single one by mounting a subapp as a middleware. For example, a blog might be an app, and the blog's admin interface might be a separate app.

The blog app might be like:

```js
blogApp.use("/articles/:id",serveArticle);
blogApp.use("/",serveIndex);
```

The admin app might be like:

```js
adminApp.use("/articles/:id",manageArticle);
adminApp.use("/comments/:id",manageComment);
adminApp.use(adminDashBoard);
```

And to combine these two:

```js
// mount the admin app at the path /admin
blogApp.use("/admin",adminApp);
```

Now if we start `blogApp` as a web server, we can access `adminApp`  by using the `/admin` prefix:

```
GET /admin => adminDashBoard
GET /admin/articles/1 => manageArticle(req.params.id == 1)
GET /admin/comments/1 => manageComment(req.params.id == 1)
```

The blogApp can be accessed as usual:

```
GET / => serveIndex
GET /articles/1 => serveArticle(req.params.id == 1)
```

# The path-to-regexp Package

Install `path-to-regexp`:

```
npm install path-to-regexp@0.1.2
```

This package [path-to-regexp](https://www.npmjs.org/package/path-to-regexp) turns an Express-style path string such as `/user/:name` into a regular expression. A regular expression is a pattern that can be used to match strings to see if they fit a pattern. If you are unfamiliar, see: [CondeAcademy's Regular Expression Tutorial](http://www.codecademy.com/courses/javascript-intermediate-en-NJ7Lr/0/1).

The module exports a function that takes 3 arguments. See [documentation](https://github.com/component/path-to-regexp#pathtoregexppath-keys-options) for `pathToRegexp(path, keys, options)`.

Let's see a few examples.

**1 By default it returns a regex that matches the path exactly:**

```
var p2re = require("path-to-regexp");
re = p2re("/foo");
re;                   // re => /^\/foo\/?$/i
re.test("/foo");      // => true
re.test("/foo/");     // => true
re.test("/fooo");     // => false
re.test("/foo/bar");  // => false
```

**2 It can return a regex that parses parameters from a path:**

```
var p2re = require("path-to-regexp");
re = p2re("/foo/:a/:b");
re // => /^\/foo\/(?:([^/]+?))\/(?:([^/]+?))\/?$/i
```

That looks complicated. Let's break it down:

```
/^                # the beginning
  \/foo           # /foo
  \/(?:([^/]+?))  # /:a
  \/(?:([^/]+?))  # /:b
  \/?             # optional trailing slash
$/i               # the end
```

The symbols `^`, `$`, `\/`, `+`, `?` are all commonly seen in regex. Please see [Regular Expressions](https://developer.mozilla.org/en/docs/Web/JavaScript/Guide/Regular_Expressions) if you don't know what they mean.

`?:` is more rare. `()` is used to group a regexp pattern, and by default it "remebers" the matched string (see [Using Parentheses](https://developer.mozilla.org/en/docs/Web/JavaScript/Guide/Regular_Expressions#Using_Parentheses)). `(?:pattern)` groups the pattern without remembering the matched string.

We can use this regexp to match request paths:

```js
m = re.exec("/foo/apple/samsung")
// returns
[ '/foo/apple/samsung',
  'apple',
  'samsung',
  index: 0,
  input: '/foo/apple/samsung' ]

m[0] // the matched path
m[1] // the parameter :a
m[2] // the parameter :b
```

**3 It can tell us the name of the parameters:**

```js
var p2re = require("path-to-regexp");
var names = [];
re = p2re("/foo/:a/:b",names);
// the value of names is:
[ { name: 'a', optional: false },
  { name: 'b', optional: false } ]
```

It modifies the array `names` passed into it to return the arguments in the path.

This is a really weird API for Javascript, and smells like C. In general, it's a terrible idea to modify the parameter value passed in. Don't design your own API like this. If you need to return mutliple values, it would be better to return an object:

```js
p2re("/foo/:a/:b")
// returns
{re: /^\/foo\/(?:([^/]+?))\/(?:([^/]+?))\/?$/i,
 params: [ { name: 'a', optional: false },
           { name: 'b', optional: false } ]}
```

**4 It can return a regex that matches the prefix:**

We can use the `end` option to get a regex that matches the prefix instead the whole path:

```js
var names = [];
var re = p2re("/foo/:a/:b",names,{end: false});
re.test("/foo/apple/samsung");        // => true
re.test("/foo/apple/samsung/xiaomi"); // => true
```

For fun, you might be interested to see what the implementation is like. See [path-to-regexp/index.js](https://github.com/component/path-to-regexp/blob/821b3df30b3a4fadcf7f2b4021fbefddc840b28a/index.js). Ya, I don't understand that gibberish either.

# Running Lesson Tests

The test repo is at `https://github.com/hayeah/fork2-myexpress-verify`

You should `git pull` to get new tests.

# Implement Path Parameters Extraction

Let's modify Layer to use `path-to-regexp` to support parameters extraction from path.

**Implement**: Layer should support path with parameters

Example:

```js
layer = new Layer("/foo/:a/:b",middleware)

layer.match("/foo")
// => undefined

layer.match("/foo/apple")
// => undefined

layer.match("/foo/apple/xiaomi")
// returns =>
{path: "/foo/apple/xiaomi",
 params: {
   a: "apple",
   b: "xiaomi"
 }}

layer.match("/foo/apple/xiaomi/htc")
// returns =>
{path: "/foo/apple/xiaomi",
 params: {
   a: "apple",
   b: "xiaomi"
}}
```

Hint: Pass `{end: false}` as option to path-to-regexp so it matches prefix path instead of exact path.

Hint: path-to-regexp doesn't work well for paths with trailing slashes. Instead of "/foo/" use "/foo". Instead of "/" use "". (ya, it's stupid)

Hint: If you type `https://github.com/ab cd` into Chrome, it would turn into `https://github.com/ab%20cd`. This is [URI encoding](http://en.wikipedia.org/wiki/Percent-encoding). Use `decodeURIComponent` to turn a parameter like `xiao%20` into `xiao mi`.

Pass: `mocha verify -R spec -g 'Path parameters extraction'`

```
Path parameters extraction
  ✓ returns undefined for unmatched path
  ✓ returns undefined if there isn't enough parameters
  ✓ returns match data for exact match
  ✓ returns match data for prefix match
  ✓ should decode uri encoding
  ✓ should strip trialing slash
```

# Implement req.params

If a middleware matches a path with parameters, it should be able to get the parameters at `req.params`.

**Implement**: middleware should get the path parameters in `req.params`

Hint: `request.params` should defualt to `{}`

Example:

```
app.use("/foo/:a",function(req,res,next) {
  res.end(req.params.a);
});

app.use("/foo",function(req,res,next) {
  res.end(""+req.params.a);
});
// GET /foo/google => google
// GET /foo => undefined
```

Pass: `mocha verify -R spec -g "Implement req.params"`

```
Implement req.params
  ✓ should make path parameters accessible in req.params
  ✓ should make {} the default for req.params
```

**Commit**

```
git commit
```

# Implement Prefix Path Trimming

The request url of an embedded app should have the layer path prefix removed. For example:

```js
app.use("/foo",subapp)
```

+ `GET /foo/` should match `/foo`, and the `request.url` in subapp should be `/`.
+ `GET /foo/bar` should match `/foo`, and the `request.url` in subapp should be `/bar`.

There is a small problem, though. How can we tell between a "app" and a normal middleware? If it's a normal middleware, we should not modify `request.url` when calling it. If it's an app, we should trim prefix. But they are both functions and we can't tell them apart.

Express solves this problem by assuming that an `app` would have a `handle(req,res,next)` method, and a normal middleware wouldn't (since it's just a plain function):

```js
if(typeof fn.handle === "function") {
  console.log("is an app");
} else {
  console.log("is a function");
}
```

So we should ensure that the `app` we create has a `handle` method, such that:

```
app(req,res,next); // => delegates to:
app.handle(req,res,next);
```

**Implement**: Make sure that app has the `handle` method.

Pass: `mocha verify -R spec -g "app should have the handle method"`

```
app should have the handle method
  ✓ should have the handle method
```
**Implement**: Prefix path trimming for embedded app.

Example:

```js
subApp.use("/bar",function(req,res) {
  res.end("embedded app: "+req.url);
});
app.use("/foo",subApp);
app.use("/foo",function(req,res) {
  res.end("handler: "+req.url);
});

// GET /foo/bar => embedded app: /bar
// GET /foo => handler: /foo
```

Hint: The `request.url` is "untrimmed" when going from embedded app to the next middleware.

Hint: Ensure that the trimmed path starts with "/". Make sure that this edge case works:

```js
subApp.use("/",function() {
  res.end("/bar");
});
app.use("/bar",subApp);

// GET /bar => /bar
// GET /bar/ => /bar
```

Pass: `mocha verify -R spec -g "Prefix path trimming"`

```
Prefix path trimming
  ✓ trims request path prefix when calling embedded app
  ✓ restore trimmed request path to original when going to the next middleware
  ensures leading slash
    ✓ ensures that first char is / for trimmed path
```

**Commit**

```
git commit
```