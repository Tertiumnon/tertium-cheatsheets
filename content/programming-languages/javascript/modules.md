# JavaScript Modules

## IIFE (Immediately Invoked Function Expression)

```javascript
(function(){
    // ...
})()
```

## UMD (Universal Module Definition)

```javascript
(function (root, factory) {
    if (typeof define === 'function' && define.amd) {
        // AMD
        define(['jquery'], factory);
    } else if (typeof exports === 'object') {
        // Node, CommonJS-like
        module.exports = factory(require('jquery'));
    } else {
        // Browser globals (root is window)
        root.returnExports = factory(root.jQuery);
    }
}(this, function ($) {
    // methods
    function myFunc(){};
    // exposed public method
    return myFunc;
}));
```

## RequireJS / AMD (Asynchronous Module Definition)

```javascript
define(["foo"], function(foo) {
    foo.doStuff();
})
```

```javascript
// add.js
define([] , function () {
    return {
        add: function(a, b) {
        return a + b
    }
});

// numbers.js
define([] , function () {
    return {
        number1: 3,
        number2: 5
});

// whatever.js
define([
    './path/to/add',
    './path/to/numbers'
] , function (math, numbers) {
    var X = math.add(numbers.number1, numbers.number2);
    return X;
});
```

## NodeJS / CommonJS

```javascript
// foo.js
function foo() {};
module.exports = foo;

// index.js
var foo = require('foo');
foo.doStuff();
```

```javascript
// add.js
module.exports = {
    add: function(a,b) {
        return a + b
    }
}

// numbers.js
module.exports = {
    number1: 3,
    number2: 5
};

// whatever.js
var math = require('./path/to/add.js');
var numbers = require('./path/to/numbers.js')

module.exports = {
    X: math.add(numbers.number1, numbers.number2);
};
```

## ES6

```JavaScript
// add.js
export default function add (a,b) {
    return a + b
};

// numbers.js
export const number1 = 3;
export const number2 = 5;

// whatever.js
import add from './path/to/add.js';
import { number1, number2 } from './path/to/numbers.js';

const X = add(number1, number2);

export const X;
```
