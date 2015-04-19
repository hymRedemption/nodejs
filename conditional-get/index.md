# req.send and Conditional Get

We will implement `req.send`, a utility function that helps to simplify the details of sending a response.

By working with `req.send`, we will also learn how server can avoid sending a document to a client if the document has not changed.

# Implement req.send

See the documentation for [res.send](http://expressjs.com/4x/api.html#res.send) to understand what it does.

Here's a list of things `res.send` does:

+ Can handle String or Buffer body
+ Sets default Content-Type
  + default to text/html for String body
  + default to application/octet-stream for Buffer body
+ Sets Content-Length
+ If only a status code is given use http.STATUS_CODES[code] as body.
+ Respond in JSON format if given an object.

+ Calculates ETag (for conditional get)

**Implement**: `req.send`

**support buffer and string body:**

A [buffer](http://nodejs.org/api/buffer.html) is a byte array. It is more efficient for binary data like images and videos than string.

+ Should set `Content-Type`
  + default to text/html for String body
  + default to application/octet-stream for Buffer body

Example:

```js
app.use("/buffer",function(req,res) {
  res.send(new Buffer("binary data"));
  // Content-Type: application/octet-stream
});
app.use("/string",function(req,res) {
  res.send("string data");
  // Content-Type: text/html
});
app.use("/json",function(req,res) {
  res.type("json");
  res.send("[1,2,3]");
  // Content-Type: application/json
});
```

Pass: `mocha verify/send_spec.js -R spec -g "support buffer and string body"`

```
res.send:
  support buffer and string body:
    ✓ responds to buffer
    ✓ responds to string
    ✓ should not override existing content-type
```
**set Content-Length**

`Content-Length` should be the number of bytes the body has.

Hint: The length of a Unicode string is not the same as its byteLength:

```js
> s = "你好吗"
'你好吗'
> s.length
3
> Buffer.byteLength(s)
9
```

Pass: `mocha verify/send_spec.js -R spec -g "sets content-length"`

```
res.send:
  sets content-length:
    ✓ responds with the byte length of unicode string
    ✓ responds with the byte length of buffer
```

**can optionally specify status code**

Example:

```js
app.use("/foo",function(req,res) {
  res.send("foo ok"); // default status is 200
});
app.use("/bar",function(req,res) {
  res.send(201,"bar created");
});
app.use("/201",function(req,res) {
  res.send(201);
});
// GET /foo => 200 foo ok
// GET /bar => 201 bar created
// GET /201 => Created
```

Hint: If only the status code is given, use `http.STATUS_CODES` to return a default body:

```js
var http = require("http");
http.STATUS_CODES
{ '100': 'Continue',
  '101': 'Switching Protocols',
  '102': 'Processing',
  '200': 'OK',
  '201': 'Created',
  '202': 'Accepted',
  ...
  '506': 'Variant Also Negotiates',
  '507': 'Insufficient Storage',
  '509': 'Bandwidth Limit Exceeded',
  '510': 'Not Extended',
  '511': 'Network Authentication Required' }
```

Pass: `mocha verify/send_spec.js -R spec -g "sets status code"`

```
res.send:
  sets status code:
    ✓ defaults status code to 200
    ✓ can respond with a given status code
    ✓ reponds with default status code body is only status code is given
```

**can respond with JSON**

```js
app.use(function(req,res) {
  res.send({foo: [1,2,3]});
});
// GET /
// =>
// Content-Type: application/json
//
// {"foo": [1,2,3]}
```

Hint: Use [`JSON.stringify`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify) to convert an object to JSON string.

Pass: `mocha verify/send_spec.js -R spec -g "JSON response"`

```
res.send:
  JSON response:
    ✓ returns a JSON as response
```

# Conditional Get

HTTP conditional get if a way for the client to ask the server: "I already have this document, but give me the new one if it changed." If the document have not changed, the server would respond with "304 - Not Modified" with an empty body to avoid sending the document back to the client again.

Let's see how this works in practice. We'll get the Github API data for `@hayeah`:

```
> curl -i https://api.github.com/users/hayeah
HTTP/1.1 200 OK
Server: GitHub.com
Date: Sun, 13 Apr 2014 13:20:50 GMT
...
Cache-Control: public, max-age=60, s-maxage=60
Last-Modified: Thu, 10 Apr 2014 15:02:07 GMT
ETag: "3e8e89e3cdb89c8574e761cb3bcb9b3e"
Vary: Accept
...

{
  "login": "hayeah",
  "id": 50120,
  ...
  "public_repos": 82,
  "public_gists": 46,
  "followers": 127,
  "following": 17,
  "created_at": "2009-01-29T05:33:38Z",
  "updated_at": "2014-04-10T15:02:07Z"
}
```

Pay attention to these two headers:

```
Last-Modified: Thu, 10 Apr 2014 15:02:07 GMT
ETag: "3e8e89e3cdb89c8574e761cb3bcb9b3e"
```

+ `Last-Modified` tells us when the data was last changed.
+ [`ETag`](http://en.wikipedia.org/wiki/HTTP_ETag) is a unique identifier for the document. It the data changes the ETag also changes.
  +  the ETag can be any string. Often, a fast checksum like [CRC32](http://en.wikipedia.org/wiki/Cyclic_redundancy_check) is used.

The client can use either of these to make a conditional request.

We can try using the `-H` flag to add the `If-Modified-Since` header:

```
> curl -i https://api.github.com/users/hayeah -H "If-Modified-Since: Thu, 10 Apr 2014 15:02:07 GMT"
HTTP/1.1 304 Not Modified
Server: GitHub.com
Date: Sun, 13 Apr 2014 13:25:09 GMT
Status: 304 Not Modified
...
```

The server tells us that the data did not change since the Last-Modified time.

We can try adding the `If-None-Match` header:

```
> curl -i https://api.github.com/users/hayeah -H 'If-None-Match: "3e8e89e3cdb89c8574e761cb3bcb9b3e"'
HTTP/1.1 304 Not Modified
Server: GitHub.com
Date: Sun, 13 Apr 2014 13:26:22 GMT
Status: 304 Not Modified
...
```

The server tells us that the newest data is still the same as the ETag "3e8e89...", so nothing changed.

# Implement Conditional Get

We will use CRC32 to generate ETags. Install crc32:

```
npm install buffer-crc32@0.2.1 --save
```

A CRC32 checksum is a 32bit number. It should generate a different number for different strings:

```js
> crc32 = require('buffer-crc32');
> crc32.unsigned("cba");
3635344512
> crc32.unsigned("abc");
891568578
```

But a CRC32 checksum isn't perfect. Although unlikely, it can generate the same number for different strings:

```js
> crc32.unsigned("plumless")
1306201125
> crc32.unsigned("buckeroo")
1306201125
```

Depending on your application, [hash collision](http://en.wikipedia.org/wiki/Hash_collision) may or may not be a problem. There are other choices for ETag:

+ MD5
+ Last-Modified timestamp

For our implementation we'll accept the fact that ETag might be the same for different output.

**Implement**: Use `crc32` to calculate ETag of body in `req.send`

+ Should only generate ETag only for GET requests
+ Don't override existing ETag header.

```js
app.use("/plumless",function(req,res) {
  res.send("plumless");
});
app.use("/buckeroo",function(req,res) {
  res.setHeader("Etag","buckeroo");
  res.send("buckeroo");
});
// GET /plumless
//   ETag: "1306201125"
//
//   plumless

// POST /plumless
//   plumless

// GET /buckeroo
//   ETag: "buckeroo"
//
//   buckeroo
```

Pass: `mocha verify -R spec -g 'Calculate Etag'`

```
Conditional get:
  Calculate Etag:
    ✓ generates ETag
    ✓ doesn't generate ETag for non GET requests
    ✓ doesn't generate ETag if already set
    ✓ doesn't generate ETag for empty body
```

**Implement**: Respond with 304

`req.send` can decide to respond with 304 by comparing request's conditional get headers to response headers.

+ if `ETag` matches `If-None-Match` return 304
+ if `Last-Modified` < `If-Modified-Since` return 304

Example:

**Conditional get with ETag**

```js
app.use("/",function(req,res) {
  res.setHeader("Etag","foo-v1")
  res.send("foo-v1");
});
// GET /
// If-None-Match: "foo-v1"
// =>
// 304

// GET /
// If-None-Match: "foo-v0"
// =>
// 200
```

Hint: `this.req.headers["if-none-match"]` returns the request's ETag.

Pass: `mocha verify -R spec -g 'ETag 304'`

```
Conditional get:
  ETag 304:
    ✓ returns 304 if ETag matches
    ✓ returns 200 if ETag doesn't match
```

**Conditional get with Last-Modified**

```
app.use(function(req,res) {
  res.setHeader("Last-Modified","Sun, 31 Jan 2010 16:00:00 GMT")
  res.send("bar-2010");
});
// GET /
// If-Modified-Since: Sun, 31 Jan 2010 16:00:00 GMT
// => 304

// GET /
// If-Modified-Since: Thu, 31 Jan 2008 16:00:00 GMT
// => 200
```

Hint: You can compare dates with the `<` operator

Pass: `mocha verify -R spec -g 'Last-Modified 304'`

```
Conditional get:
  Last-Modified 304:
    ✓ returns 304 if not modified since
    ✓ returns 200 if modified since
```