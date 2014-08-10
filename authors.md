# Authoring Ractive.js components

*Visit the [introductory README](https://github.com/ractivejs/component-spec/blob/master/README.md) to learn what components are, why you should use them, and how to load them into your app*

This page documents the specification for authoring Ractive.js components, together with examples. It is a **living standard**, which means that *it may change* - though those changes will be kept to a minimum. If you have ideas for improvements, please [open an issue](https://github.com/ractivejs/component-spec/issues/new) on this repo!

Components are designed to be completely agnostic about their environment. Use AMD? Browserify? Webpack? None of the above? Doesn't matter: you can share component files between projects with completely different setups.


## Example component

```html
<!-- optionally, import any sub-components needed by this component -->
<link rel='ractive' href='foo.html' name='foo'>

<!-- follow that with a template -->
<h1>{{title}}</h1>
<p>This is an imported 'foo' component: <foo/></p>

<!-- optionally, include a style block (this will be encapsulated,
     so it doesn't leak into the page) -->
<style>
  p { color: red; }
</style>

<!-- optionally, include a script block defining behaviours etc -->
<script>
  // Dependencies are declared with `require()` - the exact meaning of this
  // depends on the loader being used. E.g. with the AMD loader it means
  // 'load this dependency, then initialise the component', with the Ractive.load()
  // plugin it means 'return `Ractive.lib['myLibrary']` or `window['myLibrary']`'
  var myLibrary = require( 'myLibrary' );

  // `component.exports` should basically be what you'd normally use
  // with `Ractive.extend(...)`, except that the template is already specified
  component.exports = {
    init: function () {
      alert( 'initing component' );
    },

    data: {
      letters: [ 'a', 'b', 'c' ]
    }
  };
</script>
```

We'll go through each of those things in turn. Below, 'top-level' refers to elements that don't live inside other elements.


## `<link>` tags - imported components (optional)

If a template contains one or more top-level `<link rel='ractive'>` tags, they will be treated as import declarations:

```html
<link rel='ractive' href='foo.html' name='foo'>
```

This line says 'load the foo.html file, and register it as the foo component'. Thereafter, we can write `<foo/>` in our template to refer to it. If you omit the `name` attribute, the filename will be used instead - so this line is effectively identical to the previous one:

```html
<link rel='ractive' href='foo.html'>
```

Note that components are **not made globally available** via `Ractive.components` - `<foo/>` only exists inside this component and its children. (Much like passing in a `components` property to `Ractive.extend()`. In fact, exactly like that.) This allows you to easily avoid naming collisions.

It is up to the component loader to determine what the `href` attribute is relative to (unless it begins `./` or `../`, in which case it is relative to the current file). For example in [ractive-load](https://github.com/ractivejs/ractive-load) it's relative to `Ractive.load.baseUrl`, which defaults to the current page. In [rvc](https://github.com/ractivejs/rvc) it's relative to the RequireJS `baseUrl` setting. If in doubt, use relative URLs!


## The markup

In most cases, the template itself is the most important part of a component. Anything that is not a 'special' tag - `<link>`, `<style>` or `<script>` - is treated as part of the template:

```html
<h1>{{title}}</h1>
<p>This is an imported 'foo' component: <foo/></p>
```


## `<style>` tags - encapsulated CSS (optional)

If a template contains one or more top-level `<style>` (or `<style type='text/css'>`) tags, their contents will be added to the `css` property of the object that is passed to `Ractive.extend()`. Unless you specify `noCssTransform` in your `component.exports` object (see below), this CSS will, at render time, be transformed so that it only applies to this component and its children, so you don't need to employ over-engineered namespacing conventions:

```html
<style>
  p { color: red; }
</style>
```


## `<script>` tags and `component.exports` (optional)

If a template contains one or more top-level `<script>` (or `<script type='text/javascript'>`) tags, their contents will be wrapped in a function and executed once. (Implementation detail alert: if the component has dependencies - see below - the dependencies are loaded first.)

Inside the function you have access to two variables aside from `Ractive` itself - `require` and `component`.


### `require`

Some components may have dependencies on external libraries. These dependencies are declared using `require()`:

```html
var myLibrary = require( 'myLibrary' );
```

This is true in all environments, whether or not you're using AMD or whatever. It's the component loader's responsibility to figure out what to do with this. For example, the [ractive-load](https://github.com/ractivejs/ractive-load) plugin would first look for `Ractive.load.modules['myLibrary']`, then would fall back to `window['myLibrary']` in browsers or the native `require('myLibrary')` in node.js. The [rvc](https://github.com/ractivejs/rvc) loader, on the other hand, basically just uses the RequireJS implementation of `require()`.

*GOTCHA 1: For asynchronous loaders like rvc to work, the dependencies must be identified ahead of time, so that they can be loaded before the script executes. Rather than bundling [esprima](http://esprima.org/) and parsing the code to identify those `require()` calls, it's done with regex. For that reason, if something 'looks' like a `require()` call - such as a comment or a string - it may cause problems. See [rcu#4](https://github.com/ractivejs/rcu/issues/4).*

*GOTCHA 2: Dependencies on Ractive plugins can be declared with e.g. `require('ractive-events-tap')`. This will currently throw an error with ractive-load. See [ractive-load#15](https://github.com/ractivejs/ractive-load/issues/15).*


### `component`

Much in the same way that node modules have `module.exports`, Ractive components have `component.exports`. `component` is an empty object (`{}`) - if, once your JavaScript executes, it has an `exports` property, that object will be augmented with `template`, `css` and `components` properties, as appropriate, and passed to `Ractive.extend()`.

```js
component.exports = {
  init: function () {
    alert( 'initing component' );
  },

  data: {
    letters: [ 'a', 'b', 'c' ]
  }
};
```


