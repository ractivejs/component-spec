# Ractive.js component specification

This repo exists to document the specification for **component files** - HTML files that contain the definition of a Ractive component. If you know what you're looking for, skip to the section for [authors](https://github.com/ractivejs/component-spec/blob/master/authors.md) or [loader implementers](https://github.com/ractivejs/component-spec/blob/master/implementers.md).


## What are components?

Like many front-end libraries and frameworks, Ractive has a concept of *components* - reusable chunks of code that encapsulate templates, data, and behaviour. (Unlike most other tools, Ractive components also allow you to encapsulate styles - more on that later.)

The 'hello world' of components looks like this:

```js
var HelloWorld = Ractive.extend({
  template: '<h1>Hello world!</h1>'
});

var ractive = new HelloWorld({
  el: 'body'
});
```

This code creates a new `HelloWorld` component that inherits from `Ractive` - that is to say, you interact with it just as you would with `Ractive` itself. It has a default `template`, so when you call `new HelloWorld()` without specifying a `template` property, it knows what to do.

With `Ractive.extend()`, you can easily create complex UI components that are very simple to use - even entire applications. Consult the [documentation](http://docs.ractivejs.org/latest/ractive-extend) for more information.

You can even use components inside your templates, by *registering* them:

```js
Ractive.components['hello-world'] = HelloWorld;

var ractive = new Ractive({
  el: 'body',
  template: '<hello-world/>'
});
```


## Single-file components

Do you remember the good old days, when you'd put all your CSS in a `<style>` tag at the top of an `index.html` file, and all your code in a `<script>` block at the bottom, and that was okay? The days before starting a new project meant first creating a Rube Goldberg build process, perfecting your folder structure, wrestling with package managers and having heated Twitter debates about SemVer - long before you got to write any actual code? The days before changing a UI component meant editing some JavaScript in one place, realising you needed to hunt down and edit a template in another place, then changing a CSS file in yet another corner of your project (making sure you strictly adhere to whatever BEM/SMACSS/OOCSS guidelines you've laid down for yourself)?

Ractive remembers, and it's bringing those good times back. It turns out that all the complexity we've introduced in the name of good engineering practices has a real cost in the form of *cognitive burden*. Worse, the 'separation of concerns' mantra has been misunderstood to mean 'separation across syntax boundaries' - in other words, keep your markup, your CSS and your JavaScript separate - when, in fact, properly encapsulated UI components *require* you to consider those languages jointly, not separately. It's just much, *much* more efficient, and easier on your overworked programmer's brain, to (for example) define a CSS class in the same place as you're using it.

A Ractive component file is an HTML file that includes all the markup, data, styles and behaviour necessary to create a component. It can import other Ractive components, and use external libraries (regardless of whether you're using module loaders or Browserify, or whatever). And if your app *does* have a build step, it's straightforward to optimise your components and convert them to pure JavaScript.


## Hello world, again

For this example, we'll use the [ractive-load](https://github.com/ractivejs/ractive-load) plugin:

**hello-world.html**
```html
<h1>Hello {{name}}!</h1>
```

**index.html**
```html
<html>
<head><title>Ractive component demo</title></head>
<body>
  <main><!-- the component will be rendered here --></main>
  <script src='ractive.js'></script>
  <script src='ractive-load.js'></script>
  <script>
    var ractive;

    Ractive.load( 'hello-world.html' ).then( function ( HelloWorld ) {
      ractive = new HelloWorld({
        el: 'main',
        data: { name: 'world' }
      });
    });
  </script>
</body>
</html>
```

**Visit the [page for component authors](https://github.com/ractivejs/component-spec/blob/master/authors.md)** to learn how to write more complex components.


## Loading components

By itself, Ractive doesn't know what to do with a component file - you have to use a *component loader*. This is because the mechanism for turning a component file into an object that can be passed to `Ractive.extend()` differs from one environment to the next.

In the example above, we used `Ractive.load()`, which is the easiest way to use components (it only depends on Ractive itself). That's totally fine, and if you're not building a complex app with many components you shouldn't worry about this too much, but it does have two drawbacks: an HTTP request needs to be made for every component file you end up using, and there's no way to *optimise* the component. So if you're using a build step in your app, you have some other options, listed below.


### Available loaders

* Plain JavaScript
  * [ractive-load](https://github.com/ractivejs/ractive-load)
* RequireJS
  * [rvc](https://github.com/ractivejs/rvc) - works with the [RequireJS optimiser](http://requirejs.org/docs/optimization.html)
* Browserify
  * [ractify](https://github.com/marcello3d/node-ractify)
  * [ractiveify](https://npmjs.org/package/ractiveify) - similar, but with support for compile-to-JS/CSS languages like CoffeeScript
  * [ractive-componentify](https://github.com/blackgate/ractive-componentify) - similar to ractiveify but with support for sourcemaps and partial imports
* Broccoli
  * [broccoli-ractive](https://github.com/ractivejs/broccoli-ractive) - can output AMD, CommonJS or ES6 modules
* Webpack
  * [ractive-component-loader](https://github.com/thomsbg/ractive-component-loader)

## Creating loaders

If your needs aren't met by the existing loaders, you can create your own. **[Consult the page for implementers](https://github.com/ractivejs/component-spec/blob/master/implementers.md) for more information**. If you create a new component loader, let us know via an issue/pull request on this repo, or tell the [mailing list](groups.google.com/forum/#!forum/ractive-js) and [@RactiveJS](http://twitter.com/RactiveJS) on Twitter. Thanks!
