

Here are some of the best practices, patterns and anti-patterns that I've come across in my own development as well as some of those encountered frequently by others.

> Promises are about making asynchronous code retain most of the lost properties of synchronous code such as flat indentation and one exception channel.
> 
-- **Petka Antonov** (Bluebird Promise Library)

With this in mind, let's first talk about some common anti-patterns seen in the wild when using Promises.

## Promise Anti-Patterns
These anti-patterns, or bad use cases, can be seen in plenty of code. I've been guilty of some of them in the past, as likely you have too. It's ok, the first goal is to understand the basics of Promises (*which we did in our previous post*); and the second is to just acknowledge what practices don't really fit with the idea and implementation of Promises so we don't perpetuate them in the future.

### #1 Treating Promises as fancy Callbacks
If you're coming from the land of using callbacks, as many of us have, then it's easy to fall back into the routine of just treating Promises like they are glorified callbacks.

For instance, the following code resembles usual callback style code where you would do `somethingAsync(function(err, result))`; so you just go ahead and use the form below taking advantage of `.then()`'s ability to take a success and an error handler.

```javascript
somethingAsync().then(function(result) {
    // handle success
},
function(err) {
    // handle error
});
```
This is just awkward, given that Promises are implemented to allow a single channel for errors, you almost never want to use the `.then(successFn, errorFn)` form.  Rather, it's more idiomatic, and cleaner to express the above to take advantage of the channel that `.then` provides in control flow.

```javascript
somethingAsync()
  .then(function(result) { 
    // handle success 
  })
  .catch(function(err) {
    // handle error
  });
```

Using `.then(successFn, errorFn)` can also lead to other problems, such as when nesting Promises (*which is almost never needed as well*).

```javascript
firstThingAsync().then(function(result1) {
    secondThingAsync().then(function(result2) {
        // do something with result1 and result2
    });
},
function(err) {
   // Errors from secondThingAsync() don't end up here!
});
```
Since you need to do something with the results of both Promises, you decided to use nesting to create a closure around the first results. However, any error or exception occurring in the `secondThingAsync()` call or its own Promise chain won't be caught by your error handler in the second parameter to `firstThingAsync().then()`.  Those errors just get swallowed and never seen.

### #2 Nested Promises
Let's take a look at our previous example again, which has another Promise call nested inside the `.then` handler of a first Promise call due to the dependency between the two.

```javascript
firstThingAsync().then(function(result1) {
  secondThingAsync().then(function(result2) {
    // do something with result1 and result2
  });
},
function(err) {
  // Errors from secondThingAsync() don't end up here!
});
```
The problem here is that we need to do something with the results of both Promises (`firstThingAsync` and `secondThingAsync`).  We can actually resolve this using `Promise.all()`.

```javascript
Promise.all([firstThingAsync, secondThingAsync])
  .then(function(results) {
    // do something with result1 and result2
    // available as results[0] and results[1] respectively
  })
  .catch(function(err) { /* ... */ });
```
`Promise.all()` allows us to pass an array of promises to execute; only when they are all fulfilled does it pass the results on, as an array, to any handler.  And, by moving the error handler out to a final `.catch()` we can now handle any errors from either Promise or the inline expression handler.

But this gets complicated if the second Promise function needs the results of the first Promise as a parameter.  How do we handle that while still retaining a shallow call chain?  Wouldn't we have to nest the Promises again to pass the results of the first to the second? Not necessarily.

```javascript
firstThingAsync()
  .then(function(result1) {
    return Promise.all([Promise.resolve(result1), secondThingAsync(result1)]); 
  })
  .then(function(result1, result2) {
    // do something with result1 and result2
  })
  .catch(function(err){ /* ... */ });
```
In this case we can still use `Promise.all()`; but we use it from a wrapper handler that lets us pass on the first results along with the second Promise that needs to use the first results as a parameter.  Remember, `.then` handlers can return Promises too, not just values.  

### #3 Deferred Anti-Pattern

For those of us familiar with jQuery's implementation of Promises and their Deferred objects, it might be hard to recognize this as an anti-pattern. But, the Promise/A+ spec clearly defines how Promises work and there's really no need to use a Deferred, when you have an actual Promise already.

For example, you are building an API that allows the caller to pass in a function that should run asynchronously and the API function will further handle its results.  We'll use jQuery's deferred in the example.

```javascript
// pseudo API implementation
var url = 'http://www.api.com/v1/widgets';
function apiGetSomething(callerGetFn) {
  callerGetFn(url).then(function(results) {
     // do something further with results
  }
}

// Caller
function getSomethingAsync(url) {
  var deferred = $.Deferred();
  $.getJSON(url).then(function(json) {
    // do some stuff ...
    deferred.resolve(JSON.parse(json));
  });
  return deferred.promise();
}

apiGetSomething(getSomethingAsync);
```
The `deferred` object being created is superfluous here, we can simply use a real Promise and return it.  If using a third-party Promise-like, non-spec compliant library (*like jQuery*) you can wrap or '*promisify*' their API call (*most Promise libraries offer a way to do this as well*).

```javascript
// Wrap the $.getJSON() call to return a real Promise
// properly handling rejection with the available `.fail` method
var fetchJSON = function(url) {  
  return new Promise((resolve, reject) => {
    $.getJSON(url)
      .done((json) => resolve(json))
      .fail((xhr, status, err) => reject(status + err.message));
  });
} 
```

Now, we can simply create and return the Promise in our called function and get rid of the deferred object:

```javascript
// Caller
function getSomethingAsync(url) {
  return fetchJSON(url).then(function(json) {
    return JSON.parse(json);
  });
}

apiGetSomething(getSomethingAsync);
```
There are likely very *rare* occasions where you would need to use a deferred. I can't think of one; but if you can, leave it in the comments for me.  

Even the `delay()` method that wraps `setTimeout` mentioned by Petka Antonov on the [Bluebird anti-patterns wiki](https://github.com/petkaantonov/bluebird/wiki/Promise-anti-patterns#the-thensuccess-fail-anti-pattern) can be done without using a Deferred object.

```javascript
function delay(ms) {  
  return new Promise(function(resolve, reject) {
    setTimeout(resolve, ms);
  });
}
```
But, I agree with Petka in that if you have a third-party library api that has a function that can't be generically wrapped using your Promise library, it is likely an implementation issue and poor design of the API and you should likely report it so they can fix the issue.

## Reminders and Good Ideas
We covered quite a few anti-patterns in ES6 Promises above; and it's time to get cheery again and stop talking about how bad our code has been up until now.

The following items are a set of reminders and best practices to keep in mind when working with Promises.  These are just a handful; but they're the ones I need to recall when writing code, so I'm making a note of them here in the hopes that they help you as well.

### Don't forget to `.catch`
Remember, we want to take advantage of Promise's single flow of data and exceptions; and, ensure we don't let any errors get swallowed or dropped on the floor.

Avoid the `.then(successFn, errorFn)` pattern and keep your Promise chains flat with a trailing `.catch()` to properly handles errors.  If you're over confident of your code's fulfillment, you'll end up missing something. If nothing else, just use a common default `.catch()` so at least they show up in the console.

```javascript
doThing()
  .then(doNextThing)
  .then(doAnotherThing)
  .catch(console.log.bind(console));  // just catch everything here
```

### Avoid side-effects
Don't make assumptions about how long asynchronous code will take within a Promise before fulfilling. Your `.then` handlers should return something specifically.

```javascript
doFirstThingAsync().then(function(result) {
   doSecondThingAsync(result);
})
.then(function() {
  doThirdThingAsync();  // did doSecondThingAsync() resolve?
});
```
The `doSecondThingAsync()` Promise has likely *not* resolved by the time your `.then()` handler executes. You should always return one of the following from your `.then` handlers:

* a new Promise - which would solve our case above
* a synchronous value or `undefined`
* or, throw an Error or Exception

Doing so allows us to correct the previous assumption by returning `doSecondThingAsync()` which itself is a Promise.

```javascript
doFirstThingAsync().then(function(result) {
   return doSecondThingAsync(result);
})
.then(function() {
  doThirdThingAsync();  // doSecondThingAsync has resolved?
});
```

### Don't forget about immediately resolved/rejected Promises

The ES6 Promise spec defines two functions that can be useful sometimes for creating immediately resolved or rejected Promises using a static value: `Promise.resolve()` and `Promise.reject()`.

These can be useful for handling synchronous code that might throw an error that needs to be used in a Promise chain.

```javascript
function makeSyncAsync() {
  return Promise.resolve().then(function(){
    // execute synchronous code that might throw
    return value;
  });
}
```

### Executing Promises in Series
We've seen `Promise.all()` as a way to execute a list of Promises in parallel, waiting for all of them to fulfill before continuing processing.  This is handy if you're building Promises dynamically and don't know exactly how many you might be executing.  But what if we don't want to run in parallel; but instead run in series, each Promise chained to the previous and receiving its results?

```javascript
// Promise returning functions to execute
function doFirstThing(){ return Promise.resolve(1); }
function doSecondThing(res){ return Promise.resolve(res + 1); }
function doThirdThing(res){ return Promise.resolve(res + 2); }
function lastThing(res){ console.log("result:", res); }

var fnlist = [ doFirstThing, doSecondThing, doThirdThing, lastThing];

// Execute a list of Promise return functions in series
function pseries(list) {
  var p = Promise.resolve();
  return list.reduce(function(pacc, fn) {
    return pacc = pacc.then(fn);
  }, p);
}

pseries(fnlist);
// result: 4
```
Note that the functions in our list are essentially "*factories*" that return a Promise.  We don't want to pass an array of Promises directly to our `pseries()` method, because as soon as a Promise is created it begins executing. Had we done that, we couldn't guarantee the execution order in series, as we need to be able to pass a function to the `.then` inside our reduce as we chain each Promise to the previous one. 

Had we passed Promises directly, the results would not be what we expect at all:

```javascript
// list of Promises, not factories now
var fnlist = [ doFirstThing(), doSecondThing(), doThirdThing(), lastThing() ];

pseries(fnlist);
// result: undefined
```

## Summary
We've covered a number of anti-patterns and gotchas when using ES6 Promises above; as well as a number of good practices to keep in mind.  If you have questions regarding any of the details of the Promise spec or the functions above check out the resources listed below.


---
More Promise Patterns & Resources:

* [Promise Anti-Patterns](https://github.com/petkaantonov/bluebird/wiki/Promise-anti-patterns#the-thensuccess-fail-anti-pattern) - Petka Antonov, Bluebird Wiki
* [Promise Anti-Patterns](http://taoofcode.net/promise-anti-patterns/) - Tao of Code
* [We have a problem with Promises](http://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html) - Nolan Lawson
* [My five Promise Patterns](https://remysharp.com/2014/11/19/my-five-promise-patterns) - Remy Sharp
* [Promise Patterns](https://www.promisejs.org/patterns/) - Forbes Lindesay
