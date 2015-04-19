# Content Negotiation

In this lesson we will use Accept and Content-Type to implement content negotiation so the server can send different file types as response to a client.

# About The Content Type Header

The [MIME type](http://en.wikipedia.org/wiki/Internet_media_type) as specified by the `Content-Type` header is a way for the server to indicate what type of file "/foo" is. It's similar to how file extensions like ".jpg" and ".html" can indicate the type of file.

The reason that HTTP needs to set the `Content-Type` header is because the same request path `/foo` may respond with HTML, XML, or JSON, depending on what the client asks for (see: [content negotiation](http://en.wikipedia.org/wiki/Content_negotiation)).

We can use the [mime](https://www.npmjs.org/package/mime) package to map from file extensions to mime types:

```
mime.lookup("txt") => 'text/plain'
mime.lookup("html") => 'text/html'
mime.lookup("json") =>'application/json'
mime.lookup("png") =>'image/png'
```

# Implement res.type

Install mime: `npm install mime@1.2.11 --save`

Let's write two helper methods to help us set the `Content-Type` header:

+ `res.type(".html")` sets the Content-Type to be `text/html`
+ `res.default_type(".html")` sets the Content-Type to be `text/html` if it wasn't already set.

**Implement**: `res.type(ext)`

Example:

```js
app.use(function(req,res) {
  res.type("json");
  res.end("[1,2,3]");
});
// GET / => [1,2,3]
// Content-Type: application/json
```

Pass: `mocha verify/nego_spec.js -R spec -g 'sets the content-type'`

```
Setting Content-Type
  ✓ sets the content-type
```

**Implement**: `res.default_type(ext)`

Example:

```js
app.use(function(req,res) {
  res.default_type("text");
  res.default_type("json");
  res.end("[1,2,3]");
});
// GET / => [1,2,3]
// Content-Type: text/plain
```

Pass: `mocha verify/nego_spec.js -R spec -g 'sets the default content type'`

```
Setting Content-Type
  ✓ sets the default content type
```

# About The Accept Header

The Accept header is a way for the client to ask the server to respond in file formats that the client prefers.

For an excellent description of how the `Accept` header works, and how different browsers use different `Accept` header, read: [Unacceptable Browser HTTP Accept Headers](http://www.newmediacampaigns.com/blog/browser-rest-http-accept-headers).

Firefox's default Accept header is this:

```
Accept:text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
```

If the quality `q` is missing, it defaults to 1. So the order of client preference is:

+ `text/html;q=1`, HTML
+ `application/xhtml+xml;q=1`, XHTML
+ `application/xml;q=0.9`, XML
+ `*/*;q=0.8`, anything else

In human language, Firefox is asking:

> I want it in an HTML or XHTML format. If you cannot serve me this way, I'll take an XML instead. If you can't even give it to me in XML I'll take anything you've got!

The server should try to make the client as happy as possible by offering its most prefered format.

# Implement res.format

[res.format](http://expressjs.com/4x/api.html#res.format) performs content-negotiation on the request's Accept header.

Install the `accepts` package:

```
npm install accepts@1.0.1 --save
```

The `accepts` package makes content negotiation easier. Given an array of possible file types, it returns the one most preferred by the client:

```
var accept = accepts(req);
// Accept: text/*, application/json
accept.types(['html']);
// => "html"
accept.types(['json', 'text']);
// => "json"
```

See [Accept.prototype.types](https://github.com/hayeah/accepts/blob/a1cdfa93e71d5a0ac61e21f878e8d7221932931f/index.js#L17-L54) for more examples.

**Implement**: res.format

**1 Respond with different formats**

Example:

```
app.use(function(req,res) {
  res.format({
    text: function() {
      res.end("text hello");
    },

    html: function() {
      res.end("html <b>hello</b>");
    }
  });
})

// Accept: text/plain, text/html
//   => text hello

// Accept: text/html, text/plain
//   => html <b>hello</b>
```

Hint: Use [`Object.keys`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/keys) to get keys of res.format's argument.

Pass: `mocha verify/nego_spec.js -R spec -g 'Respond with different formats'`

```
res.format
  Respond with different formats
    ✓ responds to text request
    ✓ responds to html request
```

**2 Respond with `406 Not Acceptable` if there's no matching type for accept**

Example:

```
app.use(function(req,res) {
  res.format({});
});
// GET /
//  => 406 Not Acceptable
```

Hint: set `err.statusCode` to 406 before throwing it. You should adjust the error handler to return this statusCode instead of 500.

```js
var err = new Error("Not Acceptable");
err.statusCode = 406;
throw err;
```

Pass: `mocha verify/nego_spec.js -R spec -g 'responds with 406'`

```
res.format
  ✓ responds with 406 if there is no matching type
```
