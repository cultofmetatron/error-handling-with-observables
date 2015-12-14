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
If apiCall fails, it can recover by making a second attempt. There's a hidden advantage to this approach as well; one that improves on the original try catch semantics.

The Promise is **composable**!

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
        /*
         promise.resolve() to run the delay
         before calling the function a second
         time
        */
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

Plenty of things can be wrapped by Promises such as http calls, fileIO etc. Unfortunately, Promises aren't for everything.  Asynchronous functions that call their callback repeatedly cannot be encapsulated by a Promise. A Promise resolves or rejects once and cannot change state after that.

Repeated calls had to be handled using less composable means of asynchronous event management such as basic node callbacks style or event emitters.

### Promises are one-shot observables

Where a Promise encapulates the lifcycle of a single asynchronous event, An Observable represents a sequence of values that do not yet exist. It helps to think of it as an array of values being pulled from the future.

Rx has some great features for composing Observables. lets say that apiCall now returns an observables.

For now, lets keep it idiomatic to the promise based version.

```javascript
let rx = require('rx');
let { reduce } = require('lodash')
//fn() is assumed to return an Observable
let catcher = functon(timeouts, fn) {
  return function() {
    let args = Array.prototype.slice.call(arguments);
    return reduce(timeouts,
      (observable, timeout) => observable
        .catch((err) => rx.Observable
            .just(null)
            .delay(timeout * 1000)
            .map(() => fn.apply(this, args))
            .concatAll())
      fn.apply(this, args))
    )
  }
}

catcher([1, 1, 2, 3, 5, 8], apiCall)(data)
.subscribe(doSomething, handleError);

```

Observables solve a more general class of problem than Promises. This results in some consequential differences.

Promise.prototype.catch() takes an error handling function. It returns a promise for either the return value of the error handler or if the error handler returns a promise, the eventual value of the returned promise.

Observable.prototype.catch() cannot make such assumptions. It is always assumed to return an observable because observables are sequences. The smallest unit therefore would be a sequence of one value.

In order to run only delays when an error occurs, we start with Observable.just(null) no indicate an observable of just one value, null. We map over it to call the api key. This introduces a complication. Our error handler recovers from an Observable of results to an Observale of an Observable of results.

```

[res1, res2, res3] VS [[res1, res2, res3]]

```

**Observable.prototype.concatAll()** flattens this into an Observable of results. With this returned to catch, on an error, the Observable will fall back to recalling the apiCall function similarly to the promise based version.

### Retrying sequences

Observables handle a much wider array of situations than promises. There are consequently, more ways in which we may need to aggregate and work on errors.

By default, an error interrupts the original observable. Catch handles recovery by switching the source stream to a new Observable. There will be situations where we simply want to recover from errors by skipping the erroneous value and going to the next value in the observable sequence.

In this contrived example, an observable sequence of numbers from 0 to 1000 is run. We map over it; throwing an error if the value is not even. The retry() call means that we ignore the error and continue next
value in the sequence.

```javascript

let rx = require('rx');
let evens = rx.Observable.range(0, 1000)
  .map((val) => (val % 2 === 0) ?
    rx.Observable.just(val) :
    rx.Observable.throw(new Error('not even')))
  .retry()
  .concatAll()

evens.subscribe(onVal, onError);

```

Now lets try something a bit more realistic. say a long polled connection to
get a series of tweets. More importantly, we want to serially retry every 2000 seconds and
ignore errors.


```javascript

let rx = require('rx');
let tweetStream = Observable.interval(2000)
  .map(() => getLatestTweets())
  .concatAll()
  .retry()
  .scan((tweet) => {
    tweets.push(tweet);
    return tweets;
  }, []);

tweetStream.subscribe((tweets) => pushToView(tweets))

```
In very little code, we've created code that calls an api repeatedly every 2 seconds. If it errors,
it simply retries again eventually building up a list of tweets like you'd find in flux or redux.
