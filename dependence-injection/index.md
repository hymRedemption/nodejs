# Express Dependency Injection

In this lesson we'll impement a dependency injection mechanism similiar to to @luin's express-di project.

# What is Express DI?

Read [express-di README](https://github.com/luin/express-di)

Read [Express 框架 middleware 的依赖问题与解决方案](http://zihua.li/2014/03/using-dependency-injection-to-optimise-express-middlewares)

# Running Lesson Tests

The test repo is at `https://github.com/hayeah/fork2-myexpress-verify`

You should `git pull` to get new tests

# Implementation Plan

There are three steps:

1. Be able to define dependencies by calling `app.factory(name,fn)`
2. Create an injector that can transform any function into a proper request handler
  + `inject(app,function(dep1,dep2,next) { })` => `function(req,res,next) {}`
3. `app.inject` can create an injected request handler

We do most of the work in the second step.

# Implement app.factory

The `app.factory(name, fn)` method is used to define a dependency.

**Arguments:**

+ `name`: The name of the dependency.
+ `fn`: A function that is like a typical express middleware, takes 3 arguments, `req`, `res` and `next`, with a subtle difference that the `next` function takes 2 arguments: an error(can be null) and the value of the dependency.

We'll store the factories in `app._factories`.

**Implement**: `app.factory(name,fn)` should add a factory function to `app._factories`

Example:

```js
getStatus = function(req,res,next) {
  next(null,"ok");
}
app.factory("status",getStatus);

app._factories[name] == getStatus
// => true
```

Pass: `mocha verify/di_spec.js -R spec -g "app.factory"`

```
app.factory
  ✓ should add a factory in app._factories
```

# Implement The Injector

Given a handler that needs its dependencies injected:

```js
handler = function(dep1,dep2,next) { }
```

We want to be able to transform it into something like this:

```js
injector = function(req,res,next) {
  // 1. build dependency values with factories
  // 2. call handler with values
}
```

Note that `injector` can be called as a normal request handler.

We'll define the `createInjector` function to do this transformation. For example:

```js
// Transforms the handler into an injector
injector = createInjector(handler,app);

// Call the injector as a request handler
injector(req,res,next);
```

There are three parts to make this work:

1. Analyze the inner handler's arguments for dependencies.
2. Build the dependency values from factories when the injector is called.
3. Call the inner handler with the dependency values.

We'll create the a module `lib/injector.js`, which exports the function `createInjector`.

## Handler Dependencies Analysis

**Implement**: Extract handler parameters by calling `injector.extract_params()`

Example:

```js
handler = function(foo,bar,baz) {

};
injector = createInjector(handler);
inject.extract_params()
// => ["foo","bar","baz"]
```

Hint: Use @luin's `getParameters` function to extract parameters from a function. [See the code.](https://github.com/luin/express-di/blob/a94e70813d34a6ec0def450a47530a999774aadb/lib/utils.js#L30-L53)

Pass: `mocha verify/di_spec.js -R spec -g "Handler Dependencies Analysis"`

```
Handler Dependencies Analysis:
  ✓ extracts the parameter names
```

## Implement Dependencies Loader

Injector needs to load the values of dependencies by calling the factories. We'll create the `injector.dependencies_loader` method to build a function that loads the dependencies:

```js
// create a loader function
loader = injector.dependencies_loader();

// loader passes the values to a callback
loader(function(err,values) {
  // "values" is an array of dependency values
});
```

**Implement**: `injector.dependencies_loader`

Example:

**1 values loading**

```js
// create the "foo" and "bar" factories
app.factory("foo",function(req,res,next) {
  next(null,"foo value");
});

app.factory("bar",function(req,res,next) {
  next(null,"bar value");
});

handler = function(bar,foo) {};
injector = createInjector(handler,app);
loader = injector.dependencies_loader();
loader(function(err,values) {
  // values => ["bar value","foo value"]
});
```

Hint: modify `createInjector(handler,app)` to accept two arguments.

Hint: For now, use `undefined` for `req` and `res` when calling factory.

Hint: write a recursive `next` function to call the factories.

Pass: `mocha verify/di_spec.js -R spec -g "load named dependencies"`

```
  Implement Dependencies Loader:
    load named dependencies:
      ✓ loads values
```

**2 error handling**

```js
app.factory("foo",function(req,res,next) {
  next(new Error("foo error"));
});

app.factory("bar",function(req,res,next) {
  throw new Error("bar error");
});

loader(function(err,values) {
  // these handlers would cause error:
  // function(foo) { }
  // function(bar) { }
  // function(baz) { }
});
```

+ `function(foo) { }` and `function(bar) { }` would cause errors because their factories cause errors.
+ `function(baz) { }` would cause an error because the `baz` factory doesn't exist.

Pass: `mocha verify/di_spec.js -R spec -g "dependencies error handling"`

```
Implement Dependencies Loader:
  dependencies error handling:
    ✓ gets error returned by factory
    ✓ gets error thrown by factory
    ✓ gets an error if factory is not defined
```

**3 builtin dependencies**

A handler can specify the dependencies `req`, `res`, and `next` in any order.

Let's extend `dependencies_loader` so it takes three arguments `req`, `res`, and `next`. The loader it returns would return these arguments as builtin dependencies.

```js
loader = injector.dependencies_loader(req,res,next);
handler = function(next,res,req) { }
loader(function(err,values) {
  // values => [next,res,req]
}
```

Hint: `loader` is a closure that captures the parameters of `dependencies_loader`.

Pass: `mocha verify/di_spec.js -R spec -g "load bulitin dependencies"`

```
Implement Dependencies Loader:
  load bulitin dependencies:
    ✓ can load req, res, and next
```

**4 pass req and res to factories**

Now that `dependencies_loader` has `req` and `res` as arguments, we should pass them into factories.

Pass: `mocha verify/di_spec.js -R spec -g "pass req and res to factories"`

```
Implement Dependencies Loader:
  pass req and res to factories
    ✓ can calls factories with req, res
```

## Implement Injector Invokation

Now let's make it possible to call an injector as a request handler.

**Implement**: Can call injector as request handler

Example:

```js
app.factory("foo",function(req,res,next) {
  next(null,"foo value");
});
handler = function(res,foo) {
}
injector = createInjector(handler,app);
injector(req,res,next); // call handler with dependencies
```

Pass: `mocha verify/di_spec.js -R spec -g "Implement Injector Invokation"`

```
Implement Injector Invokation:
  ✓ can call injector as a request handler
  ✓ calls next with error if injection fails
```

# Implement app.inject

We'll use a slightly different depenency injection API than express-di. Instead of implicitly injecting for every route handler, we'll use `app.inject` to explicitly inject a handler.

**Implement**: `app.inject(handler)` should return an injector.

```
app.factory("foo",function(res,req,cb) {
  cb(null,"hello from foo DI!");
})
app.use(app.inject(function (res,foo) {
  res.end(foo);
}));
// GET / => "hello from foo DI"
```

Pass: `mocha verify/di_spec.js -R spec -g "Implement app.inject"`

```
Implement app.inject
  ✓ can create an injector
```

# Bonus: Implement DI Refinements

luin's express-di has a few important optimizations to make dependency injection efficient:

+ Add dependency caching (per request)
  + See [Caching](https://github.com/luin/express-di/tree/a94e70813d34a6ec0def450a47530a999774aadb#cache)

+ Skip injection for well-known handler signatures: (req), (req,res), (req,res,next).
  + Hint: use [`needInject`](https://github.com/luin/express-di/blob/a94e70813d34a6ec0def450a47530a999774aadb/lib/utils.js#L76-L91)

+ Make it possible to for a subapp to use the factories of its parent app
  + See [Sub App](https://github.com/luin/express-di/tree/a94e70813d34a6ec0def450a47530a999774aadb#sub-app)
