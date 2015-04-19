# Setting Up Mocha For Testing

In this lesson we'll setup testing for the project using the [Mocha test framework](http://visionmedia.github.io/mocha)

# Install Mocha As Package Dependency

[Mocha](visionmedia.github.io/mocha) is a test framework for NodeJS.

Let's install `mocha` locally (using the `--save-dev` option to add it to `package.json` as a package dependency):

```
> npm install mocha --save-dev
npm WARN package.json greet@0.0.0 No repository field.
npm WARN package.json greet@0.0.0 No README data
npm http GET https://registry.npmjs.org/mocha
npm http 200 https://registry.npmjs.org/mocha
npm http GET https://registry.npmjs.org/mocha/-/mocha-1.18.2.tgz
npm http 200 https://registry.npmjs.org/mocha/-/mocha-1.18.2.tgz
...
mocha@1.18.2 node_modules/mocha
├── diff@1.0.7
├── debug@0.7.4
├── growl@1.7.0
├── commander@2.0.0
├── mkdirp@0.3.5
├── jade@0.26.3 (commander@0.6.1, mkdirp@0.3.0)
└── glob@3.2.3 (inherits@2.0.1, graceful-fs@2.0.3, minimatch@0.2.14)
```

Now mocha is installed in the local `./node_modules/mocha` directory.

And in `package.json` you can see the development dependencies:

```
"devDependencies": {
  "mocha": "^1.18.2"
}
```

Executables installed by node packages are available in `./node_modules/.bin/`:

```
$ ./node_modules/.bin/mocha --version
1.18.2
```

This is actually a symbolic link to a file in `./node_modules/mocha`:

```
$ ls -l ./node_modules/.bin/mocha
./node_modules/.bin/mocha -> ./node_modules/mocha/bin/mocha
```

If we want to run the locally installed `mocha` executable as `mocha` instead of `./node_modules/.bin/mocha`, we need to add `./node_modules/.bin` to the `PATH` environment variable:

```
$ export PATH=./node_modules/.bin:$PATH
$ which mocha
./node_modules/.bin/mocha
$ mocha --version
1.18.2
```

Add `export PATH=./node_modules/.bin:$PATH` to `~/.bashrc` if you want to be able to run locally installed npm executables in the future.

# Running Test

The `scripts` key is used to specify tasks that the package can run (see: [list of available tasks](https://www.npmjs.org/doc/misc/npm-scripts.html)). In `package.json` we have:

```js
{
  "name": "greet",
  ...
  "scripts": {
    "test": "mocha"
  },
  ...
}
```

In this example, the `test` task runs the `mocha` command. We can try running the test:

```
$ npm test
> greet@0.0.0 test /Users/howard/workspace/node-bootcamp/greet
> mocha


/Users/howard/workspace/node-bootcamp/greet/node_modules/mocha/bin/_mocha:432
      if (!files.length) throw new Error("cannot resolve path (or pattern) '"
                               ^
Error: cannot resolve path (or pattern) 'test'
...
```

This error is because we don't have the `test` directory. Let's create it:

```
$ mkdir test
$ npm test
npm test

> greet@0.0.0 test /Users/howard/workspace/node-bootcamp/greet
> mocha

  0 passing (1ms)
```

Cool it works now.

# The BDD Testing Style

The Mocha test framework allows you to write your tests in many different styles. We'll choose to write our tests using the BDD style and the "expect" syntax of the [Chai assertion library](http://chaijs.com).

An example test looks like:


```js
var expect = require("chai").expect;
describe('Array', function(){
  before(function(){
    // ...
  });

  describe('#indexOf()', function(){
    it('should return -1 when not present', function(){
      expect([1,2,3].indexOf(4)).to.equal(-1);
    });
  });
});
```

# Writing The First Test

Install both mocha and chai as development dependencies:

```
$ npm install mocha chai --save-dev
...
chai@1.9.1 node_modules/chai
├── assertion-error@1.0.0
└── deep-eql@0.1.3 (type-detect@0.1.1)

mocha@1.18.2 node_modules/mocha
├── diff@1.0.7
├── debug@0.7.4
├── growl@1.7.0
├── commander@2.0.0
├── mkdirp@0.3.5
├── jade@0.26.3 (commander@0.6.1, mkdirp@0.3.0)
└── glob@3.2.3 (inherits@2.0.1, graceful-fs@2.0.3, minimatch@0.2.14)
```

There's no test yet:

```
$ mocha
  0 passing (1ms)
```

The `mocha` command find files that match the pattern `test/*.js` and run the tests in those files. Let's Create the first test file at `test/greet_spec.js`:

```js
var expect = require("chai").expect
describe('greet', function(){
  it("is a dummy success case", function(){
    expect(1).to.eql(1);
  });
});
```

Let's run mocha (with the `-R` option to get more detailed output):

```
$ mocha -R spec

  greet
    ✓ is a dummy success case

  1 passing (6ms)
```

**Implement**: Write the actual test cases for the `greet` function. There are two cases:

+ your test file should require the greet module
+ should greet a person by name
+ should greet a person flirtatiously if drunk

Hint: [Chai BDD API reference](http://chaijs.com/api/bdd).

The test output should look like:

```
$ mocha -R spec

  greet
    ✓ should greet a person by name
    ✓ should greet a person flirtatiously if drunk
  2 passing (6ms)
```

# spec helper

As your test suite grows bigger into multiple files, you might find it helpful to be able to share code across all files.

In our case instead of having to require `expect` in every test file, let's create a spec helper to load it.

Remove `var expect = require("chai").expect` from `test/greet_spec.js`. The test suite should now fail:

```
$ mocha

  ․․

  0 passing (4ms)
  2 failing

  1) greet should greet a person by name:
     ReferenceError: expect is not defined

  2) greet should greet a person flirtatiously if drunk:
     ReferenceError: expect is not defined
```

In `test/support/helper.js` add the following:

```js
global.expect = require("chai").expect;
```

Let's run mocha again with the `--require` option to load the helper:

```
$ mocha --require test/support/helper
․․
  2 passing (4ms)
```

It works.

**Question**: How is `var expect = ...` different from `global.expect = ...`? Does it work if in `helper.js` you use `var expect = require("chai").expect;`

Hint: See [global](http://nodejs.org/api/globals.html#globals_global)

# mocha.opts

It's cumbersome to have to type `--require test/support/helper` everytime you run a test. You can set mocha's default options by creating the `mocha.opts` in the test directory.

In `test/mocha.opts` add:

```
--require test/support/helper
--reporter spec
```

Now you can run mocha without specifying options:

```
$ mocha
  greet
    ✓ should greet a person by name
    ✓ should greet a person flirtatiously if drunk
  2 passing (6ms)
```

You can also run the test suite as an npm task:

```
$ npm test
npm test
> greet@0.0.0 test /Users/howard/workspace/node-bootcamp/greet
> mocha
  greet
    ✓ should greet a person by name
    ✓ should greet a person flirtatiously if drunk
  2 passing (6ms)
```

# Bonus

If you'd like to gain more experience using Mocha, you are encouraged to read [Better CoffeeScript Testing With Mocha](http://code.tutsplus.com/tutorials/better-coffeescript-testing-with-mocha--net-24696) is a great tutorial that guides you through building a todo list using TDD.

It uses the `should` syntax instead of `expect`.
