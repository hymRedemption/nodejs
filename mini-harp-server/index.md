# Mini Harp Static Server

We are going to implement a web-server that serves static files and preprocesses jade to HTML, less to css, and CoffeeScript to JavaScript. It is inspired by [Harp](http://harpjs.com).

We'll use [connect](https://github.com/senchalabs/connect) to do this, and practice using middlewares to build a web-server.

In this lesson we'll build a web-server that only serves static files. The command `mini-harp` should be able to server static files from the path `project-root`:

```
$ mini-harp project-root --port 4000
```

# Create mini-harp project

Let's create an empty node project called `mini-harp` which has a bin called `mini-harp`.

+ `git init`
+ Create `package.json` using `npm init`:
+ Create an empty `bin/mini-harp.js` and add it to `package.json` as a bin.
+ `npm link` the project so you can use it as you develop:

```bash
$ npm link
~/.nvm/v0.10.26/bin/mini-harp -> ~/.nvm/v0.10.26/lib/node_modules/mini-harp/bin/mini-harp.js
~/.nvm/v0.10.26/lib/node_modules/mini-harp -> ~/mini-harp

$ which mini-harp
~/.nvm/v0.10.26/bin/mini-harp
```

# Running An Empty Web Server

Let's add the latest Connect (version 3) as project dependency:

```
$ npm install connect@3.0.1 --save
```

In `empty.js` we'll create a web-server that runs the simplest connect app possible:

```js
var connect = require('connect');

// create a connect app
var app = connect();

// start the app as an http server on port 4000
console.log("Starting http server on http://localhost:4000");
app.listen(4000);
```

To run it:

```
$ node empty.js
Starting http server on http://localhost:4000
```

All it does is to respond to any request with `404 Not Found`. We can use `curl -i` to print both the content and the headers of an http response:

```
$ curl -i http://localhost:4000
HTTP/1.1 404 Not Found
Content-Type: text/html
Date: Sun, 30 Mar 2014 14:02:42 GMT
Connection: keep-alive
Transfer-Encoding: chunked

Cannot GET /
$ curl  -i http://localhost:4000/abc
HTTP/1.1 404 Not Found
Content-Type: text/html
Date: Sun, 30 Mar 2014 14:03:03 GMT
Connection: keep-alive
Transfer-Encoding: chunked

Cannot GET /abc
```

# Implement The Mini-Harp Module

We want the mini-harp module to export a factory function that builds a connect app. (The factory shouldn't run the app as an http server.)

**Implement**: `index.js` should export a function that creates a connect app.

Example:

```js
var createMiniHarp = require("mini-harp")
  , app = createMiniHarp();
console.log("Starting mini-harp on http://localhost:4000");
app.listen(4000);
```

**Commit**

```
git commit
```

# Implement The mini-harp Command

The `mini-harp` should create a mini-harp app, and listen on a specified port (4000 is default). Use `minimist` to parse command arguments.

**Implement** The mini-harp command should start a mini-harp server.

Example:

```
$ mini-harp
Starting mini-harp on http://localhost:4000
```

with port specified:

```
$ mini-harp --port 5000
Starting mini-harp on http://localhost:5000
```

**Commit**

```
git commit
```

# Implement A Current Time Middleware

If you don't know to use middleware, read [Meet the Connect Framework](http://code.tutsplus.com/tutorials/meet-the-connect-framework--net-31220).

Let's make our app do something. We'll add a middleware to report current time.

**Implement** The app should responds to request path `/current-time` with the current time.

Hint: `app.use(function(request,response,next) {...})` to add a middleware.

Hint: should respond with `(new Date()).toISOString()`.

Hint: `request.url` is the request url.

Example:

```
$ curl http://localhost:4000/current-time
2014-03-30T14:59:41.249Z
```

But it should not respond to other paths:

```
$ curl http://localhost:4000/abcd
Cannot Get /abcd
```

**Commit**

```
git commit
```

# Serving Static Content

Let's enable mini-harp to serve static files by using the [`server-static`](https://github.com/expressjs/serve-static) middleware.

`server-static`'s README is not very good. Instead, [look at its jsdoc](https://github.com/expressjs/serve-static/blob/e7c792749fd2e3f482a5963f43c4a05d42e4863e/index.js#L17-L42).

```
$ npm install serve-static@1.0.3 --save
```

**Implement**: mini-harp should serve static content from a given root path.

Example:

Using `mini-harp` as a module:

```
var miniHarp = require("mini-harp");

var root = process.cwd(); // current directory
var app = miniHarp(root);
app.listen(4000);
```

Running the server from the project directory you should be able to get project files via the http server:

```
$ curl -i http://localhost:4000/package.json
HTTP/1.1 200 OK
Accept-Ranges: bytes
ETag: "391-1396192611000"
Date: Sun, 30 Mar 2014 15:36:08 GMT
Cache-Control: public, max-age=0
Last-Modified: Sun, 30 Mar 2014 15:16:51 GMT
Content-Type: application/json
Content-Length: 391
Connection: keep-alive

{
  "name": "mini-harp",
  "version": "0.0.0",
  "description": "",
  "main": "index.js",
  ...
}
```

It should 404 for a non-existent file:

```
$ curl -i http://localhost:4000/i-dont-exist.js
HTTP/1.1 404 Not Found
Content-Type: text/html
Date: Sun, 30 Mar 2014 15:39:42 GMT
Connection: keep-alive
Transfer-Encoding: chunked

Cannot GET /i-dont-exist.js
```

**Implement**: change the mini-harp bin to accept an optional argument as root path (or default to `process.cwd`)

Example:

To use the current directory as root path:

```
$ mini-harp
```

To use a given path as root path:

```
$ mini-harp project-path
```

**Commit**

```
git commit
```