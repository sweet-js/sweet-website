---
layout: default
toc: true
use_anchors: true
---

# Introduction

Sweet brings the hygienic macros of languages like Scheme and Rust to JavaScript.
Macros allow you to sweeten the syntax of JavaScript and craft the language you’ve always wanted.

## Installation and Getting Started

Install Sweet with npm:

```sh
$ npm install -g @sweet-js/sweet-cli @sweet-js/sweet-helpers
```

This globally installs the `sjs` binary, which is used to compile Sweet code.

For example, say you'd like to sweeten JavaScript with a simple hello world macro.
You can write it down as the following:

```js
// sweet_code.js
syntax hi = function (ctx) {
  return #`console.log('hello, world!')`;
};
hi
```

Then, you can use the `sjs` command to compile the sweetened code into plain JavaScript:

```sh
$ sjs sweet_code.js
console.log('hello, world!')
```


## Babel Backend

Note that Sweet uses [Babel](https://babeljs.io/) as a backend. After Sweet has done its work of finding and expanding macros, the resulting code is run through Babel.

By default Babel performs no transformations so you will need to configure it according to your needs. The easiest way to do this is via a [`.babelrc`](https://babeljs.io/docs/usage/babelrc/) file. A minimal configuration looks something like:

```js
{
    "presets": ["es2015"]
}
```

Where you've installed the es2015 preset via npm:

```
npm install babel-preset-es2015
```

If you do not want to use Babel at all, pass the `--no-babel` flag to `sjs`.

# Sweet Hello

How do macros work?
Well, in a sense macros are a bit like compiletime functions; just like functions, macros have definitions and invocations which work together to abstract code into a single location so you don't keep repeating yourself.

Consider the hello world example again:

```js
syntax hi = function (ctx) {
  return #`console.log('hello, world!')`;
};
hi
```

The first three lines make up the macro definition. The `syntax` keyword is a bit like `let` in that it creates a new variable in the current block scope. Rather than create a variable for a runtime value, `syntax` creates a new variable for a _compiletime value_. In this case, `hi` is the variable bound to the compiletime function defined on the first three lines.

In this example, `syntax` sets the variable to a function, but the variable can be set to any JavaScript value. Currently, this point is rather academic since Sweet does not provide a way to actually _use_ anything other than a compiletime function. However, this feature will be added eventually.

Once a macro has been defined, it can be invoked. On line three above the macro is invoked simply by writing `hi`.

When the Sweet compiler sees the `hi` identifier bound to the compiletime function, the function is invoked and its return value is used to replace the invoking occurrence of `hi`. In this case, that means that `hi` is replaced with `console.log('hello, world!')`.

Compiletime functions defined by `syntax` must return an array of syntax objects. You can easily create these with a _syntax template_. Syntax templates are template literals with a `\#` tag, which create a List (see the [immutable.js docs](https://facebook.github.io/immutable-js/docs/#/List) for its API)
of syntax objects.

A _syntax object_ is  Sweet's internal representation of syntax. Syntax objects are somewhat like tokens from traditional compilers except that delimiters cause syntax objects to nest. This nesting gives Sweet more structure to work with during compilation. If you are coming from Lisp or Scheme, you can think of them a bit like s-expressions.


# Sweet New

Let's move on to a slightly more interesting example.
Pretend you are using an OO framework for JavaScript where instead of using `new` we want to call a `.create` method that has been monkey patched onto `Function.prototype` (don't worry, I won't judge... much). Rather than manually rewrite all usages of `new` to the `create` method you could define a macro that does it for you.

```js
syntax new = function (ctx) {
  let ident = ctx.next().value;
  let params = ctx.next().value;
  return #`${ident}.create ${params}`;
};

new Droid('BB-8', 'orange');
```

```js
Droid.create('BB-8', 'orange');
```

Here you can see the `ctx` parameter to the macro provides access to syntax at the macro call-site. This parameter is an iterator called the _macro context_.

The macro context has the type:

```
{
  next: () -> {
    done: boolean,
    value: Syntax
  }
}
```

Each call to `next` returns the successive syntax object in `value` until there is nothing left in which case `done` is set to true. Note that the context is also an iterable so you can use `for-of` and related goodies.

Note that in this example we only call `next` twice even though it looks like there is more than two bits of syntax we want to match. What gives? Well, remember that delimiters cause syntax objects to nest. So, as far as the macro context is concerned there are two syntax objects: `Droid` and a single paren delimiter syntax object containing the three syntax objects `'BB-8'`, `,`, and `'orange'`.

After grabbing both syntax objects with the macro context iterator we can stuff them into a syntax template. Syntax templates allow syntax objects to be used in interpolations so it is straightforward to get our desired result.

# Sweet Let

Ok, time to make some ES2015. Let's say we want to implement `let`.
We only need one new feature you haven't seen yet:

```js
syntax let = function (ctx) {
  let ident = ctx.next().value;
  ctx.next(); // eat `=`
  let init = ctx.expand('expr').value;
  return #`
    (function (${ident}) {
      ${ctx} // <2>
    }(${init}))
  `
};

let bb8 = new Droid('BB-8', 'orange');
console.log(bb8.beep());
```

```js
(function(bb8) {
  console.log(bb8.beep());
})(Droid.create("BB-8", "orange"));
```

Calling `expand` allows us to specify the grammar production we want to match; in this case we are matching an expression. You can think matching against a grammar production a little like matching an implicitly-delimited syntax object; these matches group multiple syntax object together.


# Sweet Cond

One task we often need to perform in a macro is looping over syntax. Sweet helps out with that by supporting ES2015 features like `for-of`. To illustrate, here's a `cond` macro that makes the ternary operator a bit more readable:

```js
import { unwrap, isKeyword } from '@sweet-js/helpers' for syntax;

syntax cond = function (ctx) {
  let bodyCtx = ctx.contextify(ctx.next().value);

  let result = #``;
  for (let stx of bodyCtx) { // <2>
    if (isKeyword(stx) && unwrap(stx).value === 'case') {
      let test = bodyCtx.expand('expr').value;
      // eat `:`
      bodyCtx.next();
      let r = bodyCtx.expand('expr').value;
      result = result.concat(#`${test} ? ${r} :`);
    } else if (isKeyword(stx) && unwrap(stx).value === 'default') {
      // eat `:`
      bodyCtx.next();
      let r = bodyCtx.expand('expr').value;
      result = result.concat(#`${r}`);
    } else {
      throw new Error('unknown syntax: ' + stx);
    }
  }
  return result;
};

let x = null;

let realTypeof = cond {
  case x === null: 'null'
  case Array.isArray(x): 'array'
  case typeof x === 'object': 'object'
  default: typeof x
}
```

```js
var x = null;
var realTypeof = x === null ? "null" :
                 Array.isArray(x) ? "array" :
                 typeof x === "undefined" ? "undefined" : typeof x);
```

Since delimiters nest syntax in Sweet, we need a way to get at the nested syntax. The `contextify` method on the macro context provides this functionality. Calling `contextify` with a delimiter will wrap the syntax inside the delimiter in a new macro context object that you can iterate over.

Utility functions are available for import at `'@sweet-js/helpers'` to inspect the syntax you are processing. Note the `for syntax` on the import declaration, which is required to make the imports available inside of a macro definition. Modules and macros are described in the modules section of this document.

The two imports used by this macro are `isKeyword` and `unwrap`. The `isKeyword` function does what it sounds like: it tells you if a syntax object is a keyword. The `unwrap` function returns the primitive value of a syntax object. In the case of unwrapping a keyword, it returns the string representation (if the syntax was a numeric literal `unwrap` would return a number). Since some syntax objects do not have a reasonable primitive representation (e.g. delimiters), the `unwrap` function ether returns an object with a single `value` property or an empty object.

The full API provided by the helper module is described in the [reference documentation](reference.html).

# Sweet Class

So putting together what we've learned so far, let's make the sweetest of ES2015's features: `class`.

```js
import { unwrap, isIdentifier } from '@sweet-js/helpers' for syntax;

syntax class = function (ctx) {
  let name = ctx.next().value;
  let bodyCtx = ctx.contextify(ctx.next().value);

  // default constructor if none specified
  let construct = #`function ${name} () {}`;
  let result = #``;
  for (let item of bodyCtx) {
    if (isIdentifier(item) && unwrap(item).value === 'constructor') {
      construct = #`
        function ${name} ${bodyCtx.next().value}
        ${bodyCtx.next().value}
      `;
    } else {
      result = result.concat(#`
        ${name}.prototype.${item} = function
            ${bodyCtx.next().value}
            ${bodyCtx.next().value};
      `);
    }
  }
  return construct.concat(result);
};

class Droid {
  constructor(name, color) {
    this.name = name;
    this.color = color;
  }

  rollWithIt(it) {
    return this.name + " is rolling with " + it;
  }
}
```

```js
function Droid(name, color) {
  this.name = name;
  this.color = color;
}

Droid.prototype.rollWithIt = function(it) {
  return this.name + " is rolling with " + it;
};
```

# Sweet Modules

Now that you've created your sweet macros you probably want to share them! Sweet supports this via ES2015 modules:

```js
'lang sweet.js';
export syntax class = function (ctx) {
  // ...
};
```

```js
import { class } from './es2015-macros';

class Droid {
  constructor(name, color) {
    this.name = name;
    this.color = color;
  }

  rollWithIt(it) {
    return this.name + " is rolling with " + it;
  }
}
```

The `'lang sweet.js'` directive lets Sweet know that a module exports macros, so you need it in any module that has an `export syntax` in it. This directive allows Sweet to not bother doing a lot of unnecessary expansion work in modules that do not export syntax bindings. Eventually, this directive will be used for other things such as defining a base language.

## Modules and Phasing

In addition to importing macros Sweet also lets you import runtime code to use in compiletime code.

As you've probably noticed, we have not seen a way to define a function that a macro's compiletime function can use:

```js
let log = msg => console.log(msg);

syntax m = ctx => {
  log('doing some Sweet things'); // ERROR: unbound variable `log`
  // ...
};
```

We get an unbound variable error in the above example because `m`'s definition runs at a different _phase_ than the surrounding code: namely it runs at compiletime while the surrounding code is invoked at runtime.

Sweet solves this problem by allowing you to import values for a particular phase:

```js
'lang sweet.js';

export function log(msg) {
  console.log(msg);
}
```

```js
import { log } from './log.js' for syntax;

syntax m = ctx => {
  log('doing some Sweet things');
  // ...
};
```

Adding `for syntax` to an import statement lets Sweet know that it needs to load the values being imported into the compiletime environment and make them available for macro definitions.

NOTE: Importing for syntax is currently only supported for Sweet modules (i.e. those that begin with `'lang sweet.js'`). Support for non-Sweet modules is coming soon.


# Sweet Operators

In addition to the macros we've seen so far, Sweet allows you to define custom operators. Custom operators are different from macros in that you can specify the precedence and associativity but you can't match arbitrary syntax; the operator definition is invoked with fully expanded expressions for its operands.

Operators are defined via the `operator` keyword:

```js
operator >>= left 1 = (left, right) => {
  return #`${left}.then(${right})`;
};

fetch('/foo.json') >>= resp => { return resp.json() }
                   >>= json => { return processJson(json) }
```

```js
fetch("/foo.json").then(resp => {
  return resp.json();
}).then(json => {
  return processJson(json);
});
```

The associativity can be either `left` or `right` for binary operators and `prefix` or `postfix` for unary operators. The precedence is a number that specifies how tightly the operator should bind. The builtin operators range from a precedence of 0 to 20 and are defined [here](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence).

The operator definition must return an expression.
