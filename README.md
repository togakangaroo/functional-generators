**gimgen** is a javascript micro-library that uses generators to invert program flow and generate coroutines.
This is especially useful when you need to make decisions based on a stream of events that come from multiple sources.

So its kind of like [js-csp](https://github.com/ubolonton/js-csp) but smaller, and focused on control flow rather than on channels.

So its kind of like a limited form of functional reactive programming with [baconjs](https://baconjs.github.io/) but you can use
javascript constructs for control flow

[For examples check out the demo page](https://togakangaroo.github.io/gimgen)

[Interactive slide deck explaining many of the underlying concepts](http://georgemauer.net/gimgen-preso)

# Disclaimer

I do not yet know exactly *what* I have here. It's interesting, it feels very useful, and it allows me to create very complex code that works correctly on the first attempt far more often than it has any right to.

However, I hesitate to promote it for use in production as it simply hasn't been very vetted by very many people yet. Please help and share your opinions on the issue tracker or by contacting me!

# Environment Support

**gimgen** comes with a umd wrapper and is usable with amd, commonjs, or globals. While it is written with es2015, it is transpiled to
regular es5. It should therefore work in [any environment that supports or transpiles to javascript generators](http://kangax.github.io/compat-table/es6/#test-generators).

# Background

The key to understanding how this library works is to [understand javascript generators](http://www.2ality.com/2015/03/es6-generators.html).

While we won't go into this in depth, generators are fundamentally this

* A generator declaration (the `function *` thing) gives you a function that when invoked returns an iterator
* That iterator has `.next()` method.
* Every time `iterator.next()` is invoked, the function executes until it hits a yield keyword. If the yield keyword is passed a value on its right hand side (eg `yield 5`), `iterator.next()` returns that value
* The next time `iterator.next()` is called, the function runs starting with where it left off at the previous yield keyword.

If you've ever [looked into](http://www.2ality.com/2015/03/no-promises.html) how libraries like [bluebird](http://bluebirdjs.com/docs/features.html#async), [co](https://www.npmjs.com/package/co),
or the `await` keyword work, you realize that that all they do is that if `iterator.next()` returns a promise, the library/language will wait
until that promise resolves before invoking `next()` again.

```js
co(function * () {
  consle.log(`starting`)                                              //run this
  const customers = yield $.get(`/customers`)                         //start query, yield promise. co waits for promise to resolve until resuming
  const bestCustomer = customers[0]                                   //this runs after the first next() call
  const details = yield $.get(`/customer/${customers[0].id}/details`) //again start query, yield promise.
  console.log(details)                                                //co runs this after the second next() call
})                                                                    //done - returns a promise that resolves when execution finishes
```

**gimgen** simply builds on this concept by providing some structure around the returned promises. Rather than return a simple promise,
when wrapping a generator with **gimgen** you return *signals* where **a signal is an object with a `createPromise`**
method. Because signals are *promise factories* they can be yielded multiple times (enabling looping) and since they are objects, they
can maintain state and (varying what happens when `createPromise` is called).

This has some very powerful consequences.

# Usage

The core function to start using **gimgen** is

`gimgen(generator) -> function`

This takes a generator and returns a function. Invoking the function will start an instance of the generator and running through its steps. The generator
should yield back signals. So the following will start two coroutines that will each log every 300 and 2000 milliseconds respectively

```js
const { gimgen, timeoutSignal } = window.gimgen

const logTimes = gimgen(function * (name, frequency) {
  const elapsed = timeoutSignal(frequency)
  while(true) {
    yield elapsed
    console.log(`${name}: says hi at ${Date()}`)
  }
})

logTimes("logger one", 300)
logTimes("logger b", 2000)
```

In the case where you would like to start the coroutine immediately, a function is provided that will simply start it running

`runGimgen(generator) -> void`

## Signals

At its heart a signal is a simple object with a `createPromise` method which returns any then-able object.

```js
{ createPromise: () -> Promise }
```

**gimgen** comes bundled with many useful signals. The below are signal factory functions. Invoking these will return a signal object.
This signal object can then be yielded back within your methods. [See demos](https://togakangaroo.github.io/gimgen) for usage examples of the below

* `timeoutSignal(milliseconds)` - emits X milliseconds after being yielded
* `domEventToSignal(domNode, eventName)` - emits the next time the given event occurs after being yielded. The signal has a `getLastEvent()->Event` method which is useful for accessing the dom event object.
* `promiseToSignal(promise)` - when yielded will emit when the wrapped promise becomes resolved or immediately if the promise is already resolved
* `manualSignal()` - returns a signal object with a `.trigger(x)` method. When yielded emits the next time the object's `trigger()` method is invoked. The first parameter to the trigger is returned by the yield.
* `anySignal(...signals)` - when yielded, returns the ifrst of the passed in signals to occur in this structure: `{signal, result}` where `signal` is a reference to the signal that happened, and `result` is its last payload. Useful when one of several things might happen next. (See demos for usages)
* `controlSignal(function * ({emit}))` - Takes a generator that is a **gimgen** coroutine. This will start executing immediately. When yielded, the signal will emit whenever the coroutine invokes the injected `emit` method. This is very useful for aggregating events. (see ping pong demo)

### Creating your own signals

If you would like to create your own signal factories **gimgen** provides a convenience method `createSignalFactory` which takes one of two forms

```js
createSignalFactory(name, ({state, setState}, ...args)-> Promise) ->  (() -> signal)
```

This takes a name (used for a `toString` implementation) and a `createPromise` function. It returns a function that when invoked will create your signal

```js
export const timeoutSignal = createSignalFactory('timeoutSignal', (_, ms) =>
                              new Promise(resolve => setTimeout(resolve, ms) )
                            )
```

The other form is an en extension of this where you pass in a configuration object containing a `createPromise` function

```js
createSignalFactory(name, {
  createPromise: ({state, setState}, ...args)-> Promise,
  getInitialState?: ({}) -> state,
  ...any other functions..
}) ->  (() -> signal)
```

The first parameter into any functions on the configuration object will always be an object with a `state` and `setState` property to read the current state of the signal and to adjust it respectively.
An additional `getInitialState` function can be provided to set the state when a signal instance is created.

Any other functions or fields on the configuration object will be copied to each signal instance with all functions being re-bound to receive the
state object as the first parameter before any others

```js
export const manualSignal = createSignalFactory('manualSignal', {
  getInitialState: () => [],
  createPromise: ({state: toNotify, setState}) =>
    new Promise(resolve => setState([resolve, ...toNotify])),
  trigger: ({state: toNotify, setState}, ...args) => {
    if(args.length > 1)
      setState([])
    toNotify.forEach(fn => fn(...args))
  }
})
```

### `invokableGimgen`

A primary motivation for **gimgen** was as an experiment to find a generator-based way of implementing [common functional helpers](http://underscorejs.org/#functions)
such as `throttle` and `debounce`. In order for this to be possible, we must be able to generate both a signal and a regular javascript function the invoking of
which will cause a signal to emit.

To assist with this we have

`invokableGimgen(({invokedSignal}) => generator) -> function`

`invokeableGimgen` itself returns a function. Unlike other **gimgen** runners, it doesn't take a generator directly but rather a closure that
returns a generator `({invokedSignal}) => generator`. Within the generator you are free to yield `invokedSignal` which will be a signal that emits
whenever the returned function runs.

For sample usages see [function decoration demo](https://togakangaroo.github.io/gimgen/demo/function-decoration.html) and [gimgen.implementations file](https://github.com/togakangaroo/gimgen/blob/master/src/gimgen-implementations.js).
