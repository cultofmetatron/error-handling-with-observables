# Error Handling With Observables
### A short tutorial on error handling with es6 observables.

#### Why observables?

Javascript's exception handling has never been particularly great. The built in trycatch handler more or less works but it has one flaw. Consider the following.

```javascript
try {
  fn(data, function(err, result) {
    throw err;
  }); //this throws something
} catch(err) {
  console.log(err);
}

```

This code would only work if fn is an asychronous function calling its sb synchrously as well. The trycatch operators can only pick up errors thrown synchronously. The second it becomes async, the error is lost resulting in undefined behavior.

Node's callback style deals with this by passing err or null as the first argument to callbacks to indicate the error. While fast, it forces to step back to dealing with errors explicitly where they first originate. We lose teh ability to have fine grained control over where we handle errors. We also end up with more boilerplate code as a result.

One of the great parts of Promises is the ability to capture errors and propagate them through the promise chain.

Lets say the function apiCall takes some data and returns a promise. However, the function calls an external service and has a 10% chance of failure. Promises let us catch() the error and handle it in a manner that looks synchronous.

```javascript

apiCall(data)
  .catch((err) => apiCall(data))
  .then((data) => something(data));

```
If apiCall fails, it can recover by making a second attempt. There's a hidden advantage to this approach as well; one that improves on the original try catch semantics. The Promise is composable!

The function catcher, takes a set of timeouts and a function that returns a promise. It then returns a function that calls the argument function such that for each element in timeouts, it will
retry repeatedly after a delay finally letting the error through when all retries are exhausted.

```javascript
let { reduce } = require('lodash');
let Promise    = require('bluebird');
let catcher = function(timeouts, fn) {
  return function() {
    let args = Array.prototype.slice.call(arguments);
    return reduce(timeouts,
      (memo, timeout) => memo
        /* promise.resolve() to run the delay before calling
         the function a second time */
        .catch((err) => Promise
          .resolve(null)
          .delay(timeout * 1000) //convert seconds to milisec
          .then(() => fn.apply(this, arguments))),
      fn.apply(this, args));
  }
};

//will attempt up to 6 times with timeouts.
catcher([1, 1, 2, 3, 5], apiCall)(data)
  .then(doSomething)
  .catch(handleError);

```
