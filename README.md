# Advanced Node
- Node.js is asynchronous

## Asynchronous Patterns

### Callback Pattern
A callback is a block of instruction wrapped in a function to be called when an asynchronous call has completed.

Consider, Continuation Parsing Style (CPS):
(this is **synchronous**)
```javascript
function hideString(str, done) {
    done(str.replace(/[a-zA-Z]/g, 'X'));
}

hideString("Hello World", (hidden) => {
    console.log('hidden');
});

console.log( hidden );
console.log('end');
// XXXXX XXXXX
// end
```
Example callback (i.e. done):
```javascript
function hideString(str, done) {
    process.nextTick(() => {
        done(str.replace(/[a-zA-Z]/g, 'X'));
    });
}

hideString("Hello World", (hidden) => {
    console.log(hidden);
});

console.log('end');
// end
// XXXXX XXXXX
```

Example of sequential execution with callbacks:
(NOTE: this is an anti-pattern called Callback-Hell/Pyramid of doom)
```javascript
function delay(seconds, callback) {
    setTimeout(callback, seconds*1000);
};

console.log('starting delays');
delay(2, () => {
    console.log('2 seconds');
    delay(1, () => {
        console.log('3 seconds');
        delay(1, () => {
            console.log('4 seconds');
        });
    });
});
// starting delays
// 2 seconds
// 3 seconds
// 4 seconds
```

Example of promise:
```javascript
var delay = (seconds) => new Promise((resolves, rejects) => {
    setTimeout(resolves, seconds*1000);
});

delay(1)
    .then(() => console.log('the delay has ended'));
    
console.log('end first tick');
// end first tick
// the delay has ended
```
Same as above:
```javascript
var delay = (seconds) => new Promise((resolves, rejects) => {
  setTimeout(() =>{
      resolves('the long delay has ended');
  }, seconds*1000);
});

delay(1)
    .then((message) => console.log(message));

console.log('end first tick');
// end first tick
// the long delay has ended
```

Same as above:
```javascript
var delay = (seconds) => new Promise((resolves, rejects) => {
  setTimeout(() =>{
      resolves('the long delay has ended');
  }, seconds*1000);
});

delay(1)
    .then(console.log);

console.log('end first tick');
// end first tick
// the long delay has ended
```

Chaining:
```javascript
var delay = (seconds) => new Promise((resolves, rejects) => {
  setTimeout(() =>{
      resolves('the long delay has ended');
  }, seconds*1000);
});

delay(1)
    .then(console.log)
    .then(() => 42)
    .then((number) => console.log('hello world: ${number}'));

console.log('end first tick');
// end first tick
// the long delay has ended
// hello world: 42
```

Handling error in promises:
```javascript
var delay = (seconds) => new Promise((resolves, rejects) => {
    throw new Error('argh');
    setTimeout(() =>{
        resolves('the long delay has ended');
    }, seconds*1000);
});

delay(1)
    .then(console.log)
    .then(() => 42)
    .then((number) => console.log(`hello world: ${number}`))
    .catch((error) => console.log(`error: ${error.message}`));

console.log('end first tick');
// end first tick
// error: argh
```

Rejecting error:
```javascript
var delay = (seconds) => new Promise((resolves, rejects) => {
    if(seconds > 3){
        rejects(new Error(`${seconds} is too long!`))
    }
    setTimeout(() =>{
        resolves('the long delay has ended');
    }, seconds*1000);
});

delay(4)
    .then(console.log)
    .then(() => 42)
    .then((number) => console.log(`hello world: ${number}`))
    .catch((error) => console.log(`error: ${error.message}`));

console.log('end first tick');
// end first tick
// error: 4 is too long!
```

Promisify:\
(allows us to convert callback functions into promises)
```javascript
var { promisify } = require('util');

var delay = (seconds, callback) => {
    if (seconds > 3) {
        callback(new Error(`${seconds} seconds it too long!`));
    } else {
        setTimeout(() =>
            callback(null, `the ${seconds} second delay is over.`),
            seconds
        );
    }
}

var promiseDelay = promisify(delay);

promiseDelay(5)
    .then(console.log)
    .catch((error) => console.log(`error: ${error.message}`));

// error: 5 seconds it too long!
```

Another example of using `promisify`:
```javascript
var fs = require('fs');
var { promisify } = require('util');

var writeFile = promisify(fs.writeFile);
writeFile('sample.txt', 'This is a sample')
    .then(() => console.log('file successfully created'))
    .catch((error) => console.log('error creating file'));
```

Parallel Execution:
```javascript
var fs = require('fs');
var { promisify } = require('util');
var writeFile = promisify(fs.writeFile);
var unlink = promisify(fs.unlink);
var readdir = promisify(fs.readdir);
var beep = () => process.stdout.write("\x07");
var delay = (seconds) => new Promise((resolves) => {
    setTimeout(resolves, seconds*1000);
})
//  Also have access to promise.race
Promise.all([
    delay(5),
    delay(2),
    delay(3),
    delay(5)
]).then(() => readdir(__dirname))
  .then(console.log); 
```

Concurrent Tasks:

```javascript
import logUpdate from 'log-update';
var toX = () => 'X';
var delay = (seconds) => new Promise((resolves) => {
    setTimeout(resolves, seconds*1000);
});

var tasks = [
  delay(4),
  delay(6),
  delay(4),
  delay(3),
  delay(5),
  delay(7),
  delay(9),
  delay(10),
  delay(3),
  delay(5)
];

class PromiseQueue {
    constructor(promises=[], concurrentCount=1) {
        this.concurrent = concurrentCount;
        this.total = promises.length;
        this.todo = promises;
        this.running = [];
        this.complete = [];
    }

    get runAnother() {
        return (this.running.length < this.concurrent) && this.todo.length;
    }

    graphTasks() {
        var { todo, running, complete } = this;
        logUpdate(`
            todo: [${todo.map(toX)}]
            running: [${running.map(toX)}]
            complete: [${complete.map(toX)}]
        `);
    }

    run() {
        while(this.runAnother){
            var promise = this.todo.shift();
            promise.then(() => {
                this.complete.push(this.running.shift());
                this.graphTasks();
                this.run();
            })
            this.running.push(promise);
            this.graphTasks();
        }
    }
}

var delayQueue = new PromiseQueue(tasks, 2);
delayQueue.run();
```

## Streams