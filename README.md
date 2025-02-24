<h1 align="center">
  <img alt="threads.js" src="./docs/assets/logo-label.png" width="75%" />
</h1>
<p align="center">
  <a href="https://travis-ci.org/andywer/threads.js" target="_blank"><img alt="Build status" src="https://img.shields.io/travis/andywer/threads.js/v1.svg?style=flat-square"></a>
  <a href="https://www.npmjs.com/package/threads" target="_blank"><img alt="npm (tag)" src="https://img.shields.io/npm/v/threads.svg?style=flat-square"></a>
</p>

<br />

Offload CPU-intensive tasks to worker threads in node.js, web browsers and electron using one uniform API.

Uses web workers in the browser, `worker_threads` in node 12+ and [`tiny-worker`](https://github.com/avoidwork/tiny-worker) in node 8 to 11.

### Features

* First-class support for **async functions** & **observables**
* Write code once, run it **on all platforms**
* Manage bulk task executions with **thread pools**
* Use **require()** and **import**/**export** in workers
* Works great with **webpack**

### Version 0.x

You can find the old version 0.12 of threads.js on the [`v0` branch](https://github.com/andywer/threads.js/tree/v0). All the content on this page refers to version 1.0 which is a rewrite of the library with a whole new API.

## Installation

```
npm install threads tiny-worker
```

*You only need to install the `tiny-worker` package to support node.js < 12. It's an optional dependency and used as a fallback if `worker_threads` are not available.*

## Platform support

<details>
<summary>Run on node.js</summary>

<p></p>

Running code using threads.js in node works out of the box.

Note that we wrap the native `Worker`, so `new Worker("./foo/bar")` will resolve the path relative to the module that calls it, not relative to the current working directory.

That aligns it with the behavior when bundling the code with webpack or parcel.

</details>

<details>
<summary>Webpack build setup</summary>

#### Webpack config

Use with the [`threads-plugin`](https://github.com/andywer/threads-plugin). It will transparently detect all `new Worker("./unbundled-path")` expressions, bundles the worker code and replaces the `new Worker(...)` path with the worker bundle path, so you don't need to explicitly use the `worker-loader` or define extra entry points.

```sh
  npm install -D threads-plugin
```

Then add it to your `webpack.config.js`:

```diff
+ const ThreadsPlugin = require('threads-plugin');

  module.exports = {
    // ...
    plugins: [
+     new ThreadsPlugin()
    ]
    // ...
  }
```

#### Node.js bundles

If you are using webpack to create a bundle that will be run in node (webpack config `target: "node"`), you also need to specify that the `tiny-worker` package used for node < 12 should not be bundled:

```diff
  module.exports = {
    // ...
+   externals: {
+     "tiny-worker": "tiny-worker"
+   }
    // ...
}
```

Make sure that `tiny-worker` is listed in your `package.json` `dependencies` in that case.

#### When using TypeScript

Make sure the TypeScript compiler keeps the `import` / `export` statements intact, so webpack resolves them. Otherwise the `threads-plugin` won't be able to do its job.

```diff
  module.exports = {
    // ...
    module: {
      rules: [
        {
          test: /\.ts$/,
          loader: "ts-loader",
+         options: {
+           compilerOptions: {
+             module: "esnext"
+           }
+         }
        }
      ]
    },
    // ...
  }
```

</details>

<details>
<summary>Parcel bundler setup</summary>

<p></p>

You need to import `threads/register` once at the beginning of your application code (in the master code, not in the workers):

```diff
  import { spawn } from "threads"
+ import "threads/register"

  // ...

  const work = await spawn(new Worker("./worker"))
```

This registers the library's `Worker` implementation for your platform as the global `Worker`. This is necessary, since you cannot `import { Worker } from "threads"` or Parcel won't recognize `new Worker()` as a web worker anymore.

Be aware that this might affect any code that tries to instantiate a normal web worker `Worker` and now instead instantiates a threads.js `Worker`. The threads.js `Worker` is just a web worker with some sugar on top, but that sugar might have unexpected side effects on third-party libraries.

Everything else should work out of the box.

</details>

## Getting Started

### Basics

```js
// master.js
import { spawn, Thread, Worker } from "threads"

const auth = await spawn(new Worker("./workers/auth"))
const hashed = await auth.hashPassword("Super secret password", "1234")

console.log("Hashed password:", hashed)

await Thread.terminate(auth)
```

```js
// workers/auth.js
import sha256 from "js-sha256"
import { expose } from "threads/worker"

expose({
  hashPassword(password, salt) {
    return sha256(password + salt)
  }
})
```

### spawn()

The `hashPassword()` function of the `auth` object in the master code proxies the call to the `hashPassword()` function in the worker:

If the worker's function returns a promise or an observable then you can just use the return value as such in the master code. If the function returns a primitive value, expect the master function to return a promise resolving to that value.

### expose()

Use `expose()` to make a function or an object containing methods callable from the master thread.

In case of exposing an object, `spawn()` will asynchronously return an object exposing all the object's functions. If you `expose()` a function, `spawn` will also return a callable function, not an object.

## Usage

<p>
  Find the full documentation on the <a href="https://threads.js.org/" rel="nofollow">website</a>:
</p>

- [**Quick start**](https://threads.js.org/getting-started)
- [**Basic usage**](https://threads.js.org/usage)
- [**Using observables**](https://threads.js.org/usage-observables)
- [**Thread pools**](https://threads.js.org/usage-pool)
- [**Advanced**](https://threads.js.org/usage-advanced)

## Webpack

Threads.js works with webpack. Usually all you need to do is adding the
[`threads-plugin`](https://github.com/andywer/threads-plugin).

See [Build with webpack](https://threads.js.org/getting-started#build-with-webpack)
on the website for details.

<!--
## API

TODO
-->

## Debug

We are using the [`debug`](https://github.com/visionmedia/debug) package to provide opt-in debug logging. All the package's debug messages have a scope starting with `threads:`, with different sub-scopes:

- `threads:master:messages`
- `threads:master:spawn`
- `threads:master:thread-utils`
- `threads:pool:${poolName || poolID}`

Set it to `DEBUG=threads:*` to enable all the library's debug logging. To run its tests with full debug logging, for instance:

```
DEBUG=threads:* npm test
```

## License

MIT
