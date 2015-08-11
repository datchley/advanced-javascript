
# Working with Lists of Things

Most javascript programs involve lists, or arrays.  Understanding how to work with lists in a declarative and efficient way is important. 

## Using `.map()`, `.reduce()` and `.filter()`
Making use of Javascript's `map()`, `reduce()` and `filter()` Array methods can help you write more declarative, fluent style code. Rather than building up `for` loops and nesting to process data in lists and collections, you can take advantage of these methods to better organize your logic into building blocks of functions, and then chain them to create more readable and understandable implementations.

Javascript's `Array.prototype` has a number of methods available to make working with lists easier.  The primary one that most developers use is `.forEach()` for looping over items in a list.

Consider the process of adding 1 to each integer in a list:
```
var numbers = [1,2,3,4,5,6,7,8,9,10];  
numbers.forEach((n, index) => {  
   numbers[index] = n + 1;
});
// => [2,3,4,5,6,7,8,9,10,11]
```
Here, we're looping through the array and adding one to each item. This approach also has the side effect that it mutates the original list of data.

As good developers, we want to reduce side effects (*avoid mutating state*) and be more declarative with the code we're writing.  We can do this using `.map()`.

```
var plusone = numbers.map((n) => n+1);  
// => numbers: [1,2,3,4,5,6,7,8,9,10]
// => plusone: [2,3,4,5,6,7,8,9,10,11]
```


