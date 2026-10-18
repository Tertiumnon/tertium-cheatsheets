# JavaScript Patterns

## Singleton

### Singleton (ES5)

```javascript
var Singleton = function() {
  function publicMethod() {
    console.log('publicMethod');
  }

  return { publicMethod: publicMethod }
}();

// fact checking
Singleton.publicMethod();
```

### Singleton (ES6)

```javascript
/* singleton.js */

const Singleton = {
  publicMethod: () => console.log('Public Method')
}

Object.freeze(Singleton);
export default Singleton;

/* app.js */

// fact checking
let s1 = new Singleton();
console.log(s1);
let s2 = new Singleton();
console.log(s2);
```
