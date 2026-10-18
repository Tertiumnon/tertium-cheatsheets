
# Methods

##

### Methods: process.nextTick(), setImmediate() and setTimeout()

- *process.nextTick()* postpones the execution of action until the next pass around the event loop, or it simply calls the callback function once the event loop's current execution is complete
- *setTimeout()* setTimeout() method schedules code execution after a specified delay, measured in milliseconds
- *setImmediate()* executes a callback on the next cycle of the event loop and returns control to the event loop for any I/O operations. setImmediate() has a higher priority than setTimeout().

### Methods: fork() and spawn()

- *fork()* is a particular case of spawn() that generates a new instance of a V8 engine. Multiple workers run on a single node code base for multiple tasks.
- *spawn()* launches a new process with the available set of commands. This method doesn’t generate a new V8 instance, and only a single copy of the node module is active on the processor.

### Methods: async.queue()

*async.queue()* returns a queue object which is capable of concurrent processing i.e processing multiple items at a single time.

The first parameter is executed on the element (function) added to the queue. The second parameter tells the queue, the number of elements to be processed at a particular time.

### Methods: console.time(), console.timeEnd() and performance.now()

The *console.time* and *console.timeEnd* methods allow you to measure the duration of a block of code. The console.time method is used to start the timer and the console.timeEnd method is used to stop the timer and log the duration to the console.

The *performance.now* method provides a more precise way to measure the duration of async operations. It returns the current timestamp in milliseconds, which can be used to calculate the duration of a task.
