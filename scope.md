It's common for Javascript developers at all levels to deal with scope and its associated implications.  In this post we'll cover the basics of Javascript scope and some of the issues commonly encountered.

## Lexical Scope
Scope, in general, refers to how the browser's javascript engine looks up identifier names at run time to in order to set how they will be looked up during execution.  That definition implies that there is a *lexing* phase of the engine which is done prior to executing.

Javascript has two lexical scopes: *global* and *function* level. With ES6 there is also *block* level scoping; but more on that later. These lexical scopes are also nested. For instance:

![](js-es5-scope-2.png)

Here we have three separate lexical scopes. The default global scope which contains declarations for `a` and `foo`, the lexical scope declared within the `foo()` function which contains `x`, `b` and `bar`, and the lexical scope declared within `bar()` which contains `y` and `c`.  

Notice that outside of a function, the default scope of declared variables is the global scope (*`window` in the browser*). Also, that scopes can be nested.  Nested scopes have access to the scope's they are declared in, which has two important implications:

* nested scopes can access variables declared in their parent scopes (*there is a scope lookup chain*)
* nested scopes can *shadow* variables declared in their parent scope.

### Closures
The first point allows *closures*, meaning we can define a function which will retain access to it's enclosing scope, even outside of that enclosing scope.
```
var times = function(n) {
    return function(x) {
        return x * n;
    };
}
var times2 = times(2);
console.log(times2(4));
// => 8
```
Here, the function returned by times allows us to create functions that multiply their argument by a given value, `n`. The function returned retains a reference to `n`, even though the lexical scope of that function is not accessible outside of the actual function. `n`, in effect, becomes a *free* variable that is only accessible to the function defined and returned within the scope in which `n` was declared.

### Shadowing
Scope lookup during the lexical phase also stops once it finds the first match. This means you can *shadow* a variable further up the scope chain.

```
var a = 4;
function foo(x) {
   var a = x;  // shadows parent 'a' declaration
   console.log(a);
}
console.log(foo(6));   // => 6
console.log(a);        // => 4
```
Here, the locally declared `a` *shadows* the `a` declared in the parent, global scope.  Without the `var` keyword, we would have been reassigning to the variable `a` in the parent scope.

```
var a = 4;
function foo(x) {
   a = x;    // woops!
   console.log(a);
}
console.log(foo(6));   // => 6
console.log(a);        // => 6
```
### Hoisting
Hoisting also plays a factor with Javascript's scoping. In Javascript, `var` and `function(){}` declarations are hoisted to the top of the current scope; and hence, those identifiers are available to any code in that scope.

```
(function(){
   console.log(inc(4));  // 5
   console.log("a =",a); // a = undefined?

   var a = 1;
   function inc(n){ return ++n; }
})();
```
Uh, oh. So we can clearly access `inc()` which was hoisted and it looks like we can use `a` without a Reference Error; but, `a` is undefined? 

Javascript only hoists the declaration and not the initialized value.  The value will still end up being assigned at the same spot you have the original declaration. The above code actually gets modified and works as follows:

```
(function(){
    var a;
    function inc(n){ return ++n; }
    
    console.log(inc(4));   // 5
    console.log("a = ", a); // a = undefined

    a = 4; // Ah-ha!
})();
```
This is the same for variables assigned function expressions as well. Because function expressions are just values in an assignment statement.

```
(function(){
   console.log(inc(4)); // 5
   console.log(dec(4)); // ReferenceError: dec undefined
   
   var dec = function(n) { return --n; }
   function inc(n){ return ++n; }
})();
```

### ES6 Scoping Additions
With the new ES6 specification, lexical scope has been altered in a handful of ways, primarily with the new `let` and `const` declarations and with the shorter `=>` function syntax.

In ES5, as discussed above, if you wanted to declare a lexically scoped block of code, you could do so by creating a closure, typically with an *immediately invoked function expression* or *IIFE*.  ES5 was essentially lexically scoped to *functions*.

```
var greet = (function() {
   var greeting = "Hello!";
   return function(name) { 
      console.log(greeting + ", " + name + "!");
   };
})();
greet("Cap'n Tight Pants");
// => "Hello, Cap'n Tight Pants!"
```

#### `let`
With ES6's `let` declaration we can now have lexically scope *blocks* of code, without using functions or IIFEs.

```
var a = 1, b = 2, c = 3;
{
  let a = 4, b = 5, c = 6;
  console.log("a,b,c = ", a, b, c);
}
console.log("a,b,c = ", a, b , c);
// => a,b,c = 4 5 6
// => a,b,c = 1 2 3
```

Something you might encounter with `let` declarations also, is that according to the ES6 spec, `let` declarations don't *hoist* like `var` and `function` declarations.

With `let` declarations, you'll get a reference error if you try to access a `let` declared variable *before* its declaration in the code (*according to the ES6 spec*).
```
console.log("a = ", a);
console.log("b = ", b);
var a = 4;
let b = 6;
// => a = undefined
// => b = ReferenceError!
```
However, if you are using Traceur or Babel(6to5), this will not be the case, as transpiling this dead zone behavior would require much more work and nearly reimplementing the Javascript run-time to handle those cases.

#### `const`
`const` is the other new, block scope declaration type, which, as you would suspect creates *constant* variables.

```
const a = 4;
console.log("a = ", a);
a = 3;    // TypeError! a is readonly
```

A `const` declared variable must be initialized with a value when it is declared.  Declaring `const a;` and trying to assign a value to it later on will throw an error.  Also, `const` works by limiting the assignment to a variable, not by freezing a variable's value. So, if you assign a reference type to it, you can still modify the underlying properties of that reference type:
```
const a = { name: 'Mal', ship: 'Serenity' };
a = 4;   // TypeError
a.name = 'Wash'  // no problem!

const b = [1,2,3];
b.unshift(0);  // b = [0,1,2,3]
```
`const` declared reference variables hold a constant reference to that variable, not a constant value - so the underlying reference assigned to a const can change.

### Lexical `this`
We've already encountered the idea that Javascript `var` and `function` declarations are lexically scoped to the enclosing function or scope they are contained in.  But, what about the `this` keyword inside of functions? What does it reference?

Most familiar is the use of `this` in functions used as constructors with the `new` keyword.  When using `new`, `this` refers to the eventual object that will be returned from the function.

```
function Tribble(color) {
  this.color = color;
}
var my_tribble = new Tribble("brown");
my_tribble.color;  // "brown"
```
When calling an method on an object, `this` refers to the object that is the *context* that that function is being called with; in most cases, this is the object prototype or instance the function is assigned to.

```
var tardis = {
  where: "Gallifrey",
  go: function() { 
    console.log("Off to " + this.where + ", Allons-y!"); 
  }
};
tardis.go();  // "Off to Gallifrey, Allons-y!"
```
When calling `tardis.go()`, the `this` references the current context of that function, which is the `tardis` object.  But, even without a context object, `this` still refers to the functions context, which turns out to be `window` for browsers.

```
var where = "Gallifrey";
function go() {
  console.log("Off to " + this.where + ", Allons-y!"); 
}
go();  // "Off to Gallifrey, Allons-y!"
```

`this`, it seems, is fairly flexible and potentially dubious in Javascript depending on what you're trying to do.

#### ES6 `=>` functions
ES6 specifies a short-hand syntax for declarations for functions using the *fat arrow* (`=>`) syntax.  However, `=>` functions bind `this` to their enclosing scope, unlike regular functions which create a new scope entirely.

```
var even = (x) => x % 2 == 0;
console.log(even(2));  // true
console.log(even(3));  // false

var a = 4,
    list = [1,2,3].map((n) => n + a);
// list = [5,6,7]
```

The left side of the `=>` declares the function's arguments and the right side declares the return value, or a block of code as the function body.

```
let evens = [1,2,3,4,5,6].filter((n) => {
  if (even(n)) {
    return n;
  }
});
console.log(evens);  // [2,4,6]
```

There is much more to Javascript scoping than this (*ha, get it?*); and I would suggest you take a look at Kyle Simpson's [You Don't Know JS: Scope & Closures](http://www.amazon.com/You-Dont-Know-JS-Closures/dp/1449335586) and the [ES6 spec](https://people.mozilla.org/~jorendorff/es6-draft.html#sec-declarations-and-the-variable-statement) related to scoping of `let` and `const` as well for a more detailed coverage and understanding.

If you are using [Babel](https://babeljs.io/) or [Traceur](https://github.com/google/traceur-compiler), be sure and reference their documentation for any missing parts or inconsistencies from the ES6 specification as well, which will make it easier when you transition for transpilers to native ES6 in the browser.

