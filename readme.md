# web-imports

> Enables the browser to import from `node_modules`

### How

* Use bare specifiers when importing from `node_modules` in your `src` code.
* The browser receives transformed JS code which has import paths absolutely resolved to file locations on the origin file server.
* The file server serves `node_modules`.
* If the requested file came from under the `/node_modules/` root path, then any import paths in it are also absolutely resolved to the file location.

## Usage

`web-imports` can be used as one of two ways: SSG or SSR.

In both cases, `web-imports` transforms JS files so that any imports with bare specifiers are converted into absolute paths starting with `/node_modules/`. This allows the origin server to easily find the file location during a request, thus the browser can import from `node_modules`, and it allows your `node_modules` dependencies to also have their own `node_modules` dependencies.

For example,

```js
import grainstack from 'grainstack'
```

becomes

```js
import grainstack from '/node_modules/grainstack/esm/index.js'
```

### Static Site Generation

For SSG, run `web-imports` as a CLI tool, and pass it a glob of all the files to transform. 

In the example below, it is assumed that your `build` script has created a folder called `artifact` containing everything that will be uploaded to the static site hosting provider. So there should be a copy of `node_modules` at `artifact/node_modules`. 

```json5
// package.json
{
  "scripts": {
    "predeploy": "web-imports --glob 'artifact/**/*.{js,mjs}'"
  }
}
```

### Server Side Rendering

For SSR, run `web-imports` as a function within your request handler.

The following example uses [`express`](https://www.npmjs.com/package/express):

```js
import {transformImports} from 'web-imports'
import express from 'express'

const app = express()

app.get(/^\/node_modules\/(.*)\/?$/i, async (req, res) => {
  const filepath = req.params["0"]
  if (
    !filepath?.endsWith(".js") &&
    !filepath?.endsWith(".mjs")
  ) {
    res.status(404).send("Has unsupported extension: " + filepath)
    return
  }
  const packagePath = path.resolve(__dirname, `../node_modules/${filepath}`)
  let js = fs.readFileSync(packagePath, { encoding: "utf-8" })

  js = transformImports(js)
  
  res.setHeader("content-type", "application/javascript")
  res.setHeader("x-content-type-options", "nosniff")
  res.send(js)
})

app.listen(3000)
```

## API

### CLI Usage

```
npx web-imports --glob 'artifact/**/*.{js,mjs}' [--prefix '/node_modules/']
```

* `--glob` - Required. A glob selecting all files to transform.
* `--prefix` - Optional. Equal to `/node_modules/` by default.

### Programmatic Usage

```js
import {transformImports} from 'web-imports'
const prefix = '/node_modules/' // optional parameter
js = transformImports(js, prefix)
```

`transformImports` is not expected to throw any errors.

## Other Details

If the package name cannot be found, then the import statement will not be transformed.

Commented import statements will still be transformed if the package name can be found.

In order to have a statically served site support importing from `node_modules`, both your code in `build` and the code in `node_modules` needs to be transformed from bare specifiers into absolute paths starting with `/node_modules/`. These transformations must consider subpath exports and conditional exports using `resolve-pkg-maps`. Statically uploaded code should all have `/node_modules/` prepended.

A different deployment procedure allows for a SSR scheme where instead of prepending `/node_modules/` before uploading, files are transformed just before the server responds to a request. Note that if the server is running both front-end and back-end files, then `node_modules` might contain dependencies for one or the other, or both. But since the transformation happens not on disk, but just before response, there will be no clashing.