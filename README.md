# assetflow

Assetflow is an asset deployment tool. It supports md5 hash comparison with **S3** and enables you to create powerful asset flows easily and fast. It is a [Grunt task][grunt] and applies solid cache-busting techniques transparently.

> If you are not familiar with Grunt check out the [Grunt's Getting Started guide][Getting Started].

[![Build Status](https://travis-ci.org/verbling/assetflow.png?branch=master)](https://travis-ci.org/verbling/assetflow)

## Overview

A typical deployment flow using assetflow:

* Scan assets and generate MD5 hashes.
* Create the `manifest.json` file.
* Copy all assets and rename them with their md5 hash to a temporary location.
* Perform `HEAD` operations with **S3** and compare hashes using S3's `ETAG`.
* Upload all assets that did not have a matching `ETAG`.

Optionally there are two more tasks you can perform:

* Search & replace any set of assets based on a custom keyword, i.e. `__ASSET(img/logo.jpg)`.
* Create the `clientManifest.js` file, a client optimized subset of the manifest.

## Install

```shell
npm install assetflow --save-dev
```

## Table Of Contents

* [Grunt Task `assets`](#grunt-task-assets) :: Creates the manifest file and copies your assets to a temp folder.
* [Grunt Task `assetsReplace`](#grunt-task-assetsreplace) :: Replaces defined keywords in files using the manifest file.
* [Grunt Task `assetsBundle`](#grunt-task-assetsbundle) :: Create a front-end optimized manifest file.
  - [Using assetflow on the web](#using-assetflow-on-the-front-end)
* [Grunt Task `assetsS3`](#grunt-task-assetss3) :: Compare assets' hashes with S3 and upload new and changed files.
* [Using Assetflow on Node](#using-assetflow-on-node)

## Grunt Task `assets`

The `assets` task performs these operations:

* Scans all the defined assets.
* Generates **md5** hashes for each asset.
* Create the `manifest.json` file.
  - If a manifest file exists, it will compare the hashes.
* Copy all new or updated assets and rename them with their md5 hash to a temporary location.

When this task finishes all your assets have been copied to a new temporary folder that you defined. This folder will contain your assets renamed with their own hash, like so:

`app.js` --> `app-h522md41d.js`

The `manifest.json` file generated by this task keeps a reference to all your assets so their names can be properly resolved in all environments.

### Options

#### `manifest`
**Type**: `string` **Default**: `manifest.json`

Define the location of the manifest file.

#### `cdnurl`
**Type**: `string` **Default**: *none*

Add the url of your CDN to prepend it to all assets.

#### `rel`
**Type**: `string` **Default**: *none*

The `rel` option will perform directory subtraction on the source to calculate the relative path to the asset. Consider this case:

Your folder of static assets is under `assets/`, so the path to your logo would be `assets/img/logo.png` which would be accessed by the browser as `/img/logo.png`.

Declaring the `assets` folder as a `rel` path will make sure that all assets have the proper url.

Example
```js
assets: {
  options: {
    rel: 'assets/'
  },
  all: {
    src: 'assets/**',
    dest: 'temp/assets'
  }
}
```

#### `truncateHash`
**Type**: `number` **Default**: *none*

The **md5** hash is 32 bytes long, you don't need all of it, use this option to truncate the hash down to *n* chars.

#### `prepend`
**Type**: `string` **Default**: *none*

This option will prepend a value to the asset's key. It is mostly used to prepend a slash and make the asset key absolute, for example:

By default, the `assets` task will create records in the `manifest.json` file as relative web paths: `img/logo.png`. If you need the key to be an absolute path then you have to use `prepend`.

```js
options: {
  prepend: '/'
}
```

#### `maxOperations`
**Type**: `number` **Default**: `100`

The maximum number of concurrent operations, in this case the operations are file copying.

> Set this to 0 to disable throttling. Set it to 1 to make all operations serial.

#### `progress`
**Type**: `boolean` **Default**: `false`

A fancy progress bar.

#### `debug`
**Type**: `boolean` **Default**: `false`

Print extra debugging information.

### Example

Here's an annotated configuration for the `assets` task:

```js
assets: {
  // global task options
  options: {
    // don't output debug information
    debug: false,
    // trancate the hash length to 8 chars.
    truncateHash: 8,
    // define the location of the manifest file.
    manifest: 'temp/manifest.json',
    // define the location of the cdn.
    cdnurl: 'http://d3s3z9buwru1xx.cloudfront.net/assets/',
    // Set maximum number of file copying concurent operations.
    maxOperations: 100,
    // Show a fancy progress indicator
    progress: true,
    // Set path to substract so the relative path for the assets can be calculated.
    rel: 'lib/'
  },

  all: {
    // local task options
    options: {
      // change the rel path to test/case
      rel: 'assets/'
    },
    src: [
      // all files under folder assets
      'assets/**',
      // except all files in less folder
      '!assets/less/**',
      // except all files in handlebars folder
      '!assets/handlebars/**'
    ],
    dest: 'temp/assets'
  }
}
```
<sup>[↑ Back to TOC](#table-of-contents)</sup>

## Grunt Task `assetsReplace`

The `assetsReplace` task will search and replace the contents of your assets. It is useful for cases where you don't have the ability of a '*helper*' to resolve your assets.

LESS files are a typical example, use a custom keyword to include your assets and run the `assetsReplace` task to populate the asset urls in your `.less` files. For example if the custom keyword is `__ASSET()`:

```less
@bg-dot-light: url(__ASSET(img/pdf-icon-cv.png)) repeat 0 0 #2a2a2a;
```

After the `assetsReplace` task is executed the same line will look like this:

```less
@bg-dot-light: url(http://d3s3z9buwru1xx.cloudfront.net/assets/img/pdf-icon-cv-fk44j2s.png) repeat 0 0 #2a2a2a;
```

> The `assetsReplace` task is based on [grunt-string-replace][grunt-replace] by [@erickrdch][erickrdch]


### Options

#### `manifest`
**Type**: `string` **Default**: `manifest.json`

Define the location of the manifest file.

#### `key`
**Type**: `string` **Default**: *none*

Define the keyword that will be searched for replacement. Use the `%` char as a placeholder for the asset name. e.g.:
```js
options: {
  key: '__ASSET(%)'
}
```

#### `keyRegex`
**Type**: `string` **Default**: *none*

Use this to define a regex as a keyword. The type must be string, but the string will be evaluated as a regex. The `%` char is a placeholder for the asset name and the reason why this option needs to be a string instead of a regex type.

So take case to double escape what you need escaped, e.g.

```js
// this regex
var reg = /match[\s]space/;

// is represented like that as a string:
var strReg = 'match[\\\s]space';
```

#### `prepend`
**Type**: `string` **Default**: *none*

This option will prepend a value to the asset's key. It is mostly used to prepend a slash and make the asset key absolute.

#### `debug`
**Type**: `boolean` **Default**: `false`

Print extra debugging information.

### Example

Here's an annotated configuration for the `assetsReplace` task:

```js
assetsReplace: {
  // global task options
  options: {
    // don't output debug information
    debug: false,
    // define the location of the manifest file.
    manifest: 'temp/manifest.json',
  },
  // the less target
  less: {
    options: {
      // define the keyword for this target
      key: '__ASSET(%)'
    },
    files: {
      // Search & replace all .less files under the assets/less folder
      // and output the result in the temp/less folder
      'temp/less/': ['assets/less/**/*.less']
    }
  },
  // the handlebars target
  handlebars: {
    options: {
      // a regex for lax matching {{asset "%"}} allowing for spaces in between
      // and single quotes.
      keyRegex: '\\\{\\\{[\\\s]*asset[\\\s]+[\\\'\\\"]{1}%[\\\'\\\"]{1}[\\\s]*\\\}\\\}',
      // prepend the slash on every asset query
      prepend: '/'
    },
    // all the files from the assets/handlebars folder
    src: 'assets/handlebars/**/*.hbs',
    // output to temp/handlebars folder
    dest: 'temp/handlebars/'
  }
}
```
<sup>[↑ Back to TOC](#table-of-contents)</sup>

## Grunt Task `assetsBundle`

The assetsBundle task will create a compact version of the manifest file, optimized for transferring to the client as a javascript file. The `manifest.json` file is pretty large and not suited for getting transfered. Furthermore it might be the case where you have several hundreds, if not thousands of assets but you only need a handful to be consumed by your front-end app. This is the task that enables you to do this.

### Options

#### `manifest`
**Type**: `string` **Default**: `manifest.json`

Define the location of the manifest file.

#### `amd`
**Type**: `boolean` **Default**: `false`

export the assets as an AMD module.

#### `commonjs`
**Type**: `boolean` **Default**: `false`

export the assets using commonjs pattern. e.g: module.exports=..

#### `ns`
**Type**: `string` **Default**: `ASSETS`

Define a namespace to export using the global `window` Object. You can use dot notation.

#### `assets`
**Type**: `Array` **Default**: `[]`

An array of strings where you define the asset filename keys you want included.

#### `debug`
**Type**: `boolean` **Default**: `false`

Print extra debugging information.


### Example

Here's an annotated configuration for the `assetsBundle` task:

```js
assetsBundle: {
  // global task options
  options: {
    // define the location of the manifest file.
    manifest: 'temp/manifest.json',
    // export as AMD
    amd: true
  },

  // Export as amd
  amd: {
    dest: 'temp/bundles/clientManifest.amd.js'
  },

  // Export using namespaces
  ns: {
    options: {
      // Export assets on this global namespace:
      ns: 'app.assets'
    },
    dest: 'temp/bundles/clientManifest.ns.js'
  }
}
```

### Using assetflow on the Front-End

The javascript file generated by the `assetsBundle` task will export an Object that contains key value pairs to the assets. If you are using AMD to export the assets this is how you'd access them:

```js
var assets    = require('assets');

var asset = assets['/img/logo.png'];
// --> https://d3s3z9buwru1xx.cloudfront.net/assets/img/logo-fh422j4f.png
```

> This is a weak part in the API and will surely see changes in the future.

<sup>[↑ Back to TOC](#table-of-contents)</sup>

## Grunt Task `assetsS3`

The `assetsS3` task will read the `manifest.json` file and upload all the assets to **S3**. Although it's optional, it is highly advised to use the `chechS3Head` option which enables md5 hash checking between S3 and your local files.

> The `assetsS3` task is based on [grunt-S3][grunt-S3] by [@pifantastic][pifantastic]. [All options from that task](https://github.com/pifantastic/grunt-s3#options) are available in this one too.


### Options

These are the options that are only available in the `assetsS3` task of Assetflow.

#### `manifest`
**Type**: `string` **Default**: `manifest.json`

Define the location of the manifest file.

#### `checkS3Head`
**Type**: `boolean` **Default**: `false`

Perform md5 hash comparisons between the local assets and the ones in S3. It is highly advisable that you enable this option so you only upload files that have been updated.

#### `rel`
**Type**: `string` **Default**: *none*

The `rel` option will perform directory subtraction on the source to calculate the relative path to the asset.


#### `maxOperations`
**Type**: `number` **Default**: `100`

The maximum number of concurrent operations, in this case the operations are network uploads.

> Set this to 0 to disable throttling. Set it to 1 to make all operations serial.

#### `progress`
**Type**: `boolean` **Default**: `false`

A fancy progress bar.

#### `debug`
**Type**: `boolean` **Default**: `false`

Print extra debugging information.


### Grunt-S3 Options

These options are available by [Grunt-S3][], sic:

The grunt-s3 task is now a [multi-task](https://github.com/gruntjs/grunt/wiki/Creating-tasks); meaning you can specify different targets for this task to run as.

The following are the default options available to each target.

* **key** - (*string*) An Amazon S3 credentials key
* **secret** - (*string*) An Amazon S3 credentials secret
* **bucket** - (*string*) An Amazon S3 bucket
* **region** - (*string*) An Amazon AWS region (see http://docs.aws.amazon.com/general/latest/gr/rande.html#s3_region)
* **maxOperations** - (*number*) max number of concurrent transfers - if not specified or set to 0, will be unlimited.
* **encodePaths** - (*boolean*) if set to true, will encode the uris of destinations to prevent 505 errors. Default: false
* **headers** - (*object*) An object containing any headers you would like to send along with the
transfers i.e. `{ 'X-Awesomeness': 'Out-Of-This-World', 'X-Stuff': 'And Things!' }`
* **access** - (*string*) A specific Amazon S3 ACL. Available values: `private`, `public-read`, `
public-read-write`, `authenticated-read`, `bucket-owner-read`, `bucket-owner-full-control`
* **gzip** - (*boolean*) If true, uploads will be gzip-encoded.
* **gzipExclude** - (*array*) Define extensions of files you don't want to run gzip on, an array of strings ie: `['.jpg', '.jpeg', '.png']`.
* **upload** - (*array*) An array of objects, each object representing a file upload and containing a `src`
and a `dest`. Any of the above values may also be overriden.
* **download** - (*array*) An array of objects, each object representing a file download and containing a
`src` and a `dest`. Any of the above values may also be overriden.
* **del** - (*array*) An array of objects, each object containing a `src` to delete from s3. Any of
the above values may also be overriden.
* **debug** - (*boolean*) If true, no transfers with S3 will occur, will print all actions for review by user

### Example

Here's an annotated configuration for the `assetsS3` task:

```js
assetsS3: {
  options: {
    // no debug info
    debug: false,
    // enable checking md5 hashes by performing S3 HEAD requests.
    checkS3Head: true,
    // the manifest file
    manifest: 'temp/manifest.json',

    // aws credentials
    key: 'AWS-KEY',
    secret: 'AWS-SECRET',
    bucket: 'S3-BUCKET',

    // Enable public access
    access: 'public-read',

    // don't show the fancy progress indicator.
    progress: false
  },
  all: {
    // These options override the defaults
    options: {
      // limit concurent uploads to 100
      maxOperations: 100
    },
    // the 'upload' option key is required
    upload: {
      // all files from temp/assets
      src: 'temp/assets/**',
      // a prefix folder on S3
      dest:  'assets/',
      // the rel option
      rel: 'temp/assets/',
      // gzip uploaded files
      gzip: true,
      // excluded these extensions
      gzipExclude: ['.jpeg', '.jpg', '.png', '.gif', '.less', '.mp3',
          '.mp4', '.mkv', '.webm', '.gz'],
      // upload assets with a looong expire Cache-Control header.
      headers: {'Cache-Control': 'max-age=31536000, public'}
    }
  }
}
```
<sup>[↑ Back to TOC](#table-of-contents)</sup>

## Using Assetflow on Node

You can use the Assetflow library on node:

```js
// mind the () in the end!
var assets = require('assetflow')();

assets.config({
  manifest: __dirname + '/assetManifest.json'
});

var assetUrl = assets.asset('/img/logo.png');
```

> Like the client API, node's API is weak and may change in the future.

<sup>[↑ Back to TOC](#table-of-contents)</sup>

## Authors

* [@thanpolas][thanpolas]

## Release History
- **v0.1.4**, *08 May 2013*
  - Updated Knox so Assetflow will work for node 0.10.x
  - Fixed bug when no rel path is used.
- **v0.1.2**, *17 April 2013*
  - S3 paths now get normalized using `path.normalize()`.
- **v0.1.1**, *19 March 2013*
  - Added support for commonjs for browsers.
- **v0.1.0**, *Mid March 2013*
  - Big Bang

## License
Copyright 2012 Verbling (Fluency Forums Corporation)

Licensed under the MIT License

[grunt]: http://gruntjs.com/
[Getting Started]: https://github.com/gruntjs/grunt/wiki/Getting-started
[Gruntfile]: https://github.com/gruntjs/grunt/wiki/Sample-Gruntfile "Grunt's Gruntfile.js"
[grunt-replace]: https://github.com/erickrdch/grunt-string-replace "Grunt string replace"
[grunt-S3]: https://github.com/pifantastic/grunt-s3 "grunt-s3 task"
[pifantastic]: https://github.com/pifantastic "Aaron Forsander"
[erickrdch]: https://github.com/erickrdch "Erick Ruiz de Chavez on GitHub"
[thanpolas]: https://github.com/thanpolas "Thanasis Polychronakis"
