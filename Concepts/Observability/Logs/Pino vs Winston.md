---
tags:
  - logging
  - observability
---

# Pino vs Winston — The Real Difference

Before comparing them, you need two mental models locked in. If you already know what the event loop is and what a process is, skip ahead to [The Comparison](#the-comparison).

---

## Prerequisites

### 1. What is the Node.js Event Loop?

Node.js runs your entire application in **a single thread**. There is no parallel execution by default. Everything runs one at a time, in sequence.

The event loop is the mechanism that keeps this single thread alive and responsive. You can think of it as a loop that constantly asks:

> "Is there any work to do right now? If yes, do it. If no, wait."

Work items in the loop look like:

- Run this function (a callback, a promise resolution, etc.)
- Handle this incoming HTTP request
- Read the result of this file read

The critical rule is: **while a piece of work is running, nothing else can run.** If your code takes 500ms to do something, the entire application is frozen for 500ms. No requests are handled, no responses are sent.

This is what people mean by "blocking the event loop" — you're hogging the single thread so long that other work piles up waiting.

```
┌──────────────────────────────────┐
│           Event Loop             │
│                                  │
│  → pick next task from queue     │
│  → run it (nothing else runs!)   │
│  → when done, pick the next      │
│  → repeat forever                │
└──────────────────────────────────┘
```

**Example of blocking:**

```javascript
// This blocks for ~2 seconds. Nothing else can happen.
const start = Date.now();
while (Date.now() - start < 2000) {}
```

**Example of NOT blocking (async I/O):**

```javascript
// This does NOT block. Node hands the work to the OS,
// then continues the event loop. When the OS is done,
// the callback gets queued.
fs.readFile('bigfile.txt', (err, data) => {
  console.log('done!');
});
// Node is free to handle other work here immediately
```

---

### 2. What is a Process?

A **process** is a running program. When you run `node app.js`, your OS starts a process.

Each process has:

- Its own memory space
- Its own CPU time allocation
- Its own "standard streams" (stdin, stdout, stderr)

Two processes run truly in parallel on modern CPUs. They don't share memory. They communicate by passing messages or through pipes.

**A pipe** is a connection between two processes where one's output becomes the other's input:

```bash
node app.js | pino-pretty
#    ^               ^
#  process 1      process 2
#  writes JSON    reads JSON, formats it
```

Here `app.js` and `pino-pretty` are two separate processes running at the same time, on different CPU cores if available. What one does has zero impact on the event loop of the other.

---

### 3. What is `stdout`?

Every process has three standard file descriptors (numbered channels for I/O):

|Number|Name|Default destination|
|---|---|---|
|0|stdin|keyboard|
|1|stdout|terminal screen|
|2|stderr|terminal screen|

`console.log()` in Node.js writes to **stdout** (fd 1). Writing to stdout is a low-level OS operation — it's one of the fastest things a process can do.

When you pipe processes together (`node app.js | pino-pretty`), the OS connects stdout of process 1 to stdin of process 2. Process 1 just writes to its own stdout as fast as it can — it doesn't wait for process 2 to do anything.

---

## The Comparison

Now that the foundations are in place, here's what each logger actually does when you call `log.info(...)`.

---

### Winston: Everything Happens Inside Your Process

When you call `logger.info('payment failed', { userId: 42 })`, Winston does all of this **inside your Node.js process, on the event loop:**

```
Your code calls logger.info()
        │
        ▼
Winston runs your message through a format pipeline
  - apply timestamp format
  - apply colorize (if configured)
  - apply JSON or printf format
  - apply any custom transforms you added
        │
        ▼
Winston sends the formatted string to each transport
  - Console transport: write to stdout
  - File transport: write to disk
  - HTTP transport: make a network request
  - (all of these happen here, now, in your event loop)
        │
        ▼
Control returns to your code
```

Every step above is synchronous work happening in the event loop. In a high-traffic application calling `log.info()` thousands of times per second, all of that formatting work adds up and occupies the event loop — time your application could have spent handling requests.

This is not necessarily a problem for most apps. Winston is fast enough for the vast majority of use cases. But at scale (tens of thousands of log calls per second), the cost is measurable.

---

### Pino: Formatting is Pushed Outside Your Process

When you call `logger.info({ userId: 42 }, 'payment failed')`, Pino does almost nothing inside your event loop:

```
Your code calls logger.info()
        │
        ▼
Pino does ONE thing: serialise the object to a JSON string
  (this is extremely fast — just string concatenation)
        │
        ▼
Pino writes the raw JSON string to stdout (fd 1)
  (a single OS syscall — also extremely fast)
        │
        ▼
Control returns to your code
```

That's it. Pino's job is done. The event loop is free.

Meanwhile, in a **completely separate process**, a transport (like `pino-pretty`) is reading from stdin, receiving that raw JSON, and doing whatever formatting/routing/colouring it needs to do:

```
┌─────────────────────────┐       OS pipe        ┌──────────────────────────┐
│    Your Node.js app     │ ──── JSON lines ────▶ │  pino-pretty (or any     │
│                         │                       │  transport process)       │
│  Pino writes JSON fast  │                       │  Formats, colours,        │
│  Event loop stays free  │                       │  writes to file, etc.     │
└─────────────────────────┘                       └──────────────────────────┘
       process 1                                         process 2
```

Because process 2 runs separately, its CPU work never touches your event loop. Your app stays fast regardless of how complex the formatting is.

---

### The Core Philosophical Difference

||Winston|Pino|
|---|---|---|
|Design goal|Rich, flexible logging inside your app|Minimum work in your app, push the rest out|
|Where formatting happens|Inside your process (event loop)|Outside your process (separate worker)|
|Transport model|Transports are objects inside your app|Transports are separate processes connected by pipes|
|Log format out of the box|Configurable (JSON, human-readable, custom)|Always JSON (formatting is someone else's job)|
|Configuration style|Code — create a logger with options objects|Code for the app, shell for routing (pipe to tools)|
|Flexibility|Very high — rich plugin ecosystem, custom formats|High — but via external tools, not in-process|
|Performance at scale|Good|Excellent|

---

### What "Minimum Work" Actually Means in Pino

Here is what Pino avoids doing in your process that Winston does:

**1. No format pipeline.** Winston lets you chain formatters:

```javascript
// Winston
const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.colorize(),
    winston.format.printf(({ level, message, timestamp }) => {
      return `${timestamp} [${level}]: ${message}`;
    })
  )
});
```

Every one of those `combine()`, `timestamp()`, `colorize()` calls runs in your event loop on every single log statement. Pino skips all of this — it only JSON-stringifies and writes.

**2. No in-process transports.** Winston's file transport, HTTP transport, etc. all live inside your app. A slow HTTP transport (sending logs to an external service) can hold up your event loop. Pino pushes this to a separate worker thread or process — your app just writes to stdout and moves on.

**3. Fast JSON serialisation.** Pino uses a technique called **fast-json-stringify** — it generates a dedicated serialisation function for your log schema ahead of time, rather than using `JSON.stringify()` on every call. This alone makes it meaningfully faster.

---

### What Winston is Better At

Being simpler and more flexible to configure without needing shell pipes:

```javascript
// Winston: easy multi-destination setup, all in one place
const logger = winston.createLogger({
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ]
});
```

With Pino, doing the same thing requires either shell piping or Pino's newer worker-thread-based transport API — more powerful but a higher learning curve.

Winston is also more established, has a larger ecosystem of community transports (Slack, email, Datadog, etc.), and its format pipeline model is very intuitive for developers coming from other languages.

---

### When the Difference Actually Matters

**Use either — they're both fine for:**

- Most web applications
- APIs handling up to a few hundred requests per second
- Applications where logging is a small fraction of total work

**Pino's advantage is real for:**

- High-throughput APIs (thousands of requests per second)
- Applications where log volume is very high (debug-level logging in prod, audit logging of every event)
- Latency-sensitive services where every millisecond matters

**Winston's flexibility is worth it when:**

- You need rich in-process log routing (different log levels to different destinations)
- You want easy integration with many third-party services without setting up shell pipes
- Your team finds the configuration model more readable and maintainable

---

## Quick Code Comparison

### Winston

```javascript
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'app.log' })
  ]
});

logger.info('payment processed', { userId: 42, amount: 99.99 });
```

Output (goes through the format pipeline, then to console + file):

```json
{"level":"info","message":"payment processed","userId":42,"amount":99.99,"timestamp":"2024-01-15T10:23:11.000Z"}
```

---

### Pino

```javascript
import pino from 'pino';

const logger = pino(); // defaults to JSON → stdout

logger.info({ userId: 42, amount: 99.99 }, 'payment processed');
```

Output (raw JSON, immediately to stdout):

```json
{"level":30,"time":1705313391000,"pid":1234,"hostname":"server-1","userId":42,"amount":99.99,"msg":"payment processed"}
```

To pretty-print it, you pipe the process — formatting never touches your app:

```bash
node app.js | pino-pretty
```

---

## Summary

The difference is **where work happens**:

Winston does everything inside your Node.js process — formatting, transformation, routing to destinations. This is convenient and flexible but costs event loop time.

Pino does almost nothing inside your process — it only converts to JSON and writes to stdout. All heavier work (formatting, routing, pretty-printing) is delegated to a separate process via a pipe, so your event loop stays free.

For most applications the difference is invisible. At scale, or in latency-sensitive services, Pino's approach gives you measurably better throughput.