# HTTP Verbs

In this lesson we will implement the http verbs that are often used to implement RESTful HTTP API.

# Restful Architecture

RESTful architecture isn't just writing a web API using "GET", "POST", "DELETE", etc. It is a whole philosophy of how to exchange data in a scalable and flexible way. This [stackoverflow answer](http://stackoverflow.com/a/671508/1068220) is a good starting point to learn about REST.

[Roy Fielding's thesis on REST](http://www.ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf) can help us to understand how technologies like HTTP, DNS and URI had help the Web to scale so incredibly successfully.

Roy Fielding's thesis is probably the best way to understand what REST is about. It is also not very hard to read. Highly recommended.

# "RESTful" Verbs For Express

For this lesson, we'll just focus ourselves on implementing express's API for HTTP verbs.

See [app.VERB](http://expressjs.com/4x/api.html#app.VERB) for details about the API.

Let's look at an example:

```
// Define a GET handler on /foo
app.get("/foo",function(req,res) {
  res.end("Got Foo");
});

// Define a POST handler on /foo
app.post("/foo",function(req,res) {
  res.end("Posted To Foo");
});
app.listen(4000);
```

If we try to GET `/foo` (using the `-X` flag to specify the GET verb), the first handler responds:

```sh
curl -X GET localhost:4000/foo
Got Foo
```

If we try to POST to `/foo` (using the `-X` flag to specify the POST verb), the second handler responds:

```sh
$ curl -X POST localhost:4000/foo
Posted To Foo
```

# Running Lesson Tests

The test repo is at `https://github.com/hayeah/fork2-myexpress-verify`

You should `git pull` to get new tests

# Implement GET

When we add a request handler with `app.get(path,handler)`, layer is responsible of matching the path. But we'll need something else to do the `request.method` matching, so a request handler is called only for the right HTTP verb.

Let's create a `makeRoute(verb,handler)` function so it takes a request handler and return another handler:

```
// The "get" request handler
handler = function(req,res,next) {
  // ...
}

makeRoute("get",handler);

// returns another handler =>

function(req,res,next) {
  // calls handler if verb matches "get"
}
```

`makeRoute` is very simple at the moment. In the next lesson we'll extend it to provide more functionalities.

**Implement**: Create the `makeRoute(verb,handler)` constructor.

Make the `lib/route.js` module export `makeRoute`.

The returned function should be `function(req,res,next)`. We'll make error handlers work in next lesson.

(There's no test written for this.)

**Implement**: Implement `app.get`

Hint: `app.get(path,handler)` should do whole path matching, not prefix matching. Pass `{end: true}` to `Layer` as option.

Example:

```
a.get("/foo",function(req,res) {
  res.end("foo");
});
// GET /foo => foo
// GET /foo/bar => 404
// POST /foo => 404
```

Pass: `mocha verify -R spec -g "App get method"`

```
App get method:
  ✓ should respond for GET request
  ✓ should 404 non GET requests
  ✓ should 404 non whole path match
```

# Implement Other HTTP Verbs

For other HTTP verbs we just have to do exactly same thing. Instead of implementing each one by hand (tedious and error prone), let's generate these methods dynamically from a list of HTTP verbs.

We can get the list of HTTP verbs that node's http parser supports from the `methods` module:

```
npm install methods@0.1.0 --save
```

Then in node console:

```js
> methods = require("methods")
[ 'get',
  'post',
  'put',
  'head',
  'delete',
  'options',
  'trace',
  'copy',
  'lock',
  'mkcol',
  'move',
  'propfind',
  'proppatch',
  'unlock',
  'report',
  'mkactivity',
  'checkout',
  'merge',
  'm-search',
  'notify',
  'subscribe',
  'unsubscribe',
  'patch',
  'search' ]
```

Some of these verbs you don't see in a typical web server.

For example, `m-search` and `notify` are used by the [Simple Service Discovery Protocol (SSDP)](http://en.wikipedia.org/wiki/Simple_Service_Discovery_Protocol). Interestingly, SSDP uses the HTTP protocol on UDP instead of the typical TCP transport.

[WebDAV](http://en.wikipedia.org/wiki/WebDAV) adds the `propfind`, `proppatch`, `mkcol`, `copy`, `move`, `lock`, `unlock` verbs. WebDAV is like a really really really crappy Git over HTTP. In a way, Github fulfills the dream of "Web Distributed Authoring and Versioning" WebDAV was designed for.

**Implement**: Create methods for http verbs

Loop over the verbs and generate methods, like so:

```js
methods.forEach(function(method){
  // app[method] = function() { ... }
});
```

Hint: You can also generate your tests by looping over the http verbs.

Hint: [supertest](https://github.com/visionmedia/supertest)'s method for DELETE is `del`

Pass: `mocha verify/verbs_spec.js -R spec -g "All http verbs"`

```
All http verbs:
  ✓ responds to get
  ✓ responds to post
  ✓ responds to put
  ✓ responds to head
  ✓ responds to delete
  ✓ responds to options
  ✓ responds to trace
  ✓ responds to copy
  ✓ responds to lock
  ✓ responds to mkcol
  ✓ responds to move
  ✓ responds to propfind
  ✓ responds to proppatch
  ✓ responds to unlock
  ✓ responds to report
  ✓ responds to mkactivity
  ✓ responds to checkout
  ✓ responds to merge
  ✓ responds to m-search
  ✓ responds to notify
  ✓ responds to subscribe
  ✓ responds to unsubscribe
  ✓ responds to patch
  ✓ responds to search
```