# Using CoffeeScript

In this lesson we'll convert the `greet` module to use [CoffeeScript](http://coffeescript.org/).

+ Convert JavaScript sources to CoffeeScript
+ `npm pack` should package only the `.js` files
+ Keep CoffeeScript as a development dependency only
+ Convert Mocha tests to CoffeeScript

# Convert JavaScript To CoffeeScript

We will put all the CoffeeScript source files under the `src` directory, and compile them into JavaScript files into the `lib` directory.

(If this is your first time writing CoffeeScript, you can use [http://js2coffee.org](http://js2coffee.org) to help you.)

Let's install CoffeeScript:

```
$ npm install coffee-script --save-dev
$ coffee -v
CoffeeScript version 1.6.2
```

To compile all the CoffeeScript source files in `src`, run:

```
$ coffee --compile --output lib src
```

You can use the `--watch` option to get the coffee command to automatically recompile when a CoffeeScript file is modified.

(See `coffee --help` for descriptions of the command options).

## Convert index.js

**Implement** Create `src/index.coffee` and remove `index.js`.

Hint: Change the `main` value in `package.json`.

Hint: Change `bin/greet.js` so it requires the right path.

Tests should still pass:

```
$ mocha
mocha
  greet
    ✓ should greet a person by name
    ✓ should greet a person flirtatiously if drunk
  2 passing (6ms)
```

Command should still work:

```
$ greet howard
hello, howard
```

**Commit**:

(Don't commit the compiled JavaScript files in `lib`)

```
git commit
```

## Convert bin/greet.js

Because we don't have CoffeeScript as a dependency when a user installs the `greet` module, we need to keep `bin/greet.js` as a JavaScript file that node knows how to execute.

Let's keep `bin/greet.js` minimal:

```
#!/usr/bin/env node
var command = require("../lib/command");
command();
```

**Implement**  Create `src/command.coffee`, which exports a function that runs the command logic when called.

**Commit**:

(Don't commit the compiled JavaScript files in `lib`)

```
git commit
```

# NPM Pack & .npmignore

As a package author, you'd want to keep your package as small as possible. Right now the `greet` package contains a lot of crap:

```
$ npm pack
greet-0.0.0.tgz
```

And it contains the files:

```
package/package.json
package/bin/greet.js
package/lib/command.js
package/lib/index.js
package/src/command.coffee
package/src/index.coffee
package/test/greet_spec.js
package/test/mocha.opts
package/test/support/helper.js
```

Let's exclude the files in `src` and `test` from our package. Create `.npmignore`:

```
.npmignore
.git*
.DS_Store
*.tgz
test/
src/
```

We build the package again, and it now contains only the files we actually need:

```
$ tar -ztf greet-0.0.0.tgz
package/package.json
package/bin/greet.js
package/lib/command.js
package/lib/index.js
```

**Commit**:

```
git commit
```

# Build Project With Makefile

There's a bunch of fancy task runners / project builders available for NodeJS, like

+ [GruntJS](http://gruntjs.com)
+ [gulpjs](http://gulpjs.com/)
+ [CoffeeScript Cakefile](http://coffeescript.org/#cake)

For a small project a Makefile is a lot simpler than all of the above because we just need to run simple shell commands.

(Read [task automation with npm run](http://substack.net/task_automation_with_npm_run) to see how a more complicated project with css and javascript could be built with simple shell scripts.)

Create `Makefile`:

```
compile:
  coffee --compile --output lib src

.PHONY: compile
```

Few things to notice about `Makefile`:

+ Indentations **MUST** be tabs
+ `compile` is our only build target
+ the `compile` target runs the coffee compilation
+ `.PHONY` is a list of build targets that don't produce a file (this can be invoked as a task). [See details](http://stackoverflow.com/questions/2145590/what-is-the-purpose-of-phony-in-a-makefile).

Now we can use make to compile our project:

```
$ make compile
coffee --compile --output lib src
```

(If you get an error like "Makefile:2: *** missing separator.  Stop.", make sure you are not using spaces for indentation.)

**Implement** Create a `test` task to run test.

HINT: The `test` task should depend on the `compile`

HINT: To make the `task` foo depend on the task `bar`:

```
# task foo
foo: bar
  echo foo

# task bar
bar:
  echo bar
```

**Implement** Create a `package` to build project package.

HINT: The `package` task should depend on `test`. It should fail if the test suite fails.

Expected output:

```
$ make package
coffee --compile --output lib src
mocha
  greet
    ✓ should greet a person by name
    ✓ should greet a person flirtatiously if drunk
  2 passing (6ms)
npm pack
greet-0.0.0.tgz
```

**Commit**

```
git commit
```

# Convert Tests To CoffeeScript

To run tests written in CoffeeScript, add the `--compilers` option to `mocha.opts`:

```
--compilers coffee:coffee-script/register
```

Hint: If your CoffeeScript version is `<= 1.6`, the `--compilers` option is `--compilers coffee:coffee-script`. See: [Cannot run Mocha with CoffeeScript](http://stackoverflow.com/questions/9940838/cannot-run-mocha-with-coffeescript)

**Implement**: Convert to `test/greet_spec.coffee` and `test/support/helper.coffee`

**Commit**

```
git commit
```