# Building The Express App

In this lesson we'll start to build our own express. As a first step, it does nothing and will respond to all requests with 404.

We'll also start practicing test-driven development. This is a short lesson, so you should have the time to try TDD on your own.

Remember, we are going to reimplement `connect`, so don't use `connect` in this lesson.

# Starting The Web Server

To create an app without any middleware:

```js
var express = require("myexpress");
var app = express();
```

To start a web server running that app we can use the `http` module:

```js
var http = require("http");
var server = http.createServer(app);
server.listen(4000);
```

Or use a shortcut:

```js
// start the web server on port 4000
app.listen(4000)
```

# HTTP Status Code

You should take a look at [List of HTTP status codes](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes).

Some of the more frequent status codes you'd often see are:

+ 200 a request is successful
+ 301, 302 redirection to another url to get resource
+ 304 a resource is not modified

If client made an error, should respond with code in the 400s:

+ 400 bad request
+ 403 forbidden
+ 404 not found

If server made an error, should respond with code in the 500s:

+ 500 Internal Server Error (when you have an uncaught exception)
+ 502 Bad Gateway (when your reverse proxy (like nginx) fails to get a response from app server)

These are not arbitrary numbers, and you should know what these codes mean instead of returning random numbers for your http responses.

# Start The Project

+ `git init`
+ `npm init` and name the project `myexpress`

# Running Lesson Tests

In project root:

```
git clone https://github.com/hayeah/fork2-myexpress-verify.git verify
```

Add the following dev dependencies for testing:

```
"devDependencies": {
  "mocha": "^1.18.2",
  "chai": "^1.9.1",
  "supertest": "^0.10.0"
}
```

# Doing TDD On Your Own

Although tests are available in `verify`, you are going to write your own tests as well.

When doing TDD you should follow this work process:

1. Understand what you need to implement for a feature
2. Write a test case BEFORE you write code
3. Pass the test case you've written with the simplest code
4. Repeat step 2 by writing another test case

Each test case you write should test only one thing. Keep adding test cases until you've implemented a feature.

You should run the tests in `verify` only after you've completed a task.

# The Empty Express App

**Implement**: The empty app that responds with 404

Example:

```js
// the module is a factory function that builds an express app
var express = require("myexpress");

// factory function returns an express app
var app = express();

// start the http server
var http = require("http");
var server = http.createServer(app);
server.listen(4000);
```

Hint: What type should `app` be? Look at what value [`http.createServer`](http://nodejs.org/api/http.html#http_event_request) accepts.

Before you start writing code, please write the test file in `test/app_spec.js`:

```js
var express = require("../");
var request = require("supertest");

describe("app",function() {
  describe("create http server",function() {
    // write your test here
  });
});
```

You should read about how to write [asynchronous test](http://mochajs.org/#asynchronous-code) using mocha. For async API, there are two things you need to do:

1. Add a `done` argument to the testing function. It is a callback.
  + If you forget to do this, the test will end too early.
2. Call the `done` when the async test completes.
  + If you forget to do this, the test will then timeout.
3 Call `done` with err if there's an error.

Here's a simple example of how you'd write an async test:

```js
describe("an async function foo",function(done) {
  // foo is an async API
  foo(function(err) {
    // the test fails if err is not null
    done(err);
  });
});
```

Hint: `supertest` makes it easier to write expectations for http requests. [Read doc for supertest](https://github.com/visionmedia/supertest).

Hint: The expectation is something like `request(server).get(...).expect(...).end(done)`.

Pass: `mocha verify/app_spec.js -R spec -g 'as handler to http.createServer'`

```
Implement Empty App
  as handler to http.createServer
    ✓ responds to /foo with 404
```

# The Listen Method

**Implement** Create the `app.listen` method to start up an http server. It should return an `http.Server`.

Example:

```
var express = require("myexpress");
var app = express();
// start the http server
app.listen(4000);
```

Add new tests in `test/app_spec.js`:

```
describe("app",function() {
  // ... previous tests
  describe("#listen",function() {
    // write your test here
  });
});
```

Hint: `request("http://localhost:7000").get("/foo")` to make an http request to port 7000.

Pass: `mocha verify/app_spec.js -R spec -g 'Defining the app.listen method'`

```
Implement Empty App
  Defining the app.listen method
    ✓ should return an http.Server
    ✓ responds to /foo with 404
```