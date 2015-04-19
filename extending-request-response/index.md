# Monkey Patching Request And Response

In this lesson we will monkey patch `request` and `response` to extend them with express functionalities.

# Extending The Request and Response Objects

We want to add the `isExpress` property to `req` and `res`:

```js
req.isExpress // => true
res.isExpress // => true
```

We can do it by extending [http.IncomingMessage](http://nodejs.org/api/http.html#http_http_incomingmessage) and [http.ServerResponse](http://nodejs.org/api/http.html#http_class_http_serverresponse):

```js
http.IncomingMessage.prototype.isExpress = true;
http.ServerResponse.prototype.isExpress = true;
```

But it's bad practice to monkey patch standard classes. Another approach is to add properties to each request we serve:

```js
// before serving each request
req.isExpress = true;
res.isExpress = true;
handler(req,res,next);
```

This avoids monkey patching the standard classes. However, it can be inefficient if we have to copy a lot of properties:

```js
// before serving each request
for(k in reqProps) {
  // copy many properties
  if(reqProps.hasOwnProperty(k)) {
    continue;
  }
  req[k] = reqProps[k];
}

for(k in resProps) {
  // copy many properties
  if(resProps.hasOwnProperty(k)) {
    continue;
  }
  res[k] = resProps[k];
}
handler(req,res,next);
```

A third way is to insert a prototype into the prototype chain by manipulating the `__proto__` property.

# Inserting A Prototype To Lookup Chain

Given a simple class A:

```js
function A() { }
A.prototype.a = 1;
var a = new A();
var a2 = new A();
```

The `__proto__` of `a` is the constructor's prototype:

```js
a.__proto__ === A.prototype;
// => true
```

How can we add the properties `b: 2` and `c: 3` only to a?

If we say `A.prototype.b = 3`, `a2` would also get the `b` property. We don't want that because we only want to extend `a`.

Let's think about this. When we try to access the `b` property, the look up order is:

```
// does it have b?
a.b // no. look up __proto__

// a.__proto__ => { a: 1 }
// does it have b?
a.__proto__.b // no. look up __proto__

// a.__proto__.__proto__ => {}
// does it have b?
a.__proto__.__proto__.b // no. look up __proto__

// a.__proto__.__proto__.__proto__ => null
// end of proto chain, return undefined
```

What if we change a's `__proto__`?

```js
extension = {b: 2, c: 3};
a.__proto__ = extension;

// does a have b?
a.b // no. look up __proto__

// a.__proto__ => {b: 2, c: 3}
// does it have b?
a.__proto__.b // yes. return 2
```

There is one problem. `a.a` is now undefined because `A.prototype` is no longer in the prototype chain. Let's add it back in.

```js
extension = {b: 2, c: 3};
extension.__proto__ = A.prototype;
a.__proto__ = extension;
```

The full example is:

```js
function A() { }
A.prototype.a = 1;
var a = new A();
var a2 = new A();

// monkey patch a
extension = {b: 2, c: 3};
extension.__proto__ = A.prototype;
a.__proto__ = extension;
a.a; // => 1
a.b; // => 2
a.c; // => 3

a2.a; // => 1
a2.b; // => undefined
a2.c; // => undefined
```

# Implement Request & Response Monkey Patching

Let's add the `isExpress` property to request and response.

**Implement**: `app.monkey_patch(req,res)` should extend request and response with `isExpress`.

In both `lib/request.js` and `lib/response.js` add:

```
var proto = {};
proto.isExpress = true;
// proto.__proto__ = ???
module.exports = proto;
```

Example:

```js
app.use(function(req,res) {
  app.monkey_patch(req,res);
  res.end(req.isExpress + "," + res.isExpress);
});

// GET / => true,true
```

Pass: `mocha verify/monkey_spec.js -R spec -g "Monkey patch req and res"`

```
Monkey patch req and res
  ✓ adds isExpress to req and res
```

**Implement**: Extend request and response before serving a request.

Now we want the app to automatically monkey patch req and res

Example:

```js
app.use(function(req,res) {
  res.end(req.isExpress + "," + res.isExpress);
});
// GET / => true,true
```

Hint: Change `app(req,res,next)` to call `monkey_patch` before calling middlewares.

Pass: `mocha verify/monkey_spec.js -R spec -g "Monkey patch before serving"`

```
Monkey patch before serving
  ✓ adds isExpress to req and res
```

# Implement req.app

Whenever an app is called to serve a request, it should set `req.app` to itself.

Handling subapp is slightly trickier. When exiting from a subapp, `req.app` should be retored to the parent app.

**Implement**: setting `req.app` when entering an app.

Hint: To restore `req.app` when exiting a subapp, wrap the `next` function passed to the subapp in another function.

Example:

```js
app.use(function(req,res,next) {
  req.app; // => app
  next();
});
subapp.use(function(req,res,next) {
  req.app; // => subapp
  next();
});
app.use(subapp);
app.use(function(req,res,next) {
  req.app; // => app
  res.end("ok");
});
```

Pass: `mocha verify/monkey_spec.js -R spec -g "Setting req.app"`

```
Setting req.app
  ✓ sets req.app when entering an app
  ✓ resets req.app to parent app when exiting a subapp
```

Make sure other tests still pass.

Pass: `mocha verify`

```
  ․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․․
  ․․․․․․․․․

  143 passing
```

# Implement req.res and req.req

For convenience, we'd like to be able to access the response object from the request object, and vice versa.

**Implement**: request and response should be accessible to each other

Example:

```
app.use(function(req,res) {
  req.res == res; // true
  res.req == req; // true
});
```

Pass: `mocha verify/monkey_spec.js -R spec -g 'req.res and res.req'`

```
req.res and res.req
  ✓ makes request and response accessible to each other
```

# Implement res.redirect

HTTP redirection is very simple. It adds `Location` header in the response for where to redirect the request to. The `Location` value can be:

+ an absolute url, e.g. `https://www.github.com`
+ a relative url, e.g. `/sign_up`

There are two status codes for redirection:

+ 301 - Moved Permanently.
+ 302 - Found, for temporary redirection.

See [http://stackoverflow.com/a/15702472/1068220](http://stackoverflow.com/a/15702472/1068220) to understand the difference between 301 and 302.

As an example, let's see how a request for `http://www.github.com` gets redirected to `https://github.com`. We use curl with the `-L` flag to follow redirects:

```
> curl -I -L http://www.github.com
HTTP/1.1 301 Moved Permanently
Content-length: 0
Location: https://www.github.com/
Connection: close

HTTP/1.1 301 Moved Permanently
Content-length: 0
Location: https://github.com/
Connection: close

HTTP/1.1 200 OK
Server: GitHub.com
Date: Sun, 13 Apr 2014 10:29:02 GMT
Content-Type: text/html; charset=utf-8
Status: 200 OK
...
```

+ The first request redirected from http to https
+ The second request redirected from `www.github.com` to `github.com`
+ The third request is a normal 200.

**Implement**: `res.redirect` to respond with HTTP redirection.

Example:

```js
app.use("/foo",function(req,res) {
  res.redirect("/baz"); // default status code is 302
});
app.use("/bar",function(req,res) {
  res.redirect(301,"/baz");
});
```

Pass: `mocha verify/monkey_spec.js -R spec -g "HTTP redirect"`

```
HTTP redirect:
  ✓ redirects with 302 by default
  ✓ redirects with the given status code
  ✓ returns empty body
```