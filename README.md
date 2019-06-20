CPU Intensive Node.js: Part 1 - https://codeburst.io/cpu-intensive-node-js-part-1-1218b102e5ec
-----------------------------
We explore the issues and solutions around running CPU intensive code in Node.js; in particular in a web server.

Node.js’ single-threaded approach to running user code (the code that you write) can pose a problem when running CPU intensive code.

Problem:
--------
We first illustrate the problem of running CPU intensive code in a typical Node.js web application. In this case, we modify Node.js’ hello world example using the sleep package to simulate a web server where each request takes five seconds running CPU intensive code (complete example available).

index.js

    /* eslint-disable no-console */
    const http = require('http');
    const { sleep } = require('sleep');
    const hostname = '127.0.0.1';
    const port = 3000;
    const server = http.createServer((req, res) => {
      sleep(5); // ARTIFICIAL CPU INTENSIVE
      res.statusCode = 200;
      res.setHeader('Content-Type', 'text/plain');
      res.end('Hello World\n');
    });
    server.listen(port, hostname, () => {
      console.log(`Server running at http://${hostname}:${port}/`);
    });

The reason that this is problematic is that each request blocks the JavaScript event loop for five seconds. For example, say two requests come in roughly at the same time. The first request gets immediately handled and completes in the expected five seconds. The second request, however, gets stuck in the event queue for five seconds and then completes five seconds later (takes ten seconds).

This example experiences performance issues once we than more than one request in a five second period; pretty restrictive.

Worse:
------
To further illustrate the problem of blocking the JavaScript event loop, we can create a second web application example based on the Express hello world example. In this case, we have two endpoints one that is not CPU intensive and one that is (complete example available).

index.js

    /* eslint-disable no-console */
    const express = require('express');
    const { sleep } = require('sleep');
    const app = express();
    app.get('/', (req, res) => res.send('Hello World!'));
    app.get('/intense', (req, res) => {
      sleep(5); // ARTIFICIAL CPU INTENSIVE
      res.send('Hello Intense!');
    });
    app.listen(3000, () => console.log('Example app listening on port 3000!'));
Like the previous example, this example experiences performance issues (for five seconds) once we have a request to the intense endpoint. In this case, all requests to all endpoints are delayed until the CPU intensive code finishes; yuck!

Fork:
----
Similar to other programming languages, Node.js can fork new child processes to handle intensive CPU tasks without blocking the parent process’s event loop. Maybe we can use this approach in rewriting our web application (full example available).

index.js

    /* eslint-disable no-console */
    const express = require('express');
    const { fork } = require('child_process');
    const app = express();
    app.get('/', (req, res) => res.send('Hello World!'));
    app.get('/intense', (req, res) => {
      const worker = fork('./worker');
      worker.on('message', ({ fruit }) => {
        res.send(`Hello Intense ${fruit}!`);
        worker.kill();
      });
      worker.send({ letter: 'a' });
    });
    app.listen(3000, () => console.log('Example app listening on port 3000!'));
    worker.js

    const { sleep } = require('sleep');
    process.on('message', ({ letter }) => {
      sleep(5); // ARTIFICIAL CPU INTENSIVE
      let fruit = null;
      switch (letter) {
        case 'a':
          fruit = 'apple';
          break;
        default:
          fruit = 'unknown';
      }
      process.send({ fruit });
    });
This all looks good until you dig a little deeper.

Note:
It is important to keep in mind that spawned Node.js child processes are independent of the parent with exception of the IPC communication channel that is established between the two. Each process has its own memory, with their own V8 instances. Because of the additional resource allocations required, spawning a large number of child Node.js processes is not recommended.
— Node.js Team

Looking at my OS’s process monitor, I observed that the parent process consumed around 14MiB of memory and each child process (while running) consumed around 7MiB of memory. With my machine showing it has around 1740MiB of free memory, around 250 simultaneous calls to the CPU intensive endpoint would crash it.

note: As comparison, the Go language with goroutines would only uses several KiB per call (thus could handle 250,000 simultaneous calls).

This solution does not scale well for a Node.js web server; yuck.

Next Steps:
----------
There is a common solution to this sort of Node.js problem (web server with CPU intensive endpoints): setting up a queue and a pool of worker processes. In the next part, CPU Intensive Node.js: Part 2 we explore this solution.

CPU Intensive Node.js: Part 2 - https://codeburst.io/cpu-intensive-node-js-part-2-f1610a17f40a
-----------------------------
Using Kue and Redis, we develop and effectively infinitely scalable solution for running CPU intensive code in Node.js; in particular in a web server.
A common solution of the problem of a web server with CPU intensive endpoints is setting up a queue and a pool of worker processes. In particular, we are going to use the library Kue to do this.

Kue is a priority job queue backed by redis, built for node.js.
— Kue Team

First a bit on what and why Redis and what and why Kue.

Clustered Node.js processes do not share a global scope; so in this case the queue data has to be in another process anyway
Redis is a commonly used, feature rich, and highly optimized in-memory data structure store
Kue provides a robust priority queue using the raw key/value structure of Redis.
This article assumes that you have access to a Redis server; in my case, I installed it into my local development environment; didn’t bother installing it into a system directory; just ran it out of the downloaded folder.

Hello
-----
This first example illustrates Kue’s core concepts through a command-line interface application (the full example is available). We start with writing index.js that connects to the queue, creates a job in the queue, waits for the job to complete (or fail), and then exits.

index.js

    /* eslint-disable no-console */
    const kue = require('kue');
    const queue = kue.createQueue();
    console.log('INDEX CONNECTED');
    const job = queue.create('mytype', {
      title: 'mytitle',
      letter: 'a',
    })
      .removeOnComplete(true)
      .save((err) => {
        if (err) {
          console.log('INDEX JOB SAVE FAILED');
          process.exit(0);
          return;
        }
        job.on('complete', (result) => {
          console.log('INDEX JOB COMPLETE');
          console.log(result);
          process.exit(0);
        });
        job.on('failed', (errorMessage) => {
          console.log('INDEX JOB FAILED');
          console.log(errorMessage);
          process.exit(0);
        });
      });
We run index.js. At this point, we can use the following utility provided by Kue to monitor the queue. It shows a single job waiting to be processed.

    node_modules/kue/bin/kue-dashboard -p 3050 -r redis://127.0.0.1:6379

We then run the worker.js. It connects to the queue, processes a job off of it, and then wait for the next job.

worker.js

    /* eslint-disable no-console */
    const kue = require('kue');
    const { sleep } = require('sleep');
    const queue = kue.createQueue();
    console.log('WORKER CONNECTED');
    queue.process('mytype', (job, done) => {
      sleep(5);
      console.log('WORKER JOB COMPLETE');
      switch (job.data.letter) {
        case 'a':
          done(null, 'apple');
          break;
        default:
          done(null, 'unknown');
      }
    });

Hello-Web
---------
In this example, we refactor the earlier web example using Kue (full example available).

index.js

    /* eslint-disable no-console */
    const express = require('express');
    const kue = require('kue');
    const app = express();
    const queue = kue.createQueue();
    app.get('/', (req, res) => res.send('Hello World!'));
    app.get('/intense', (req, res) => {
      const job = queue.create('mytype', {
        title: 'mytitle',
        letter: 'a',
      })
        .removeOnComplete(true)
        .save((err) => {
          if (err) {
            res.send('error');
            return;
          }
          job.on('complete', (result) => {
            res.send(`Hello Intense ${result}`);
          });
          job.on('failed', () => {
            res.send('error');
          });
        });
    });
    app.listen(3000, () => console.log('Example app listening on port 3000!'));

The first observation is that if the job is not processed (to completion or failure) in two minutes (default) Node.js will automatically close the response.

By running worker.js as in the last example, we have a better solution to our CPU intensive problem. With a single worker, this solution can service up to 24 simultaneous calls to the CPU intensive endpoint (120 seconds / 5 seconds) before calls fail; but each successive call takes incrementally longer. Node.js will simply close the responses that take over 120 seconds to complete.

This solution:
--------------
The non-CPU intensive endpoints work without interruption.
The solution is stable (will not crash server) regardless of the number of simultaneous calls.
The CPU intensive endpoints, however, still have scaling issues; effectively fails after about 6 simultaneous connections (people don’t want to wait over 30 seconds for a result).
Scaling

We wrap this series up by thinking about how we can scale up the number of simultaneous calls we can handle.

First, if our worker.js example was not running a CPU intensive process, e.g., was simply hitting (and waiting for) a third-party API (say an email API), we could simply use Kue’s process concurrency feature to enable the single worker to simultaneously handle multiple active jobs.

In our artificial example, however, we have a CPU intensive problem. Because Node.js is single threaded we cannot use the process concurrency feature in this case. Instead we want to spin up many worker processes.

There are several ways of thinking of this:

One is to only use as many workers as you have CPUs; this would have the effect to delivering the first few requests as quickly as possible but delaying everyone else’s.
Second is to use many workers to spread out the load across all requests; this would have the effect to delaying all the requests depending on the load.
In the case of using matching the number of workers to the number of CPUs, you can use the Node.js child processes feature to automate this (full example available). We only need to add a workers.js file and run it instead of worker.js.

workers.js

  const { cpus } = require('os');
  const { fork } = require('child_process');
  const numWorkers = cpus().length;
  for (let i = 0; i < numWorkers; i += 1) {
    fork('./worker');
  }
Another way to think of this problem is to leverage the fact that Redis is accessible over a network. By refactoring the workers.js application to use the network we can effectively infinitely scale the solution, e.g., you can have hundreds of computers running the workers.js application (and thus one worker per CPU).

Wrap Up
-------
While there are some additional documented operational details, e.g., queue maintenance, to create a more reliable solution, we have demonstrated how one can relatively easily scale up CPU intensive Node.js web applications.



