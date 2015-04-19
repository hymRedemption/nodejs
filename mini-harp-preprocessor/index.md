# Mini Harp Preprocessors

In this lesson we'll create middlewares that extend the static mini-harp application so it can:

+ render `.jade` templates as HTML.
+ render `.coffeescript` as JavaScript.
+ render `.less` as CSS.

Each of the preprocessor is in its own independent middleware.

# Running Tests

In project root:

```
git clone https://github.com/hayeah/fork2-mini-harp-verify.git verify
```

Add the following dev dependencies for testing:

```
"devDependencies": {
  "mocha": "^1.18.2",
  "chai": "^1.9.1",
  "supertest": "^0.10.0"
}
```

For these tests we assume that the root path of our mini-harp server is at `verify/assets`.

# Rendering Jade Templates

Install Jade: `npm install jade@1.3.0 --save`

[Jade](http://jade-lang.com) is an indentation based html template for nodejs. It looks like:

```
doctype html
html(lang="en")
  head
    title= pageTitle
    script(type='text/javascript').
      if (foo) {
         bar(1 + 5)
      }
  body
    h1 Jade - node template engine
    #container.col
      if youAreUsingJade
        p You are amazing
      else
        p Get on it!
      p.
        Jade is a terse and simple
        templating language with a
        strong focus on performance
        and powerful features.
```

Let's add a middleware so that if a request is made to `/foo.html`, and `/foo.jade` is a Jade template, then we should render it as the response of the request.

**Implement**: Add jade preprocessing by creating the middleware factory `makeJade`.

Hint: use `fs.readFile(jadeFile,{encoding: "utf8"},callback)` to read a file.

Hint: use `path.extname(req.url)` to test if a request is for an html path.

Hint: call the `next` function if the middleware doesn't know how to handle a request.

Hint: `jade.render(source, options)` See [API doc](http://jade-lang.com/api/).

In `lib/processor/jade.js` add:

```
module.exports = makeJade;

function makeJade(root) {
  // TODO
}
```

Pass: `mocha verify -R spec -g 'Implement jade template rendering'`

```
Implement jade template rendering
  ✓ should return 200 for /foo.html
  ✓ should render foo.jade for /foo.html
  ✓ should 404 for /not-found.html
```

**Implement**: Add jade preprocessor to the mini-harp app

If both `/bar.html` and `/bar.jade` exist, the server should server `/bar.html`.

Pass: `mocha verify -R spec -g 'Add jade preprocessor to the mini-harp app'`

```
Add jade preprocessor to the mini-harp app
  ✓ should serve /foo.html if foo.html does not exist but foo.jade does
  ✓ should serve /bar.html if bar.html exists
```

**Commit**

```
git commit
```

# Rendering Less Templates

Install Less:

```
npm install less@1.7.0 --save
```

[Less](http://lesscss.org/) extends the CSS language, adding features that allow variables, mixins, functions, etc. It looks like:

```css
@base: #f938ab;

.box-shadow(@style, @c) when (iscolor(@c)) {
  -webkit-box-shadow: @style @c;
  box-shadow:         @style @c;
}
.box-shadow(@style, @alpha: 50%) when (isnumber(@alpha)) {
  .box-shadow(@style, rgba(0, 0, 0, @alpha));
}
.box {
  color: saturate(@base, 5%);
  border-color: lighten(@base, 30%);
  div { .box-shadow(0 0 5px, 30%) }
}
```

Let's add a middleware so that if a request is made to `/foo.css`, and `/foo.less` is a `.less` file, then we should render it as the response of the request.

**Implement**: Add less preprocessing by creating the middleware factory `makeLess`.

In `lib/processor/less.js` add:

```js
module.exports = makeLess;

function makeLess(root) {
  // TODO
}
```

Hint: [How to compile less to css](http://lesscss.org/#using-less-usage-in-code). Use the `less.render` function.

Pass: `mocha verify -R spec -g 'Implement less template rendering'`

```
Implement less template rendering
  ✓ should return 200 for /foo.css
  ✓ should render foo.less for /foo.css
  ✓ should 404 for /not-found.css
```

**Implement**: Add less preprocessor to the mini-harp app

If both `/bar.css` and `/bar.less` exist, the server should server `/bar.css`.

Pass: `mocha verify -R spec -g 'Add less preprocessor to the mini-harp app'`

```
Add less preprocessor to the mini-harp app
  ✓ should serve /foo.css if foo.css does not exist but foo.less does
  ✓ should serve /bar.css if bar.css exists
```

**Commit**

```
git commit
```

# Serving The Root Path

You can try serving the HarpJS welcome page with mini-harp (the assets are in the `verify/harp-welcome` directory):

```
$ mini-harp verify/harp-welcome
Starting http server on http://localhost:4000
```

You should be able to open the index page in browser and see a nicely formatted page, at: [http://localhost:4000/index.html](http://localhost:4000/index.html).

But you visit a 404 if you visit [http://localhost:4000](http://localhost:4000). Let's fix that.

**Implement**: Requesting for the root should render `/index.html`

Hint: Add a middleware that rewrites `request.url`

Pass: `mocha verify -R spec -g 'The root path should render index.html'`

```
The root path should render index.html
  ✓ should render index (38ms)
```

**Commit**

```
git commit
```

# Reject Stupid Requests

You can download the `jade` template source because it's served as static file:

```
$ curl localhost:4000/index.jade
doctype
html
  head
    link(rel="stylesheet", href="/main.css")
  body
    h1 Welcome to Mini-Harp.
    h3 This is yours to own. Enjoy.
```

Let's prevent that.

**Implement**: Add a middleware that returns 404 if the request path ends with `.jade` or `.less`

Pass: `mocha verify -R spec -g 'Should not respond to .jade or .less'`

Hint: set `response.statusCode`

```
Should not respond to .jade or .less
  ✓ should return 404 for /index.jade
  ✓ should return 404 for /foo.less
```

# Setting HTTP Headers

We compare the HTTP headers produced by harp:

```
$ curl -i localhost:4000
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 161
Date: Mon, 31 Mar 2014 13:32:21 GMT
Connection: keep-alive
```

And the http headers produced by our mini-harp (when rendering `index.jade`):

```
$ curl -i localhost:4000
HTTP/1.1 200 OK
Date: Mon, 31 Mar 2014 13:33:55 GMT
Connection: keep-alive
Transfer-Encoding: chunked
```

There are a few differences in mini-harp:

1. Transfer-Encoding is `chunked`
1. No Content-length
1. No Content-type

## Turning Off Chunked Transfer

[Chunked Transfer](http://en.wikipedia.org/wiki/Chunked_transfer_encoding) is for streaming dynamically generated content.

We might have a service that generates numbers from 1 to infinity:

```js
var http = require("http");

var server = http.createServer(function(req,res) {
  var i = 0;
  function tick() {
    i++;

    res.write(String(i)+"\n");
    setTimeout(tick,500);
  }

  tick();
});

server.listen(4000);

```

You can see the streaming output using curl:

```
$ curl -i --no-buffer localhost:4000
HTTP/1.1 200 OK
Date: Mon, 31 Mar 2014 15:15:39 GMT
Connection: keep-alive
Transfer-Encoding: chunked

1
2
3
4
5
6
7
8
9
^C
```

You won't be able to see this in Chrome because it buffers the streaming data. But in Firefox you can see the numbers are they are generated.

It is better to turn off chunked encoding unless you are dynamically generating content, and you don't know the how much content there is.

**Implement**: Turn off chunking in the jade and less preprocessing middlewares.

Hint: To turn off chunking, set the `Content-Length` header for reponse.

Pass: `mocha verify -R spec -g 'No chunked transfer for .jade or .less'`

```
No chunked transfer for .jade or .less
  ✓ disables chunking for jade
  ✓ disables chunking for less
```

## Setting Content Type

Without setting the content type, the browser has to guess what kind of file a response is. Your browser is smart enough to guess correctly the content type of our rendered jade and less files, but we should still say what type of file a reponse is by setting the [mime type](http://en.wikipedia.org/wiki/Internet_media_type) explicitly.

In HTTP a resource at the path `/foo.html` doesn't mean it's an html file. It's an html file only if the response header `Content-Type` says it is.

**Implement**: Set the content type for preprocessed jade and css.

Hint: The mime type for rendered html should be `text/html; charset=UTF-8`

Hint: The mime type for rendered css should be `text/css; charset=UTF-8`

Pass: `mocha verify -R spec -g 'Set content type for .jade or .less'`

```
Set content type for .jade or .less
  ✓ set content type for jade
  ✓ set content type for less
```

**Commit**

```
git commit
```