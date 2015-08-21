

## What are Promises
Promises, have been around quite a while and are defined by a spec called [Promise/A+]().  ES6 has adopted this spec for its Promise implementation; but there are other Promise libraries out there such as [Q](https://github.com/kriskowal/q), [Bluebird](https://github.com/petkaantonov/bluebird), [RSVP](https://github.com/tildeio/rsvp.js/) and others that adhere to this spec and offer other features on top of it.

Promises give us a way to handle asynchronous processing in a more synchronous fashion. They represent a value that we can handle at some point in the future. And, better than callbacks here, Promises give us guarantees about that future value, specifically:

1. No other registered handlers of that value can change it (*the Promise is immutable*)
2. We are guaranteed to receive the value, regardless of when we register a handler for it, even if it's already resolved (*in contrast to events, which can incur race conditions*).

For example:
```
// an immediately resolved promise
var p2 = Promise.resolve("foo"); 

// can get it after the fact, unlike events
p2.then((res) => console.log(res)); 

var p = new Promise(function(resolve, reject) {
   setTimeout(() => resolve(4), 2000);
});

// handler can't change promise, just value
p.then((res) => {
  res += 2;  
  console.log(res);
});

// still gets 4
p.then((res) => console.log(res));
```

But more than just allowing us to handle future values, they give us a much better way to control asynchronous program flow than plain callbacks.  Plain callbacks don't really let us use our usual language features like `return` and `throw` to handle value and error processing in synchronous flows.

Let's get into the details...

## Creating Promises
The standard way to create a Promise is by using the `new Promise` constructor which accepts a handler that is given two functions as parameters. The first handler (*typically named* `resolve`) is a function to call with the future value when it's ready; and the second handler (*typically named* `reject`) is a function to call to reject the Promise if it can't resolve the future value.

```
var p = new Promise(function(resolve, reject) {
   if (/* condition */) {
      resolve(/* value */);  // fulfilled successfully
   }
   else {
      reject(/* reason */);  // error, rejected
   }
});
```

In this way, a Promise itself has one of the following three states:

1. `Pending` - until a Promise is fulfilled it is in pending state
2. `Fulfilled` - when the first handler is called the Promise is considered fulfilled with the value passed to that handler.
3. `Rejected` - if the second handler is called, the Promise is considered rejected with the value passed to that handler.

A Promise can only be "settled" (meaning it has been *fulfilled* or *rejected*) once.  Other consumers, as we stated previously, can not change the settled value.

You can also create an immediately resolved Promise by using the `Promise.resolve()` method.

```
var p = Promise.resolve(42);
```

## Consuming Promises
Once we have a Promise, it can be passed around as a value.  The Promise is a stand-in for a future value; and so it can be returned from a function, passed as a parameter and used in any other way a standard value would be used.

To consume the Promise - meaning we want to process the Promises value once fulfilled - we attach a handler to the Promise using it's `.then()` method. This method takes a function that will be passed the resolved value of the Promise once it is fulfilled.

```
var p = new Promise((resolve, reject) => resolve(5));
p.then((val) => console.log(val)); // 5
```

A Promise's `.then()` method actually takes two possible parameters. The first is the function to be called when the Promise is fulfilled and the second is a function to be called if the Promise is rejected.

```
p.then((val) => console.log("fulfilled:", val),
       (err) => console.log("rejected: ", err));
```

You can omit either handler in a `.then()`, so sending a `null` as the first handler and providing the second is the same as the standard `Promise.catch()`, which takes a single handler to be called when a promise is rejected.  

The following are equivalent:

```
p.then((val) => console.log("fulfilled:", val))
 .catch((err) => console.log("rejected:", err));

p.then((val) => console.log(fulfilled:", val))
 .then(null, (err) => console.log("rejected:", err));
```

And, as shown above `.then()` calls can be chained.  If a handler returns a value in the a `.then()` call it is automatically wrapped in a Promise when returned and then properly unwrapped to pass the value to further chained `.then()` calls.

## Dealing with Errors
You should use `.catch()` for handling errors, rather than `.then(null, fn)`.  Using `.catch()` is more explicit and idiomatic; and when chaining you can have a single `.catch()` at the end of the chain to handle any rejection or thrown exceptions from either the original promise or any of it's handlers.

Throwing an exception in a Promise will automatically reject that Promise as well.  This is the same for `.then()` handlers and their results and return values as well - a thrown error is wrapped in a Promise and treated as a rejection.  For example:

```
// A Promise that throws, rather than explicitly reject
var p1 = new Promise((resolve, reject) => {
  if (true)  
    throw new Error("rejected!"); // same as rejection
  else
    resolve(4);
});

// trailing .catch() handles rejection
p1.then((val) => val + 2)
 .then((val) => console.log("got", val))
 .catch((err) => console.log("error: ", err.message));
// => error: rejected!

// A fulfilled promise
var p2 = new Promise((resolve, reject) => {
  resolve(4);
});

// Second .then throws error, .catch() still handles it
// as rejection anywhere in the processing chain of promise
p2.then((val) => val + 2)
 .then((val) => { throw new Error("step 2 failed!") }) 
 .then((val) => console.log("got", val))
 .catch((err) => console.log("error: ", err.message));
// => error: step 2 failed!
```

Note that the exception thrown in the second block above (*on Promise `p2`'s chain*), within the handler, does not reject the original promise. So if we had setup another process flow on that same promise, it would continue to work as long as no handler in its chain threw an exception.

```
// Second chain attached to fulfilled Promise
p2.then((val) => val + 4)
 .then((val) => console.log("got:", val))
 .catch((err) => console.log("error:", err.message));

// => got: 8
```

## Composing Promises
Sometimes we're working with multiple Promises and we need to be able to start our processing when all of them are fulfilled. This is where `Promise.all()` comes in.  `Promise.all()` takes an array of Promises and once all of them are fulfilled it fulfills its returned Promise with an array of their fulfilled values.

For example, let's say we have a function wrapper around jQuery's `.getJSON()` method to fetch JSON results from a url which returns a Promise.

```
var fetchJSON = function(url) {
  return new Promise((resolve, reject) => {
    $.getJSON(url)
      .done((json) => resolve(json))
      .fail((xhr, status, err) => reject(status + err.message));
  });
}
```
Now we can setup an array of promises which will fulfill with the JSON results of fetching the response from each of the urls in our `itemUrls` array.  `Promise.all()` will not fulfill until *all* the Promises in the array have fulfilled.  If any of those promises are rejected (*or throw an exception*) then the `Promise.all()` Promise will reject and the `.catch()` below will be triggered.
```
var itemUrls = {
    'http://www.api.com/items/1234',
    'http://www.api.com/items/4567'
  },
  itemPromises = itemUrls.map(fetchJSON);

Promise.all(itemPromises)
  .then(function(results) {
     // we only get here if ALL promises fulfill
     results.forEach(function(item) {
       // process item
     });
  })
  .catch(function(err) {
    // Will catch failure of first failed promise
    console.log("Failed:", err);
  });
```
Keep in mind, a failure (*rejection or thrown exception*) of *any* of the Promises in the array passed to `Promise.all()` will cause the Promise it returns to be rejected.

Sometimes, we don't need to wait on all the Promises in our array; but we simply want to get the results of the first Promise in the array to fulfill.  We can do this with `Promise.race()`, which, like `Promise.all()`, takes an array of promises; but unlike `Promise.all()` will fulfill its returned Promise as soon as the first Promise in that array fulfills. 

As an example, let's say we want to fetch JSON from a url to process. However, we don't want to wait forever if that url is slow to respond to our request. In that case, we just want to use a default value instead.

We can accomplish this as follows:

```
// A Promise that times out after ms milliseconds
function delay(ms) {
  return new Promise((resolve, reject) => {
    setTimeout(resolve, ms);
  });
}

// Which ever Promise fulfills first is the result passed to our handler
Promise.race([
  fetchJSON('http://www.api.com/profile/currentuser'),
  delay(5000).then(() => { user: 'guest' })
])
.then(function(json) {
   // this will be 'guest' if fetch takes longer than 5 sec.
   console.log("user:", json.user);  
})
.catch(function(err) {
  console.log("error:", err);
});
```

With the `delay()` function above, we're simply returning a new Promise that resolves (*fulfills*) with no value, since we just care about the timing, after a given number of milliseconds. We then attach a handler to that returned Promise to just return our default user object.

## ES6 Promises in Summary
These are the basics of the ES6 Promise specification in a nutshell.  

1. Promises give us the ability to write asynchronous code in a synchronous fashion, with flat indentation and a single exception channel.  
1. Promises help us unify asynchronous APIs and allow us to wrap non-spec compliant Promise APIs or callback APIs with real Promises.
1. Promises give us guarantees of no race conditions and immutability of the future value represented by the Promise (unlike callbacks and events).

But, Promises aren't without some drawbacks as well:

1. You can't cancel a Promise, once created it will begin execution. If you don't handle rejections or exceptions, they get swallowed.
1. You can't determine the state of a Promise, ie whether it's pending, fulfilled or rejected. Or even determine where it is in it's processing while in pending state.
1. If you want to use Promises for recurring values or events, there is a better mechanism/pattern for this scenario called *streams*.


