
In this section, we're going to focus on the following:

- how Javascript's prototype based objects work
- building up functionality through *pseudo-classical inheritance* via constructor functions and the prototype chain
- and a simpler, more idiomatic way to build up functionality using composition, delegation and mixins.


## What is this `prototype` thing?

When most developers think of Object Oriented (OO) programming, they are usually coming from languages like Java and C++ where the concept of a *class* represents a kind of *blueprint* for a particular type of object's behavior. In those languages, a class is a separate, (*typically*) static representation of how an object should be created; and this is much different than an *instance* of an object created from that class.

Despite what some people might tell you, Javascript does not have classes.  What some developers refer to as a "class", is actually just a function, with other functions assigned to its `.prototype` property; which, in turn, is called with `new` in order to treat the function like a constructor.

```
// Not a 'class', just a function
function Person(name) {
  this.name = name;
}
Person.prototype.hello = function() {
  console.log("Hello, my name is " + this.name);
};

// invoke our function with `new` for magical constructor behavior
var dave = new Person("Dave");
dave.hello();
// "Hello, my name is Dave"
```

The `.prototype` property of `Person` above is an actual property on the `Person` function (*functions are objects*). This property is treated as an object and used to create the actual prototype to which our new object is linked.  

**Note**: In some browsers you can access an objects prototype using `__proto__` property - typically referred to using the `[[Prototype]]` symbol. 

> The actual prototype of an object "*is just another object*" to which it is linked

Using the `.prototype` property of a function we can assign new properties so that when a new object is created from that constructor function, those properties are available on that object's actual prototype.  

To wit, the above code snippet yields the following relationships:

![](js-prototype-example--1-.png)

These prototypes in Javascript are really just a way to compose functionality. They allow an object to *delegate* behavior to its *prototype*, which is simply just another object that it links to via an internal, single path chain.  

This resembles "inheritance" in the sense that you can call a function on your object that isn't directly defined on that object, so long as it is defined on an object that your function links to in the prototype chain. 

You may have heard this referred to as *prototypal inheritance*; but, in reality, this is simply Ojects Linking to Other Objects (*OLOO*), as [Kyle Simpson](http://davidwalsh.name/javascript-objects-deconstruction) correctly termed it. The actual work is being accomplished via:

- **composition** - your object contains another object, called its prototype, and
- **delegation** - when you access a property on your object, if it's not directly defined on it, javascript will search the prototype chain and delegate that behavior to the first object it finds that defines that property.

### `.constructor` 
Just to make OO concepts even muddier in Javascript, we also have the `.constructor` property. This property will reference a function used to create one of the objects in a given object's prototype chain. 

When you call a function with `new` the `.constructor` property is set for you on the returned object's prototype.

In the case of our example above, we can inspect the `.constructor` property and see that it holds a reference to the `Person()` function, which we used to construct the object. 

```
dave.__proto__.constructor; // function Person() 
dave.__proto__.__proto__.constructor; // function Object()
```
![](js-prototype-example--2-.png)

However, this direct relationship doesn't always hold true; especially when we start trying to implement *inheritance* using `new` and assigning prototypes. 

```
function Foo(){}

function Bar(){
    // Call our parent constructor on our object
    Foo.call(this);
}
// "inherit" from Foo's prototype
Bar.prototype = Object.create(Foo.prototype);

var bar = new Bar();

console.log(bar.constructor);
// function Foo() !WOOPS, that's not our constructor
```
By re-assigning the prototype, we end up with the `.constructor` property of the prototype object referencing `Foo()`, not the constructor of our `bar` object, which we expected to be `Bar()`. To fix that, we have to manually reassign the constructor right after setting its prototype.

```
Bar.prototype = Object.create(Foo.prototype);
Bar.prototype.constructor = Bar;
```

### `instanceof`
When using `new` for constructor functions, Javascript allows us to use `instanceof` to check if an object is an instance of a particular constructor function.  More precisely, `instanceof` checks if an object's prototype chain contains an instance of a given constructor's prototype.

For example:
```
function A(){}

function B(){}
B.prototype = Object.create(A.prototype);

var obj = new B();
console.log(obj instanceof B); // true
console.log(obj instanceof A); // true
console.log(obj instanceof Object);  // true
console.log("%o", obj.constructor.prototype); // A {}
```
However, `instanceof` can be problematic in some circumstances and return incorrect results:

- the prototype of the object in question has changed
- you are comparing across frame/window execution contexts (*object from one frame, say an Array, checked in another frame will return false when doing `obj instanceof Array`*)
- you don't have a constructor function to use to introspect the object (*say, a third party library, module or closure*)

For instance:
```
// Related objects whose constructors are out of scope
var a = (function(){
    function A(){};
    return new A();
})();
var b = (function(proto) {
    function B(){};
    B.prototype = proto;
    return new B();
})(a);

// Can we determine if b is related to a?
function F(){}
F.prototype = a;
console.log("Related? ", b instanceof F);
// Related? true
```
In this case, we have two objects whose constructor functions weren't available for comparison using `instanceof`; so we create a throw-away function and assign the base object `a` to it's `.prototype` and then use `instanceof` to determine if `b` is an instance of the function `F()`. Rather, if the prototype of `F()` is anywhere in the prototype chain of `b`.

Clearly the object `b` was not constructed by the function `F`, but we still get a false positive because of how `instanceof` tests for ancestry using the constructor's prototype.

> `obj instanceof F` is semantically misleading. It does not imply that `obj` was created by `F()` or is even remotely related to `F()`; only that the prototype `F` will use when invoked with `new` is somewhere in the prototype chain of `obj`.

## Implementing inheritance in Javascript
So far we've seen that Javascript provides us with a number of "features" that make it *seem* as if it supports thinking about and implementing objects using classical inheritance. But, we've also seen the caveats that surround those features like `instanceof`, `new` to invoke functions as constructors and the `.constructor` and `.prototype` properties. 

To implement classical inheritance, we need to understand the details of how it's implemented behind the scenes to fully appreciate that Javascript isn't really using "classes" or inheritance at all.

Let's give it a go with a simple, albeit contrived, example of inheritance with a `Dog` which inherits from `Animal`.

```
// Our base "class" Animal
function Animal(name) {
    this.me = name;
}
// Some base "class" methods
Animal.prototype.who = function() {
    return "I am " + this.me;
};
Animal.prototype.speak = function() {
    // What should this do? I dunno...
    console.log("(Animal) speak!");
}

// A child "class" that "inherits" from Animal
function Dog(name) {
    Animal.call(this, name);
}
// Ensure we properly inherit from our base class
Dog.prototype = Object.create( Animal.prototype );
Dog.prototype.constructor = Dog;

// Add our own 'speak' method and call our base class 'speak' as well
Dog.prototype.speak = function() {
    Animal.prototype.speak.call(this); 
    console.log("Hello, " + this.who() + "." );
};

// Puppies! Awwwww
var fluffy = new Dog( "Fluffy" );
var spot = new Dog( "Spot" );
```
So, yeah, we now have a couple of talking dogs. Nothing to see here, move along...

Get a cup of coffee (*or your favorite beverage*) and sit down for second. Take some deep breaths; because as straight forward (*not so much*) as the above code looks, the following diagram shows what's actually happening under the hood:

![](classical-inheritance-js-1.png)

Seems like we're going to a lot of effort to treat Javascript like it has classes by 

- using `new` to invoke functions as constructors
- mimicking methods by assigning them to the function's `.prototype` property
- ensuring that we create the proper relationships by setting the correct prototype for child classes and ensuring we call our parent class's constructor in the right context
- and re-wiring the `.constructor` property of our `.prototype` to ensure it points to our actual constructor function

In fact, this all boils down to jumping through hoops to basically ensure our objects properly compose behavior via creating the right linkage to another object via composition in the prototype chain.

I hear you asking, "*There must be a better, more idiomatic, way to do this in Javascript?*" 

Yes, there is.

## Delegation using `Object.create()`
We used `Object.create()` above when creating the function prototypes. `Object.create()` does exactly what we really want - creating a linkage between objects by building an object based on another object.

So, maybe we can just use "plain objects", instead of constructor functions?

```
var foo = { a: 1 };

var bar = Object.create(foo);
bar.b = 2;

var baz = Object.create(bar);
baz.c = 3;

console.log("I can haz 'a' and 'b'? ", !!(baz.a && baz.b));
// true
```
`Object.create()` properly sets up the prototype linkage and gives us a new object; which we can then link to another object.  We don't have to worry about `instanceof` because we're not using functions as constructors; nor do we have to deal with setting a `.prototype` or `.constructor` property correctly.

So let's go back to our `Animal` and `Dog` example and see how we would compose that same functionality using delegation through `Object.create()` instead of constructors. 

We'll use `Object.assign()` to initialize the objects in the our chain. Most browsers don't support `Object.assign()` yet, so you can use this simple polyfill to run the examples:

```
if (!Object.prototype.assign) {
    Object.prototype.assign = function() {
        var args = [].slice.call(arguments),
            target = args.shift();
        
        return args.reduce(function(base, obj) {
            Object.keys(obj).forEach(function(prop) {
                if (obj.hasOwnProperty(prop)) {
                    base[prop] = obj[prop];
                }
            });
            return base;
        }, target);
    }
}
```

The functionality we want to compose using delegation will be similar to our original; but delegation is a fundamentally different approach and pattern than inheritance.

- Delegation builds functionality by composing (*via links*) regular objects, not by using constructor functions.
- Delegation uses more specific method names on the delegator objects that are reflective of the actions they perform. Inheritance keeps method names fairly generic, as sub-classes tend to re-implement specific behavior that override the base classes method. 
- State is typically maintained at the delegator level, not in the delegatee objects.

```
var Animal = {
    who: function() { return this.name; },
    speak: function(s) { console.log(this.who() + ": " + s); }
}

var Dog = Object.create(Animal, {
    bark: {
        value: function() { this.speak("woof!"); }
    }
});

var spot = Object.assign(Object.create(Dog), { name: "Spot" });
var fluffy = Object.assign(Object.create(Dog), { name: "Fluffy" });

spot.bark();
fluffy.bark();
```

That's much less code; and here's the new mental model you'll have to grok.  Much easier!

![](js-object-create-example--2-.png)

We're composing functionality here using `Object.create()`. And the `bark()` method of our `Dog` objects delegates to `Animal.speak()`, available via its prototype chain. 

The approach with the delegation pattern is to use an object as a base "type" that contains common behavior and have other objects link to that object to use that functionality. 

In many respects, what we're trying to do is simply *compose behavior*, not create types, like inheritance does.  The relationship between two objects is explicit via composition using `Object.create()`, rather than implicit via an internal inheritance mechanism.

So, if we're wanting to really just compose behavior; and those behaviors might be common to a number of different objects, there is actually another pattern we can use: **mixins**. 

### Composing Behavior through Mixins
The concept of *mixins* is fairly wide spread and found in a number of languages.  The idea is to factor out common, reusable behavior into their own objects and then integrate those objects into more specific objects that need to use them.

```
// A base object so we can create people
var Person = {
    who: function(){ return this.name; },
    init: function(name) {
        this.name = name;
    }
};

// Factor out common functionality into their own
// objects
var canSpeak = {
    speak: function(s) { console.log(this.who() + ": " + s); }
};
var canWalk = {
    walk: function() { console.log(this.who() + " is walking..."); }
};
var canBuild = {
    tools: ['hammer', 'pliers'],
    use: function(tool) { this.tools.push(tool); },
    build: function(thing) { 
        var withTool = parseInt(Math.round(Math.random() * this.tools.length));
        console.log(this.who() + " is building a " + thing + " using " + this.tools[withTool]);
    }
};

// Can we build it?...
var bob = Object.assign(Object.create(Person), canSpeak, canWalk, canBuild);
bob.init("Bob the Builder");
bob.speak("Hi there!");
bob.walk();
bob.use("stapler");
bob.build("web site");
```
This pattern improves things by 

- further simplifying the mental model used to create objects, 
- improves the readability of the code,
- makes it very easy to isolate functionality in specific objects for better reuse. (*DRY, Separation of Concerns*)  

Here's the inside view of using mixins.

![](js-prototype-example--3-.png)

What an object can do is explicitly stated through the use of mixins. Plus, we can add as many behaviors/mixins as make sense to an object.

We're still using delegation here, via the prototype chain; but mixins give us the ability to inherit from objects in a different way.  Mixins represent *concatenative inheritance*, meaning that instead of delegating behavior to another object, we copy that behavior directly. You can see this in the above diagram, as all the mixins' properties now exist on our new object.

## Summary
There are many libraries available that wrap the implementation of *inheritance* making it easier to program in Javascript if you come from a strict OO background or "class" centric language. But using inheritance and trying to maintain that mental model in Javascript is complex and hard to manage due to various problems.

With the end goal of simply being able to compose functionality, we found that the more idiomatic way to do that in Javascript is through simple objects, the prototype chain, and using new patterns of thinking in our design like *delegation* and *mixins*.

 
