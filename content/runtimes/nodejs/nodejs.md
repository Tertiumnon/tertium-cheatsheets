# NodeJS

## Pros & Cons

Pros:

- Non-blocking operations

Cons:

- Is not suitable for CPU-intensive tasks

## V8

Instead of employing an interpreter, V8 converts JavaScript code into more efficient machine code to increase performance. It turns JavaScript code into machine code during execution by utilizing a JIT (Just-In-Time) compiler.

## API function types

- *Asynchronous*, non-blocking functions
- *Synchronous*, blocking functions

## Q&A

### Event

An event is a notification of something that happened in the system. Node uses EventEmitter class for handling events.

```javascript
const EventEmitter = require('events');
const emitter = new EventEmitter();

emitter.on('data', (message) => console.log(message));
emitter.emit('data', 'Hello');  // Triggers listener
```

### Event-driven architecture

Node.js is built on event-driven, non-blocking I/O. Operations like file reads, network calls don't block execution.

```javascript
// Non-blocking: code continues immediately
fs.readFile('file.txt', (err, data) => {
  if (err) throw err;
  console.log(data);
});
console.log('Reading file...'); // Prints before file is read
```

### Worker processes

Run JavaScript in parallel using Worker Threads or child processes.

```javascript
const { Worker } = require('worker_threads');

const worker = new Worker('./worker.js');
worker.on('message', (result) => console.log(result));
worker.postMessage({ task: 'compute', value: 100 });
```

### Demultiplexer

Also called "reactor pattern." Monitors multiple I/O operations and dispatches events when operations complete.

```javascript
// libuv library handles demultiplexing internally
// File reads, network requests, timers all handled efficiently
```

### Callback Hell

Nested callbacks become difficult to read (Pyramid of Doom).

```javascript
// Bad: Callback Hell
readFile('file.txt', (err, data) => {
  parseData(data, (err, result) => {
    saveResult(result, (err) => {
      console.log('Done');
    });
  });
});
```

### Promise vs Async/Await

Promises and async/await both handle asynchronous operations, but async/await is cleaner.

```javascript
// Promise
function getUser(id) {
  return fetchUser(id)
    .then(user => fetchProfile(user.id))
    .then(profile => console.log(profile))
    .catch(error => console.error(error));
}

// Async/Await (cleaner)
async function getUser(id) {
  try {
    const user = await fetchUser(id);
    const profile = await fetchProfile(user.id);
    console.log(profile);
  } catch (error) {
    console.error(error);
  }
}
```
