# Implement CoffeeScript's Class System

In this lesson we'll practice working with prototype and constructor by implementing a basic class system similar to CoffeeScript's class.

In the next lesson we'll implement inheritance.

# Start Project

Let's start a new project called `Class`.

+ `git init` and `npm init` in a new directory.
+ `npm install mocha chai --save-dev`

# A CoffeeScript Zoo

Let's use the [example class hierarchy](http://coffeescript.org/#classes) from CoffeeScript's documentation.

In `animals.coffee` add:

```coffeescript
alert = (msg) -> console.log(msg)

class Animal
  constructor: (@name) ->

  move: (meters) ->
    alert @name + " moved #{meters}m."

class Snake extends Animal
  move: ->
    alert "Slithering..."
    super 5

class Horse extends Animal
  move: ->
    alert "Galloping..."
    super 45

sam = new Snake "Sammy the Python"
tom = new Horse "Tommy the Palomino"

sam.move()
tom.move()
```

Then run it:

```
$ coffee animals.coffee
Slithering...
Sammy the Python moved 5m.
Galloping...
Tommy the Palomino moved 45m.
```

You can see the the compiled JavaScript:

```js
$ coffee --compile --print --bare animals.coffee
var Animal, Horse, Snake, alert, sam, tom,

// ...

alert = function(msg) {
  return console.log(msg);
};

Animal = (function() {
  function Animal(name) {
    this.name = name;
  }

  Animal.prototype.move = function(meters) {
    return alert(this.name + (" moved " + meters + "m."));
  };

  return Animal;

})();
// ...
```

# Our Own Class Method

We'll imitate CoffeeScript's `class` by implementing our own `Class` function in JavaScript. It looks like:

```js
Animal = Class({
  constructor: function(name) {
    this.name = name;
  },
  move: function(meters) {
    alert(this.name + "moved " + meters + "m.");
  }
});

Snake = Class({
  move: function() {
    alert("Slithering...");
    this.super("move",5);
  }
}, Animal);

Horse = Class({
  move: function() {
    alert("Galloping...");
    this.super("move",45);
  }
}, Animal);

sam = new Snake("Sammy the Python");
tom = new Horse("Tommy the Palamino");
```

# Checking Your Work Is Correct

Clone the test suite into your project (it should be the "verify" directory):

`git clone https://github.com/hayeah/fork2-node-class-spec.git verify`

You can check your answer against the test suite.

# Implement Class Constructor

Let's look at a very simple class `Foo`:

```js
function Foo() {
  this.a = 10;
}
Foo.prototype.b = 20;
foo = new Foo();
foo2 = new Foo();
foo.a // => 10
foo.b // => 20
foo.b = 30
foo.b // => 30

foo2.a // => 10
foo2.b // => 20
Foo.prototype.b = 40;
foo.b // => ?
foo2.b // => ?
```

**Question**: What is `foo.constructor`?

**Question**: Why is `foo2.b` 20 even though you didn't assign a value to it?

**Question**: What are the values of `foo.b` and `foo2.b` after you assign 40 to `Foo.prototype.b`?

If you are not sure how this works, read [JavaScript constructors, prototypes, and the new keyword](http://pivotallabs.com/javascript-constructors-prototypes-and-the-new-keyword).

**Implement**: Can define classes with the `Class` function.

Hint: A "class" in JavaScript is just a constructor function. So `Class` should return a function.

Example:

```js
var Foo = Class({
  initialize: function(a,b) {
    this.a = a;
    this.b = b;
  }
});
```

It should also be possible to define a class without a constructor:

```js
var Bar = Class({});
bar = new Bar();
```

Pass: `mocha verify -R spec -g 'Implement Class Constructor'`

```
Implement Class Constructor
  ✓ should return a class constructor function
  ✓ should be able define a class
  ✓ should be able define a class without constructor
```

**Commit**

```
git commit
```

# Implement Instance Methods

Methods are just properties that are functions. To define a method, assign a function to the constructor's prototype:

```js
// The Animal class
function Animal() {}
// Define the move method
Animal.prototype.move = function() {
  // ...
};
```

**Question:** How is assigning a function in the constructor different? Like:

```js
// The Animal class
function Animal() {
  // Define the move method
  this.move = function() {
    // ...
  };
}
```

**Implement**: Can define instance methods

Example:

```js
Foo = Class({
  initialize: function(a,b) {
    this.a = a;
    this.b = b;
  },

  getA: function() {
    return this.a;
  },

  getB: function() {
    return this.b;
  }
});

foo = new Foo(1,2);
foo.getA(); // => 1
foo.getB(); // => 2
```

Pass: `mocha verify -R spec -g 'Implement Instance Methods'`

```
Implement Instance Methods
  ✓ should be able to define methods
  ✓ should not define `initialize` as a method
```

**Commit**

```
git commit
```
