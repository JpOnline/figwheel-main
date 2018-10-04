---
title: NPM Modules
layout: docs
category: docs
order: 10
---

# NPM Modules

<div class="lead-in"> There is a tremendous amount of code that is now
stored in the <a href="https://www.npmjs.com/">NPM</a> package
repository. This guide will show you how to include and consume NPM
packages in your ClojureScript codebase.</div>

This guide is modeled after the
[ClojureScript Webpack Guide](https://clojurescript.org/guides/webpack). If
you prefer a more concise guide feel free to head over there now. This
guide will also demonstrate how to use the
[`:npm` config option](../config-options#npm)

## What? 

[NPM][npm] is a package repository for the JavaScript
ecosystem. Almost all available JavaScript libraries are packaged,
stored, and retrieved via NPM.

We want to use these libraries inside our ClojureScript codebase but
there is some natural friction because ClojureScript embraced the
Google Closure Compiler and its method of declaring libraries which is
quite different than NPM's.

We could get into a debate about why ClojureScript designers decided
to embrace the less popular ecosystem, but that is largely academic at
this point. I will say that the advantages of effortless interactive
development via hot-reloading and the amazing capabilities of the
Google Closure Compiler's advanced mode are direct benefits of using
the GCC's (Google Closure Compiler's) method of defining modules via
simple JavaScript object literals. In other words, without the GCC
there would most likely not be a Figwheel.

Nevertheless experiencing friction importing libraries from the
dominant JavaScript ecosystem is a vary unfortunate trade-off.

However, with recent changes in the ClojureScript compiler (along with
changes in Figwheel) it is now becoming much more straight forward to
include NPM modules in your codebase.

## Importing NPM libraries into your project

We are going to assume you are starting from the
[base `hello-world.core` example][base-example-gist] which is used
throughout this documentation.

It is assumed that you have installed Node along with `npm` in your
development environment.

I'm going to use `npm` for this example but if you prefer `yarn` go
ahead and use that. It doesn't really matter for this.

There are four steps that we are going to do to add some libraries to
our `hello-world.core` project.

1. add the needed libraries as NPM dependencies
2. configure Webpack to bundle the needed dependencies into a single JS file
3. create an `index.js` file to export the needed libraries to the global context
4. configure our ClojureScript build to use the bundle generated by Webpack

## Add the needed libraries

We will need to initialize `npm` for our `hello-world` project.

Make sure you are in the root directory of the project and execute:

```
$ npm init -y
```

Let's say we want to use the `react` and `react-dom` libraries in our
application.

We'll use `npm` to add them in the usual manner:

```sh
$ npm add react react-dom
```

This should download and install the needed libraries. Now there will
be a `package.json` file and a `node_modules` directory in your
project.

## Configure Webpack to bundle the dependencies

First we will install `webpack` and `webpack-cli`:

```sh
$ npm add webpack webpack-cli
```

Next we will create a basic Webpack configuration file for our
bundle. In `webpack.config.js` place the following code:

```javascript
module.exports = {
  entry: './src/js/index.js',
  output: {
    filename: 'index.bundle.js'
  }
}
```

If you aren't familiar with Webpack the above config file is stating
that when we run `webpack` it will bundle all the resources needed in
`src/js/index.js` into a single `dist/index.bundle.js` file.

## Create the `index.js` file

The `src/js/index.js` file is where we will require the libraries that
we need and export them to the global context so that we can access
them from ClojureScript.

Continuing with the example let's export the `react` and `react-dom`
NPM libraries to the global `window` context of the browser.

Place the following code in the `src/js/index.js` file:

```javascript
import React from 'react';
import ReactDOM from 'react-dom';
window.React = React;
window.ReactDOM = ReactDOM;
```

The above code imports the NPM libraries we need and makes them
available in the global context on `window`.

> Keep in mind that the `import` statements are dependent on how the
> individual JavaScript library exports its functionality.  You will
> need to refer to the documentation for the given library to
> understand how to import it properly.

We can now use Webpack to bundle all the NPM libraries needed for the
above `index.js` file into a single file. We will use the `npx`
command to run Webpack:

```sh
npx webpack
```

If all goes well you will see that a `dist/index.bundle.js` file was
created.

> Important: When you are managing your own NPM dependencies, as we
> are here, and you have a `node_modules` directory in the root of
> your project directory, you will need to set `:npm-deps` to `false`
> in your build config file (`dev.cljs.edn` in this
> example). Otherwise, the ClojureScript compiler will scan it and
> make other decisions based on its presence, and this can lead to
> confusing errors.

Let's do this now and change the `dev.cljs.edn` file and set
`:npm-deps` to `false`:

```clojure
{:main hello-world.core
 :npm-deps false}
```

## Configure our ClojureScript build to use libraries in the bundle

Everything we have done up until this point is very similar to what we
would normally do if we were using NPM and Webpack for a simple
JavaScript project.

Now we need to let the ClojureScript compiler know that we are using
some globally exported libraries in `dist/index.bundle.js`.

In the `dev.cljs.edn` file we'll add the following `:foreign-libs`
entry:

```clojure
{:main hell-world.core
 :npm-deps false
 :infer-externs true
 :foreign-libs [{:file "dist/index.bundle.js"
                 :provides ["react" "react-dom"]
                 :global-exports {react React
                                  react-dom ReactDOM}}]}
```

Understanding the above configuration is important, so I'm going to
explain each part.

The
[`:foreign-libs` compiler option](https://clojurescript.org/reference/compiler-options#foreign-libs)
helps you map *foreign* libraries to libraries that you can require
and use from inside your ClojureScript source code. Foreign libraries
are dependencies that don't follow the Google Closure way of defining
a library.

Since we want to be able to require and use `react` NPM library from
our ClojureScript code. We have to provide the compiler and the
Closure bootstrapping/require process where our custom file falls in
the dependency tree.

So `:foreign-libs` is a list of data structures that provide this
information for individual JavaScript files.

Let's look at our entry in `:foreign-libs` and see how this plays out.

```clojure
{:file "dist/index.bundle.js"
 :provides ["react" "react-dom"]
 :global-exports {react React
                  react-dom ReactDOM}}
```

Well it's pretty obvious that we need a `:file` key as we are adding
meta information to a specific JavaScript file. The ClojureScript
compiler does not need to this file to be on the classpath and it will
copy the file into the `:output-dir` at `dist/index.bundle.js`.

Next let's look at the `:provides` key. The `:provides` key tells the
compiler that this file *provides* the listed libraries. In this case
`:provides` key tells the compiler that when someone requires `react`
or `react-dom` then the `"dist/index.bundle.js` must be supplied.

The names that you list in the `:provides` key are the same names that
you will use in your `:require` expressions in your namespace
declarations.

```clojure
(ns hello-world.core
  (:require [react]
            [react-dom]))
```

Actually, using the `:file` and `:provides` keys is enough to start
using Reactjs from our source code, but we will only be able to
reference it via JavaScript not via the `react` namespace.

For example this works:

```clojure
(ns hello-world.core
  (:require [react]
            [react-dom]))
			
(js/console.log js/React)			
```

This works because the compiler now knows to include the
`dist/index.bundle.js` file when `react` is required, and this bundle
file exports React to `window.React` and thus we can reference it. But
the following will not work:

```clojure
(ns hello-world.core
  (:require [react]
            [react-dom]))
			
(js/console.log react)			
```

Referring to `react` directly in the source code won't work if we only
use the `:provides` key. This won't work because there is no way for
the compiler to know what should be bound to `react`.

This is where the `:global-exports` key comes in, it maps the name you
specify in the require statement to a specific global resource in your
JavaScript environment.

Looking at the `:global-exports` key and our `src/js/index.js` file
together, let's see how they map to one another.

```clojure
:global-exports {react React
                 react-dom ReactDOM}
```

```javascript
import React from 'react';
import ReactDOM from 'react-dom';
window.React = React;
window.ReactDOM = ReactDOM;
```

The `React` value in the global exports map refers specifically to the
presence of `window.React` in the `index.js` file.

To further understand what is happening here let's look at how this example
compiles when we have specified `:global-exports`.

This `src/hello_world/core.cljs` file:

```clojure
(ns hello-world.core
  (:require [react]
            [react-dom]))
			
(js/console.log react)			
```

compiles to the following JavaScript when using `:optimizations` level
`:none`:

```javascript
goog.provide('hello_world.core');
goog.require('cljs.core');
goog.require('react');
goog.require('react_dom');
hello_world.core.global$module$react = goog.global["React"];   // <--
hello_world.core.global$module$react_dom = goog.global["ReactDOM"]; // <--
console.log(hello_world.core.global$module$react);
```

Looking at the above compiled JavaScript you can see the two lines
that grab `React` and `ReactDOM` from the global context
(`goog.global` is `window` in this case) and binds them to a "local"
name. You can then see how the local
`hello_world.core.global$module$react` is used in the `console.log`
statement.

If you don't use `:global-exports` and only use `:provides` this name
binding doesn't happen and thus you are required to refer to your
libraries via the `js/` prefix.

Now you should have a good idea of how to specify `:foreign-libs`
entries to make NPM libraries available for ClojureScript consumption.

Next let's look at how Figwheel can automate the generation of these
`:foreign-libs` entries for you.

> You can learn more about importing JavaScript [via `:foreign-libs`
> here](https://clojurescript.org/reference/dependencies)

## Automated `:foreign-libs` entries for NPM

When you look at the shape of a `:foreign-libs` entry and the format
of our example `src/js/index.js` file you may wonder if we can just
automate this.

If you want, Figwheel will try to read your `index.js` file and
generate a `:foreign-libs` entry for you. This is intended to help cut
out some of the pain of consuming NPM libraries.

The [`:npm` figwheel option](../config-options#npm) will do this for
you. Here is an example of the configuration in our `dev.cljs.edn`
before and after using the `:npm` key.

Before:

```clojure
{:main hell-world.core
 :npm-deps false
 :infer-externs true
 :foreign-libs [{:file "dist/index.bundle.js"
                 :provides ["react" "react-dom"]
                 :global-exports {react React
                                  react-dom ReactDOM}}]}
```

After:

```clojure
^{:npm {:bundles {"dist/index.bundle.js" "src/js/index.js"}}}
{:main hell-world.core}
```

This may not seem like much of a reduction but when you consider the
fact that once you define your bundle in the `:npm` key you no longer
have to change the `dev.cljs.edn` file when you add a new NPM library
in your `index.js` file.

You will still need to re-run webpack when you change the `index.js`
file but you won't have to add an entry to `:global-exports` for each
new library.

When you supply an `:npm > :bundles` configuration Figwheel will add
both `:infer-externs true` and `:npm-deps false` to your compile
options as well, but only if they are not already defined.

**How `:npm > :bundles` reads your `index.js` file to create a `:foreign-libs` entry**

If you supply an `index.js` file with lines in it that start with
`window.` as is the case in the example above with the lines that
start `window.React` and `windows.ReactDOM`. The Figwheel will take
the `React` and `ReactDOM` names and
[**kebab-case**](http://wiki.c2.com/?KebabCase) them into `react` and
`react-dom`. It will then use these identifiers in the
`:global-exports` like so:

```clojure
:global-exports {react React
                 react-dom ReactDOM}
```

It will then take the keys of the `:global-exports` map and turn them
into a `:provides` entry like this:

```clojure
:provides ["react" "react-dom"]
:global-exports {react React
                 react-dom ReactDOM}
```

Then Figwheel will get the `:file` from the name of the Webpack bundle
in the `:npm > :bundles` declaration to create the complete
`:foreign-libs` entry before sending it to the compiler.

```clojure
:foreign-libs [{:file "dist/index.bundle.js"
                :provides ["react" "react-dom"]
                :global-exports {react React
                                 react-dom ReactDOM}}]
```

Sometimes the kebab-case can fail to generate the library name that
you need, you can precisely control the library name by using
`window["react"] = React;` form to assign the library to the global
context instead. For instance you could use this to override some code
that is relying on `cljsjs.react` via `window["cljsjs.react"] =
React;` and now when something requires `cljsjs.react` they will get
your NPM version of Reactjs.

The `:bundles` functionality can also read `goog.global.ReactDOM =
ReactDOM;` and `goog.global["cljs.react-dom"] = ReactDOM;` in your
`index.js` file.

If you prefer to use the `goog.global` form you should protect against
it not being there under advanced compile. Like this:

```javascript
;; ensure that goog global exists under advanced compile
var goog = goog || {global = window}; 
import React from 'react';
import ReactDOM from 'react-dom';
goog.global.React = React;
goog.global.ReactDOM = ReactDOM;
```









[base-example-gist]: https://gist.github.com/bhauman/a5251390d1b8db09f43c385fb505727d
[npm]: https://www.npmjs.com/

