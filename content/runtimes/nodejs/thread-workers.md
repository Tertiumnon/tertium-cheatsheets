# Worker Threads

In Node.js, you can explicitly use worker threads for CPU-bound tasks to avoid blocking the single main thread. The worker_threads module allows you to do this.

```js
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
  const worker = new Worker(__filename);
  worker.on('message', (message) => {
    console.log(`Received message from worker: ${message}`);
  });
  worker.postMessage('Hello, worker!');
} else {
  parentPort.on('message', (message) => {
    parentPort.postMessage(`Worker received: ${message}`);
  });
}
```
