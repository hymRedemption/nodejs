# Building The Middlewares Stack

In this lesson we'll make it possible to compose an app by adding middlewares. We will make it possible to:

+ Add middlewares to build a stack
+ Add error handling middlewares
+ Add express apps as a middleware

The `next` function transfers control to the next middleware in the stack.

# Stacking Middlewares

Install Express 4:

```
npm install express@4.0.0-rc4
```

Let'e experiment with express's middleware stack. A very simple middleware stack looks like:

```js
var app = require("express")();

app.use(function(req,res,next) {
  // the first middleware
  next();
});

// the second middleware
app.use(function(req,res,next) {
  res.end("hello from the second middleware");
});

app.listen(4000);
```

A request to this stack would get the response `hello from the second middleware`. The first middleware simply passes over to the second middleware by calling `next`.

## Error Handling

If there is an error, call `next(err)` to pass the error down the middleware stack. When there is an error, only the error handlers should be called. The normal middlewares should be skipped.

```js
// erorr-stack.js
var app = require("express")();

// the first middleware creates an error
app.use(function(req,res,next) {
  var error = new Error("an error");
  next(error);
});

// skip this if there is an error
app.use(function(req,res,next) {
  console.log("second middleware");
  next();
});

app.use(function(error,req,res,next) {
  console.log("first error handler");
  next(error);
});

// skip this if there is an error
app.use(function(req,res,next) {
  console.log("third middleware");
  next();
});

app.use(function(error,req,res,next) {
  console.log("second error handler");
  res.end("hello from second error handler");
});

app.listen(4000);
```

Start the server and curl:

```
$ curl localhost:4000
hello from second error handler
```

And the server should output these messages:

```
$ node error-stack.js
first error handler
second error handler
```

# Running Lesson Tests

The test repo is at `https://github.com/hayeah/fork2-myexpress-verify`

You should `git pull` to get new tests.

# Implement Middleware Stack

Remember that `app` is a `function(res,req)` that node's http server calls to handle a request. When it is called it should create a `next` function, and pass it as the last argument to middlewares in the stack.

Let's think about how the function `next` should behave:

1. The first time next is called, it should call the first middleware
  + e.g. `stack[0](req,res,next)`
2. The second time next() is called, it should call the second middleware
  + e.g. `stack[1](req,res,next)`
3. The nth time next is called, it calls the nth middleware
  + e.g. `stack[n](req,res,next)`
4. When there is no more middleware, it should end with a default 404
  + e.g. `res.end("404 - Not Found")`

The stacktrace of function calls when serving a request is like:

```
app(res,req) # http calls this to handle a request
  next() # calls the first middleware
    stack[0](req,res,next)
      next() # the first middleware calls the second middleware
        stack[1](req,res,next)
          next() # the second middleware calls the third middleware
            // keep going like this recursively until there's no more
            stack[n](req,res,next)
              next()
              // if there's no more middleware
                res.end("404 - Not Found")
```

So `next` is a recursive function that is called repeatedly as a request travels down the stack.

**Implement**: `app.use(middleware)` should add a middleware to `app.stack = []`

Example:

```js
var m1 = function() {};
var m2 = function() {};
app.use(m1);
app.use(m2);
```

Add your own test in `test/app_spec.js`:

```js
describe(".use",function() {
  /// your own test
});
```

Pass: `mocha verify -R spec -g 'Implement app.use'`

```
Implement app.use
  ✓ should be able to add middlewares to stack
```

**Implement**: `app(res,req)` should call the middlewares in `app.stack`

This is a more complicated feature, so it's a perfect opportunity to practice TDD. We'll break it down into simpler cases. You should:

1. Add a test for the case.
2. Write code to pass the test.
3. Refactor the code.

Add your own tests in `test/app_spec.js`. You should use `beforeEach` to create a new app instance for each test case.

```
describe("calling middleware stack",function() {
  var app;
  beforeEach(function() {
    app = new myexpress();
  });

  // test cases
});
```

Examples:

1 Should be able to call a single middleware:

```js
var m1 = function(req,res,next) {
  res.end("hello from m1");
};
app.use(m1);
// => hello from m1
```

2 Should be able to call `next` to go to the next middleware:

Hint: See [Recursive Next Hints](https://gist.github.com/hayeah/efd773bdbae2e9fa9130) if you need help.

```js
var m1 = function(req,res,next) {
  next();
};

var m2 = function(req,res,next) {
  res.end("hello from m2");
};
app.use(m1);
app.use(m2);
// => hello from m2
```


3 Should 404 at the end of middleware chain:

```js
var m1 = function(req,res,next) {
  next();
};

var m2 = function(req,res,next) {
  next();
};
app.use(m1);
app.use(m2);
// => 404 - Not Found
```

4 Should 404 if no middleware is added:

```js
app.stack = []
// => 404 - Not Found
```

Pass: `mocha verify -R spec -g 'Implement calling the middlewares'`

```
Implement calling the middlewares
  ✓ Should be able to call a single middleware
  ✓ Should be able to call `next` to go to the next middleware
  ✓ Should 404 at the end of middleware chain
  ✓ Should 404 if no middleware is added
```

# Implement Error Handling

There are two ways in which a middleware might cause an error.

It might raise an error:

```js
app.use(function() {
  throw new Error("oops");
});
```

Or it might get an async error from an api:

```js
app.use(function(res,req,next) {
  fs.readFile(filePath,function(err,content) {
    if(err)
      next(err);
  });
});
```

The async version calls next with the error as an argument.

We should handle both of these cases by passing this error to error handling middlewares (and skip the normal middlewares). An error handler is a function that accepts four arguments:

```js
err_handler = function(err,req,res,next) { ... }
```

See [Error Handling](http://expressjs.com/guide.html#error-handling) for more.

**Implement**: Error handling

Again, this is a complicated feature. We'll break it down into simpler cases. You should:

1. Add a test for the case.
2. Write code to pass the test.
3. Refactor the code.

Add your own tests in `test/app_spec.js`.

Examples:

1 should return 500 for unhandled error

```js
var m1 = function(req,res,next) {
  next(new Error("boom!"));
}
app.use(m1)
// => 500 - Internal Error
```

2 should return 500 for uncaught error

```js
var m1 = function(req,res,next) {
  raise new Error("boom!");
};
app.use(m1)
// => 500 - Internal Error
```

3 should skip error handlers when `next` is called without an error

```js
var m1 = function(req,res,next) {
  next();
}

var e1 = function(err,req,res,next) {
  // timeout
}

var m2 = function(req,res,next) {
  res.end("m2");
}
app.use(m1);
app.use(e1); // should skip this. will timeout if called.
app.use(m2);
// => m2
```

4 should skip normal middlewares if `next` is called with an error

```js
var m1 = function(req,res,next) {
  next(new Error("boom!"));
}

var m2 = function(req,res,next) {
  // timeout
}

var e1 = function(err,req,res,next) {
  res.end("e1");
}

app.use(m1);
app.use(m2); // should skip this. will timeout if called.
app.use(e1);
// => e1
```

Pass: `mocha verify -R spec -g 'Implement Error Handling'`

```
Implement Error Handling
  ✓ should return 500 for unhandled error
  ✓ should return 500 for uncaught error
  ✓ should ignore error handlers when `next` is called without an error
  ✓ should skip normal middlewares if `next` is called with an error
```

# Implement App Embedding As Middleware

Lastly, we want to be able to add an app as a middleware. Here's an example:

```js
app = new express();
subApp = new express();

app.use(m1);
app.use(subApp);
app.use(m2);

subApp.use(m3);
subApp.use(m4);
app.listen(4000)
```

The calling order is like:

```
app(req,res) // called by http server
  m1(req,res,next)
    subApp(res,req,next) // called as a middleware by app
      m3(res,req,next2) // next2 is the internal stack traversal function for subApp
        m4(res,req,next2)
          m2(res,req,next) // back from subapp
```

Notice a few things:

+ the sub-app behaves just like a normal middleware function.
+ when the sub-app is at the end, it shouldn't 404 or 500. It should return to the parent app by calling next.

**Implement** App embedding as middleware

Example:

1 should pass unhandled request to parent

```js
app = new express();
subApp = new express();

function m2(req,res,next) {
  res.end("m2");
}

app.use(subApp);
app.use(m2);
=> m2
```

2 should pass unhandled error to parent

```js
app = new express();
subApp = new express();

function m1(req,res,next) {
  next("m1 error");
}

function e1(err,req,res,next) {
  res.end(err);
}

subApp.use(m1);

app.use(subApp);
app.use(e1);
=> m1 error
```

Pass: `mocha verify -R spec -g 'Implement App Embedding As Middleware'`

```
Implement App Embedding As Middleware
  ✓ should pass unhandled request to parent
  ✓ should pass unhandled error to parent
```