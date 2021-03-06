### Source maps

The Webpack [SourceMapDevToolPlugin](http://webpack.github.io/docs/list-of-plugins.html#sourcemapdevtoolplugin)
creates [source maps](https://github.com/ryanseddon/source-map/wiki/Source-maps:-languages,-tools-and-other-info)
which allows a developer to view / debug developer-friendly source code instead
of the optimized, mangled, and minified JS bundle of a frontend web app. Source
maps should be enabled for both development and production.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Source Maps Primer](#source-maps-primer)
- [Integrating Source Maps](#integrating-source-maps)
  - [Public Apps](#public-apps)
  - [Private Apps](#private-apps)
- [Basic Example](#basic-example)
- [Source Map Example](#source-map-example)
- [Advantages](#advantages)
- [Disadvantages](#disadvantages)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

##### Source Maps Primer

Modern JavaScript on the frontend practically _guarantees_ that the original
source code developers write is different from the code distributed and
executed on web browsers. These days, code is likely to be transformed by things
like:

* Babel transforms of ESnext/JSX code to ES5 code.
* Developer-friendly full code to minified gibberish for smaller download size
  and some protection of application IP.

The skew between what developers write and what a browser executes makes
debugging JavaScript applications quite tedious and complicated. Fortunately,
modern browsers implement **source maps** whereby minified code can be
mapped to real developer sources before execution to give developers a much
better debugging experience.

If you are not familiar with how source maps work, please take a moment to
review a couple introductory articles:

* [How do source maps work?](http://www.mattzeunert.com/2016/02/14/how-do-source-maps-work.html)
* [Introduction to JavaScript Source Maps](http://www.html5rocks.com/en/tutorials/developertools/sourcemaps/)
* [List of Source Maps Resources](https://github.com/ryanseddon/source-map/wiki/Source-maps:-languages,-tools-and-other-info)
* [Debuggable JavaScript in production with Source Maps](https://blog.sentry.io/2015/10/29/debuggable-javascript-with-source-maps.html)

The brief summary of how source maps work is as follows:

* During bundling, an application is configured to output a source mapping file
  corresponding to the application bundle.
* The real application bundle is downloaded and parsed by a browser.
* If the browser encounters a special control comment like:
  `//# sourceMappingURL=PATH/TO/bundle.js.map` **and** the developer console
  is open, then the source map file is downloaded from the URL in the control
  comment.
* Once downloaded, the debugger maps execution of the real minified code to the
  attached source maps to give the appearance / simulation of running the raw
  original code.

This functionality is useful in both development and production. In
**development** an application may not be minified, but it is bundled into a
single / couple of application chunks. Source maps can give developers a
debugging experience corresponding to the _actual raw source files_ at issue.
[SurviveJS](http://survivejs.com/webpack/developing-with-webpack/enabling-sourcemaps/)
provides a very good tutorial in utilizing source maps for easier development
and we won't touch on the topic further, as we're focused on the production
story for this current document.

In **production** source maps are the lifeline that allows application
developers to not have to make sense of gobs of minified variables and code.
The source maps are not just a convenience, they're practically essential for
being able to debug a production app for any sizeable bundle in the modern
web. The rest of this page will examine enabling your critical "out" for bugs
you find in production.

**Side Note: source maps for end users**: There is sometimes a concern that for
most end users of an application that they shouldn't have the penalty of either
downloading or 404'ing source maps in the non-development use of a production
application. Fortunately, source map integration in browsers works such that
a browser will not even attempt to download source maps unless the development
console of the browser is open, using that as a good heuristic of when to add
the enhanced debugging capabilities of source maps. Thus, for normal end users
there is absolutely no penalty for having a source map integration strategy
in production applications.

##### Integrating Source Maps

With the above background in mind, the big issue is where and how to serve
source maps when focusing on production.

###### Public Apps

For *public applications*, this is fairly easy -- just serve the source
map alongside the application code. There's no worry about the real source
code getting out to the public, so for an application like `app1.js` and a
source map file of `app1.js.map`, a simple `sourceMappingURL` comment like:

```js
//# sourceMappingURL=app1.js.map
```

is sufficient to make the tight minified bundle execute for most users while
still having source maps available for any users that have a development
console open.

###### Private Apps

For *private applications*, it is a little more complicated. Essentially, the
source maps need to be served, but _just_ to trusted members of a private team.
And the source mapping control comment needs to integrate this choice.

There are several reasons why an application may need private source code:

* To protect intellectual property in the application code.
* To obfuscate application code to slightly impede hackers and miscreants.
* To prevent the world from seeing your atrocious last-minute hacks. `;)`

The usual options for integrating privately available source maps include:

**Option 1: Locally serve the maps**

The easiest option to wire together is to point the source mapping comment to
localhost, either by direct reference or an alias that developers have to
enter in `/etc/hosts` or whatnot. For example:

```js
//# sourceMappingURL=http://127.0.0.1:3000/PATH/TO/app1.js.map
//# sourceMappingURL=http://localhost:3000/PATH/TO/app1.js.map
//# sourceMappingURL=http://localhost-alias.com:3000/PATH/TO/app1.js.map
```

The application build process should then go through these steps:

1. Ensure that the version number or git hash uniquely identifying the specific
   point in the application code is available in the production app.
   (E.g., `window.__VERSION = "2.0.3"`).
2. On building the application, potentially store the source mapping files for
   later serving. This is not strictly necessary as a checked out project may
   rebuild the source map files, but there is some risk of version skew in the
   build tools affecting the resulting bundle / source. The best practice is
   to *store the maps that built the code in production*.
3. Provide a simple development static server to serve the source maps at a
   given port (in our example port `3000`) so that a project checkout can
   provide the maps via a server.
4. Ensure that the build process writes a correct `sourceMappingURL` comment
   that corresponds to the correct locally hosted map server path.

Then, when a developer needs to debug production code, they:

1. Identify the version or git hash at issue.
2. Download the project (git, npm, deploy artifact) at the proper version.
3. Rebuild the map files if not already provided.
4. Start the local static map server.

After these steps, local source maps will be available for the production code.

Advantages:

* Easy implementation for existing infrastructure.
* Don't need any specialized private map servers.

Disadvantages:

* Slow, tedious. Developers need to figure out the deployed version and check
  out that version locally.
* Rebuilding source maps risks correctness issues.

**Option 2: Privately served maps**

The more complicated and robust option is to have source mapping comments point
to an internal server so that developers just have to "get on the network" and
have access to source maps while the general public does not.

For example, control comments in this scenario could look like:

```js
//# sourceMappingURL=http://vpn-only-server.com:3000/PATH/TO/app1.js.map
//# sourceMappingURL=http://authenticated-server.com:3000/PATH/TO/app1.js.map
```

The application build process should then go through these steps:

1. On building the application, publish the map files to a private server and
   capture the published URL of each map file.
2. Ensure that the build process writes a correct `sourceMappingURL` comment
   that corresponds to the privately hosted map server path.

Then, when a developer needs to debug production code, they:

1. Gain access to the private maps server. This could be, logging in to
   the VPN, authenticating to a publicly-accessible server, firing up an SSH
   tunnel to a non-public server, etc.

After these steps, privately hosted source maps will be available for the
production code.

Advantages:

* Developers can just "go online" and magically have the correct maps.
  No checking out versions, rebuilding, etc.

Disadvantages:

* Need to publish to private servers during build process.
* Need to host / control private map servers.

##### Basic Example

(Example source available at: [github.com/FormidableLabs/formidable-playbook/tree/master/examples/frontend/src/es5](https://github.com/FormidableLabs/formidable-playbook/tree/master/examples/frontend/src/es5))

Let's start with the familiar source code used in our
[shared library](./webpack-shared-libs.md) and
[code splitting example](./webpack-source-maps.md) examples.

[`foo.js`](../../examples/frontend/src/es5/foo.js)

```js
module.exports = function (id, msg) {
  return "<h1 id=\"" + id + "\">" + msg + "</h1>";
};
```

[`app1.js`](../../examples/frontend/src/es5/app1.js)

```js
var foo = require("./foo");

document.querySelector("#content").innerHTML += foo("app1", "App 1");
```

[`app2.js`](../../examples/frontend/src/es5/app2.js)

```js
var foo = require("./foo");

document.querySelector("#content").innerHTML += foo("app2", "App 2");
```

##### Source Map Example

(Example build / dist code available at: [github.com/FormidableLabs/formidable-playbook/tree/master/examples/frontend/webpack-source-maps](https://github.com/FormidableLabs/formidable-playbook/tree/master/examples/frontend/webpack-source-maps))

Our Webpack configuration uses the
[`SourceMapDevToolPlugin`](https://webpack.github.io/docs/list-of-plugins.html#sourcemapdevtoolplugin)
directly for maximum flexibility. Note however that there are many catch-all
[`devtool` options](https://webpack.github.io/docs/configuration.html#devtool)
for simpler / development configuration. In our case here, we:

- Enable minification with the `UglifyJsPlugin` so that we have a good example
  to see the impact of source maps with.
- Override the `append` option to `SourceMapDevToolPlugin` so that we can insert
  a full URL in the outputted `//# sourceMappingURL` comment.

[`webpack.config.js`](../../examples/frontend/webpack-source-maps/webpack.config.js)

```js
var path = require("path");
var webpack = require("webpack");

module.exports = {
  context: path.join(__dirname, "../src/es5"),
  entry: {
    app1: "./app1.js",
    app2: "./app2.js"
  },
  output: {
    path: path.join(__dirname, "dist/js"),
    filename: "[name].js"
  },
  plugins: [
    new webpack.optimize.UglifyJsPlugin({
      // Preserve the source map comment in minified code.
      sourceMap: true
    }),
    new webpack.SourceMapDevToolPlugin({
      // https://webpack.github.io/docs/configuration.html#output-sourcemapfilename
      filename: "[name].js.map",

      // Output sourceMappingURL comment.
      //
      // `[url]` is replaced with the `PATH/TO/<filename>`.
      //
      // Examples:
      // 1. Default to: `"\n//# sourceMappingURL=[url]"` for relative hosting
      append: ""
      // 2. Localhost + local repo checkout.
      // append: "\n//# sourceMappingURL=http://localhost:3000/" +
      //   "examples/frontend/webpack-source-maps/dist/js/[url]"
      // 3. Internal VPN URL.
      // append: "\n//# sourceMappingURL=http://my-vpn-url.com/PATH/[url]"
    })
  ]
};
```

This produces four files:

* [`dist/js/app1.js`](../../examples/frontend/webpack-source-maps/dist/js/app1.js):
  The `app1` entry point.
* [`dist/js/app1.js.map`](../../examples/frontend/webpack-source-maps/dist/js/app1.js.map):
  The `app1` source map file.
* [`dist/js/app2.js`](../../examples/frontend/webpack-source-maps/dist/js/app1.js):
  The `app2` entry point.
* [`dist/js/app2.js.map`](../../examples/frontend/webpack-source-maps/dist/js/app2.js.map):
  The `app2` source map file.

Let's look at the `app1` files in detail (the `app2` files are analogous):

[`dist/js/app1.js`](../../examples/frontend/webpack-source-maps/dist/js/app1.js)

```js
!function(n){function r(e){/*... SNIPPED (OTHER MINIFIED CODE) ...*/document.querySelector("#content").innerHTML+=e("app1","App 1")}]);
//# sourceMappingURL=app1.js.map
```

The `app1.js` file contains a minified bundle of the `app1` entry point with the
key `sourceMappingURL` control comment at the bottom. As it stands now, an
`app1.js.map` file is expected to be served from the same directory as the
application bundle.

As discussed in the [Integrating Source Maps](#integrating-source-maps) section,
there are many ways to serve and write the `sourceMappingURL`. Here, we choose
"Option 1" -- serving the maps alongside code -- which is appropriate for this
publicly available code as part of an open source playbook.

For private applications, see the commented out options for `append` that
correspond to locally checked out or private served source maps. We don't
provide more of a specific guide to implementing the private application source
maps support which are typically bespoke and customized to a specific
organization's needs and infrastructure. Nonetheless, the implementation steps
should follow logically from our [discussion](#private-apps) of the private
serving alternatives above.

[`dist/js/app1.js.map`](../../examples/frontend/webpack-source-maps/dist/js/app1.js.map)

```js
// Prettified and truncated.
{
  "version": 3,
  "sources": [
    "webpack:///app1.js",
    "webpack:///webpack/bootstrap 05941fb93e61871b9ba6?c490",
    "webpack:///./foo.js?60db",
    "webpack:///./app1.js"
  ],
  "names": [
    "modules",
    "__webpack_require__",
    "moduleId",
    /* ... SNIPPED ... */
    "querySelector",
    "innerHTML"
  ],
  "mappings": "CAAS,SAAUA,GCInB,QAAAC,/* ... SNIPPED ... */,WAAAH,EAAA",
  "file": "app1.js",
  "sourcesContent": [
    /* ... SNIPPED ... */
    "module.exports = function (id, msg) {\n  return \"<h1 id=\\\"\" + id + \"\\\">\" + msg + \"</h1>\";\n};\n\n\n\n//////////////////\n// WEBPACK FOOTER\n// ./foo.js\n// module id = 0\n// module chunks = 0 1",
    "var foo = require(\"./foo\");\n\ndocument.querySelector(\"#content\").innerHTML += foo(\"app1\", \"App 1\");\n\n\n\n//////////////////\n// WEBPACK FOOTER\n// ./app1.js\n// module id = 1\n// module chunks = 1"
  ],
  "sourceRoot": ""
}
```

This [JSON structure](http://www.html5rocks.com/en/tutorials/developertools/sourcemaps/#toc-anatomy)
corresponds to the source map [V3 spec](https://docs.google.com/document/d/1U1RGAehQwRypUTovF1KRlpiOFze0b-_2gc6fAH0KY0k/edit?hl=en_US&pli=1&pli=1).
The essential information in this file includes:

* The names of the individual files.
* The full original source code of the individual code files.
* Names of variables used and mappings of production code location to individual
  source file locations.

With this file available, a browser can download the map file, attach it to the
execution environment, and developers are now off to the debugging races with
the original development source!

Once we build these files, we can load the independent applications with a
standard index page:

[`index.html`](../../examples/frontend/webpack-source-maps/index.html)

```html
<!DOCTYPE html>
<html>
  <body>
    <div id="content" />
    <script src="./dist/js/app1.js"></script>
    <script src="./dist/js/app2.js"></script>
  </body>
</html>
```

And with these steps, both `app1` and `app2` have individualized full source
maps and offer a fighting chance for developers debugging live in production.

##### Advantages

**Summary**: You **absolutely should** be enabling source maps in production
and probably in development too. The disadvantages provided below are really
just the "costs" of doing the right thing.

* **Debug against original source files**: This is huge. Debugging single
  application bundles is awkward, and debugging large minified code chunks is
  downright impossible.
* **Integrates easily into public apps**: For development-time or projects that
  don't care if the full source is publicly served via source maps, integrating
  source maps is very straightforward.

##### Disadvantages

* **Debugging has limitations**: Source maps are not a _perfect_ representation
  of the minified code execution. For example, you cannot always place a
  breakpoint on a specific line of original source. Additionally, local variable
  names are not effectively unmangled, leaving you to debug with awkward,
  one-character variable names to step through.
  See the "[Limitations of source maps](http://www.mattzeunert.com/2016/02/14/how-do-source-maps-work.html#limitations-of-source-maps)"
  section of the article "[How do source maps work](http://www.mattzeunert.com/2016/02/14/how-do-source-maps-work.html)".
* **Complicates build**: The build infrastructure for private source maps in
  production code is complicated and must have an appropriate strategy. But
  the costs are well worth it.
