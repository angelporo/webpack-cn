A loader is a node module exporting a `<span class="function"><span class="keyword">function</span></span>`.

This function is called when a resource should be transformed by this loader.

In the simple case, when only a single loader is applied to the resource, the loader is called with one parameter: the content of the resource file as string.

The loader can access the [loader API](loaders.html) on the `<span class="keyword">this</span>` context in the function.

A sync loader that only wants to give a one value can simply `<span class="keyword">return</span>` it. In every other case the loader can give back any number of values with the `<span class="keyword">this</span>.callback(err, values...)` function. Errors are passed to the `<span class="keyword">this</span>.callback` function or thrown in a sync loader.

The loader is expected to give back one or two values. The first value is a resulting JavaScript code as string or buffer. The second optional value is a SourceMap as JavaScript object.

In the complex case, when multiple loaders are chained, only the last loader gets the resource file and only the first loader is expected to give back one or two values (JavaScript and SourceMap). Values that any other loader give back are passed to the previous loader.

## [→](#examples)Examples

    // Identity loader
    module.exports = function(source) {
      return source;
    };

    // Identity loader with SourceMap support
    module.exports = function(source, map) {
      this.callback(null, source, map);
    };

## [→](#guidelines)Guidelines

(Ordered by priority, first one should get the highest priority)

Loaders should

### [→](#do-only-a-single-task)do only a single task

Loaders can be chained. Create loaders for every step, instead of a loader that does everything at once.

This also means they should not convert to JavaScript if not necessary.

Example: Render HTML from a template file by applying the query parameters

I could write a loader that compiles the template from source, execute it and return a module that exports a string containing the HTML code. This is bad.

Instead I should write loaders for every task in this use case and apply them all (pipeline):

*   jade-loader: Convert template to a module that exports a function.
*   apply-loader: Takes a function exporting module and returns raw result by applying query parameters.
*   html-loader: Takes HTML and exports a string exporting module.

### [→](#generate-modules-that-are-modular)generate modules that are modular

Loader generated modules should respect the same design principles like normal modules.

Example: That’s a bad design: (not modular, global state, …)

    require("any-template-language-loader!./xyz.atl");

    var html = anyTemplateLanguage.render("xyz");

### [→](#flag-itself-cacheable-if-possible)flag itself cacheable if possible

Most loaders are cacheable, so they should flag itself as cacheable.

Just call `cacheable` in the loader.

    // Cacheable identity loader
    module.exports = function(source) {
        this.cacheable();
        return source;
    };

### [→](#not-keep-state-between-runs-and-modules)not keep state between runs and modules

A loader should be independent of other modules compiled (expect of these issued by the loader).

A loader should be independent of previous compilations of the same module.

### [→](#mark-dependencies)mark dependencies

If a loader uses external resources (i. e. by reading from filesystem), they **must** tell about that. This information is used to invalidate cacheable loaders and recompile in watch mode.

    // Loader adding a header
    var path = require("path");
    module.exports = function(source) {
        this.cacheable();
        var callback = this.async();
        var headerPath = path.resolve("header.js");
        this.addDependency(headerPath);
        fs.readFile(headerPath, "utf-8", function(err, header) {
            if(err) return callback(err);
            callback(null, header + "\n" + source);
        });
    };

### [→](#resolve-dependencies)resolve dependencies

In many languages there is some schema to specify dependencies. i. e. in css there is `@import` and `url(...)`. These dependencies should be resolved by the module system.

There are two options to do this:

*   Transform them to `require`s.
*   Use the `<span class="keyword">this</span>.resolve` function to resolve the path

Example 1 css-loader: The css-loader transform dependencies to `require`s, by replacing `@import`s with a require to the other stylesheet (processed with the css-loader too) and `url(...)` with a `require` to the referenced file.

Example 2 less-loader: The less-loader cannot transform `@import`s to `require`s, because all less files need to be compiled in one pass to track variables and mixins. Therefore the less-loader extends the less compiler with a custom path resolving logic. This custom logic uses `<span class="keyword">this</span>.resolve` to resolve the file with the configuration of the module system (aliasing, custom module directories, etc.).

If the language only accept relative urls (like css: `url(file)` always means `.<span class="regexp">/file</span>`), there is the `~`-convection to specify references to modules:

    url(file) -> require("./file")
    url(~module) -> require("module")

### [→](#extract-common-code)extract common code

don’t generate much code that is common in every module processed by that loader. Create a (runtime) file in the loader and generate a `require` to that common code.

### [→](#should-not-embed-absolute-paths)should not embed absolute paths

don’t put absolute paths in to the module code. They break hashing when the root for the project is moved. There is a method [`stringifyRequest` in loader-utils](https://github.com/webpack/loader-utils#stringifyrequest) which converts an absolute path to an relative one.

Example:

    var loaderUtils = require("loader-utils");
    return "var runtime = require(" +
      loaderUtils.stringifyRequest(this, "!" + require.resolve("module/runtime")) +
      ");";

### [→](#use-a-library-as-peerdependencies-when-they-wrap-it)use a library as `peerDependencies` when they wrap it

using a peerDependency allows the application developer to specify the exact version in `package.json` if desired. The dependency should be relatively open to allow updating the library without needing to publish a new loader version.

    "peerDependencies": {
        "library": "^1.3.5"
    }

### [→](#programmable-objects-as-query-option)programmable objects as `query`-option

there are situations where your loader requires programmable objects with functions which cannot stringified as `query`-string. The less-loader, for example, provides the possibility to specify [LESS-plugins](https://github.com/webpack/less-loader#less-plugins). In these cases, a loader is allowed to extend webpack’s `options`-object to retrieve that specific option. In order to avoid name collisions, however, it is important that the option is namespaced under the loader’s camelCased npm-name.

Example:

    // webpack.config.js
    module.exports = {
      ...
      lessLoader: {
        lessPlugins: [
          new LessPluginCleanCSS({advanced: true})
        ]
      }
    };

The loader should also allow to specify the config-key (e.g. `lessLoader`) via `query`. See [discussion](https://github.com/webpack/less-loader/pull/40) and [example implementation](https://github.com/webpack/less-loader/blob/39f742b4624fceae6d9cf266e9554d07a32a9c14/index.js#L49-51).

### [→](#be-added-to-the-list-of-loaders)be added to the [list of loaders](list-of-loaders.html)

## [→](#read-more)Read more

Read more about [loaders](loaders.html).