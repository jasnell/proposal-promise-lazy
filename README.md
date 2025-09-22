# Deferred Promises

Today, Promises in JavaScript are eager. For instance, given the example:

```js
const promise = new Promise((res, rej) => {
  res(123);
});
promise.then(() => {});
```

The callback passed into the Promise constructor is invoked immediately by the Promise constructor itself. For most cases this is just fine. However, it does present a challenge in some edge cases where the work might be relevant only if someone is actually interested in the result or when it is beneficial to defer the work until after a complete promise chain has been set up. This proposal introduces lazily-evaluated Promises.

** Note: The actual syntax `Promise.defer(...)` here is not part of the actual proposal here, it's simply a stand-in to make the examples more concrete **

```js
let called = false;
let val = 123;

// Strawman-syntax...
const promise = Promise.defer((res) => {
  // do something only when the promise is awaited
  called = true;
  console.log(val);  // 'abc'
  res(123);
});

// At this point, the Promise exists but the callback has not yet been evaluated.
val = 'abc';
console.log(called); // false because the promise has not been awaited yet.

const result = await promise;

console.log(result);  // 123
console.log(called);  // true
```

While the `Promise` returned by `Promise.defer(...)` has no attached reactions, the callback remains pending.

Once a reaction is attached to the `Promise` (using `await`, `then(...)`, `catch(...)`, or `finally(...)`), the callback passed in to `Promise.defer(...)` is scheduled to run as a microtask. The signature of the callback function passed to `Promise.defer(...)` is identical to that passed to the Promise constructor.

Currently, this model is implementable using a custom thenable:

```js
// Current approach that this proposal is seeking to replace....
class DeferredPromise {
  #callback = undefined;
  constructor(callback) {
    this.#callback = callback;
  }
  then(resolve, reject) {
    queueMicrotask(() => {
      try {
        this.#callback(resolve, reject);
      } catch (err) {
        reject(err);
      }
    });
  }
}
```

`Promise.defer(...)` simplifies this pattern, eliminating boilerplate and using a native Promise rather than a custom thenable.

The reason for scheduling the callback to run as a microtask is to make it possible to construct a complete promise chain prior to actually evaluating the callback:

```js
Promise.defer((res) => res(123))
   .then(doSomething)
   .catch(handleTheError)
   .finally(doOneLastThing);

// The entire promise chain is created while the callback is scheduled to run on the
// next microtask drain
```

Like the callback passed to the Promise constructor, the callback is expected to be a regular function that returns nothing. Any return value is ignored.

The arguments passed into the callback are the `resolve()` and `reject()` functions, just as in the Promise construtor.

If the deferred Promise is never awaited the callback is never evaluated.

### Problem Statement

The problem statement here is: JavaScript should allow for a first-class lazy Promise variant that defers execution as a queued task only when there are continuations attached.

## `Promise.defer(...)` and `AsyncContext`

The callback is invoked with the `AsyncContext` that is in scope at the time `Promise.defer(...)` is called.

```js
const ac = new AsyncContext.Variable();

const p = ac.run(123, () => Promise.defer((res) => res(ac.get())));

ac.run('abc', () => {
  // The callback is not scheduled to run until then is called, but runs with
  // the context captured when Promise.defer was called...
  p.then((val) => console.log(val)); // 123
});
```

## `promise.deferredThen(thenHandler[, catchHandler])`
## `promise.deferredCatch(catchHandler)`
## `promise.deferredFinally(finallyHandler)`

These variations on `then()`, `catch()`, and `finally()` return Promises whose continuations are evaluated only when they are actually followed.

For example, in the following example, the `then` callback is invoked immediately on the next drain of the microtask queue after the `promise` is resolved:

```js
const p = Promise.resolve(1).then((val) => console.log(val));
```

However, using `deferredThen(...)`, the callback passed into `deferredThen(...)` is not invoked until a continuation is attached to the promise returned.

```js
const p = Promise.resolve(1).deferredThen((val) => console.log(val));

{ drain the microtask queue } ... the then function is not actually invoked yet.

p.then(() => {});  // the continuation is now scheduled to run on the next microtask queue drain.
```

The key idea is to defer the actual invocation of a Promise handler or continutation until there is something explicitly waiting on the result.

## Other name options

* `Promise.lazy()`
* `promise.prototype.thenLazy()` or `promise.prototype.lazyThen(...)`, etc

## Prior art

* https://austingil.com/lazy-promises/
* https://www.npmjs.com/package/p-lazy

## Use case: Task Queue

One use case for deferred promises is to implement a task queue in which the work to evaluate a promise is not started immediately:

```js
// Set up a task queue of GET requests to perform.
const tasks = [
  Promise.defer((res) => res(fetch('http://example.com/1'))),
  Promise.defer((res) => res(fetch('http://example.com/2'))),
  Promise.defer((res) => res(fetch('http://example.com/3'))),
  Promise.defer((res) => res(fetch('http://example.com/4'))),
];

// Perform each fetch in sequence
while (tasks.length > 0) {
  await tasks.shift();
}
```

While this specific example is a bit silly on it's own, it's not difficult to imagine a more complex (and useful) implementation.

## Use case: Avoiding Unhandled Rejection Bookkeeping Overhead

In many many implementations of the Web platform standard `unhandledRejection` event, it is often necessary to implement additional book keeping when a promise rejection is not handled immediately since the mechanisms for reporting unhandled rejections occur synchronously.

* https://chromium.googlesource.com/chromium/src/+/1edacd5d66d5b1642a4c4d9a411717150be8e8e9/third_party/WebKit/Source/bindings/core/v8/RejectedPromises.cpp?pli=1#
* https://github.com/nodejs/node/blob/a8e9f634d3e5b0fb1a5c3f5e22a8193adba30cff/lib/internal/process/promises.js#L439

Because promise rejection events are typically reported synchronously immediately when the promise is rejected, the additional book keeping typically causes the `'unhandledrejection'` event to be deferred until after the current execution context completes just in case the `catch` handler was added immediately after the rejection:

```js
const rejected = Promise.reject(new Error('boom'));
// do other stuff
rejected.catch(() => {}); // handle the promise rejection
```

In this example, the JS engine will typically notify the runtime about the unhandled rejection immediately when it occurs when `Promise.reject(...)` is called, but then will report a *second* event indicating that, nevermind, the promise rejection was handled afterall.

With `Promise.defer(...)` this additional book keeping can be avoided entirely because the rejection will not occur until after the catch handler has a chance to be installed:

```js
const deferred = Promise.defer((res, rej) => rej(new Error('boom')));
// The rejection has not occurred yet, and therefore we can avoid the additional book keeping necessary to track the rejection
deferred.catch(() => {}); // Adding the catch handler causes the deferred promise to be scheduled.
// When the rejection occurs the handler is already attached.
```

Unfortunately hosts cannot completely eliminate the additional book keeping since the current existing Promise handling model remains unchanged, but use of `Promise.defer` by an application can eliminate/reduce the need for the application to use the `unhandledrejection` and `rejectionhandled` events and save the runtime the cost of the additional book keeping in scenarios where it makes sense.

## Use cases: Examples in the wild

Both of these make use of custom thenables to intentionally defer work until the promise is actually awaited/consumed

* avvio - https://github.com/fastify/avvio/blob/18f924a748df0959824752bba0e584017b62e2d5/lib/thenify.js#L50
* fastify - https://github.com/fastify/fastify/blob/87f9f20687c938828f1138f91682d568d2a31e53/lib/reply.js#L455

