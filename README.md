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
/*
Final Result:
todo: []
running: []
complete: [X,X,X,X,X,X,X,X,X,X]
*/
```

## Streams
Streams help to create performant applications.

Readable Streams:
(Below snippet creates a class that can use streams readable to create our own stream)
```javascript
const { Readable } = require('stream');

const peaks = [
    "Tallac",
    "Ralston",
    "Rubicon",
    "Twin Peaks",
    "Castle Peak",
    "Rose",
    "Freel Peak"
];

class StreamFromArray extends Readable {
    constructor(array) {
        super({ objectMode: true });
        this.array = array;
        this.index = 0;
    }

    _read() {
      if(this.index < this.array.length){
          const chunk = {
            data: this.array[this.index],
            index: this.index
          };
          this.push(chunk);
          this.index += 1;
      }
      else {
      this.push(null);
      }
    }
}
const peakStream = new StreamFromArray(peaks);
peakStream.on('data', (chunk) => console.log(chunk));
peakStream.on('end', () => console.log('done'));
```

Working with `fs` read stream:
```javascript
const fs = require('fs');
const readStream = fs.createReadStream('./powder-day.mp4');

readStream.on('data', (chunk) => {
  console.log('reading little chunk\n', chunk.length);
});

readStream.on('end', () => {
  console.log('read stream finished');
});

readStream.on('error', (error) => {
  console.log('an error has occurred');
  console.log(error);
});

readStream.pause();
process.stdin.on('data', (chunk) => {
  if(chunk.toString().trim() === 'finish') {
    readStream.resume();
  }
  readStream.read();
});
```

Making a copy using writable streams:
```javascript
const { createReadStream, createWriteStream } = require('fs');

const readStream = createReadStream('./powder-day.mp4');
const writeStream = createWriteStream('./copy-of-powder-day.mp4');

readStream.on('data', (chunk) => {
  writeStream.write(chunk);
});


readStream.on('error', (error) => {
  console.log('an error has occurred');
  console.log(error);
});

readStream.on('end', () => {
  writeStream.end();
});

writeStream.on('close', () => {
  process.stdout.write('file copied\n');
});
```

Managing `backpressure` and using a custom `highWaterMark`:

```javascript
const { createReadStream, createWriteStream } = require('fs');

const readStream = createReadStream('../../powder-day.mp4');
const writeStream = createWriteStream('./copy.mp4', { 
    highWaterMark: 1628920 
});

readStream.on('data', (chunk) => {
    const result = writeStream.write(chunk);
    if(!result) {
        console.log('backpressure');
        readStream.pause();
    }
});

readStream.on('error', (error) => {
    console.log('an error occurred', error.message);
});

readStream.on('end', () => {
    writeStream.end();
});

writeStream.on('drain', () => {
    console.log('drained');
    readStream.resume();
});

writeStream.on('close', () => {
    process.stdout.write('file copied\n');
});

```

Reducing amount of code with `pipe`:

```javascript
const { createReadStream, createWriteStream } = require('fs');

const readStream = createReadStream('../../powder-day.mp4');
const writeStream = createWriteStream('./copy.mp4', { 
    highWaterMark: 1628920 
});

readStream.pipe(writeStream).on('error', console.error); // automatically handles back pressure

// In terminal -> echo "Hello World" | node .
// or
// cat ./sample.txt | node .
```

Duplex Streams:
(Implements both readable and writable - use to compose streams into complex pipelines)
```javascript
const { PassThrough } = require('stream');
const { createReadStream, createWriteStream } = require('fs');

const readStream = createReadStream('../../powder-day.mp4');
const writeStream = createWriteStream('./copy.mp4');

const report = new PassThrough();

var total = 0;
report.on('data', (chunk) => {
    total += chunk.length;
    console.log('bytes:', total);
})

readStream.pipe(report).pipe(writeStream);
```

Transform Stream:
```javascript
const { Transform } = require('stream');

class ReplaceText extends Transform {
    constructor(char) {
        super();
        this.replaceChar = char;
    }
    _transform(chunk, encoding, callback) {
        const transformChunk = chunk.toString().replace(/[a-z]|[A-Z]|[0-9]/g, this.replaceChar);
        this.push(transformChunk);
        callback();
    }

    _flush(callback) {
        this.push('more stuff is being passed...');
        callback();
    }
}

var xStream = new ReplaceText('x');
process.stdin
    .pipe(xStream)
    .pipe(process.stdout);
```

## HTTP Streaming
Streaming to the browser:
```javascript
const { createServer } = require('http');
const { stat, createReadStream } = require('fs');
const { promisify } = require('util');
const fileName = './powder-day.mp4';
const fileInfo = promisify(stat);

createServer( async (req, res) => {
  const { size } = await fileInfo(fileName);
  res.writeHead(200, {
    'Content-Length': size, 
    'Content-Type': 'video/mp4' });
  createReadStream(fileName).pipe(res);
}).listen(3000, () => console.log('server running - 3000'));
```

Handling range requests:
```javascript
const { createServer } = require('http');
const { stat, createReadStream } = require('fs');
const { promisify } = require('util');
const fileName = './powder-day.mp4';
const fileInfo = promisify(stat);

createServer( async (req, res) => {
  const { size } = await fileInfo(fileName);
  const range = req.headers.range;
  if(range) {
    let = [start, end] = range.replace(/bytes=/, '').split('-');
    start = parseInt(start, 10);
    end = end ? parseInt(end, 10) : size - 1;
    res.writeHead(206, {
      'Content-Range': `bytes ${start}-${end}/${size}`,
      'Accept-Ranges': 'bytes',
      'Content-Length': (end-start) + 1,
      'Content-Type': 'video/mp4'
    })
    createReadStream(fileName, { start, end }).pipe(res);
  } else {
    res.writeHead(200, {
      'Content-Length': size, 
      'Content-Type': 'video/mp4' });
    createReadStream(fileName).pipe(res);
  }
}).listen(3000, () => console.log('server running - 3000'));
```

Forking and uploading streams:
```javascript
const { createServer } = require('http');
const { stat, createReadStream, createWriteStream } = require('fs');
const { promisify } = require('util');
const fileName = './powder-day.mp4';
const fileInfo = promisify(stat);

const respondWithVideo = async (req, res) => {
  const { size } = await fileInfo(fileName);
  const range = req.headers.range;
  if(range) {
    let = [start, end] = range.replace(/bytes=/, '').split('-');
    start = parseInt(start, 10);
    end = end ? parseInt(end, 10) : size - 1;
    res.writeHead(206, {
      'Content-Range': `bytes ${start}-${end}/${size}`,
      'Accept-Ranges': 'bytes',
      'Content-Length': (end-start) + 1,
      'Content-Type': 'video/mp4'
    })
    createReadStream(fileName, { start, end }).pipe(res);
  } else {
    res.writeHead(200, {
      'Content-Length': size, 
      'Content-Type': 'video/mp4' });
    createReadStream(fileName).pipe(res);
  }
}

createServer((req, res) => {
  if (req.method === 'POST') {
    req.pipe(res);
    req.pipe(process.stdout);
    req.pipe(createWriteStream('./upload.file'));
  } else if(req.url === '/video'){
    respondWithVideo(req, res);
  } else {
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(`
      <form enctype="multipart/form-data" method="POST" action="/">
        <input type="file" name="upload-file" />
        <button>Upload file</button>
      </form>
    `);
  }
}).listen(3000, () => console.log('server running - 3000'));
```

Multipart form data:
```javascript
const { createServer } = require('http');
const { stat, createReadStream, createWriteStream } = require('fs');
const { promisify } = require('util');
const multiparty = require('multiparty');
const fileName = './powder-day.mp4';
const fileInfo = promisify(stat);

const respondWithVideo = async (req, res) => {
  const { size } = await fileInfo(fileName);
  const range = req.headers.range;
  if(range) {
    let = [start, end] = range.replace(/bytes=/, '').split('-');
    start = parseInt(start, 10);
    end = end ? parseInt(end, 10) : size - 1;
    res.writeHead(206, {
      'Content-Range': `bytes ${start}-${end}/${size}`,
      'Accept-Ranges': 'bytes',
      'Content-Length': (end-start) + 1,
      'Content-Type': 'video/mp4'
    })
    createReadStream(fileName, { start, end }).pipe(res);
  } else {
    res.writeHead(200, {
      'Content-Length': size, 
      'Content-Type': 'video/mp4' });
    createReadStream(fileName).pipe(res);
  }
};

createServer((req, res) => {
  if (req.method === 'POST') {
    let form = new multiparty.Form();
    form.on('part', (part) => {
      part
        .pipe(createWriteStream(`./${part.filename}`))
        .on('close', () => {
          res.writeHead(200, { 'Content-Type': 'text/html' });
          res.end(`<h1>File uploaded: ${part.filename}</h1>`)
        })
    });
    form.parse(req);
  } else if(req.url === '/video') {
    respondWithVideo(req, res);
  } else {
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(`
      <form enctype="multipart/form-data" method="POST" action="/">
        <input type="file" name="upload-file" />
        <button>Upload file</button>
      </form>
    `);
  }
}).listen(3000, () => console.log('server running - 3000'));
```