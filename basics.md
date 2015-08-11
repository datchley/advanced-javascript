An exercise...

{% exercise %}
Define the `isEven`, `getSquare` and `addToSum` function expressions below to correctly build the `res` result set and `sum` variables.
{% initial %}
var data = [1,2,3,4,5,6,7,8,9,10],
    isEven = /* TODO: filter function */,
    getSquare = /* TODO: map function */,
    addToSum = /* TODO: reduce function */,
    res, sum;

res = data
        .filter(isEven)
        .map(getSquare);

sum = res.reduce(addToSum, 0);


{% solution %}
var data = [1,2,3,4,5,6,7,8,9,10],
    isEven = function(n){ return n % 2 == 0; },
    getSquare = function(n) { return n*n; },
    addToSum = function(sum, n) { return sum += n; },
    res, sum;

res = data
        .filter(isEven)
        .map(getSquare);

sum = res.reduce(addToSum, 0);
{% validation %}
assert(res.toString() == [4,16,36,64,100], sum == 220);
{% context %}
// This is context code available everywhere
// The user will be able to call magicFunc in his code
function magicFunc() {
    return 3;
}
{% endexercise %}