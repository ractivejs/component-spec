# Ractive.js component file specification

## Background

Complexity introduced in the name of good engineering practices has a real cost in the form of *cognitive burden*. In particular, "separation of concerns" has widely been misunderstood to mean "separation of syntax". It's common seeing developers keep HTML, CSS and JS separate when, in fact, keeping them together is much, _much_ more efficient. Especially under tight deadlines, it's easier to define CSS in the same file as the HTML that uses it, rather than having to hunt down the right file to place it.

## Ractive component files

Remember the good old days? When all CSS went in `<style>` elements in `<head>`? When all JS went in `<script>` elements just before `</body>`? When all HTML was written in Mustache inside inert `<script>` elements? When it felt good when everything just worked after a page refresh? Ractive remembers, and it's bringing those good times back with component files.

Ractive component files are simply a self-contained HTML files that define a component and contains all the markup, data, styles and logic it needs. It's also designed with module management in mind, allowing it to declare library and component dependencies. Best of all, component files are written in the same way regardless of the development process involved, build step or none.


```html
<!-- Example component file -->

<!-- Import a component named Foo from the file foo.html. -->
<link rel='ractive' href='foo.html' name='foo'>

<!-- Define the markup for this component. -->
<h1>{{ title }}</h1>

<!-- Use imported foo component -->
<p>This is an imported 'foo' component: <foo/></p>

<!-- Define the styles for this component. -->
<style>
  p { color: red; }
</style>

<!-- Define the behavior for this component. -->
<script>
const $ = require( 'jquery' );

component.exports = {
  onrender: function () {
    $('<p />').text('component rendered').insertAfter($this.find('p'));
  },
  data: {
    title: 'Hello World!'
  }
};
</script>
```

## Terminology

- **Loader** - A tool that converts component files either into component constructors or source code.
- **Loader-specific** - A behavior that is not defined by the spec and is open to custom implementation.
- **Top-level** - A position where an element is not nested inside another element.
- **Transform** - The process of converting source code to another form.

## Specification

### `<link rel="ractive">`

Top-level `<link rel="ractive">` elements must be treated as component import declarations.

`href` is a required attribute that defines the location of the import. Paths that start with `./` or `../` must be resolved relative to the current component file. Resolution of paths that do not start with `./` or `../` is loader-specific.

`name` is an optional attribute that defines the registered name of the component. If present, its value must be used as the imported component's registered name. If absent, the file name of the imported component's file, as defined in `href`, must be used instead.

Components imported via `<link rel="ractive">` must only be registered to the importing component. It must neither be registered specifically for an instance nor globally. This is equivalent to using the `components` initialization option inside a `Ractive.extend()`.

### `<style>`

Top-level `<style>` elements must be treated as component CSS.

If more than one top-level `<style>` is found on the component file, their contents must be concatenated in the order of appearance.

Styles defined by a top-level `<style>` must only be registered to the current component. It must neither be registered specifically for an instance nor globally. This is equivalent to using the `css` initialization option inside a `Ractive.extend`.

### `<script>`

Top-level `<script>` tags must be treated as the component script definition.

There must only be one top-level `<script>` tag in a component file. Should there be more than one defined inside the component file, the loader must throw an error.

During evaluation of the script, `component`, `require` and `Ractive` must be present in the scope of the script.

`require` must be a function that accepts a dependency ID and returns a reference to the dependency that corresponds to that ID. This function must be _synchronous/imperative_. The implementation of the underlying dependency management system, including the determination, timing, resolution, and registration of dependencies, is entirely loader-specific. Dependency IDs may not necessarily represent paths to the dependencies.

`component` must be an empty object prior to the script's evaluation. After evaluation, if an `exports` property is present and is an object, that object's properties must be used as the component's initialization options and must be augmented with the following if present:

  - `css` from the contents of the `<style>` elements.
  - `components` from `<link rel="ractive">` elements.
  - `template` from the template markup.

`Ractive` must be the currently loaded Ractive.

### Template

Top-level elements that are neither `<link rel="ractive">`, `<style>`, nor `<script>` must be treated as part of the component's template.

Top-level template elements must only be registered to the current component. It must neither be registered specifically for an instance nor globally. This is equivalent to using the `template` initialization option inside a `Ractive.extend`.

## Loaders

By itself a component file is just an HTML file. Alone, it's useless to Ractive, to browsers and to tools. In order to utilize component files, a *component loader* is required.

Loaders may be implemented in any way imaginable, be it a runtime library, a cli tool, or a build-step plugin, as long as it is able to do at least one of two things:

- Transform a component file into a **component constructor**. This type of loader is typically used in apps where the environment does all the heavy lifting at runtime. A typical implementation would have the loader, on runtime, recursively load a component file and its dependencies, create constructors, and return them to the requiring code.

    ```js
    // Rough concept of how a constructor is created from a component file.

    const componentScript = '/* the contents of <script> */';
    const options = {
      css: '/* contents of <style> elements */';
      template: '/* extracted markup */';
      components: { /* name-constructor pairs processed from <link rel="ractive"> */ }
    };

    const component = {};
    const require = id => loadedDependencies[id];
    const factory = new Function('component', 'require', 'Ractive', scriptContents);

    // Execute component script
    factory.call(null, component, require, Ractive);

    // Merge gathered options with those from the script
    if ( typeof component.exports === 'object' ) {
      for ( prop in component.exports ) {
        if ( !component.exports.hasOwnProperty( prop ) ) continue;
        options[ prop ] = prop === 'css' ? options.prop + component.exports[ prop ]
          : prop === 'components' ? Object.assign(options.components, component.exports[ prop ])
          : component.exports[ prop ];
      }
    }

    return Ractive.extend(options);
    ```

- Transform a component file into **JavaScript source code**. This type of loader is typically used as part of a build step or cli. It's normally found at the very beginning to convert component files into JS, usually ES, AMD or CJS modules, so that tools down the pipeline can process them like regular JS (i.e. transpile, bundle, minify, etc.).

    This type of loader also has the benefit of being able to incorporate pre-processors and post-processors, allowing the the component file to be written in different languages (i.e Jade, SASS, ES6+), have templates be pre-parsed with `Ractive.parse()`, as well as have the CSS compacted.

    Taking the example component file at the very begining of this document, a loader could convert it to...

    ```js
    // An ES module
    import foo from './foo';
    import $ from 'jquery';

    export default Ractive.extend({
      components: { foo },
      data: { title: 'Hello World!' },
      onrender: function () {
        $('<p />').text('component rendered').insertAfter($this.find('p'));
      },
      template: {"v":4,"t":[{"t":7,"e":"h1","f":[{"t":2,"r":"title"}]}," ",{"t":7,"e":"p","f":["This is an imported 'foo' component: ",{"t":7,"e":"Foo"}]}]},
      css: 'p{color:red;}'
    });
    ```

    ```js
    // An AMD module
    define(function(require, exports, module){
      const Ractive = require('ractive');
      const $ = require('jquery');
      const foo = require('foo.html');

      return Ractive.extend({
        components: { foo },
        data: { title: 'Hello World!' },
        onrender: function () {
          $('<p />').text('component rendered').insertAfter($this.find('p'));
        },
        template: {"v":4,"t":[{"t":7,"e":"h1","f":[{"t":2,"r":"title"}]}," ",{"t":7,"e":"p","f":["This is an imported 'foo' component: ",{"t":7,"e":"Foo"}]}]},
        css: 'p{color:red;}'
      });
    });
    ```

    ```js
    // A CommonJS module
    const Ractive = require('ractive');
    const $ = require('jquery');
    const foo = require('foo.html');

    module.exports = Ractive.extend({
      components: { foo },
      data: { title: 'Hello World!' },
      onrender: function () {
        $('<p />').text('component rendered').insertAfter($this.find('p'));
      },
      template: {"v":4,"t":[{"t":7,"e":"h1","f":[{"t":2,"r":"title"}]}," ",{"t":7,"e":"p","f":["This is an imported 'foo' component: ",{"t":7,"e":"Foo"}]}]},
      css: 'p{color:red;}'
    });
    ```
