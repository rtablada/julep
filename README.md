# Julep.js
## Something to make the Broccoli(.js) go down a little easier.

> **WARNING** This project is currently under heavy development and is mostly being created to match some of the thoughts and planned APIs listed in this documentation.
> Right now it probably doesn't work.
> But I'd love your feedback on different proposals and the way the API is looking.

## Reasoning

Broccoli.js is an incredibly powerful tool for managing and serving your asset builds during development.
Unfortunately, the bare API can be really verbose and often even [simple convention driven Brocfiles](https://github.com/jayphelps/broccoli-babel-boilerplate).
This isn't the fault of the plugin creators or Babel: Jo Liss even said she hoped that other creators would create more opinionated build frameworks on top of Broccoli - so here's my go at things.

## Inspiration

Coming from the Ember community and being one of the early users of Ember CLI, I love the way that addons are able to be automatically registered and give developers a ton of power to get things done.
Similarly, I like the simplicity of tools like Laravel's Elixir which are convention driven and have a small API to allow developers to get the usual stuff done quick.

## Blue Sky Execution

Ideally, Julep.js will let a project's Brocfile go from something like this: 

```js
var funnel = require('broccoli-funnel');
var concat = require('broccoli-concat');
var mergeTrees = require('broccoli-merge-trees');
var esTranspiler = require('broccoli-babel-transpiler');
var compileSass = require('broccoli-sass-source-maps');
var pkg = require('./package.json');

var src = 'src';
var styleDir = 'sass';

var indexHtml = funnel('public', {
  files: ['index.html']
});

var js = esTranspiler(src, {
  stage: 0,
  moduleIds: true,
  modules: 'amd',

  // Transforms /index.js files to use their containing directory name
  getModuleId: function (name) { 
    name = pkg.name + '/' + name;
    return name.replace(/\/index$/, '');
  },

  // Fix relative imports inside /index's
  resolveModuleSource: function (source, filename) {
    var match = filename.match(/(.+)\/index\.\S+$/i);

    // is this an import inside an /index file?
    if (match) {
      var path = match[1];
      return source
        .replace(/^\.\//, path + '/')
        .replace(/^\.\.\//, '');
    } else {
      return source;
    }
  }
});

var styles = compileSass('.', styleDir . 'app.scss', 'app.css');

var main = concat(js, {
  inputFiles: [
    '**/*.js'
  ],
  outputFile: '/' + pkg.name + '.js'
});

module.exports = mergeTrees([main, indexHtml, styles]);
```

Into something that I think is a lot clearer and gives some good defaults to cover most small to mid-sized projects:

```js
var julep = require('julep');

var publicDir = 'public';
var srcDir = 'src';
var styleRoot = 'sass';

// Merges and serves files without further build steps
julep.serve(publicDir);

// Babel Compiles js with a lot of sane defaults
// Then Concatinates to a 
julep.babel(srcDir).concat();

// Compiles Sass scoped to directory for entry file
julep.sass(styleDir, 'app.scss');

module.exports = julep.getMerged();
```

## First Thoughts

Similar to Broccoli, Laravel Elixer, and even Ember-CLI, Julep will have a fairly small set of responsibilities:

- Autoloading Addons
- Tracking and merging trees added by addons
- Allowing Tree Chaining to allow things like multi-stage builds on a single tree & hooks for addons to return their results without polluting other build trees
- `julep.serve`

### Autoloading

Looking at the way that Ember-CLI loads in addons I think a similar approach could work for Julep Addons.
Essentially the Julep main would look for dependencies in your `package.json` file, map over those dependencies and check to see if they have `julep-addon` in their `keywords` array.
If so, then there will be some loading to bring that module in and add a method on the `julep` object returned when you run `require('julep')`.

### Merging Trees

One of the most common steps that I have found when writing large Broccoli files by hand is forgetting to merge all of the trees that I have processed.
My hope is that when a Julep addon is finished with processing, it Juelp will log that finished tree and then at the end of your `Brocfile.js` when you call `julep.getMerged`, it will take all of the trees you have worked on and merge them so you never miss a merge!

### Build Chaining

Sometimes you want to do multiple things to a single directory of files, such as Babel preprocessing and then concatinating and then uglifying.
Ideally you should be able to quickly and easily run these multi-step processes without having to worry about breaking another part of your tool chain and without having to revert to working with regular broccoli trees again.

### `julep.serve`

In almost every project, you will need to pull some unprocessed files in to your builds.
Usually to do this you have to use `broccoli-static-compiler` or `broccoli-funnel`.
Since this is so common, I think it would only make sense for `julep.serve` or `julep.passthrough` to be included with the main Julep library and autoloader.
This reduces repetitive dependencies.

