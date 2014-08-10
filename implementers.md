# Implementing Ractive.js component loaders

*Visit the [introductory README](https://github.com/ractivejs/component-spec/blob/master/README.md) to learn what components are, why you should use them, and how to load them into your app*

This page explains how to create new component loaders to integrate Ractive with different project setups. Once you've absorbed the information herein, take a look at the [existing component loaders](https://github.com/ractivejs/component-spec/blob/master/README.md#available-loaders) to see examples.

If you haven't already, you should familiarise yourself with the [page for component authors](https://github.com/ractivejs/component-spec/blob/master/authors.md).


## The component loader's job

A loader is something that takes an HTML file containing a component definition, and either

* turns it into a **constructor** that inherits from `Ractive`, or
* transforms it into **JavaScript source code** as part of a build process

For example [ractive-load](https://github.com/ractivejs/ractive-load) turns it into a constructor right there and then, because there's no build step happening. The [rvc](https://github.com/ractivejs/rvc) RequireJS plugin does *both* - during development, it turns HTML into a constructor, but in production (if you use the [RequireJS optimiser](http://requirejs.org/docs/optimization.html)) it creates the source code for an AMD module.

The second bullet point is deliberately vague - 'source code' could mean a variety of things. In [rvc](https://github.com/ractivejs/rvc) it means an AMD module, but in [broccoli-ractive](https://github.com/ractivejs/broccoli-ractive) it could mean AMD, a node.js module, or even an ES6 module. [node-ractify](https://github.com/marcello3d/node-ractify) generates a node.js module, but the module exports the `options` object to pass to `Ractive.extend(options)`, rather than the result of that call. Different things make sense in different environments.

What they all have in common is the end result - a component file should render the same way in a browser (or in node.js) no matter what route it took.


## rcu - Ractive component utils

The implementation details are entirely up to you, but it's highly recommended that you use [rcu]((https://github.com/ractivejs/rcu) to create your loader, as it has a handful of utilities that make doing so much easier.

In particular, it has the `rcu.parse()` method, which transforms the HTML file into an intermediate representation that's much easier to work with. If we ran the [example component from the authors page](https://github.com/ractivejs/component-spec/blob/master/authors.md#example-component) through `rcu.parse()`, this is what we'd get:

```js
{
  imports: [{ href: 'foo.html', name: 'foo' }],
  template: {v:1,t:[ /* template goes here... */] },
  css: 'p { color: red; }',
  script: /* contents of the script tag go here */,
  modules: [ 'myLibrary' ]
}
```

From there, it's fairly easy to create JavaScript (e.g. an AMD module), and you can turn it into a constructor (if you're loading components in the browser, as opposed to transforming them during a build step) with `rcu.make()`.

Consult the [rcu documentation]((https://github.com/ractivejs/rcu) for more information.


## Terminology

For the rest of this document, we'll use the following terminology as shorthand:

* **top-level** means an element that is not nested inside another element
* **making** a component means creating a constructor (with `Ractive.extend()`) that inherits from `Ractive`
* **transforming** a component means turning the HTML file into JavaScript of some form, typically a module (whether AMD, node.js or ES6)
* **`options`** refers to the object that is passed to `Ractive.extend()`


## `<link>` tags

Any top-level `<link rel='ractive'>` tags should be imported and added to the `components` property of `options`, using the `name` as key:

```js
options.components = {
  foo: ImportedFooComponent // how you name things internally is up to you!
}
```

When *making* a component, that means those imports must be loaded first (`rcu.make()` takes care of this). When *transforming* into a JavaScript module, it means using whatever import declaration that module system provides - note that imports must *themselves* be JavaScript modules.


## The markup

This one is easy - just add the `template` property of the object returned from `rcu.parse()` to `options`:

```js
parsed = rcu.parse( template );
options.template = parsed.template;
```


## `<style> tags

Any top-level `<style>` (or `<style type='text/css'>`) tags should be concatenated and added to `options` as the `css` property.

```js
parsed = rcu.parse( template );
options.css = parsed.css;
```

Optionally, you can lint and minify CSS during transformation. (Future versions of this spec may accommodate things like autoprefixing and converting other languages such as SCSS.) You don't need to worry about style encapsulation - that happens at render time.


## `<script>` tags

Any top-level `<script>` (or `<script type='text/javascript'>`) tags should be concatenated and executed once, to generate `component.exports` ([see here](https://github.com/ractivejs/component-spec/blob/master/authors.md#component)). When making components, `rcu.make()` handles this for you.

When transforming components, you need to ensure that when the code executes, it has access to the variables `component`, `require` and `Ractive`, where `component` is an empty object (`{}`), `require` is a function that returns an external dependency, and `Ractive` is, well, Ractive.

Typically the relevant part of a transformed module might look something like this:

```js
component = {};
require = function ( id ) {
  // this is left as an exercise to the reader... in some situations,
  // e.g. RequireJS or Browserify, we simply use the existing `require`
  return dependencies[ id ];
};

// We wrap the code in an IIFE so component authors
// can't bork anything up
(function ( component, require, Ractive ) {
  /* content of script tags goes here */
}( component, require, Ractive ));

if ( typeof component.exports === 'object' ) {
  for ( prop in component.exports ) {
    if ( component.exports.hasOwnProperty( prop ) ) {
      options[ prop ] = component.exports[ prop ];
    }
  }
}
```

If you're transforming a component into an asynchronous module, such as AMD or ES6, then you'll need to load any external dependencies (i.e. anything loaded with `require()` before the code executes. `rcu.parse()` identifies those dependencies and makes them available as the `modules` property, so you can add them as import declarations to your module.
