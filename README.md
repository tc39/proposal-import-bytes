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
