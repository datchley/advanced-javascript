### Declaration, Expression, Invoking
I don't want to belabor the basics of function declaration and definition in Javascript; but there are primarily two different ways you'll see functions in Javascript:

* function declarations
* function expressions (named & anonymous)

Functions can be declared using the `function` keyword and a name, as in
```javascript
function compare(a,b) {
  return a == b ? 0 : (a < b) ? -1 : 1;
}
```
This declares a function called `compare`, which is *hoisted* to the top of its enclosing scope and available to any code in that scope or its child scopes.

Functions can also be treated as values, meaning you can assign them to variables and pass and return them to and from functions as well. These are referred to as *first-class functions* and lead to *higher-order functions* when working in a more declarative, functional style of programming.

```javascript
// function expression
var even = function(n){ return n % 2 == 0; };  

// returning a function
var log = function(base) {  // returning a function
  return function(n) {
    return Math.log(n) / Math.log(base);
  }
};
var log10 = log(10);
log10(100);   // 2
```

For a discussion of function `scope` in relation to *IIFE*s and lexical `this`, refer to [Part 1](http://www.datchley.name/basic-scope/) of this series on Scope.

### Function Arguments & Arity
The arity of a function refers to the number of arguments a function expects. This is determined by the number of declared arguments in the function declaration and is available in the function's `.length` property.

```javascript
function foo(a,b) { console.log(a, b); }
foo.length;   // 2
```

A functions arity is, and can be, necessarily different than the arguments it actually receives when called.  If we invoke a function with fewer arguments than it expects, the argument identifiers in the function for arguments not passed will be set to undefined.

```javascript
foo(4, 5);  // => 4, 5
foo(3);     // => 3, undefined
foo();      // => undefined, undefined
```

Within a function we can access all the arguments passed, both those declared in the function declaration and any extras (*you can pass more than the declared arguments to a function*) using the available `arguments` variable.

```javascript
function args(a,b) {
   console.log("a,b: ", a, b);
   console.log("all: ", arguments);
}
foo(1,2);        // a,b: 1 2
                 // all: [1,2]
foo(1,2,3,4);    // a,b: 1 2
                 // all: [1,2,3,4]
```
The `arguments` object is a local variable available within all functions and it is *array-like*.  This means it is not an instanceof the `Array` type and does not have many of it's methods.  `arguments` can be accessed by index, ie `arguments[0]`, `arguments[1]` and it has a `.length` property.

This feature allows Javascript functions to have variable arguments. To work with the arguments as a real array, you can simply convert the arguments object to an array.

```javascript
function has() {
  let args = [].slice.call(arguments);
  args.forEach((arg) => console.log(arg));
}
has(1,2,3,4);   
// 1
// 2 
// 3
// 4
```

### Invoking functions
Invoking a function is done using the `()` operator on the function name or variable holding the function expression.

```javascript
var foo(){ /* do foo */ }   // declaration
foo();   // invoke

(function baz() {           // IIFE
   console.log("baz!");
})();
// "baz!"

var bar = function(){ /* do bar */ };  // expression
bar();   // invoke
```

We can also assign a function to a property on an object and invoke it through the standard object property access method as well.

```javascript
let princess = {
   name: 'Leia Organa',
   say: function(msg) { console.log(this.name + ": " + msg); }
};
princess.say("I love you!");
// => Leia Organa: I love you!
```

In this case, when invoking a function via an object's property it's assigned to, the context of `this` in that function is the object itself.

However, using Javascript's `.call()` and `.apply()`, we can change that context when invoking the function.

```javascript
let name = 'Han Solo';
princess.say.call(this, "I know.");
// => Han Solo: I know.
```

What just happened there?  Why would Han totally underplay that kind of declaration _and_ how'd he steal her line?  The `.call()` method's first argument is an object to use as the invoking context of the function being called.  This allows us to override `princess.name` and use the global `name` variable by passing in `this`, which refers to the `window` object.

`.call()` also lets us pass in as many arguments for that function using the remaining parameters, which is how Han came across as so smug.

The only difference between `.call()` and `.apply()` is that `.apply()` only takes 2 parameters: a context object just like call, and an array of arguments (*instead of passing them as individual parameters*). Even though `.apply()` takes the arguments as an array, they are still passed normally to the function being invoked.

```javascript
function add(a,b) { console.log(a + b); }
add.apply(null, [2,3]);
// 5
```
Why did we just pass `null` as the context object and what exactly did that do? According the the ECMAScript standard, passing in `null` or `undefined` will make the function's lexically scoped `this` point to the global scope.

In most cases with single functions, the lexical context of `this` is probably not a concern, as you probably aren't referencing `this` within the function.  However, when dealing with functions assigned to object properties and invoked through them, understanding how the first argument of `.call()` and `.apply()` affect the function's `this` is important.

Why would we use `.apply()` and not just use `.call()` everywhere?  Let's say you had a function called `after()` that would wrap an existing function and ensure some code was executed every time after that function was called. Using `.call()` would be nearly impossible given that you don't know the number of parameters that function might be called with - without resorting to something potentially dangerous like using `eval()`.

```javascript
function lots(a,b,c,d) {
   console.log([a,b,c,d].join(','));
}
function after(fn) {
  var orig = fn;
  return function() {
    orig.call(null, /* how do you pass them? */);
  };
}
var lots = after(lots);
lots(1,2,3,4);
```
This is where `.apply()` and the `arguments` local variable come to the rescue:

```javascript
function after(fn) {
  var orig = fn;
  return function() {
    var args = [].slice.call(arguments);
    return orig.apply(null, args);
  };
}
var lots = after(lots);
lots(1,2,3,4);
/// 1,2,3,4
```

### Using `.bind()`
So we know now that `.call()` and `.apply()` can be used to not only invoke the given function; but to also change it's calling context and pass parameters as well.  But, those methods directly invoke the function when used. 

Javascript also gives us the `.bind()` method on functions to allow us to *bind* both a context and one or more parameters and *not* invoke the function; but instead return this newly bound function to be called later.

We can use the same example from above, but allow our `after` function to take a second parameter to specify the calling context the function should be executed with.

```javascript
var doctor = {
  name: 'Matt Smith',
  who: function named() {
   console.log(this.name);
  }
};

function after(fn, context) {
  var orig = context ? fn.bind(context) : fn;
  return function() {
    var args = [].slice.call(arguments);
    return orig.apply(null, args);
  };
}

var tenth = { name: 'David Tennant' };
var thedoctor = after(doctor.who, tenth);
doctor.who();  // Matt Smith
thedoctor();   // David Tennant
```

`.bind()` also allows us to pre-bind one or more argument parameters to the function as well.  For instance:

```javascript
function add(a,b) { return a + b; }
var add2 = add.bind(null, 2);
add2(4);   // 6
add2(3,6); // 5
```
Here, we create a new function by *partially applying* the first argument to `add()`.  Passing any subsequent parameters makes no difference.

Keep in mind that `.call()`, `.apply()` and `.bind()` can not be used with ES6's `=>` functions to change the context of `this`, as `this` is explicitly bound to the enclosing scope where the function is declared.  You can use `.apply()` and `.call()` to pass in argument parameters, but the first argument is ignored for changing context.

---
### Exercises


{% exercise %}
Use 'apply()' to call the function `add(x,y)` with the given 'operands' array as the arguments. Then, use 'bind()' to create a partially applied function named 'add4(y)' that takes a number and adds four to it.
{% initial %}
function add(x,y) {
  return x + y;
}

var operands = [5,8],
    add4 = undefined; /* TODO: implement using bind() */
    sum1, sum2;

sum1 = /* TODO: call add(x,y) using apply */;

sum2 = add4(8);

{% solution %}
function add(x,y) {
  return x + y;
}

var operands = [5,8],
    add4 = add.bind(this, 4),
    sum1, sum2;

sum1 = add.apply(this, operands);

sum2 = add4(8);
{% validation %}
assert(sum1 == 13, sum2 == 12);
{% context %}
// This is context code available everywhere
{% endexercise %}