# `Promise.lazy(...)`

Today, Promises in JavaScript are eager. For instance, given the example:

```js
const promise = new Promise((res, rej) => {
  res(123);
});
promise.then(() => {});
```

The callback passed into the Promise constructor is invoked immediately by the Promise constructor itself. For most cases this is just fine. However, it does present a challenge in some edge cases where the work might be relevant only if someone is actually interested in the result. This proposal introduces lazily-evaluated Promises.

```js
let called = false;
let val = 123;

const promise = Promise.lazy((res) => {
  // do something lazily only when the promise is awaited
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

While the `Promise` returned by `Promise.lazy(...)` has no attached reactions, the callback remains pending.

Once a reaction is attached to the `Promise` (using `await`, `then(...)`, `catch(...)`, or `finally(...)`), the callback passed in to `Promise.lazy(...)` is scheduled to run as a microtask. Errors synchronously thrown by the callback are caught and used to reject the `Promise`. The signature of the callback function passed to `Promise.lazy(...)` is identical to that passed into the Promise constructor.

Currently, this model is implementable using a custom thenable:

```js
// Current approach that this proposal is seeking to replace....
class LazyPromise {
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

`Promise.lazy(...)` simplifies this pattern, eliminating boilerplate and using a native Promise rather than a custom thenable.

The reason for scheduling the callback to run as a microtask is to make it possible to construct a complete promise chain prior to actually evaluating the callback:

```js
Promise.lazy((res) => res(123))
   .then(doSomething)
   .catch(handleTheError)
   .finally(doOneLastThing);

// The entire promise chain is created synchronously while the callback is scheduled to run on the
// next microtask drain
```

Like the callback passed to the Promise constructor, the callback is expected to be a regular function that returns nothing. Any return value is ignored.

The arguments passed into the callback are the `resolve()` and `reject()` functions, just as in the Promise construtor.

If the lazy Promise is never awaited, however, the callback is never evaluated.

## `Promise.lazy(...)` and `AsyncContext`

The callback is invoked with the `AsyncContext` that is in scope at the time `Promise.lazy(...)` is called.

```js
const ac = new AsyncContext.Variable();

const p = ac.run(123, () => Promise.lazy((res) => res(ac.get())));

ac.run('abc', () => {
  // The callback is not scheduled to run until then is called, but runs with
  // the context captured when Promise.lazy was called...
  p.then((val) => console.log(val)); // 123
});
```

## `promise.thenLazy(thenHandler[, catchHandler])`
## `promise.catchLazy(catchHandler)`
## `promise.finallyLazy(finallyHandler)`

These variations on `then()`, `catch()`, and `finally()` return Promises whose continuations are lazily evaluated only when they are actually followed.

For example, in the following example, the `then` callback is invoked immediately on the next drain of the microtask queue after the `promise` is resolved:

```js
const p = Promise.resolve(1).then((val) => console.log(val));
```

However, using `thenLazy(...)`, the callback passed into `thenLazy(...)` is not invoked until a continuation is attached to the promise returned.

```js
const p = Promise.resolve(1).thenLazy((val) => console.log(val));

{ drain the microtask queue } ... the then function is not actually invoked yet.

p.then(() => {});  // the continuation is now scheduled to run on the next microtask queue drain.
```

The key idea is to defer the actual invocation of a Promise handler or continutation until there is something explicitly waiting on the result.

## Other name options

* `Promise.lazy()` or `Promise.defer()` etc
* `promise.prototype.thenLazy()` or `promise.prototype.lazyThen(...)` or `promise.prototype.deferredThen(...)`, etc

## Prior art

* https://austingil.com/lazy-promises/
* https://www.npmjs.com/package/p-lazy

## Use case: Task Queue

One use case for lazily evaluated promises is to implement a task queue in which the work to evaluate a promise is not started immediately:

```js
// Set up a task queue of GET requests to perform.
const tasks = [
  Promise.lazy((res) => res(fetch('http://example.com/1'))),
  Promise.lazy((res) => res(fetch('http://example.com/2'))),
  Promise.lazy((res) => res(fetch('http://example.com/3'))),
  Promise.lazy((res) => res(fetch('http://example.com/4'))),
];

// Perform each fetch in sequence
while (tasks.length > 0) {
  await tasks.shift();
}
```

While this specific example is a bit silly on it's own, it's not difficult to imagine a more complex (and useful) implementation.

## Use cases: Examples in the wild

Both of these make use of custom thenables to intentionally defer work until the promise is actually awaited/consumed

* avvio - https://github.com/fastify/avvio/blob/18f924a748df0959824752bba0e584017b62e2d5/lib/thenify.js#L50
* fastify - https://github.com/fastify/fastify/blob/87f9f20687c938828f1138f91682d568d2a31e53/lib/reply.js#L455

