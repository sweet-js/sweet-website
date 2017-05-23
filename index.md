---
layout: default
---

<div style="text-align: center">
  <img src="sweetjs.png" />
</div>

<h1 class="hero">Build your dream language</h1>

Sweet brings the hygienic macros of languages like Scheme and Rust to JavaScript. Macros allow you to sweeten the syntax of JavaScript and craft the language you always wanted.

## Getting started

Install the command line app:

```
$ npm install -g @sweet-js/cli
```

Write your sweet code:

```
syntax hi = function (ctx) {
  return #`console.log('hello, world!')`;
}
hi
```

And compile:

```
$ sjs my_sweet_code.js
console.log('hello, world!')
```

## Next steps

- **Learning**: read the [tutorial](doc/tutorial.html) or check out the [reference](doc/reference.html).
- **Questions**: feel free to open an [issue](https://github.com/sweet-js/sweet-core/issues) on GitHub with any questions you might have. Folks in the [gitter](https://gitter.im/sweet-js/sweet.js) room are also very nice.
- **Contributing**: from documentation, website upkeep, bug fixes, and features we'd love your help! See the [contributing guide](https://github.com/sweet-js/sweet-core/blob/master/CONTRIBUTING.md) for pointers on how to get involved.
