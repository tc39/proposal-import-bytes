# Proposal Import Buffer

Champions: [@styfle](https://github.com/styfle)

Status: Stage 0.

Please leave any feedback in the [issues](https://github.com/styfle/proposal-import-bytes/issues), thanks!

## Synopsis

This proposal is buit on top of [import attributes](https://github.com/tc39/proposal-import-attributes) and [immutable arraybuffer](https://github.com/tc39/proposal-immutable-arraybuffer) to add the ability to import arbitrary bytes in a common way across JavaScript environments.

Developers will then be able to import the buffer as follows:

```js
import buffer from "./photo.png" with { type: "buffer" };
import("photo.png", { with: { type: "buffer" } });
```

Note: a similar proposal was mentioned in https://github.com/whatwg/html/issues/9444

## Motivation

In a similar manner to why JSON modules are useful, importing raw bytes is useful to extend this behavior to all files. This proposal provides an isomorphic way to read a file, regardless of the JavaScript environment. 

For example, a developer may want to read a `.png` file to process an image or `.woff` to process a font and pass the buffer into isomorphic tools like [satori](https://github.com/vercel/satori).

Today, the developer must detect the platform in order to read the buffer.

```js
async function getBuffer(path) {
  if (typeof Deno !== "undefined") {
    const result = await Deno.readFile(path);
    return result.buffer;
  }
  if (typeof Bun !== "undefined") {
    const buffer = await Bun.file(path).arrayBuffer();
    return buffer;
  }
  if (typeof require !== "undefined") {
    const fs = require("fs/promises");
    const result = await fs.readFile(path);
    return result.buffer;
  }
  if (typeof window !== "undefined") {
    const response = await fetch(path);
    const buffer = await response.arrayBuffer();
    return buffer;
  }
  throw new Error("Unsupported runtime");
}

const buffer = await getBuffer("./photo.png");
```

We can maximize portability and reduce boilerplate by turning this into a single line of code:

```js
import buffer from "./photo.png" with { type: "buffer" };
```

Using an import also provides opportunity for further optimizations when using a bundler. For example, bundlers can statically analyze this import and inline as base64.

```js
const buffer = Uint8Array.fromBase64("iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkqAcAAIUAgUW0RjgAAAAASUVORK5CYII=").buffer.transferToImmutable()
```

## Proposed semantics and interoperability

If a module import has an attribute with key `type` and value `buffer`, the host is required to either fail the import, or treat it as an immutable ArrayBuffer. The ArrayBuffer object is the default export of the module (which has no named exports).

In browser environments, this will be equivalent to `fetch()` such that `sec-fetch-dest` will be empty. The response `content-type` will be ignored.

In "local" desktop/server/embedded, this will be equivalent to a file read. The file extension will be ignored.

All of the import statements in the module graph that address the same module will evaluate to the same immutable ArrayBuffer object.

## FAQ

### How would this proposal work with caching?

The determination of whether the `type` attribute is part of the module cache key is left up to hosts (as it is for all import attributes).

### Is there any prior art?

Deno 2.4 added support in [July 2025](https://deno.com/blog/v2.4) to inline a Uint8Array

```js
import imageBytes from "./image.png" with { type: "bytes" };
```

Bun 1.1.7 added a similar feature in [May 2024](https://bun.sh/blog/bun-v1.1.7) to inline a string of text

```js
import html from "./index.html" with { type: "text" };
```

webpack added [asset modules](https://webpack.js.org/guides/asset-modules/) to inline a base64 data URI via [url-loader](https://www.npmjs.com/package/url-loader) in 4.x and now `asset/inline` in 5.x

```js
import logo from "./images/logo.svg"
```

esbuild added a [binary loader](https://esbuild.github.io/content-types/#binary) to inline a Uint8Array

```js
import uint8array from "./example.data"
```

Parcel added a [data url](https://parceljs.org/features/bundle-inlining/#inlining-as-a-data-url) scheme to inline a base64 data URI

```js
import background from "data-url:./background.png";
```

Moddable added a [Resource](https://www.moddable.com/documentation/files/files#resource) class to inline a host buffer for embedded systems

```js
let resource = new Resource("logo.bmp");
```

### Why not mutable?

Mutable can be problematic for several reasons:

- may need multiple copies of the buffer in memory to avoid different underlying bytes for `import(specifier, { type: "json" })` and `import(specifier, { type: "buffer" })`
- may cause unexpected behavior when multiple modules import the same buffer and detach (say `postMessage()` or `transferToImmutable()`) in one module which would cause it to become detached in the other module too
- may cause excessive RAM usage for embedded system (immutable could use ROM instead)
- may cause excessive memory when tracking source maps
- may cause an undeniable global communication channel

See discussion in Issue https://github.com/styfle/proposal-import-bytes/issues/2 and https://github.com/styfle/proposal-import-bytes/issues/5 

### Why not Uint8Array?

Uint8Array is one array-like view of an underlying ArrayBuffer, but there are many views (see TypedArray) because there are many different ways that one might want to access the content of the buffer.

See discussion in Issue https://github.com/styfle/proposal-import-bytes/issues/5

### Why not Blob?

Blob is part of the W3C [File API](https://www.w3.org/TR/FileAPI/), not part of JavaScript so it is not a viable solution to include in a TC39 Proposal.

