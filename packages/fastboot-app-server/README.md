## Ember FastBoot App Server

The FastBoot App Server is an application server for hosting Ember
FastBoot apps. It manages downloading the Ember app, starting multiple
HTTP server processes, and detecting when new versions of the
application have been deployed.

FastBoot allows Ember apps to be rendered on the server, to support
things like search crawlers and clients without JavaScript. For more
information about FastBoot, see
[FastBoot website][fastboot].

[fastboot]: https://www.ember-fastboot.com

## Extensibility

The App Server is designed to be flexible and extensible enough to run
in whatever environment you want to use to host FastBoot apps. In
particular, you can provide a custom:

* *Downloader*, to control how app builds gets downloaded
* *Notifier*, to control how new versions of the build are detected
* *HTTP Server*, to use whatever stack you prefer for serving HTTP
  requests in Node.js

## Requirements

FastBoot App Server requires Node.js v10 or later.

## Quick Start

Put the following in a `fastboot-server.js` file:

```js
const FastBootAppServer = require('fastboot-app-server');

const MY_GLOBAL = 'MY GLOBAL';

let server = new FastBootAppServer({
  distPath: 'dist',
  gzip: true, // Optional - Enables gzip compression.
  host: '0.0.0.0', // Optional - Sets the host the server listens on.
  port: 4000, // Optional - Sets the port the server listens on (defaults to the PORT env var or 3000).
  buildSandboxGlobals(defaultGlobals) { // Optional - Make values available to the Ember app running in the FastBoot server, e.g. "MY_GLOBAL" will be available as "GLOBAL_VALUE"
  log: true, // Optional - Specifies whether the server should use its default request logging. Useful for turning off default logging when providing custom logging middlewares
    return Object.assign({}, defaultGlobals, { GLOBAL_VALUE: MY_GLOBAL });
  },
  chunkedResponse: true // Optional - Opt-in to chunked transfer encoding, transferring the head, body and potential shoeboxes in separate chunks. Chunked transfer encoding should have a positive effect in particular when the app transfers a lot of data in the shoebox.
});

server.start();
```

Configure `distPath` to point to the `dist` directory you upload to
your server. (See [Application Builds](#application-builds) below.)

Run the server file:

```
$ PORT=8000 node fastboot-server.js
```

This will start an HTTP server on port 8000. To stop the server, type
`Ctrl-C`.

NOTE: If you want to continue running `ember serve` in development, name the file `fastboot-server.js` instead.

## Application Builds

When you build an Ember.js app via `ember build`, it will build the app
for production and, by default, put the resulting files in your
application's `dist` directory.

## Clustering

Because Node.js is single-threaded, you must run multiple processes to
take advantage of multi-core systems. FastBoot App Server takes
advantage of Node's clustering support out of the box, automatically
spawning one worker HTTP server per core. You can override this via `options.workerCount`.

The app server will automatically spawn a new worker if one dies while
handling a request. When a new application deploy is detected, workers
will automatically reload with the newest version.

## Custom HTTP Server
You can customize HTTP server (add middlewares, subdomains, etc.), either directly:
```js
// start.js
const FastBootAppServer = require('fastboot-app-server');
const ExpressHTTPServer = require('fastboot-app-server/src/express-http-server');

const httpServer = new ExpressHTTPServer(/* {options} */);
const app = httpServer.app;
app.use('/api', apiRoutes);
let server = new FastBootAppServer({
  httpServer: httpServer
});

server.start();
```
or extend the provided HTTP server and override any methods you need:
```js
// my-custom-express-server.js
const FastBootAppServer = require('fastboot-app-server');
const ExpressHTTPServer = require('fastboot-app-server/src/express-http-server');

class MyCustomExpressServer extends ExpressHTTPServer {
  serve(middleware) {
    // put your custom code here, don't forget to add fastboot etc.
  }
}
// start.js
const MyCustomExpressServer = require('./my-custom-express-server');
const httpServer = new MyCustomExpressServer(/* {options} */);
let server = new FastBootAppServer({
  httpServer: httpServer
});

server.start();
```

## Pre and Post FastBoot middleware hooks

If you need something less than a custom server and just want to run some middleware
before or after FastBoot runs, the server provides hooks for you to do so:

```js
// Custom Middlewares
function modifyRequest(req, res, next) { /* do pre-fastboot stuff to `req` */ };
function handleErrors(err, req, res, next) { /* do error recovery stuff */ };

const server = FastBootAppServer({
  beforeMiddleware: function (app) { app.use(modifyRequest); },
  afterMiddleware: function (app) { app.use(handleErrors); }
})
```

## Logging

We provide simple log output by default, but if you want more logging control, you can disable the
simple logger using the `log: false` option, and provide a custom middleware that suits your logging needs:

```js
let server = new FastBootAppServer({
  log: false,
  beforeMiddleware: function(app) {
    let logger = function(req, res, next) {
      console.log('Hello from custom request logger');

      next();
    };

    app.use(logger);
  },
});
```

## Downloaders

You can point the app server at a static path that you manage, but that
means taking responsibility for uploading builds to each server
whenever you want to deploy a new version.

Instead, you can provide the app server with a _downloader_, an adapter
that knows how to download the current version of your application.

For example, to use the S3 downloader that downloads a zip file from
AWS S3:

```js
const S3Downloader = require('fastboot-s3-downloader');
const FastBootAppServer = require('fastboot-app-server');

let downloader = new S3Downloader({
  bucket: 'S3_BUCKET',
  key: 'S3_KEY'
});

let server = new FastBootAppServer({
  downloader: downloader
});

server.start();
```

### Available Downloaders

* [fastboot-s3-downloader](https://github.com/ember-fastboot/fastboot-s3-downloader)
* [fastboot-gcloud-storage-downloader](https://github.com/EmberSherpa/fastboot-gcloud-storage-downloader)

### Writing a Downloader

To write your own downloader, construct an object that conforms to the
following interface:

#### `download()`

Returns a promise that resolves to the path to the downloaded `dist`
directory (which does not have to be named `dist`).

Note that `download()` may be called more than once in the lifetime of
an application, if a new version is deployed. Make sure your downloader
cleans up after itself to avoid running out of disk space.

## Notifiers

Once the FastBoot App Server is up and running, it will happily chug
away until the server dies or it reaches the inevitable heat death of the
universe. Before that happens, presumably, you may want to deploy a new
version of your application.

_Notifiers_ are responsible for detecting when a new version of an app
has been deployed and reloading the app server.

For example, here's how to use the S3 notifier, which polls the last
modified date of a file on S3 to detect new versions:

```js
const S3Notifier = require('fastboot-s3-notifier');
const FastBootAppServer = require('fastboot-app-server');

let notifier = new S3Notifier({
  bucket: S3_BUCKET,
  key: S3_KEY
});

let server = new FastBootAppServer({
  notifier: notifier
});

server.start();
```

### Available Notifiers

* [fastboot-s3-notifier](https://github.com/ember-fastboot/fastboot-s3-notifier)
* [fastboot-fs-notifier](https://github.com/iheanyi/fastboot-fs-notifier)
* [fastboot-watch-notifier](https://github.com/pwfisher/fastboot-watch-notifier)
* [fastboot-gcloud-storage-notifier](https://github.com/EmberSherpa/fastboot-gcloud-storage-notifier)

### Writing a Notifier

To write your own notifier, construct an object that conforms to the
following interface:

#### `subscribe(notify)`

The `subscribe()` method on your notifier is passed a `notify` function.
If you detect that a new version of your app has been deployed (whether
via polling or a push notification), call this function to trigger a
reload.

## Basic Authentication

You can enable Basic Authentication by providing `username` and `password` options:

```js
const FastBootAppServer = require('fastboot-app-server');

let server = new FastBootAppServer({
  username: 'tomster',
  password: 'zoey'
});
```

## Scraper Issues

### Twitter and LinkedIn

As of 2019-06-06, Twitter and LinkedIn's scrapers have a hard time extracting your site's metadata for sharing if `chunkedResponse` is set to `true` in your `server.js` file. Set `chunkedResponse: false` if your meta tags are in place but the [Twitter card validator](https://cards-dev.twitter.com/validator) shows "Card not found" or [LinkedIn's Post Inspector](https://www.linkedin.com/post-inspector/) shows a 500 error.
