# Implement Class Inheritance

In this lesson we'll implement class inheritance and super method call. In the process we'll gain more experience using the prototype chain.

# How CoffeeScript Implements Inheritance

If you look at the CoffeeScript compiled output, you'd see that it uses the `__extends` method to help setup the inheritance relationship between parent and child. Here's the `__extends` function formatted and commented:

```js
var __hasProp = {}.hasOwnProperty;
var __extends = function(child, parent) {
  // Copy "class properties" from parent to child
  for (var key in parent) {
    if (__hasProp.call(parent, key)) child[key] = parent[key];
  }

  // This is a "surrogate constructor". It's used to create the prototype of the child class.
  function ctor() {
    // The surrogate constructor should satisfy "child.prototype.constructor == child.constructor"
    this.constructor = child;
  }

  // Setting up inheritance by connecting the prototype chain from child to parent
  ctor.prototype = parent.prototype;
  child.prototype = new ctor;

  // Setting up `__super__` as a convenience property for access.
  // It doesn't affect the prototype chain.
  child.__super__ = parent.prototype;
  return child;
};
```

This is a lot to understand. Just have a rough idea about what `__extends` need to do to setup the inheritance.

### An Example Inheritance

Let's look at a Foo class that inherits from Bar:

```
function Foo() {

}
__extends(Foo,Bar);
var obj = new Foo();
```

The object graph looks like:

<img src="https://www.evernote.com/shard/s20/sh/58a2d56e-c980-4cab-ba24-ddb360540e05/c695e0de75608942dfb711a238a20189/deep/0/class-inheritance-hd.png" width="600px">

### More About The Surrogate Constructor

**Question**: Why do you need to set the constructor of the prototype object? Would this work?

```
function ctor() {};
```

Hint: `obj.constructor` is the value of `obj.prototype.constructor`.

**Question**: Why can't you use a plain empty object as the prototype?

```
child.prototype = {};
child.prototype.constructor = child
```

Hint: think about setting up the prototype chain.

# Property Lookup Examples

To see how we want the prototype chain to work let's walk through a few examples. Suppose we have the following classes:

```js
var A = Class({
  a: 1

});

var B = Class({
  b: 2
},A);
```

The `a` property is in the prototype of `A`. The `b` property is in the prototype of `B`. Suppose we have an instance of `B`:

```js
var b = new B()
b.c = 3;
```

To look up `b.c`:

```
// has the c property? yes.
b.c

// return b.c
// => 3
```

To look up `b.b`:

```
// Has the b property? No. Look up in the prototype.
b.b

// Has the b property? Yes.
b.constructor.prototype.b

// return b.constructor.prototype.b
// => 2
```

To look up `b.a`:

```
// Has the a property? No. Look up in prototype.
b.a

// Has the a property? No. Look up in the prototype.
b.constructor.prototype.a

// Has the a property? Yes.
b.constructor.prototype.constructor.prototype.a

// return b.constructor.prototype.constructor.prototype.a
// => 1
```

To look up `b.d` (it doesn't exist):

```
// Has the d property? No. Look up in prototype.
b.d

// Has the d property? No. Look up in the prototype.
b.constructor.prototype.d

// Has the d property? No. No more prototype.
b.constructor.prototype.constructor.prototype.d

// return b.constructor.prototype.constructor.prototype.d
// => undefined
```

# Checking Answer

Again, we'll use the tests written in [https://github.com/hayeah/fork2-node-class-spec](https://github.com/hayeah/fork2-node-class-spec) to verify that your work is correct.

Get the latest tests by `git pull`.

# Implement Methods Inheritance

For the first step we'll use prototype chain to setup method inheritance (without the ability to call super).

*Please do this exercise on your own, and don't look at CoffeeScript's `__extend`.*

**Implement**: Methods inheritance with prototype chain.

Example:

```js
var A = Class({
  a: function() {
    return 1;
  }
});

var B = Class({
  b: function() {
    return 2;
  }
},A);

var a = new A();
a.a(); # => 1
a.b;   # undefined

var b = new B();
b.a(); # => 1
b.b(); # => 2
```

Hint: `subclass.prototype.constructor.prototype === superclass.prototype` should be true.

Pass: `mocha verify -R spec -g 'Implement Methods Inheritance'`

```
Implement Methods Inheritance
  b
    ✓ should be an instance of B
    ✓ should be able to call method `a` through inheritance
    ✓ should not have method `a` defined directly on the object
    ✓ should not have method `a` defined directly on the prototype of B
```

**Commit**

```
git commit
```

# Implement The Super Class Property

For a subclass we want to set its `__super__` property to be its parent.

**Implement**: The `__super__` class property should return its super class (or Object by default).

Example:

```js
var A = Class({
  a: function() {
    return 1;
  }
});

var B = Class({
  b: function() {
    return 2;
  }
},A);
B.__super__ # => A
A.__super__ # => Object
```

Pass: `mocha verify -R spec -g "Implement Class __super__"`

```
Implement Class __super__
  ✓ should set the __super__ class property to the parent class
  ✓ should set Object as the default __super__ class
```

**Commit**

```
git commit
```

# Implement Super

Let's make it possible to call the parent class methods via `super`.

**Implement** `super(name,arg1,arg2,...)` should call the parent's method.

Example:

```js
var A = Class({
  foo: function(a,b) {
    return [this.n,a,b];
  }
});

var B = Class({
  foo: function(a,b) {
    return this.super("foo",a*10,b*100);
  }
},A);

var b = new B();
b.n = 1;

c.foo(2,3); // => [1,20,300]
```

Hint: What's the difference between `A.prototype.foo(1,2)` and `A.prototype.foo.call(this,1,2)`?

Hint: Use [arguments](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions_and_function_scope/arguments) and [Function.prototype.apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply) to call the super method.

Hint: To get `arg1, arg2, ...` as an array, you do `[].slice.call(arguments,1)`. [Read this](http://stackoverflow.com/questions/7056925/how-does-array-prototype-slice-call-work).

Hint: This method is fairly complicated. You should focus on passing one test before moving on to the next one.

Pass: `mocha verify -R spec -g "Implement Super call"`

```
  Implement Super
    ✓ should define the `super` method on the prototype
    ✓ should be able to call super method without arguments
    ✓ should call super method with the correct `this` context
    ✓ should be able to call super method with multiple arguments
```

**Commit**

```
git commit
```

# Problem With Calling Super's Super

Your implementation probably has a subtle bug when a super method calls its super method.

In `abc.js` put the following code:

```js
var Class = require("./index.js");

var A = Class({
  foo: function(a,b) {
    return [a,b];
  }
});

var B = Class({
  foo: function(a,b) {
    return this.super("foo",a*10,b*100);
  }
},A);

var C = Class({
  foo: function(a,b) {
    return this.super("foo",a*10,b*100);
  }
},B);

var c = new C()
c.foo(1,2); // should get [100,20000]
```

When you run it, you'd (probably) get this error:

```
$ node abc.js
RangeError: Maximum call stack size exceeded
```

What this error tells you is that you have [infinite recursion](http://stackoverflow.com/questions/6095530/maximum-call-stack-size-exceeded-error) in your code.

Let's try to find where the problem is by putting in a few console.log statements in `abc.js`:

```js
var Class = require("./index.js");

var A = Class({
  foo: function(a,b) {
    console.log("A#foo",a,b);
    return [a,b];
  }
});

var B = Class({
  foo: function(a,b) {
    console.log("B#foo",a,b);
    return this.super("foo",a*10,b*100);
  }
},A);

var C = Class({
  foo: function(a,b) {
    console.log("C#foo",a,b);
    return this.super("foo",a*10,b*100);
  }
},B);

var c = new C()
c.foo(1,2);
```

**Note** `B#foo` means the instance method `foo` of the `B` class. We borrow this notation from Ruby.

Try again:

```
$ node abc.js
...
B#foo Infinity Infinity
B#foo Infinity Infinity
B#foo Infinity Infinity
B#foo Infinity Infinity
B#foo Infinity Infinity
B#foo Infinity Infinity
B#foo Infinity Infinity

RangeError: Maximum call stack size exceeded
```

`B#foo` is being called over and over again.

**Question**: Why is the infinite recursion happening?

Hint: What is the value of `this.super` in `C#foo`? What is the value of `this.super` in `B#foo`? Are they the same or are they different?


## Why Infinite Recursion Is Happening

In your implementation, you probably defined a different `super` method on the prototype of each class:

+ `B.prototype.super` would invoke the methods in `A.prototype`
+ `C.prototype.super` would invoke the methods in `B.prototype`

This seems reasonable but it doesn't work because `this.super` would always call the same function. In our example `C.prototype.super` shadows `B.prototype.super`:

```
var B = Class({
  foo: function(a,b) {
    // this.super === C.prototype.super
    // this calls B.prototype.foo
    return this.super("foo",a*10,b*100);
  }
},A);

var C = Class({
  foo: function(a,b) {
    // this.super === C.prototype.super
    // this calls B.prototype.foo
    return this.super("foo",a*10,b*100);
  }
},B);
```

# Fixing Infinite Recursion (Part 1)

CoffeeScript solves this problem by explictly calling the super through its class.:

```coffee
class Horse extends Animal
  move: ->
    alert "Galloping..."
    super 45
```

compiles to:

```js
Horse.prototype.move = function() {
  alert("Galloping...");
  //
  return Horse.__super__.move.call(this, 45);
};
```

We could do super call in the same way. Our example would look like:

```js
var B = Class({
  foo: function(a,b) {
    return B.prototype.super("foo",a*10,b*100);
  }
},A);

var C = Class({
  foo: function(a,b) {
    // this.super === C.prototype.super
    // this calls B.prototype.foo
    return C.prototype.super("foo",a*10,b*100);
  }
},B);
```

Question: Would `this.constructor.__super__.foo.call(this,a,b)` work?

# Bonus: Fixing Infinite Recursion (Part 2)

There's an hack to make our original API work so our users don't have to explicitly state the class of a super call.

Remember that our problem was that `this.super` is the same function whether it's called in `C#foo` or `B#foo`. Even though `this.super` is the same, can we change its behaviour each time it's called?

+ The first time `this.super` is called in `C#foo`, we want it to call `C.__super__.foo`
+ The second time `this.super` is called in `B#foo`, we want it to call `B.__super__.foo`

The `super` method should maintain a state called `current_class`.

+ When executing `C#foo`, the current_class is `C`.
+ When executing `B#foo`, the current_class is `B`.
+ When executing `A#foo`, the current_class is `A`.

We can keep track of `current_class` in a variable closed over by the `super` function. For example:

```js
var current_class = C;
C.prototype.super = function(name) {
  // changes current_class when calling super
}
```

A sample stack trace in pseudocode for calling `C#foo` is:

```
current_class = C;
C#foo
  this.super (C.prototype.super)
    set current_class as current_class.__super__ (B)
    call current_class's foo (B#foo)
      this.super (C.prototype.super)
        set current_class as current_class.__super__ (A)
        call current_class's foo (A#foo)
        set current_class as B
    set current_class as C
```

Note that when the super call exits from `A#foo`, the `current_class` goes back to `B`. When the super call exits from `B#foo`, the `current_class` goes back to `C`.

**Implement** Should be able to call super's super by manipulating the current class of a super call.

Example:

```js
var A = Class({
  foo: function(a,b) {
    return [a,b];
  }
});

var B = Class({
  foo: function(a,b) {
    return this.super("foo",a*10,b*100);
  }
},A);

var C = Class({
  foo: function(a,b) {
    return this.super("foo",a*10,b*100);
  }
},B);


var c = new C();
c.foo(1,2); => [100,20000]
```

Pass: `mocha verify -R spec -g "Implement Super's Super"`

```
Implement Super's Super
  ✓ should be able to call super's super
```

**Commit**

```
git commit
```

# Bonus: JS Superman - Implement John Resig's _super Method

[John Resig](http://ejohn.org) (jQuery's inventor) came up with a very cool technique to call super methods. It looks like:

```js
var B = Class({
  foo: function(a,b) {
    // calls A.prototype.foo
    return this._super(a*10,b*100);
  }
},A);
```

`B#foo` would call `A.prototype.foo` magically. Both the class and the method name are implicit.

Read [Simple JavaScript Inheritance](http://ejohn.org/blog/simple-javascript-inheritance) to see how it's done.

**Implement** the jresig's `_super`.

**Commit**

```
git commit
```