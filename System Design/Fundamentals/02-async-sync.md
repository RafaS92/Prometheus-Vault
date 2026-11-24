# Synchronous and Asynchronous Processing

## Execution and Communication Models

Two design decisions shape how a system handles work. At the **API boundary**, the client can wait for the completed result or receive an acknowledgment while processing continues in the background. **Inside the application**, the execution thread can block while an operation is pending or become available for other work.

| Concept | Meaning |
|---|---|
| **Synchronous call** | The call returns when the operation completes. A blocking I/O call occupies its calling thread while waiting |
| **Asynchronous call** | The call starts or schedules an operation and delivers the outcome later, for example through a callback or a Promise |
| **Sequential execution** | Operations execute one after another |
| **Concurrent execution** | The lifetimes of multiple operations overlap |
| **Parallel execution** | Multiple operations execute at the same instant, for example on different CPU cores |

```text
Sequential:
Task A: [AAAA]
Task B:       [BBBB]

Concurrent:
Task A: [AA]    [AA]
Task B:     [BB]    [BB]

Parallel:
Core 1: [AAAA]
Core 2: [BBBB]
```

An API can return the completed result in the original HTTP response while using asynchronous, non-blocking database calls internally. A typical example is a simple **CRUD** endpoint for creating, reading, updating, or deleting data.

```text
Client → REST API → Database → JSON Response
```

For an endpoint that completes in about 20 ms, the direct request-response flow provides immediate confirmation and typically keeps control flow, debugging, and error handling simpler than handing the request off to a queue. A slow dependency can delay the response, and sequential dependency calls accumulate latency.

## Asynchronous I/O in JavaScript

**Input/output (I/O)** includes operations such as external API calls, database queries, file reads, and other network or disk access. Blocking I/O ties up the calling thread while it waits, although the CPU can run other threads or processes. Asynchronous I/O lets the application handle other requests while an operation is pending, reducing pressure on workers and potentially increasing throughput under I/O-heavy load.

Node.js uses an event loop to handle many concurrent network connections without dedicating an application thread to each connection. By comparison, a blocking server with 100 workers and one request per worker can actively handle about 100 requests at once. Connection count measures concurrency. Throughput measures completions per unit of time, so 10,000 open connections can produce a very different request rate. Actual capacity depends on CPU, memory, request complexity, database connections, operating system limits, and downstream services.

Using fewer threads can reduce stack memory, context switching, and worker exhaustion. For example, if each Java platform thread is configured to reserve 1 MB of stack space, 1,000 threads can reserve about 1,000 MB for stacks. Actual memory use can differ from the reserved amount. Pending asynchronous operations still consume resources.

### Callbacks

A **callback** is a function passed to another function. Its timing depends on how it is invoked. In this example, Node.js invokes the callback once the file read succeeds or fails.

```javascript
import fs from "node:fs";

fs.readFile("data.txt", "utf8", (error, data) => {
  if (error) {
    console.error(error);
    return;
  }

  console.log(data);
});
```

Nesting callbacks for several dependent operations can make the code difficult to follow and maintain, a problem often called **callback hell**.

### Promises

A **Promise** represents the eventual result or failure of an operation.

| State | Meaning |
|---|---|
| **Pending** | The Promise has yet to settle |
| **Fulfilled** | The Promise settled with a value |
| **Rejected** | The Promise settled with a reason, often an error |

Use `.then()` to chain dependent operations and `.catch()` to handle rejections. Promises also make it possible to coordinate multiple operations without deeply nested callbacks. With `fetch()`, check `response.ok` explicitly because HTTP error responses can fulfill the Promise.

```javascript
fetch("/api/users")
  .then(response => {
    if (!response.ok) {
      throw new Error("Request failed");
    }

    return response.json();
  })
  .then(users => console.log(users))
  .catch(error => console.error(error));
```

### Async/await

**Async/await** expresses Promise-based operations using familiar control flow. While a Promise is pending, `await` suspends the current async function and lets the JavaScript event loop continue. CPU-intensive JavaScript can still block the event loop. Worker threads can run that work in parallel when appropriate.

```javascript
async function getUsers() {
  try {
    const response = await fetch("/api/users");

    if (!response.ok) {
      throw new Error("Request failed");
    }

    const users = await response.json();

    return users;
  } catch (error) {
    console.error(error);
    throw error;
  }
}
```

### Handling Promise rejections

This `try/catch` catches a synchronous throw from the call. A later Promise rejection escapes it.

```javascript
try {
  generateReport();
} catch (error) {
  console.error(error);
}
```

Await the Promise inside the `try` block so its rejection reaches the handler.

```javascript
try {
  await generateReport();
} catch (error) {
  console.error(error);
}
```

When a caller intentionally continues before an operation finishes, it can attach a rejection handler directly. This remains in-process work and can be interrupted by a process restart.

```javascript
generateReport().catch(error => {
  console.error(error);
});
```

## Sequential and Concurrent Operations

Use sequential execution when one operation needs the result of another. Here, `getOrders()` needs the user's ID. Other work can proceed while either I/O operation is pending.

```javascript
const user = await getUser();
const orders = await getOrders(user.id);
```

Successive `await` expressions also make independent calls run sequentially when each call starts only after the previous one finishes.

```javascript
const users = await fetch("/api/users");
const products = await fetch("/api/products");
const orders = await fetch("/api/orders");
```

To overlap independent calls, start them together and await their combined result. Here the `fetch()` calls start when JavaScript evaluates the array. `Promise.all()` coordinates the resulting Promises.

```javascript
const results = await Promise.all([
  fetch("/api/users"),
  fetch("/api/products"),
  fetch("/api/orders"),
  fetch("/api/categories"),
  fetch("/api/reviews")
]);
```

If five independent calls each take about 200 ms, sequential execution takes about 1,000 ms. With enough downstream capacity for all five to overlap, total latency is about 200 ms plus coordination overhead. In these snippets, each `fetch()` resolves when response headers are available. Reading the body adds time.

```text
API 1 ── 200 ms ──┐
API 2 ── 200 ms ──┤
API 3 ── 200 ms ──┼── Results
API 4 ── 200 ms ──┤
API 5 ── 200 ms ──┘
```

### Promise coordination methods

| Method | Behavior |
|---|---|
| `Promise.all()` | Fulfills with all values in input order, or rejects when an input rejects |
| `Promise.allSettled()` | Fulfills with every outcome after all inputs settle |
| `Promise.race()` | Settles with the first input to fulfill or reject |
| `Promise.any()` | Fulfills with the first successful value, or rejects with an `AggregateError` if all inputs reject |

If the combined Promise settles while other operations are pending, those operations continue unless explicitly canceled. Limit concurrency when calls share a database, external API, or other constrained dependency. Excessive concurrency can increase queueing and latency.

## Transactions and Shared Data

### A bank transfer as an ACID transaction

For a $5,000 transfer between two accounts in the same database, the application can confirm success after the transaction commits.

1. Begin the transaction and validate the available balance using appropriate concurrency control.
2. Debit the source account and verify that the debit succeeded.
3. Credit the destination account.
4. Commit both changes and return confirmation.

```sql
BEGIN TRANSACTION;

-- Validate available balance
-- Debit source account
-- Credit destination account

COMMIT;
```

If a step fails before commit, roll back the transaction so a partial transfer cannot persist. Application code can await each database operation in order. The database transaction supplies the **ACID** guarantees.

| Property | Description |
|---|---|
| **Atomicity** | The transaction commits as a unit or its changes are rolled back |
| **Consistency** | A correctly implemented transaction preserves the database's rules and invariants |
| **Isolation** | The isolation level governs how concurrent transactions interact and which anomalies are possible |
| **Durability** | Committed changes survive failures covered by the database's durability guarantees |

### Race conditions and lost updates

A **race condition** occurs when correctness depends on the timing or order of concurrent operations. The following example shows a **lost update** when two requests read the same balance and later write independently computed values.

```text
Initial balance: $100

Request A reads balance: $100
Request B reads balance: $100

Request A subtracts $30
Request B subtracts $50

Request A writes: $70
Request B writes: $50

Final balance: $50
Expected balance: $20
```

This can occur in any system with concurrent access to shared mutable state. Timing-dependent failures may appear intermittently and be hard to reproduce.

| Technique | How it coordinates updates |
|---|---|
| **Database transactions** | Group related updates and use an isolation level suited to the operation |
| **Locks and mutexes** | Restrict conflicting access to shared resources or allow only one execution context into a critical section at a time |
| **Atomic operations** | Apply a change as one indivisible operation |
| **Optimistic concurrency control** | Detect conflicting changes before accepting a write |
| **Sequential processing** | Enforce order for operations that must run one at a time |

An atomic conditional update can apply a debit against the current balance and check available funds in the same statement. The application should check that the statement affected the expected row before proceeding.

```sql
UPDATE accounts
SET balance = balance - 30
WHERE account_id = 1
  AND balance >= 30;
```

## Background Processing

With **background processing**, the API accepts a job and sends an acknowledgment. A worker completes the task outside the original request. Common examples include email and notifications, reports, image and video processing, large datasets, data ingestion, and scheduled jobs.

### Example of report generation

A direct request-response flow makes the user wait for a 30-second report.

```text
Client → API → Generate Report → Response
```

A background flow returns an acknowledgment quickly so the user can continue using the application while the worker generates the report.

```text
Client → API → Message Queue → Worker
           ↓                    ↓
     202 Accepted          Generate Report
                                ↓
                         Notify User
```

HTTP **202 Accepted** indicates that the server accepted the request for processing. The task may still be pending or may later fail. A job ID or status URL lets the client check progress. The acknowledgment improves response latency even when the report still takes 30 seconds to generate.

### Common infrastructure

| Technology | Role |
|---|---|
| Amazon SQS | Managed message queues |
| RabbitMQ | Message broker and task distribution |
| Apache Kafka | Distributed event streaming |
| AWS Lambda | Event-driven function execution |
| BullMQ | Redis-based background job processing |

### Reliability and observability

Use persistent queues when accepted jobs need to survive process restarts. Retry transient failures with exponential backoff, and make workers **idempotent** so processing a duplicate message leaves the intended outcome unchanged. A **dead-letter queue (DLQ)** can retain jobs that exhaust their retry attempts for investigation or recovery. Handle ordering explicitly when it affects correctness.

Asynchronous failures can surface in a callback, another process, or a background worker after the original response. Use structured logging, monitoring, distributed tracing, and async stack traces where available. A **correlation ID** connects logs and operations belonging to the same request or workflow. Queues, workers, and recovery mechanisms add operational complexity.

### Eventual consistency

In an event-driven workflow, related changes can become visible at different times.

```text
Order Created
      ↓
Event Published
      ↓
Inventory Updated
      ↓
Confirmation Email Sent
```

For example, the order can appear before the inventory update is reflected. **Eventual consistency** means that if updates stop and propagation succeeds, the parts of the system that track the same state eventually converge. Applications should handle intermediate states and workflows that are only partially complete.
