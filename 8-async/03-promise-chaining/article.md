
# Promises chaining

Let's formulate the problem mentioned in the chapter <info:callback-hell>:

- We have a sequence of asynchronous tasks to be done one after another. For instance, loading scripts.
- How to code it well?

Promises allow a couple of recipes to do that.

[cut]

In this chapter we cover promise chaining. It looks like this:

```js run
new Promise(function(resolve, reject) {

  setTimeout(() => resolve(1), 1000);

}).then(function(result) {

  alert(result); // 1
  return result * 2;

}).then(function(result) {

  alert(result); // 2
  return result * 2;

}).then(function(result) {

  alert(result); // 4
  return result * 2;

});
```

As we can see:
- A call to `promise.then` returns a promise, that we can use for the next `.then` (chaining).
- A value returned by `.then` handler becomes a result in the next `.then`.

![](promise-then-chain.png)

So in the example above we have a sequence of results: `1` -> `2` -> `4`.

Please note the technically we can also add many `.then` to a single promise, without any chaining, like here:

```js run
let promise = new Promise(function(resolve, reject) {
  setTimeout(() => resolve(1), 1000);
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});
```

...But that's a totally different thing. All `.then` on the same promise get the same result -- the result of that promise:

![](promise-then-many.png)

So in the code above all `alert` show the same: 1.

In practice chaining is used far more often than adding many handlers to the same promise.

## Returning promises

Normally, a value returned by a handler is passed to the next `.then`. But there's an exception. If the returned value is a promise, then further execution is suspended till it settles. And then the result of that promise is given to the next `.then`.

For instance:

```js run
new Promise(function(resolve, reject) {

  setTimeout(() => resolve(1), 1000);

}).then(function(result) {

  alert(result); // 1

*!*
  return new Promise((resolve, reject) => {
    setTimeout(() => resolve(result * 2), 1000);
  });
*/!*

}).then(function(result) {

  alert(result); // 2

*!*
  return new Promise((resolve, reject) => {
    setTimeout(() => resolve(result * 2), 1000);
  });
*/!*

}).then(function(result) {

  alert(result); // 4

});
```

Here each `.then` returns `new Promise(…)`. JavaScript awaits for it to settle and then passes on the result.

So the output is again 1 -> 2 > 4, but with 1 second delay between `alert` calls.

Let's use it with `loadScript` to load scripts one by one, in sequence:

```js run
loadScript("/article/promise-chaining/one.js")
  .then(function(script) {
    return loadScript("/article/promise-chaining/two.js");
  })
  .then(function(script) {
    return loadScript("/article/promise-chaining/three.js");
  })
  .then(function(script) {
    // use variables declared in scripts
    // to show that they indeed loaded
    alert("Done: " + (one + two + three));
  });
```


We can add more asynchronous actions to the chain, and the code is still "flat", no "pyramid of doom".

## Error handling

In case of an error, the closest `onRejected` handler down the chain is called.

Let's recall that a rejection (error) handler may be assigned with two syntaxes:

- `.then(...,onRejected)`, as a second argument of `.then`.
- `.catch(onRejected)`, a shorthand for `.then(null, onRejected)`.

In the example below we use the second syntax to catch all errors in the script load chain:

```js run
function loadScript(src) {  
  return new Promise(function(resolve, reject) {
    let script = document.createElement('script');
    script.src = src;

    script.onload = () => resolve(script);
*!*
    script.onerror = () => reject(new Error("Script load error: " + src)); // (*)
*/!*

    document.head.append(script);
  });
}

*!*
loadScript("/article/promise-chaining/ERROR.js")
*/!*
  .then(function(script) {
    return loadScript("/article/promise-chaining/two.js");
  })
  .then(function(script) {
    return loadScript("/article/promise-chaining/three.js");
  })
  .then(function(script) {
    // use variables declared in scripts
    // to show that they indeed loaded
    alert("Done: " + (one + two + three));
  })
*!*
  .catch(function(error) { // (**)
    alert(error.message);
  });
*/!*
```

In the code above the first `loadScript` call fails, because `ERROR.js` doesn't exist. The initial error is generated in the line `(*)`, then the first error handler in the chain is called, that is `(**)`.

Now the same thing, but the error occurs in the second script:


```js run
loadScript("/article/promise-chaining/one.js")
  .then(function(script) {
*!*
    return loadScript("/article/promise-chaining/ERROR.js");
*/!*
  })
  .then(function(script) {
    return loadScript("/article/promise-chaining/three.js");
  })
  .then(function(script) {
    // use variables declared in scripts
    // to show that they indeed loaded
    alert("Done: " + (one + two + three));
  })
*!*
  .catch(function(error) {
    alert(error.message);
  });
*/!*
```

Once again, the `.catch` handles it.


### Implicit try..catch

Throwing an exception is considered a rejection.

For instance, this code:

```js run
new Promise(function(resolve, reject) {
*!*
  throw new Error("Whoops!");
*/!*
}).catch(function(error) {
  alert(error.message); // Whoops!
});
```

...Works the same way as:

```js run
new Promise(function(resolve, reject) {
*!*
  reject(new Error("Whoops!"));
*/!*  
}).catch(function(error) {
  alert(error.message); // Whoops!
});
```

Like there's an invisible `try..catch` around the whole code of the function, that catches errors.

That works not only in the executor, but in handlers as well, for instance:

```js run
new Promise(function(resolve, reject) {
  resolve("ok")
}).then(function(result) {
*!*
    throw new Error("Whoops!");
*/!*
})
.catch(function(error) {
  alert(error.message); // Whoops!
});
```


## Rethrowing

As we already noticed, `.catch` is like `try..catch`. We may have as many `.then` as we want, and then use a single `.catch` at the end to handle errors in all of them.

In a regular `try..catch` we can analyze the error and maybe rethrow it can't handle. The same thing is possible for promises.

A handler in `.catch` can finish in two ways:

1. It can return a value or don't return anything. Then the execution continues "normally", the next `.then(onResolved)` handler is called.
2. It can throw an error. Then the execution goes the "error" path, and the closest rejection handler is called.

Here is an example of the first behavior (the error is handled):

```js run
// the execution: catch -> then
new Promise(function(resolve, reject) {

  throw new Error("Whoops!");

}).catch(function(error) {

  alert("Handled it!");
*!*
  return "result"; // return, the execution goes the "normal way"
*/!*

*!*
}).then(alert); // result shown
*/!*
```

...And here's an example of "rethrowing":


```js run
// the execution: catch -> catch -> then
new Promise(function(resolve, reject) {

  throw new Error("Whoops!");

}).catch(function(error) {

  alert("Can't handle!");
*!*
  throw error; // throwing this or another error jumps to the next catch
*/!*

}).catch(error => {

  alert("Trying to handle again...");
  // don't return anything => execution goes the normal way

}).then(alert); // undefined
```

## Unhandled rejections

What if we forget to handle an error?

Like here:

```js untrusted run refresh
new Promise(function() {
  errorHappened(); // Error here (no such function)
});
```

Or here:

```js untrusted run refresh
new Promise(function() {
  throw new Error("Whoops!");
}).then(function() {
  // ...something...
}).then(function() {
  // ...something else...
}).then(function() {
  // ...but no catch after it!
});
```

Technically, when an error happens, the promise state becomes "rejected", and the execution should jump to the closest rejection handler. But there is none.

Usually that means that the code is bad. Most JavaScript engines track such situations and generate a global error. In the browser we can catch it using `window.addEventListener('unhandledrejection')` (as specified in the [HTML standard](https://html.spec.whatwg.org/multipage/webappapis.html#unhandled-promise-rejections)):


```js run
// open in a new window to see in action

window.addEventListener('unhandledrejection', function(event) {
  // the event object has two special properties:
  alert(event.promise); // the promise that generated the error
  alert(event.reason); // the error itself (Whoops!)
});

new Promise(function() {
  throw new Error("Whoops!");
}).then(function() {
  // ...something...
}).then(function() {
  // ...something else...
}).then(function() {
  // ...but no catch after it!
});
```

In non-browser environments there's also a similar event, so we can always track unhandled errors in promises.