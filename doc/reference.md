---
layout: default
toc: true
use_anchors: true
---

# Introduction

This is the reference document for Sweet.js. For a gentle explanation of Sweet's concepts see the [tutorial](tutorial.html).

# Command Line API

- `--out-file <file>`: write result to file
- `--out-dir <dir>`: write result to directory
- `--no-babel`: do not use babel backend

# Binding Forms

## syntax

```
syntax <name> = <init>
```

Bind `<name>` to the result of evaluating `<init>` in the compiletime environment. Scoping follows `let` (i.e. block scoped with a temporal dead zone).

If the result of evaluating `<init>` is a function, then the result is a *syntax transformer*.

## syntaxrec

```
syntaxrec <name> = <init>
```

Identical to the `syntax` form except that `<name>` is also bound inside of `<init>`. This enables recursive macro definitions.

# Syntax Transformer

```
transformer : (TransformerContext) -> List(Syntax)
```

A syntax transformer is a function bound to a compile-time name. A syntax transformer is invoked with a *transformer context* that provides access to the syntax at the call-site and returns a list of syntax objects.

### Transformer Context

A transformer context is an iterable object that provides access to syntax at the call-site of a syntax transformer.

```
TransformerContext = {
  name: () -> Syntax
  next: () -> {
    done: boolean,
    value: Syntax
  }
  expand: (string) -> {
    done: boolean,
    value: Syntax
  }
  mark: () -> Marker
  reset: (Marker?) -> undefined
}
```

Each call to `next` returns the syntax object following the transformer call.

A call to `expand` initiates expansion at the current state of the iterator and matches the specified grammar production. Matching is "greedy" so `expand('expr')` with the syntax `1 + 2` matches the entire binary expression rather than just `1`. The following productions are accepted by `expand`:

- `Statement` with alias `stmt`
- `AssignmentExpression` with alias `expr`
- `Expression`
- `BlockStatement`
- `WhileStatement`
- `IfStatement`
- `ForStatement`
- `SwitchStatement`
- `BreakStatement`
- `ContinueStatement`
- `DebuggerStatement`
- `WithStatement`
- `TryStatement`
- `ThrowStatement`
- `ClassDeclaration`
- `FunctionDeclaration`
- `LabeledStatement`
- `VariableDeclarationStatement`
- `ReturnStatement`
- `ExpressionStatement`
- `YieldExpression`
- `ClassExpression`
- `ArrowExpression`
- `NewExpression`
- `ThisExpression`
- `FunctionExpression`
- `IdentifierExpression`
- `LiteralNumericExpression`
- `LiteralInfinityExpression`
- `LiteralStringExpression`
- `TemplateExpression`
- `LiteralBooleanExpression`
- `LiteralNullExpression`
- `LiteralRegExpExpression`
- `ObjectExpression`
- `ArrayExpression`
- `UnaryExpression`
- `UpdateExpression`
- `BinaryExpression`
- `StaticMemberExpression`
- `ComputedMemberExpression`
- `AssignmentExpression`
- `CompoundAssignmentExpression`
- `ConditionalExpression`

The `name()` method returns the syntax object of the macro name at the macro invocation site. This is useful because it allows a macro transformer to get access to the lexical context at the invocation site.

A call to `mark` returns a pointer to the current state of the iterator.

Calling `reset` with no arguments returns the context to its initial state, while passing a Marker instance returns the context to the state pointed to by the marker.

```js
syntax m = function (ctx) {
  ctx.expand('expr');
  ctx.reset();

  const a = ctx.next().value;
  ctx.next();

  const marker = ctx.mark();
  ctx.expand('expr');
  ctx.reset(marker);

  const b = ctx.next().value;
  return #`${a} + ${b} + 24`; // 30 + 42 + 24
};
m 30 + 42 + 66
```



# Syntax Objects

Syntax objects represent the syntax from the source program. While syntax objects have internal structure, that structure is intentionally undocumented and subject to change (in fact, major change to syntax objects is planned for the next major version).

Introspection and manipulation functions are provided in the helpers library documented in the following section.


# Helpers

A library of helper functions for introspecting syntax objects is provided at `'@sweet-js/helpers'`. To use inside a macro definition, import `for syntax`.

```js
import { isStringLiteral } from '@sweet-js/helpers' for syntax;

syntax m = ctx => {
  return isStringLiteral(ctx.next().value) ? #`'a string'` : #`'not a string'`;
};
m 'foo'
```

```
'a string'
```

The helper library contains the following kinds of functions:

- `is*` functions that test the type of a syntax object (e.g. `isIdentifier` for identifiers and `isStringLiteral` for strings)
- `from*` functions that create new syntax objects from primitive data
- an `unwrap` function that returns the primitive representation of a syntax object

## unwrap

```js
unwrap(stx: any): {
  value?: string | number | List<Syntax>
}
```

In the case of a flat syntax object (i.e. not delimiters), `unwrap` returns an object with a single `value` property that holds the primitive representation of that piece of syntax (a string for string literals, keywords, and identifiers or a number for numeric literals).

For syntax objects that represent delimiters, `unwrap` returns an object who's `value` property is a list of the syntax objects inside the delimiter.

For all other inputs `unwrap` returns the empty object.

```js
import { unwrap } from '@sweet-js/helpers' for syntax;

syntax m = ctx => {
  let id = ctx.next().value;
  let delim = ctx.next().value;

  unwrap(id).value === 'foo';  // true
  let num = unwrap(delim).value.get(1);
  unwrap(num).value === 1;     // true
  // ...
};
m foo (1)
```

## fromIdentifier

```
fromIdentifier(other: Syntax, s: string): Syntax
```

Create a new identifier syntax object named `s` using the lexical context from `other`.

```js
import { fromIdentifier } from '@sweet-js/helpers' for syntax;

syntax m = ctx => {
  let dummy = #`dummy`.get(0);

  return #`${fromIdentifier(dummy, 'bar')}`;
};
m foo
```

```
bar
```

Be careful which syntax object you use to create a new syntax object via `fromIdentifier` and related functions since the new object will share the original's lexical context. In most cases you will want to create a "dummy" syntax object inside a macro definition and then use that as a base to create new objects. By using a dummy syntax object you are using the scope of the macro definition; usually the macro definition scope is what you want.

You may be tempted to reuse the syntax object provided by `ctx.name()` but resist that feeling! The `ctx.name()` syntax object comes from the macro call-site and so any syntax objects created from it will carry the lexical context of the call-site. Sometimes this is what you want, but most of the time this breaks hygiene!

## fromNumber

```js
fromNumber(other: Syntax, n: number): Syntax
```

Create a new numeric literal syntax object with the value `n` using the lexical context from `other`.

```js
import { fromNumber } from '@sweet-js/helpers' for syntax;

syntax m = ctx => {
  let dummy = #`dummy`.get(0);

  return #`${fromNumber(dummy, 1)}`;
};
m
```

```js
1
```

## fromStringLiteral

```
fromStringLiteral(other: Syntax, s: string): Syntax
```

Create a new string literal syntax object with the value `s` using the lexical context from `other`.

```js
import { unwrap, fromStringLiteral } from '@sweet-js/helpers' for syntax;

syntax to_str = ctx => {
  let dummy = #`dummy`.get(0);
  let arg = unwrap(ctx.next().value).value;
  return #`${fromStringLiteral(dummy, arg)}`;
}
to_str foo
```

```js
'foo'
```

## fromPunctuator

```js
fromPunctuator(other: Syntax, punc: string): Syntax
```

Creates a punctuator (e.g. `+`, `==`, etc.) from its string representation `punc` using the lexical context from `other`.

```js
import { fromPunctuator } from '@sweet-js/helpers' for syntax;

syntax m = ctx => {
  let dummy = #`dummy`.get(0);
  return #`1 ${fromPunctuator(dummy, '+')} 1`;
};
m
```

```js
1 + 1
```

## fromKeyword

```js
fromKeyword(other: Syntax, kwd: string): Syntax
```

Create a new keyword syntax object with the value `kwd` using the lexical context from `other`.

```js
syntax m = ctx => {
  let dummy = #`dummy`.get(0);
  return #`${fromKeyword(dummy, 'let')} x = 1`;
};
m
```

```js
let x = 1
```

## fromBraces

```js
fromBraces(other: Syntax, inner: List<Syntax>): Syntax
```

Creates a curly brace delimiter with inner syntax objects `inner` using the lexical context from `other`.

```js
import { fromBraces } from '@sweet-js/helpers' for syntax;

syntax m = ctx => {
  let dummy = #`dummy`.get(0);
  let block = #`let x = 1;`;
  return #`${fromBraces(dummy, block)}`;
};
m
```

```js
{
  let x = 1;
}
```

## fromBrackets

```js
fromBrackets(other: Syntax, inner: List<Syntax>): Syntax
```

Creates a bracket delimiter with inner syntax objects `inner` using the lexical context from `other`.

```js
syntax m = ctx => {
  let dummy = #`dummy`.get(0);
  let elements = #`1, 2, 3`;
  return #`${fromBrackets(dummy, elements)}`;
};
m
```

```js
[1, 2, 3]
```

## fromParens

```js
fromParens(other: Syntax, inner: List<Syntax>): Syntax
```

creates a paren delimiter with inner syntax objects `inner` using the lexical context from `other`.

```js
import { fromParens } from '@sweet-js/helpers' for syntax;

syntax m = ctx => {
  let dummy = #`dummy`.get(0);
  let expr = #`5 * 5`;
  return #`1 + ${fromParens(dummy, expr)}`;
};
m
```

```js
1 + (5 * 5)
```

## isIdentifier

``js
isIdentifier(s: Syntax): boolean
```

Returns true if the syntax object is an identifier.

## isNumericLiteral

```js
isNumericLiteral(s: Syntax): boolean
```

Returns true if the syntax object is a numeric literal.

## isStringLiteral

```js
isStringLiteral(s: Syntax): boolean
```

Returns true if the syntax object is a string literal.

## isKeyword

```js
isKeyword(s: Syntax): boolean
```

Returns true if the syntax object is a keyword.

## isPunctuator

```js
isPunctuator(s: Syntax): boolean
```

Returns true if the syntax object is a puncuator.

## isTemplate

```js
isTemplate(s: Syntax): boolean
```

Returns true if the syntax object is a template literal.

## isSyntaxTemplate

```js
isSyntaxTemplate(s: Syntax): boolean
```

Returns true if the syntax object is a syntax template literal.

## isParens

```js
isParens(s: Syntax): boolean
```

Returns true if the syntax object is a parenthesis delimiter (e.g. `( ... )`).

## isBrackets

```js
isBrackets(s: Syntax): boolean
```

Returns true if the syntax object is a bracket delimiter (e.g. `[ ... ]`).

## isBraces

```js
isBraces(s: Syntax): boolean
```

Returns true if the syntax object is a braces delimiter (e.g. `{ ... }`).


# Syntax Templates

Syntax templates construct a list of syntax objects from a literal representation using backtick. They are similar to ES2015 templates but with the special sweet.js specific `#` template tag.

Syntax templates support interpolations just like normal templates via `${...}`:

```js
syntax m = function (ctx) {
  return #`${ctx.next().value} + 24`;
};
m 42
```

The expressions inside an interpolation must evaluate to a syntax object, an array, a list, or an transformer context.
