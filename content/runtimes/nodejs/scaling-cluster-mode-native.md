# Scaling: Native Cluster Mode

Node.js includes a built-in cluster module that allows execution on multiple processor cores rather than the default single-thread execution.

The module allows you to scale your application by creating worker processes (child processes). With this approach, the application runs its primary process at launch and spawns new worker processes as additional application requests come in.

```js
const express = require("express");
const cluster = require("cluster");
const os = require("os");

// check if the process is the master process
if (cluster.isMaster) {
 // get the number of available CPU cores
 const CPUs = os.cpus().length;
 // fork worker processes for each available CPU core
 for (let i = 0; i < CPUs; i++) {
  cluster.fork();
 }
 // The of the number of cores
 console.log(`Available CPUs: ${CPUs}`);

 cluster.on("online", (worker, code, signal) => {
  console.log(`worker ${worker.process.pid} is online`);
 });
} else {
 const app = express();
 // if the process is a worker process, listen for requests
 app.get("/heavytask", (req, res) => {
  let counter = 0;
  while (counter < 9000000000) {
   counter++;
  } // Log the core that will execute this request
  process.send(`Heavy request ${process.pid}`);
  res.end(`${counter} Iteration request completed`);
 });

 app.get("/ligttask", (req, res) => {
  // Log the core that will execute this request
  process.send(`Light request ${process.pid}`);
  res.send("A simple HTTP request");
 });

 app.listen(3000, () => {
  console.log(`worker process ${process.pid} is listening on port 3000`);
 });
}
```
