# System Design: Networking, Latency, Async Processing & Reliability

## 1. Understanding Latency

**Latency** is the time required to complete an operation or receive a response.

### Types of Latency

| Type | Description | Common Optimization |
|---|---|---|
| **Network latency** | Time data takes to travel between client and server | CDN, geographically closer servers |
| **Processing latency** | Time the server spends executing application logic | Optimize algorithms and code |
| **Queue latency** | Time requests wait before being processed | Load balancing, autoscaling, backpressure |
| **Database latency** | Time spent executing queries and retrieving data | Indexes, query optimization, caching |
| **Serialization latency** | Time spent converting data into formats like JSON or Protobuf | Reduce payload size, efficient serialization |

**Important:** Network latency is limited by physical distance. Data travels through fiber at approximately 200,000 km/s.

A CDN reduces latency by serving eligible content closer to users.

### Latency vs. Throughput vs. Bandwidth

- **Latency:** How long one operation takes.
- **Throughput:** How many operations a system completes per second.
- **Bandwidth:** Maximum data transfer capacity of a connection.

High throughput does not necessarily mean low latency.

---

## 2. TCP Connection Establishment

TCP (Transmission Control Protocol) establishes a reliable, ordered connection between two endpoints.

### TCP Three-Way Handshake

1. **SYN:** Client requests a connection.
2. **SYN-ACK:** Server acknowledges and responds.
3. **ACK:** Client confirms the connection.

```text
Client                      Server
  | -------- SYN ---------> |
  | <------ SYN-ACK ------- |
  | -------- ACK ---------> |
  |                         |
  |    Connection Ready     |
```

The handshake generally requires approximately one network round-trip time (RTT), assuming no retransmissions.

For HTTPS over TCP, a **TLS handshake** also establishes encryption. Connection reuse can avoid repeated handshakes.

### Connection Timeout vs. Request Timeout

| Timeout | Meaning |
|---|---|
| **Connection timeout** | Maximum time allowed to establish a connection |
| **Request timeout** | Maximum time allowed for a request to complete, according to the client's timeout configuration |

A connection timeout may occur because of network failures, firewalls, or unavailable servers.

A request timeout may occur because of slow processing, overloaded servers, or database delays.

**Important:** A request timeout does not necessarily mean the server stopped processing the request.

---

## 3. HTTP Headers

HTTP headers contain **metadata** about requests and responses.

### Request Headers

| Header | Purpose |
|---|---|
| `Content-Type: application/json` | Indicates the request body contains JSON |
| `Accept: application/json` | Requests a JSON response |
| `Authorization: Bearer <token>` | Sends an authentication token |
| `X-Request-ID: abc-123` | Identifies a request for debugging and tracing |

### Response Headers

| Header | Purpose |
|---|---|
| `Location: /users/42` | Identifies a resource, commonly after a `201 Created` response |
| `Retry-After: 30` | Advises waiting 30 seconds before retrying |
| `Cache-Control: max-age=3600` | Allows caching for 1 hour, subject to other cache directives |
| `X-RateLimit-Remaining: 47` | Indicates remaining requests in a rate-limit window |

**Key distinction:**

- `Content-Type` describes the format of the content being sent.
- `Accept` describes which response formats the client supports.

---

## 4. Synchronous vs. Asynchronous Processing

### Synchronous Processing

Operations execute in a coordinated sequence, with the caller waiting for the result before continuing.

**Useful for:**
- Operations that depend on previous results.
- Simple request-response APIs.
- Workflows requiring immediate confirmation.

### Asynchronous Processing

Operations can continue independently while the caller performs other work or waits without blocking a thread.

**Useful for:**
- External API calls.
- Database and network I/O.
- Background jobs.
- Email notifications.
- Report generation.

### Comparison

| Characteristic | Synchronous | Asynchronous |
|---|---|---|
| Execution | Caller waits for completion | Caller can continue other work |
| Complexity | Often simpler | Requires additional coordination |
| Resource usage | Blocking operations may occupy threads | Non-blocking I/O can reduce idle threads |
| Error handling | Often straightforward | Errors may cross asynchronous boundaries |
| Best suited for | Immediate results | I/O concurrency and background processing |

**Important:** Synchronous does not necessarily mean sequential CPU execution, and asynchronous does not automatically mean parallel execution.

### JavaScript Async Concepts

**Callbacks:** Functions invoked when an operation completes. Excessive nesting can cause *callback hell*.

**Promises:** Objects representing future success or failure.

**async/await:** Syntax for working with Promises in a readable style.

```javascript
async function getUser() {
  try {
    const response = await fetch("/api/users/1");

    if (!response.ok) {
      throw new Error("Request failed");
    }

    return await response.json();
  } catch (error) {
    console.error(error);
    throw error;
  }
}
```

`await` suspends the current async function without blocking the JavaScript event loop.

### Benefits of Async

- **Throughput:** Servers can handle more concurrent I/O operations.
- **Resource efficiency:** Fewer threads are needed while waiting for I/O.
- **User experience:** Long operations can run in the background.

For example, instead of waiting 30 seconds for a report, an API can return `202 Accepted` while a worker generates it.

### Challenges of Async

- **Debugging:** Errors may occur across multiple callbacks, tasks, or services.
- **Error handling:** Background failures require explicit handling.
- **Race conditions:** Concurrent operations may access or modify shared data incorrectly.
- **Operational complexity:** Background jobs need retries, monitoring, and failure recovery.

### When Synchronous Processing Makes Sense

**Financial transactions:** A bank transfer must preserve consistency between debiting and crediting accounts.

**Database transactions:** ACID properties ensure reliable state changes.

- **Atomicity:** All changes succeed or none are committed.
- **Consistency:** Database constraints remain satisfied.
- **Isolation:** Concurrent transactions behave according to the configured isolation level.
- **Durability:** Committed changes survive failures.

**Important:** ACID transactions do not require blocking or synchronous application code. Asynchronous database APIs can still execute transactions correctly.

### When Synchronous Processing Hurts Performance

Consider five independent external API calls, each requiring 200 ms:

- Sequential execution: approximately 1,000 ms.
- Concurrent execution: approximately 200 ms, ignoring overhead.

Concurrency can significantly reduce total latency when operations are independent.

---

## 5. Timeouts, Retries, and Retry Storms

### What Is a Retry Storm?

A **retry storm** occurs when many clients repeatedly retry requests against an overloaded service.

```text
Service becomes overloaded
          |
          v
Requests start timing out
          |
          v
Clients retry requests
          |
          v
Additional server load
          |
          v
More timeouts and retries
```

This creates a feedback loop that can prolong an outage.

### How to Prevent Retry Storms

1. **Exponential backoff:** Increase the delay between retry attempts.
2. **Jitter:** Add randomness to retry delays so clients do not retry simultaneously.
3. **Maximum retries:** Limit retries, often to 2–3 attempts depending on the operation.
4. **Idempotency:** Retry operations only when duplicate execution is safe.
5. **Circuit breakers:** Temporarily stop requests to an unhealthy dependency.
6. **Backpressure:** Limit incoming work when system capacity is exhausted.

### Exponential Backoff With Jitter

```text
Attempt 1 → Wait ~1 second
Attempt 2 → Wait ~2 seconds
Attempt 3 → Wait ~4 seconds
Attempt 4 → Stop retrying
```

Actual waits should be randomized and capped.

**Important:** Retries should target transient failures, not every error. Retry limits are policies, not guarantees that another attempt would fail.

---

## 6. Idempotency

An operation is **idempotent** when performing it multiple times has the same intended effect as performing it once.

### HTTP Method Behavior

| Method | Idempotent? |
|---|---|
| GET | Yes, by HTTP semantics |
| PUT | Yes |
| DELETE | Yes |
| POST | Not necessarily |
| PATCH | Depends on the operation |

For example:

```text
PUT /users/42
{ "name": "John" }
```

Sending the same update repeatedly results in the same intended resource state.

However:

```text
POST /payments
{ "amount": 100 }
```

Repeating this request could create multiple payments.

---

## 7. Idempotency Keys

An **idempotency key** is a unique identifier that helps prevent duplicate execution of an operation.

This is especially important for payments, orders, and retried API requests.

### How It Works

1. Client generates a unique key, commonly a UUID.
2. Client includes the key in the request header.
3. Server checks whether the key was previously processed.
4. If new, the server processes the operation and stores its result.
5. If already completed, the server returns the previously stored result instead of repeating the operation.

```http
POST /payments
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

{
  "amount": 100
}
```

### Implementation Requirements

| Requirement | Purpose |
|---|---|
| **Storage** | Store keys and operation results in Redis or a database |
| **Atomicity** | Prevent concurrent requests from processing the same key |
| **Response storage** | Return the stored result for duplicate requests |
| **Expiration (TTL)** | Remove keys after a defined retention period |
| **Request validation** | Reject reuse of the same key with different request data |

A retention window such as 24–48 hours is common, but depends on the API.

### Preventing Race Conditions

A simple check followed by an insert is unsafe:

```text
Request A checks key → Not found
Request B checks key → Not found

Both execute the payment
```

Use an atomic operation such as:

- Redis `SET key value NX EX`.
- A database unique constraint and transaction.

Also track whether the operation is **in progress**, **completed**, or **failed**, and coordinate the business operation with the idempotency record to prevent duplicates after crashes.

**Key takeaway:** Idempotency keys make retries safer by preventing repeated business effects, even when clients experience network failures or timeouts.

---

## 8. System Design Principles

| Problem | Recommended Approach |
|---|---|
| High network latency | CDN, regional deployment |
| Slow application logic | Profiling and optimization |
| Overloaded server queues | Load balancing, autoscaling, backpressure |
| Slow database queries | Indexes, caching, query optimization |
| Excessive blocking I/O | Non-blocking asynchronous processing |
| Long-running tasks | Background workers and queues |
| Repeated failures | Backoff, jitter, circuit breakers |
| Duplicate payments | Idempotency keys and atomic operations |
| Concurrent data modifications | Transactions, locking, concurrency control |

**Core principle:** Optimize the actual bottleneck, use asynchronous processing when it provides a measurable benefit, and design retries so that failures do not create additional failures.


# Elasticity

**Elasticity** is the ability of a system to automatically adjust its resources to match current demand.

- **Scale out:** Add resources when traffic increases.
- **Scale in:** Remove resources when traffic decreases.

## Benefits

- Maintains performance during traffic spikes.
- Reduces costs when demand is low.
- Minimizes manual infrastructure management.
- Prevents unnecessary resource allocation.

## Scalability vs. Elasticity

| Scalability | Elasticity |
|---|---|
| Ability to handle increasing workloads. | Ability to dynamically adjust resources to demand. |
| Can be manual or automatic. | Typically automatic. |
| Focuses on capacity. | Focuses on matching capacity to demand. |

## Example

AWS Auto Scaling automatically increases the number of EC2 instances when demand rises and removes instances when demand decreases.


# Challenges of Elasticity

Although elasticity automatically adjusts resources based on demand, it introduces several challenges.

## 1. Cold Start Latency

New resources require time to become operational.

Approximate startup times:

- Virtual machines: 30–90 seconds.
- Containers: 5–30 seconds.

Startup times vary depending on configuration and infrastructure. During sudden traffic spikes, additional resources may not become available quickly enough.

**Solutions:**
- Maintain a warm pool of pre-provisioned instances.
- Use serverless functions for faster scaling, considering their own cold start limitations.

## 2. Stateful Services

Stateful services are harder to scale because they maintain persistent data.

Examples:
- Databases.
- Message brokers.
- Applications storing data locally.

Adding a database read replica can take minutes or hours because data must be synchronized.

**Elasticity works best with stateless application servers.**

## 3. Over-Scaling

Occurs when aggressive scaling policies allocate more resources than necessary.

**Consequences:**
- Increased infrastructure costs.
- Unnecessary resource consumption.

**Solutions:**
- Configure maximum instance limits.
- Avoid reacting aggressively to temporary traffic spikes.

## 4. Under-Scaling

Occurs when conservative scaling policies fail to allocate resources quickly enough.

For example, waiting until CPU utilization exceeds 95% for 15 minutes before adding instances.

**Consequences:**
- Increased latency.
- Degraded performance.
- Potential customer loss.

## Summary

| Challenge | Main Impact |
|---|---|
| Cold start | Delayed resource availability |
| Stateful services | Data synchronization complexity |
| Over-scaling | Excessive infrastructure costs |
| Under-scaling | Poor performance and user experience |

**The goal of elasticity is to balance infrastructure costs, resource availability, and application performance.**