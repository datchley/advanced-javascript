
## Types and Coercion

Javascript has types.

The types in Javascript are not nearly as strongly defined as in some other languages; but it does have *types*.  The basic types, defined by the ECMA standard are:

* `null`
* `undefined`
* `boolean`
* `number`
* `string`
* `object`
* `symbol` (defined in ES6)

However, in Javascript, variables do not have types, it is the values held in them that have types; and those types are interpreted dynamically by the engine based on context at run time.  All values that fit the above are considered *primitive* types, except for object.

### What type is that?
Determining the type of a value can usually be done using `typeof`. This operator returns seven distinct values based on the given variable's value it is checking. However, these don't line up with the seven basic *types* discussed above:

```javascript
typeof undefined;    // 'undefined'
typeof true;         // 'boolean'
typeof 4;            // 'number'
typeof "foo";        // 'string'
typeof { a: 1 };     // 'object'
typeof Symbol();     // 'symbol'
typeof function(){}; // 'function'
```

Some caveats to look out for are checking for specific types like:

* `NaN` - a special value that is not equal to itself
* `null` - which is falsey and it's type is 'object'
* `null` and `undefined` are equal to each other

Checking for `NaN` can be done in two ways: manually, or with the builtin `isNaN()` function.

```javascript
var a = NaN,
    b = 4;
    
// Builtin function     
isNaN(a);    // true
isNaN(b);    // false

// Custom function 
function _isNaN(o) {
  return o !== o;
}
_isNaN(a);   // true
_isNaN(b);   // false
```

And checking for `null` is just a compound check.

```javascript
// Since null is falsey and it's type is 'object'
function isNull(o) {
  return !o && typeof o === 'object';
}

var a = null, b = undefined, c = 42;
isNull(a);  // true
isNull(b);  // false
isNull(c);  // false
```

Checking for `undefined` is usually easier by checking for `null`, given that the two are equal and there is the rare possibility that the `undefined` value might be redefined.

```
function isUndefined(o) {
  return o == null;
}

var b;
isUndefined(b);  // true

```

### Truthy and Falsey 
There's been argument about whether to use '==' or '===' in Javascript for ages and ages.  Some people say you should always use strict equals '===' and there are those that disagree. I'm one of those that disagree; and it comes down to whether the check or comparison being made is explicit or implicit and if allowing coercion of the value being checked is actually advantageous given the context.  There is no hard and fast rule.

But understanding some of the concepts surrounding coercion involves understanding the nature of truthiness and falsiness in Javascript. This is a deep, deep subject so we'll just try and cover the important points here.

Rather than mention all the possible values that evaluate to true, or are truthy, in Javascript, it's easier to just mention all the falsey values, since there are far fewer:

* `undefined`
* `null`
* `false`
* `+0`, `-0`, and `NaN`
* `""` or `''`

That's it.  All of those values, when coerced to boolean, are false. Everything else is truthy.  Seems pretty straight forward.

### Coercion
Coercion is the process of changing one value's type to another type. This might be done in a check for truthiness of a given value, as we've been discussing above. Or, it might be done in a check for equality between two values.

We mentioned the `==` equals and `===` strict equals operators previously. Let's be sure we understand how they work:

#### `==`
Using `==` is fairly liberal, as the operator will convert one or both of the operand values before comparison. Typically, one or both are converted to a `number`.  You, and Douglas Crockford, might consider this scary; but really, it's not.

Here's the pseudo logic for the `==` operator:

```javascript
x == y;
```
| x | y | result |
| -- | -- | -- |
|…|…|`x` and `y` same type… `true`|
| `null` | `undefined` | `true` |
| `undefined` | `null` | `true` |
| `number` | `string` | `x == toNumber(y)` |
| `string` | `number` | `toNumber(x) == y` |
| `boolean` | (any) | `toNumber(x) == y` |
| (any) | `boolean` | `x == toNumber(y)` |
| `string` or `number` | `object` | `x == toPrimitive(y)` |
| `object` | `string` or `number` | `toPrimitive(x) == y` |
|…|…| otherwise…`false`|


1. if the type of `x` and `y` are the same, go to `===` algorithm.

#### `===`






