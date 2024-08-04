---
author: Saurabh Dalakoti
pubDatetime: 2024-08-03T12:47:24Z
modDatetime: 2024-08-04T08:57:04Z
title: Mighty Node JS
featured: false
draft: false
tags:
  - Engineering
  - backend
description: Node JS is great for IO Intesive applications
---

Have heard a lot, use JS/ node-js to create blazing fast server as its good for IO Intensive application. What is this IO intensive??

# TLDR

To see where JS beats Python and where Python beats JS

# The mighty Node Js

As per the [doc](https://nodejs.org/en/about) node is -

> As an asynchronous event-driven JavaScript runtime, Node.js is designed to build scalable network applications. In the following "hello world" example, many connections can be handled concurrently. Upon each connection, the callback is fired, but if there is no work to be done, Node.js will sleep.

> This is in contrast to today's more common concurrency model, in which OS threads are employed. Thread-based networking is relatively inefficient and very difficult to use. Furthermore, users of Node.js are free from worries of dead-locking the process, since there are no locks. Almost no function in Node.js directly performs I/O, so the process never blocks except when the I/O is performed using synchronous methods of Node.js standard library. Because nothing blocks, scalable systems are very reasonable to develop in Node.js.

The above para mentioned JAVA language, where a thread is employed to cater a simple API call.

# Async programming

Asynchronous programming allows a program to perform tasks concurrently without blocking the execution of other code. Instead of waiting for a task to complete (such as reading from a file or making a network request), asynchronous programming allows the program to initiate the task and continue executing other code. When the task completes, a callback function or promise is used to handle the result. This approach improves the efficiency and responsiveness of applications by enabling them to manage multiple tasks simultaneously, without being stalled by operations that take time to complete. By leveraging async constructs like promises, async/await syntax, and event-driven models, asynchronous programming ensures that applications can handle I/O operations, user interactions, and other time-consuming tasks effectively, maintaining smooth performance and responsiveness.

> This above thing runs in a cycle so fast that it's impossible to notice. We think our computers run many programs simultaneously, but this is an illusion (except on multiprocessor machines).

Normally, programming languages are synchronous and some provide a way to manage asynchronicity in the language or through libraries. C, Java, C#, PHP, Go, Ruby, Swift, and Python are all synchronous by default. Some of them handle async operations by using threads, spawning a new process.

## Js is single threaded

JavaScript is **synchronous** by default and is single threaded. This means that code cannot create new threads and run in parallel.

But JavaScript was born inside the browser, its main job, in the beginning, was to respond to user actions, like `onClick`, `onMouseOver`, `onChange`, `onSubmit` and so on. How could it do this with a synchronous programming model?

The answer was in its environment. The **browser** provides a way to do it by providing a set of APIs that can handle this kind of functionality.

More recently, Node.js introduced a non-blocking I/O environment to extend this concept to file access, network calls and so on.

## Callbacks

You can't know when a user is going to click a button. So, you **define an event handler for the click event**. This event handler accepts a function, which will be called when the event is triggered:

```js
document.getElementById("button").addEventListener("click", () => {
  // item clicked
});
```

This is the so-called **callback**.

A callback is a simple function that's passed as a value to another function, and will only be executed when the event happens. We can do this because JavaScript has first-class functions, which can be assigned to variables and passed around to other functions (called **higher-order functions**)

> But everything is not rainbows in callback world, there is something call callback hell, which was solved by async-await construct in JS

# Overview of blocking vs non blocking

> "I/O" refers primarily to interaction with the system's disk and network supported by [libuv](https://libuv.org/).

## Blocking

**Blocking** is when the execution of additional JavaScript in the Node.js process must wait until a non-JavaScript operation completes. This happens because the event loop is unable to continue running JavaScript while a **blocking** operation is occurring.

In Node.js, JavaScript that exhibits poor performance due to being CPU intensive rather than waiting on a non-JavaScript operation, such as I/O, isn't typically referred to as **blocking**. Synchronous methods in the Node.js standard library that use libuv are the most commonly used **blocking** operations. Native modules may also have **blocking** methods.

All of the I/O methods in the Node.js standard library provide asynchronous versions, which are **non-blocking**, and accept callback functions. Some methods also have **blocking** counterparts, which have names that end with `Sync`.

## Comparing code

```js
const fs = require("node:fs");

const data = fs.readFileSync("/file.md"); // blocks here until file is read
console.log(data);
moreWork(); // will run after console.log
```

```js
const fs = require("node:fs");

fs.readFile("/file.md", (err, data) => {
  if (err) throw err;
  console.log(data);
});
moreWork(); // will run before console.log
```

In the first example above, `console.log` will be called before `moreWork()`. In the second example `fs.readFile()` is **non-blocking** so JavaScript execution can continue and `moreWork()` will be called first. The ability to run `moreWork()` without waiting for the file read to complete is a key design choice that allows for higher throughput.

## Concurrency and throughput

JavaScript execution in Node.js is single threaded, so concurrency refers to the event loop's capacity to execute JavaScript callback functions after completing other work. Any code that is expected to run in a concurrent manner must allow the event loop to continue running as non-JavaScript operations, like I/O, are occurring.

As an example, let's consider a case where each request to a web server takes 50ms to complete and 45ms of that 50ms is database I/O that can be done asynchronously. Choosing **non-blocking** asynchronous operations frees up that 45ms per request to handle other requests. This is a significant difference in capacity just by choosing to use **non-blocking** methods instead of **blocking** methods.

> In above example now, this 45ms is offload off the event loop and JS runtime, and delegated to libUV, and event loop is free for this 45ms to cater more incoming requests. `libuv` manages these operations in the background using its own thread pool and system-level mechanisms.

[LibUv and node js story](https://codedamn.com/news/nodejs/libuv-architecture)

# Back to Nodejs and python benchmarks

## Simple Hello world Server

A simple hello world server on python and node JS

hey command for benchmarking
` hey -n 1000 -c 100 http://localhost:3001/data`

Summary

```txt
Summary:
  Total:	1.1748 secs
  Slowest:	0.1351 secs
  Fastest:	0.1029 secs
  Average:	0.1153 secs
  Requests/sec:	851.2449

  Total data:	13000 bytes
  Size/request:	13 bytes

Response time histogram:
  0.103 [1]	    |
  0.106 [93]	|■■■■■■■■■■■■■■■■■
  0.109 [143]	|■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.113 [206]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.116 [222]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.119 [96]	|■■■■■■■■■■■■■■■■■
  0.122 [67]	|■■■■■■■■■■■■
  0.125 [5]	    |■
  0.129 [37]	|■■■■■■■
  0.132 [50]	|■■■■■■■■■
  0.135 [80]	|■■■■■■■■■■■■■■


Latency distribution:
  10% in 0.1064 secs
  25% in 0.1100 secs
  50% in 0.1131 secs
  75% in 0.1178 secs
  90% in 0.1307 secs
  95% in 0.1330 secs
  99% in 0.1347 secs

Details (average, fastest, slowest):
  DNS+dialup:	0.0005 secs, 0.1029 secs, 0.1351 secs
  DNS-lookup:	0.0001 secs, 0.0000 secs, 0.0019 secs
  req write:	0.0000 secs, 0.0000 secs, 0.0009 secs
  resp wait:	0.1144 secs, 0.1028 secs, 0.1351 secs
  resp read:	0.0000 secs, 0.0000 secs, 0.0006 secs

Status code distribution:
  [200]	1000 responses

```

Summary for python server

```
Summary:
  Total:	1.2025 secs
  Slowest:	0.2005 secs
  Fastest:	0.1009 secs
  Average:	0.1160 secs
  Requests/sec:	831.5858

  Total data:	13000 bytes
  Size/request:	13 bytes

Response time histogram:
  0.101 [1]	    |
  0.111 [662]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.121 [139]	|■■■■■■■■
  0.131 [93]	|■■■■■■
  0.141 [15]	|■
  0.151 [2]	    |
  0.161 [8]	    |
  0.171 [16]	|■
  0.181 [12]	|■
  0.191 [23]	|■
  0.201 [29]	|■■


Latency distribution:
  10% in 0.1033 secs
  25% in 0.1051 secs
  50% in 0.1078 secs
  75% in 0.1137 secs
  90% in 0.1361 secs
  95% in 0.1815 secs
  99% in 0.1979 secs

Details (average, fastest, slowest):
  DNS+dialup:	0.0024 secs, 0.1009 secs, 0.2005 secs
  DNS-lookup:	0.0009 secs, 0.0000 secs, 0.0095 secs
  req write:	0.0000 secs, 0.0000 secs, 0.0013 secs
  resp wait:	0.1130 secs, 0.1004 secs, 0.1809 secs
  resp read:	0.0002 secs, 0.0000 secs, 0.0084 secs

Status code distribution:
  [200]	1000 responses

```

> Its marginally the same

## DB read server

Lets create a simple SQL raw query programs in python and node js and lets see how it handles IO intensive reads.

### The node JS code

See code [Here](https://github.com/Dalakoti07/practising-low-level-design/blob/main/io-bound/node-proj/dbIndex.js)

The result

```txt
Summary:
  Total:	0.2203 secs
  Slowest:	0.0962 secs
  Fastest:	0.0059 secs
  Average:	0.0211 secs
  Requests/sec:	4538.3428

  Total data:	1185000 bytes
  Size/request:	1185 bytes

Response time histogram:
  0.006 [1]	    |
  0.015 [376]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.024 [482]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.033 [45]	|■■■■
  0.042 [4]	    |
  0.051 [10]	|■
  0.060 [14]	|■
  0.069 [6]	    |
  0.078 [15]	|■
  0.087 [19]	|■■
  0.096 [28]	|■■


Latency distribution:
  10% in 0.0105 secs
  25% in 0.0114 secs
  50% in 0.0163 secs
  75% in 0.0204 secs
  90% in 0.0263 secs
  95% in 0.0768 secs
  99% in 0.0939 secs

Details (average, fastest, slowest):
  DNS+dialup:	0.0014 secs, 0.0059 secs, 0.0962 secs
  DNS-lookup:	0.0007 secs, 0.0000 secs, 0.0091 secs
  req write:	0.0000 secs, 0.0000 secs, 0.0007 secs
  resp wait:	0.0197 secs, 0.0059 secs, 0.0800 secs
  resp read:	0.0000 secs, 0.0000 secs, 0.0001 secs

Status code distribution:
  [200]	1000 responses
```

### Python code

See code [here](https://github.com/Dalakoti07/practising-low-level-design/blob/main/io-bound/py-proj/dbApp.py)

```txt

Summary:
  Total:	0.9330 secs
  Slowest:	0.1425 secs
  Fastest:	0.0346 secs
  Average:	0.0888 secs
  Requests/sec:	1071.8143

  Total data:	1269000 bytes
  Size/request:	1269 bytes

Response time histogram:
  0.035 [1]	    |
  0.045 [2]	    |
  0.056 [2]	    |
  0.067 [36]	|■■■
  0.078 [62]	|■■■■
  0.089 [569]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.099 [160]	|■■■■■■■■■■■
  0.110 [106]	|■■■■■■■
  0.121 [20]	|■
  0.132 [26]	|■■
  0.142 [16]	|■


Latency distribution:
  10% in 0.0776 secs
  25% in 0.0830 secs
  50% in 0.0866 secs
  75% in 0.0908 secs
  90% in 0.1057 secs
  95% in 0.1177 secs
  99% in 0.1358 secs

Details (average, fastest, slowest):
  DNS+dialup:	0.0018 secs, 0.0346 secs, 0.1425 secs
  DNS-lookup:	0.0006 secs, 0.0000 secs, 0.0056 secs
  req write:	0.0000 secs, 0.0000 secs, 0.0010 secs
  resp wait:	0.0868 secs, 0.0267 secs, 0.1274 secs
  resp read:	0.0002 secs, 0.0000 secs, 0.0067 secs

Status code distribution:
  [200]	1000 responses
```

Conclusion

> Node JS rocked, I mean almost 4x-5x faster

The performance discrepancy is largely justified by the fundamental differences in how Node.js and Flask handle concurrency and I/O operations. Node.js's non-blocking, event-driven architecture generally allows it to handle higher concurrency more efficiently than Flask’s default synchronous approach.

## Lets scale flask up to compete with node JS

```shell
pip install gunicorn

gunicorn -w 16 -b 0.0.0.0:3000 app:app

ab -n 1000 -c 100 http://localhost:3000/data

```

Results

```txt
Summary:
  Total:	1.0446 secs
  Slowest:	0.1562 secs
  Fastest:	0.0289 secs
  Average:	0.1002 secs
  Requests/sec:	957.2943

  Total data:	1269000 bytes
  Size/request:	1269 bytes

Response time histogram:
  0.029 [1]	    |
  0.042 [3]	    |
  0.054 [5]	    |
  0.067 [8]	    |■
  0.080 [34]	|■■■
  0.093 [284]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.105 [412]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.118 [73]	|■■■■■■■
  0.131 [70]	|■■■■■■■
  0.143 [87]	|■■■■■■■■
  0.156 [23]	|■■


Latency distribution:
  10% in 0.0818 secs
  25% in 0.0864 secs
  50% in 0.0978 secs
  75% in 0.1057 secs
  90% in 0.1330 secs
  95% in 0.1383 secs
  99% in 0.1492 secs

Details (average, fastest, slowest):
  DNS+dialup:	0.0020 secs, 0.0289 secs, 0.1562 secs
  DNS-lookup:	0.0006 secs, 0.0000 secs, 0.0052 secs
  req write:	0.0000 secs, 0.0000 secs, 0.0012 secs
  resp wait:	0.0981 secs, 0.0201 secs, 0.1491 secs
  resp read:	0.0000 secs, 0.0000 secs, 0.0012 secs

Status code distribution:
  [200]	1000 responses
```

![Image](../../assets/images/nodeJs/node_up_python.webp)

# Where Node JS loses (CPU intensive tasks)

## The Node JS code

See complete code [here](https://github.com/Dalakoti07/practising-low-level-design/blob/main/io-bound/node-proj/index.js)

```txt
Summary:
  Total:	44.1380 secs
  Slowest:	20.0076 secs
  Fastest:	0.0866 secs
  Average:	1.4339 secs
  Requests/sec:	22.6562

  Total data:	508877775 bytes
  Size/request:	538495 bytes

Response time histogram:
  0.087 [1]	    |
  2.079 [868]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  4.071 [4]	    |
  6.063 [4]	    |
  8.055 [6]	    |
  10.047 [12]	|■
  12.039 [10]	|
  14.031 [10]	|
  16.023 [9]	|
  18.015 [10]	|
  20.008 [11]	|■


Latency distribution:
  10% in 0.3425 secs
  25% in 0.3608 secs
  50% in 0.3921 secs
  75% in 0.6953 secs
  90% in 0.8628 secs
  95% in 10.6542 secs
  99% in 18.7008 secs

Details (average, fastest, slowest):
  DNS+dialup:	0.0004 secs, 0.0866 secs, 20.0076 secs
  DNS-lookup:	0.0002 secs, 0.0000 secs, 0.0048 secs
  req write:	0.0001 secs, 0.0000 secs, 0.0068 secs
  resp wait:	1.3952 secs, 0.0862 secs, 19.8778 secs
  resp read:	0.0379 secs, 0.0001 secs, 0.7945 secs

Status code distribution:
  [200]	945 responses

Error distribution:
  [55]	Get "http://localhost:3000/primes": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

> The node JS sent error for 55 requests, that is where it lost.

## Python Code

See complete code [here](https://github.com/Dalakoti07/practising-low-level-design/blob/main/io-bound/py-proj/app.py)

```txt
Summary:
  Total:	67.9959 secs
  Slowest:	7.6406 secs
  Fastest:	0.3285 secs
  Average:	6.5074 secs
  Requests/sec:	14.7068

  Total data:	538498000 bytes
  Size/request:	538498 bytes

Response time histogram:
  0.328 [1]	    |
  1.060 [12]	|■
  1.791 [8]	    |■
  2.522 [5]	    |
  3.253 [16]	|■
  3.985 [10]	|■
  4.716 [14]	|■
  5.447 [9]	    |■
  6.178 [12]	|■
  6.909 [602]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  7.641 [311]	|■■■■■■■■■■■■■■■■■■■■■


Latency distribution:
  10% in 6.2858 secs
  25% in 6.5753 secs
  50% in 6.7765 secs
  75% in 6.9578 secs
  90% in 7.1200 secs
  95% in 7.2238 secs
  99% in 7.4860 secs

Details (average, fastest, slowest):
  DNS+dialup:	0.0024 secs, 0.3285 secs, 7.6406 secs
  DNS-lookup:	0.0010 secs, 0.0001 secs, 0.0184 secs
  req write:	0.0000 secs, 0.0000 secs, 0.0020 secs
  resp wait:	6.4847 secs, 0.3111 secs, 7.6196 secs
  resp read:	0.0194 secs, 0.0002 secs, 0.3356 secs

Status code distribution:
  [200]	1000 responses
```

![Image](../../assets/images/nodeJs/python_comback.webp)

# Summary

Node JS looks hell promising on IO intensive tasks, and getting chocked on CPU intensive tasks. Long live Node JS

![Image](../../assets/images/nodeJs/match_tie.webp)

- Case 1

  - Node JS
    - Only DB reads, non blocking IO intensive work and hence delegates to libUv and highly throughput system created 🔥
  - Python
    - Its also did its best, but could not compete

- Case 2
  - Node JS
    - Now blocking CPU intensive task (77ms to calculate all primes under 1M) which will occupy event loop time, and it wont cater new incoming requests, and hence u can see that 55 requests were not catered. 😢
  - Python
    - Python just did all his JOB carefully, no response misses

# References

- [Node Js docs](https://nodejs.org/en/about)
- [Overview of blocking vs non-blocking](https://nodejs.org/en/learn/asynchronous-work/overview-of-blocking-vs-non-blocking)
- [JS callbacks and aync programming](https://nodejs.org/en/learn/asynchronous-work/javascript-asynchronous-programming-and-callbacks)
- [LibUv and node js story](https://codedamn.com/news/nodejs/libuv-architecture)
