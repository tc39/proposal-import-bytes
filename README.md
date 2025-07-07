# Proposal Import Bytes

Champions: Steven ([@styfle](https://github.com/styfle))

Status: Stage 0.

Please leave any feedback you have in the [issues](https://github.com/styfle/proposal-import-bytes/issues)!

## Synopsis

This proposal is buit on top of the the [import attributes proposal](https://github.com/tc39/proposal-import-attributes) to add the ability to import arbitrary bytes in a common way across JavaScript environments.

Developers will then be able to import bytes as follows:

```js
import bytes from "./photo.png" with { type: "bytes" };
import("photo.png", { with: { type: "bytes" } });
```

Note: a similar proposal was mentioned in https://github.com/whatwg/html/issues/9444

## Motivation

In a similar manner to why JSON modules are useful, importing raw bytes is useful to extend this behavior to all files. This provides an isomorphic way to read a file, regardless of the JavaScript environment. 

For example, a developer may want to read a `.png` file to process an image or `.woff` to process a font and pass the bytes into using isomorphic tools like [satori](https://github.com/vercel/satori).

Today, the developer must detect the platform in order to read the bytes.

```js
async function getBytes(path) {
  if (typeof Deno !== "undefined") {
    const bytes = await Deno.readFile(path);
    return bytes;
  }
  if (typeof Bun !== "undefined") {
    const bytes = await Bun.file(path).bytes();
    return bytes;
  }
  if (typeof require !== "undefined") {
    const fs = require("fs/promises");
    const bytes = await fs.readFile(path);
    return bytes;
  }
  if (typeof window !== "undefined") {
    const response = await fetch(path);
    const bytes = await response.bytes();
    return bytes;
  }
  throw new Error("Unsupported runtime");
}

const bytes = await getBytes("./photo.png");
```

We can maximize portability and reduce boilerplate by turning this into a single line of code:

```js
import bytes from "./photo.png" with { type: "bytes" };
```

Using an import also provides opportunity for further optimizations when using a bundler. For example, bundlers can statically analyze this import and inline as base64.

```js
const bytes = Uint8Array.fromBase64("iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkqAcAAIUAgUW0RjgAAAAASUVORK5CYII=")
```

## Proposed semantics and interoperability

If a module import has an attribute with key `type` and value `bytes`, the host is required to either fail the import, or treat it as a Uint8Array. The Uint8Array object is the default export of the module (which has no named exports).

In browser environments, this will be equivalent to `fetch()` such that `sec-fetch-dest` will be empty. The response `content-type` will be ignored.

In "local" desktop/server/embedded, this will be equivalent to a file read. The file extension will be ignored.

All of the import statements in the module graph that address the same module will evaluate to the same mutable Uint8Array object.

## FAQ

### How would this proposal work with caching?

The determination of whether the `type` attribute is part of the module cache key is left up to hosts (as it is for all import attributes).

### Is there any prior art?

Deno 2.4 added support in [July 2025](https://deno.com/blog/v2.4).

```js
import imageBytes from "./image.png" with { type: "bytes" };
```

Bun 1.1.5 added a similar feature in [April 2024](https://bun.sh/blog/bun-v1.1.5).

```js
import html from "./index.html" with { type: "text" };
```

Webpack has [asset modules](https://webpack.js.org/guides/asset-modules/) to inline a data URI via [url-loader](https://www.npmjs.com/package/url-loader) and now `asset/inline`.

```js
import logo from './images/logo.svg';
block.style.background = url(data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDo...vc3ZnPgo=)
```

### What about ArrayBuffer vs Uint8Array?

Both are viable solutions. Uint8Array matches the [Response.bytes()](https://developer.mozilla.org/en-US/docs/Web/API/Response/bytes) method return type as well as [Deno's implementation](https://deno.com/blog/v2.4#importing-text-and-bytes). Uint8Array is also compatible with [Node.js Buffer](https://nodejs.org/api/buffer.html#buffer) which makes it widely compatible with existing JavaScript code.

### What about Blob vs Uint8Array?

Blob is part of the W3C [File API](https://www.w3.org/TR/FileAPI/), not part of JavaScript so it is not a viable solution to include in a TC39 Proposal. Blob also includes a type and is immutable.

### What about mutable vs immutable?

Both are viable solutions. Mutable would match the behavior of existing imports from [roposal-import-attributes](https://github.com/tc39/proposal-import-attributes) but there is still a possibility of making `bytes` default to immutable given the [proposal-immutable-arraybuffer](https://github.com/tc39/proposal-immutable-arraybuffer).

Ideally there would be a separate proposal for a new `immutable` attribute.
