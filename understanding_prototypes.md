
In this section, we're going to focus on the following:

- how Javascript's prototype based objects work
- building up functionality through *pseudo-classical inheritance* via constructor functions and the prototype chain
- and a simpler, more idiomatic way to build up functionality using composition, delegation and mixins.





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

 
