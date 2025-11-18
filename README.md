# TypedArray Concatenation

ECMAScript Proposal for TypedArray concatentation

This proposal is currently [stage 1](https://github.com/tc39/proposals/blob/master/README.md) of the [process](https://tc39.github.io/process-document/).

## Problem

ECMAScript should provide a native method for concatenating TypedArrays that enables implementations to optimize through strategies that can avoid the current requirement of eagerly allocating and copying data into new buffers

It is common for applications on the web (both browser and server side) to need to concatenate two or more TypedArray instances as part of a data pipeline. Unfortunately, the mechanisms available for concatenation are difficult to optimize for performance. All require additional allocations and copying at inopportune times in the application.

A common example is a `WritableStream` instance that collects writes up to a defined threshold before passing those on in a single coalesced chunk. Server-side applications have typically relied on Node.js' `Buffer.concat` API, while browser applications have relied on either browser-compatible polyfills of `Buffer` or `TypedArray.prototype.set`.

```js
let buffers = [];
let size = 0;
new WritableStream({
  write(chunk) {
    buffers.push(chunk);
    size += chunks.length;
    if (buffer.byteLength >= 4096) {
      // Not yet the actual proposed syntax... we have to determine that still
      flushBuffer(concat(buffers, size));
      buffers = [];
      size = 0;
    }
  }
});

function concat(buffers, size) {
  const dest = new Uint8Array(size);
  let offset = 0;
  for (const buffer of buffers) {
    dest.set(buffer, offset);
    offset += buffer.length;
  }
}
```

```js
const buffer1 = Buffer.from('hello');
const buffer2 = Buffer.from('world');
const buffer3 = Buffer.concat([buffer1, buffer2]);
```

While these approaches work, they end up being difficult to optimize because they require potential expensive allocations and data copying at inopportune times while processing the information. The `TypedArray.prototype.set` method does provide an approach for concatenation that is workable, but the way the algorithm is defined, there is no allowance given for implementation-defined optimization.

## Proposal

This proposal seeks to improve the current state by providing a mechanism that provides an optimizable concatenation path for TypedArrays within the language.

As a stage 1 proposal, the exact mechanism has yet to be defined but the goal would be to achieve a model very similar to Node.js' `Buffer.concat`, where multiple input `TypedArray`s can be given and the implementation can determine the most optimum approach to concatenating those into a single returned `TypedArray` of the same type.

```js
const enc = new TextEncoder();
const u8_1 = enc.encode('Hello ');
const u8_2 = enc.encode('World!');
const u8_3 = Uint8Array.concat([u8_1, u8_2]);
```

A key goal, if a reasonable approach to do so is found, would be to afford implementations the ability to determine the most optimal approach, and optimal timing, for performing the allocations and copies, but no specific optimization would be required.

### Differences from `set`

Per the current definition of `TypedArray.prototype.set` in the language specification, the user code is responsible for allocating the destination `TypedArray` in advance along with calculating and updating the offset at which each copied segment should go. Allocations can be expensive and the book keeping can be cumbersome, particularly when the are multiple input `TypedArrays`. The `set` algorithm is also written such that each element of the copied `TypedArray` is copied to the destination one element at a time, with no affordance given to allow the implementation to determine an alternative, more optimal copy strategy.

```js
let buffers = [];
let size = 0;
new WritableStream({
  write(chunk) {
    buffers.push(chunk);
    size += chunks.length;
    if (size >= 4096) {
      // Not yet the actual proposed syntax... we have to determine that still
      flushBuffer(Uint8Array.concat(buffers, size));
      buffers = [];
      size = 0;
    }
  }
});
```
