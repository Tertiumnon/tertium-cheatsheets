# JavaScript

## Q&A

### Types, Primitives

**Primitive Types:** `number`, `string`, `boolean`, `null`, `undefined`, `symbol`, `bigint`

Primitives are immutable and passed by value. Wrapper objects (String, Number, Boolean) auto-box primitives.

### LexicalEnvironment

Lexical environment defines the scope chain and variable bindings available during execution.

#### Differences between "const" and "function"

- `const` prevents reassignment of the name while function does not.
- An arrow function doesn't have its own `lexical context`, so it won't have a scoped `this` and can't be used as a constructor while function can be.
- A `const` arrow function needs to be declared before calling it, otherwise it's undefined.
- A function can be declared after calling it (hoisting).

### Decorators, Call, Apply

**call():** Invokes function with explicit `this` context and individual arguments
```javascript
function greet(greeting) { return `${greeting} ${this.name}`; }
greet.call({ name: 'Alice' }, 'Hello'); // 'Hello Alice'
```

**apply():** Like `call()` but accepts arguments as an array
```javascript
greet.apply({ name: 'Bob' }, ['Hi']); // 'Hi Bob'
```

**bind():** Returns new function with bound `this` context
```javascript
const greetAlice = greet.bind({ name: 'Alice' }, 'Hey');
greetAlice(); // 'Hey Alice'
```

### Generators, Iterators

**Iterators:** Objects with `next()` method returning `{ value, done }`

**Generators:** Functions that yield values and pause execution
```javascript
function* counter() { yield 1; yield 2; yield 3; }
const gen = counter();
gen.next(); // { value: 1, done: false }
```

### Currying

Transform function with multiple parameters into sequence of functions with single parameter.
```javascript
const add = (a) => (b) => a + b;
add(2)(3); // 5
```

### Event Loop, Stack, Heap

**Stack:** Call stack for synchronous execution (LIFO)
**Heap:** Memory for objects and variables
**Event Loop:** Monitors stack and callback queue, executes queued callbacks when stack is empty

### First-Class Function

Functions are first-class objects: can be assigned to variables, passed as arguments, returned from functions
