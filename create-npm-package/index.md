# Create An NPM Package

In this lesson we'll build a simple npm package. The package installs the `greet` command:

```sh
$ greet howard
hello howard
```

If `greet` had had a few more martinis then it should:

```
$ greet howard --drunk
hello howard, you look sexy today
```

By building this simple package, we'll become familiar with the core tools used in a nodejs development environment.

+ How to create an npm package
+ CommonJS module system

# Using npm init

Let's use the `npm init` command to start a new npm project in the `greet` directory:

```
$ mkdir greet && cd greet
$ npm init
name: (greet)
version: (0.0.0)
description: A simple and naive greeter
entry point: (index.js)
test command:
git repository:
keywords: hello-world
author: Howard Yeh
license: (ISC) MIT
Is this ok? (yes) yes
```

It creates the file `package.json`, which describes the project. Let's take a look:

```
$ cat package.json
{
  "name": "greet",
  "version": "0.0.0",
  "description": "echo \"Error: no test specified\" && exit 1",
  "main": "index.js",
  "scripts": {
    "test": "mocha"
  },
  "keywords": [
    "hello-world"
  ],
  "author": "Howard Yeh",
  "license": "MIT"
}
```

# Create The Greet Module

Let's package a very simple `greet` function as an node module.

```js
function greet(name) {
  return "hello, " + name;
}
```

## Loading The Package Module Locally

In `package.json`, there's the `main` value:

```
{
  "name": "greet",
  ...
  "main": "index.js"
}
```

This means that `index.js` is the file that would be loaded when the module is required.

Create `index.js`. Just make it an empty file for now.

```
$ touch index.js
```

We can require this file directly (it exports an empty object by default):

```
$ node -e 'console.log(require("./index.js"))'
{}
```

Or we can require it without the `.js` extension:

```
$ node -e 'console.log(require("./index"))'
{}
```

Or we can require the project directory (which requires the `main` value of `package.json`:

```
$ node -e 'console.log(require("./"))'
{}
```

All of the above loads `index.js` as a module.

## Installing The Project As Package

Instead of loading the module by path, we want to be able to load the module by name:

```js
// require by module name
var greet = require("greet");
// is the the same as requiring
var greet = require("/path_to_greet_module/index.js");
```

Let's try to require the `greet` module:

```
$ node -e 'console.log(require("greet"))'

module.js:340
    throw err;
          ^
Error: Cannot find module 'greet'
```

We get this error because the module isn't installed. For development purposes, we can use the [`npm link`](https://www.npmjs.org/doc/cli/npm-link.html) command to install the current project directory as package:

Hint: If the global node_modules path is not writable by user, you need to run `sudo npm link`

Hint: The `NODE_PATH` environmental variable determines where the global node_module path is. If it is not set, `npm link` would result in an error. The global modules path can be different depending on where and how you install NodeJS.


```js
$ npm link
~/.nvm/v0.10.26/lib/node_modules/greet -> ~/greet
```

The installed `greet` module is a symlink to the project directory. Let's require our module again:

```
$ node -e 'console.log(require("greet"))'
{}
```

It works.

To uninstall the linked package:

```
$ npm unlink -g greet
unbuild greet@0.0.0
```

**Question**: `npm link` installs a package by creating a symbolic link. What's a symbolic link?

**Question**: `npm link` installs the current directory as a global package. What's the difference between installing a package globally and installing it locally? Read [npm 1.0: Global vs Local installation](http://blog.nodejs.org/2011/03/23/npm-1-0-global-vs-local-installation)

## The Greet CommonJS Module

Right now, our `greet` module is empty. It's just the empty object:

```
$ node
> greet = require("greet")
{}
```

NodeJS uses the CommonJS module spec. Basically, whatever value you assign to `module.exports` is the value returned by `require`. See [documentation on module](http://nodejs.org/docs/latest/api/modules.html#modules_the_module_object) for more details.

**Implement**: index.js should export the greet function

Hint: In `index.js`, complete the following:

```
// file: index.js
module.exports = ...
```

Example:

```
$ node
> greet = require("greet")
[Function: greet]
> greet("howard")
'hello, howard'
```

## Building The Greet Package

`npm link` is good for developing the package locally. [npm pack](https://www.npmjs.org/doc/cli/npm-pack.html) is used to create a user installable package.

```
$ npm pack
greet-0.0.0.tgz
```

Running the packing command creates a compressed tarball (.tgz). We can look inside to see what it contains:

```
$ tar -ztf greet-0.0.0.tgz
package/package.json
package/index.js
```

You can install the tar file:

```
$ npm install greet-0.0.0.tgz
npm install greet-0.0.0.tgz
npm WARN package.json greet@0.0.0 No repository field.
npm WARN package.json greet@0.0.0 No README data
greet@0.0.0 node_modules/greet
```

Uninstall it:

```
$ npm uninstall greet
npm uninstall greet
unbuild greet@0.0.0
```

# Create The greet Executable

We want to be able to use the greet module as a command. Add the `bin` field to `package.json` ([detailed doc](https://docs.npmjs.com/files/package.json)):

```
// in package.json
"bin" : { "greet" : "./bin/greet.js" }
```

This specifies that when the package is installed, the script `./bin/greet.js` should be installed as the executable `greet`.

Now let's create the executable script at `bin/greet.js`:

```
#!/usr/bin/env node
console.log("Hello World");
```

The first line is a <a href="http://en.wikipedia.org/wiki/Shebang_(Unix)">shebang</a> directive. It's a way for the computer to know how to run the script as a program.

(Why do we use `/usr/bin/env` instead of the node command? Because the `node` executable can be at different places on different systems, and the `env` uses the environment's `PATH` to figure out which `node` program to run.)

Change the file permission as an executable file:

```
$ chmod a+x bin/greet.js
```

Now we can execute the script:

```
$ ./bin/greet.js
Hello World
```

Let's run `npm link` again:

```
$ npm link
~/.nvm/v0.10.26/bin/greet -> ~/greet/bin/greet.js
~/.nvm/v0.10.26/lib/node_modules/greet -> ~/greet
```

Notice how `~/.nvm/v0.10.26/bin/greet` is a link to our script. Now we can run the `greet` command:

```
$ greet
Hello World
```

**Implement** modify `bin/greet.js` so `greet howard` uses the `greet` function exported by `index.js`

HINT: Use `require` with a relative path. See [File Module](http://nodejs.org/api/modules.html#modules_file_modules)

HINT: `process.argv` is an array that contains the command arguments.

Example:

```
$ greet howard
hello, howard
```

# Drink A Few Nice Martinis

Let's change the `greet` function to accept an extra argument:

```js
function greet(name,drunk) {
  if(drunk) {
    return "hello " + name + ", you look sexy today";
  } else {
    return "hello, " + name;
  }
}
```

**Implement**: Use the [minimist](https://github.com/substack/minimist) package to parse command arguments, and accept the `--drunk` option.

Example:

```
$ greet howard --drunk
hello howard, you look sexy today
```

HINT: `npm install minimist --save` to add the package as dependency.

HINT: `var parseArgs = require('minimist')` is a function.

# Module Require Path

If you are not familiar with how `require` find the right file to load, read [Loading from node_modules Folders](http://nodejs.org/api/modules.html#modules_loading_from_node_modules_folders).

# Push Repository To Github

Create a repository called `fork2-node-greet` on Github, and push your code.