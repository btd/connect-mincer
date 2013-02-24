# connect-mincer

This is an Express-compatible, Connect middleware for [Mincer](https://github.com/nodeca/mincer).

## What is Mincer and why do I want this?

Mincer is an excellent port of Sprockets, which means it is a robust and comprehensive asset manager for your Node app. However, Mincer makes no assumptions about your application so by default it requires some work to get going with a typical Express app.

Using connect-mincer, you can skip that work and simply:

* Write and serve CoffeeScript, LESS, Stylus, Sass, etc
* Have everything recompiled on each request (in development)
* Serve files with an MD5 digest (for caching)
* Use whatever directory structure you want
* Precompile all your assets and have your Connect app read from the compile manifest

If you're used to the Rails asset pipeline, using this will give your Connect/Express application almost all the same capability - the only thing missing is a built-in precompile, but that's easily added (see the [example app](https://github.com/clarkdave/connect-mincer/tree/master/examples/express) for an example precompile script).

## Let's go!

    npm install git://github.com/clarkdave/connect-mincer

Now, in your connect app:

``` javascript
var ConnectMincer = require('connect-mincer');

var connectMincer = new ConnectMincer({
  root: __dirname,
  production: process.env.NODE_ENV === 'production',
  mountPoint: '/assets',
  manifestFile: __dirname + '/public/assets/manifest.json',
  paths: [
    'assets/css',
    'assets/js',
    'vendor/js'
  ]
});

app.use(connectMincer.assets());

if (process.env.NODE_ENV !== 'production')
  app.use('/assets', connectMincer.createServer());
```

The connectMincer.assets() middleware will:

* Provide js(), css() and asset_path() helpers for your views
* In development, ensure that assets are recompiled on every request

Now, in your views, you can do this:

``` html
<head>
  <%- css('main.css') %>
  <%- js('application.js') %>
</head>
```

These helpers will output something like: `<script src='/assets/application.js'></script>`.

The second piece of middleware, `connectMincer.createServer()`, sets up a Mincer server which will send the compiled version of the asset. This is great for development, though in production you'll probably want to have these files served by nginx or from a cloud host (see more about 'in production' below).

## In more detail

Mincer and this middleware are unopinionated about where your keep your assets. When you initialise connect-mincer you pass in several options:

- **root**: this is usually the root of your app. Asset paths are relative to this.
- **production**: set to true if the app is running in production mode.
- **mountPoint**: this is what the js, css and asset_path helpers use to create the URL for each asset.
- **paths**: a list of directories where your assets are located.

A typical app folder structure might be this:

    app/
      assets/
        js/
          application.js
        css/
          main.less
          blog/
            more.css
        images/
          logo.png
      lib/
      public/
      vendor/
        js/
          jquery.js
      app.js

With this, a suitable path list would be:

    ['assets/js', 'assets/css', 'assets/images', 'vendor/js']

Now anything within these paths can be referenced in your views with the helpers like so:

    css('main.css')
    css('blog/more.css')
    js('jquery.js')

Which would become:

    <link href='/assets/main.css' />
    <link href='/assets/blog/more.css' />
    <script src='/assets/jquery.js' />

## In production

For production use, connect-mincer lets you pass in a Mincer manifest file and will use this to generate correct asset paths (with MD5 digests). Precompiling is easy, using Mincer.Manifest, and there's an example of this below. For now, let's assume you have precompiled your assets to the `/public/assets` directory. This directory will contain a `manifest.json` file which links asset filenames to their digest format.

If you pass the manifest file to connect-mincer, e.g:

    manifestFile: __dirname + '/public/assets/manifest.json',

When the helpers (js, css, asset_path) are called in your views, connect-mincer will look up the compiled name in the manifest and use that, leading to an output like this:

    <script src='/assets/application-4b02e3a0746a47886505c9acf5f8c655.js'></script>

which will correspond to the file `/public/assets/application-4b02e3a0746a47886505c9acf5f8c655.js`. Now you can set nginx up to intercept requests to `/assets` and serve the static file in `/public/assets` instead. Thanks to the MD5 digest, you can set the cache headers to maximum. The next time you deploy and precompile the digests will change, and your app will adjust its `<script>` and `<link>` tags accordingly.

### But I want to use Node to serve my static assets

OK, cool, you can do that. After precompiling all your assets are ordinary static files in `public/assets`, so you can use the `connect.static` middleware to serve them like any other static file. You can also use the `connect.staticCache` middleware to speed it up for production.

``` javascript
app.use(connect.static(__dirname + '/public'));
```

## Precompiling

Because this is a middleware, it doesn't provide anything special to handle precompiling. But that's OK, because it's easy to do with Mincer so you can create your own custom precompile routine (e.g. a grunt task).

A simple precompile script:

``` javascript
var Mincer = require('mincer');

var env = new Mincer.Environment('./');
env.appendPath('assets/js');
env.appendPath('assets/css');
env.appendPath('vendor/js');

var manifest = new Mincer.Manifest(env, './public/assets');
manifest.compile(['*', '*/**'], function(err, data) {
  console.info('Finished precompile:');
  console.dir(data);
});
```

This will precompile everything in the `assets/js`, `assets/css` and `vendor/js` directories. You can pass in more specific paths to `manifest.compile()` if you only want certain things to be included.

If you were to run this from your root app directory, it would create the folder `/public/assets`, populate it with the compiled versions of all your assets, and create a manifest file suitable for passing to connect-mincer.

A more substantial example of a precompile script is part of the express example app, [here](https://github.com/clarkdave/connect-mincer/blob/master/examples/express/bin/precompile_assets.js).

## I want to know more about Mincer (bundles, custom engines, helpers)

Mincer is, like Sprockets, really powerful. The [Mincer documentation](http://nodeca.github.com/mincer/) has you covered.

connect-mincer supports anything Mincer does, so bundles and supported engines will all work out of the box. If you want to do more custom things, like adding helpers for your assets to use, you can add them directly to the Mincer Environment:

``` javascript
var connectMincer = new ConnectMincer({ ... });
connectMincer.environment.registerHelper('version', function() {
  return require(__dirname + '/package.json').version;
});
app.use(connectMincer.assets());
```

This will add a `version` helper which will be available in any asset, to be used like so:

``` javascript
app.version: '<%= version() %>';
```

Mincer supports helpers in EJS and Stylus. Fortunately, even if your asset is something else (e.g. CoffeeScript, LESS), you can attach the EJS processor to it and have variables too by appending `.ejs` to the file. Mincer processes a filename from right to left, so the file:

    css/main.less.ejs

will first be processed by EJS (resolving things like `<%= version() %>`) and then LESS itself.

**Note:** any modifications to the environment *must* be done before the connectMincer.assets() middleware is called. When the app runs in production mode, the environment is set to a read-only index (for speed), so any modifications must be done before this happens.

# TODO

- add tests
- allow use of a remote domain (e.g. Amazon S3) in production for helper outputs

# Licence

[MIT](https://github.com/clarkdave/connect-mincer/blob/master/LICENCE)
