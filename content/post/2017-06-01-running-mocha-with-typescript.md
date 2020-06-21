---
date: "2017-06-01T00:00:00Z"
tags: ["Mocha", "TypeScript"]
title: Running Mocha with TypeScript
---

I was working on proof-of-concept to TypeScript with Mocha and I wanted to share my learning.

I was working on proof-of-concept to use TypeScript with Mocha. My objective was building a project where both the source and the tests written in TypeScript, executing tests using npm scripts and gulp and finally with a good debugging experience in both Visual Studio code and Web Storm.

TL;DR, Find the working sample [here](http://github.com/hossambarakat/mocha-with-typescript)

## Project Skeleton

The first step is to create an empty project directory and run `npm init` inside of it, then create two folders `src` and `test`:

```
- src
- test
package.json
```

## Installing TypeScript

The project will written in TypeScript so let's start by installing the typescript package:

```shell
npm install typescript --save-dev
```

The TypeScript uses a file called `tsconfig.json` in the root directory of the solution to define the compiler options so add new file to the root directory with the following content.

```json
{
    "compilerOptions":{
        "module": "commonjs",
        "sourceMap": true,
        "target": "es2015",
        "isolatedModules": true
    }
}
```

You can read more about the tsconfig [here](https://www.typescriptlang.org/docs/handbook/tsconfig-json.html)

## The Calculator

The project will be a simple calculator that can add two numbers. Add new file called `Calculator.ts` inside the `src` folder. The source code that we are going to test will be simple `Calculator` class with one method `add`:

```typescript
export default class Calculator
{
    public Add(a :number,b:number): number {
        return a + b;
    }
}
```

## The Test

The tests will be written in [Mocha](http://mochajs.org) and the assertions will be done using [Chai](http://chaijs.com) so let's start by installing them

```shell
npm install mocha --save-dev
npm install chai --save-dev
```

Now proceed with creating a new file called `calculator.spec.ts` inside the `test` directory:

```typescript
import { expect } from 'chai';
import Calculator from '../src/calculator';

describe("Calculator", () => {
    describe("Add", () => {
        it("Should return 3 when a = 1 and b = 2", () => {
            let calc = new Calculator();

            var result = calc.Add(1,2);

            expect(result).to.equal(3);
        });
    })
});
```

## Running the tests

I'd like to be able to run the test through npm scripts as well as using gulps but first we need to install another package in order to be able to use Mocha with TypeScript:

```shell
npm install ts-node --save-dev
```

> TypeScript Node is TypeScript execution environment and REPL for node

### NPM Scripts

Update the `package.json` scripts section:

```json
"scripts": {
    "test": "./node_modules/.bin/mocha --compilers ts:ts-node/register ./test/*.spec.ts"
  }
```

As you noticed in the above script we used the `--compilers` parameter to use the ts-node module to compile the TypeScript files.

Now you should be able to run the tests from command line:

```shell
npm test
```

The following result should be shown on your command window

```shell
  Calculator
    Add
      ✓ Should return 3 when a = 1 and b = 2


  1 passing (6ms)
```

### Gulp

We need to install two more packages to be able to use Gulp:

```shell
npm install gulp --save-dev
npm install gulp-mocha --save-dev
```

Then adding `gulpfile.js` to the root directory:

```javascript
const gulp = require('gulp');
const mocha = require('gulp-mocha');

gulp.task('run-tests', function(){
  return gulp.src('test/*.spec.ts')
        .pipe(mocha({
            reporter: 'nyan',
            require: ['ts-node/register']
        }));
});

gulp.task('default', [ 'run-tests' ]);
```

You can run the tests using gulp by running `gulp` command and you should see the output similar to the following:

```shell
[22:16:05] Using gulpfile ~/My files/Temp/mocha-with-typescript/gulpfile.js
[22:16:05] Starting 'run-tests'...
 1   -__,------,
 0   -__|  /\_/\
 0   -_~|_( ^ .^)
     -_ ""  ""

  1 passing (8ms)

[22:16:06] Finished 'run-tests' after 605 ms
[22:16:06] Starting 'default'...
[22:16:06] Finished 'default' after 8.96 μs
```

## Debugging

### Visual Studio Code

You can debug TypeScript tests inisde visual studio using node debug configurations with V8 inspector protocol, You can set the V8 inspect protocol by setting `protocol` to `inspect`:

```json
{
    "name": "Run mocha",
    "type": "node",
    "request": "launch",
    "program": "${workspaceRoot}/node_modules/mocha/bin/_mocha",
    "stopOnEntry": false,
    "args": ["--no-timeouts", "--compilers", "ts:ts-node/register", "${workspaceRoot}/test/**/*.spec.ts"],
    "cwd": "${workspaceRoot}",
    "protocol": "inspector"
}
```

### Web Storm

You can debug the TypeScript tests inside Web Storm by using the normal Mocha configuration but remember to include `--require ts-node/register` in the `Extra Mocha options` field:

![](/assets/2017-06-01-running-mocha-with-typescript/webstorm-config.png)

