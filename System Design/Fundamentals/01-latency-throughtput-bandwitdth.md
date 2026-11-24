# Latency, Throughput, and Bandwidth

| Metric | What it measures | Common unit |
|---|---|---|
| **Latency** | Time for an individual operation to complete | Milliseconds (ms) |
| **Throughput** | Work completed per unit of time | Requests per second (RPS) |
| **Bandwidth** | Maximum data transfer rate of a network link or path | Megabits per second (Mbps) |

## Latency

For an API, **end-to-end latency** is the elapsed time from sending a request to receiving the complete response. An API that returns its response after 200 ms has an end-to-end latency of 200 ms. Lower latency usually makes a user-facing application feel more responsive.

### Sources of latency

| Source | Where time is spent | Ways to reduce it |
|---|---|---|
| **Network** | Data travels between systems, with delays from propagation, transmission, and network equipment | Serve cached content from a content delivery network (CDN), deploy servers closer to users, reduce round trips, and reuse connections |
| **Queueing** | A request waits for a server or worker with available capacity | Add workers, increase concurrency when resources allow, balance load, apply backpressure or rate limits, and defer non-urgent work |
| **Application processing** | The server runs algorithms, performs calculations, applies business rules, or transforms data | Profile CPU-intensive work, improve algorithms, avoid unnecessary computation, and cache reusable results |
| **Database** | The application executes database operations and retrieves the results | Inspect query plans, add appropriate indexes, optimize queries, reduce round trips, and cache frequently accessed data |
| **Serialization** | The application encodes or decodes data, for example when parsing JSON or encoding Protocol Buffers | Return only necessary fields, avoid redundant transformations, and choose an efficient format when appropriate |

A simple endpoint might spend 5–10 ms in application logic, while an inefficient operation can take seconds. A primary-key lookup may take a few milliseconds, while large scans, expensive joins, or missing indexes can push a query into the hundreds of milliseconds. Serialization can account for a significant share of latency when payloads are large.

### Network propagation and round-trip time

Signals travel through fiber at roughly **200,000 km/s**, about two-thirds the speed of light. If a user in Mumbai and a server in Virginia are about 12,900 km apart, the idealized round-trip propagation time is about **129 ms**.

```text
Round-trip propagation time = (2 × 12,900 km) / 200,000 km/s
                            = 0.129 s = 129 ms
```

Real network round-trip time (RTT) is typically higher because cable routes are longer than the straight-line distance and network equipment adds delay.

### Measuring latency

Use percentiles to see both typical response times and the slow end of the distribution, which an average can obscure.

| Percentile | Meaning | Example |
|---|---|---|
| **P50** | Half of measured requests complete within this time | 100 ms |
| **P95** | 95% of measured requests complete within this time | 400 ms |
| **P99** | 99% of measured requests complete within this time | 1,200 ms |

In this example, about 1% of measured requests take longer than 1,200 ms.

## Throughput

**Throughput** is the rate of completed work under a particular workload. If an API successfully handles 5,000 requests in 10 seconds, its successful throughput is 500 RPS.

```text
Throughput = Completed requests / Elapsed time
           = 5,000 / 10 s
           = 500 RPS
```

Use a unit that matches the work you are measuring.

| Unit | Work measured |
|---|---|
| **RPS** | Requests per second, such as completed HTTP requests |
| **TPS** | Transactions per second |
| **QPS** | Queries per second, such as database queries |
| **Mbps** | Megabits per second of actual network transfer |

Other useful units include processed messages per second and completed background jobs per second.

### Measuring sustainable throughput

1. Gradually increase offered traffic in a load test with a representative workload.
2. Record successful completions per second alongside P50 and P99 latency, error rate, CPU utilization, memory usage, and queue depth.
3. Identify the highest completion rate the system can sustain while meeting its latency and error-rate targets.

As demand approaches capacity, requests can start to queue and P95 and P99 latency can rise sharply. Timeouts and errors may follow. Under severe overload, successful throughput can fall. Spare capacity helps absorb traffic spikes. A target of **60–70% utilization** can serve as a planning example. Use load tests and traffic patterns to determine the headroom the system actually needs.

### Increasing throughput

| Strategy | When it helps |
|---|---|
| **Vertical scaling** | A single server needs more resources, such as when upgrading an Amazon EC2 instance from 2 to 8 vCPUs |
| **Horizontal scaling and load balancing** | Work can be distributed across more servers or workers, such as EC2 instances behind an Application Load Balancer, and shared dependencies have spare capacity |
| **Database optimization** | Queries or database resources limit the rate of completed work |
| **Caching** | Repeated requests can reuse results and avoid expensive operations |
| **Batching** | Multiple operations can share per-operation overhead, such as a database round trip |
| **Asynchronous background processing** | Non-urgent work can run in a separately managed worker pool, freeing capacity for incoming requests |

## Bandwidth

In networking, **bandwidth** usually refers to a link or path's data-carrying capacity. A wider highway can carry more cars at once while each car still travels at the same speed. Likewise, a higher-capacity link can send a large payload faster while propagation time stays the same.

### Units and conversion

| Unit | Meaning |
|---|---|
| **bps** | Bits per second |
| **kbps** | 1,000 bits per second |
| **Mbps** | 1,000,000 bits per second |
| **Gbps** | 1,000,000,000 bits per second |

```text
1 byte = 8 bits
100 Mbps ÷ 8 = 12.5 MB/s
```

A 100 Mbps link can carry up to **12.5 megabytes per second** before accounting for protocol overhead.

### Why bandwidth matters

- **Large transfers and API responses.** Uploads, downloads, inter-service data transfers, and large JSON responses take longer as the amount of data grows. Larger responses also consume more resources.
- **Images and video.** Large media assets consume transfer capacity. Sustained throughput must keep pace with the encoded video bitrate over time to avoid draining the playback buffer.
- **Mobile networks.** Available capacity varies with signal quality, congestion, and network technology.
- **Cloud costs.** Providers such as AWS may charge for the volume of outbound data transferred.

### Using bandwidth efficiently

Compress data where appropriate, reduce protocol overhead, and serve images in suitable sizes and formats. Pagination limits how much data each API response needs to transfer. A CDN can serve cached media without fetching it repeatedly from the origin, and adaptive bitrate streaming adjusts video quality to the available throughput.

### Effective network throughput

**Network throughput** is the actual rate of data transfer. If a 100 Mbps link is 70% utilized in a given direction, its measured transfer rate in that direction is 70 Mbps, assuming both figures use the same measurement layer.

```text
Link throughput = Link bandwidth × Link utilization
                = 100 Mbps × 0.70
                = 70 Mbps
```

Application payload throughput can be lower because protocol overhead and retransmissions consume some of that rate. Congestion and transport limits can also reduce how much of the link capacity a transfer achieves.

## How the Metrics Interact

| Scenario | Performance implication |
|---|---|
| A fast server has limited concurrency | Individual requests can finish quickly even if the server completes few requests per second |
| Multiple servers process requests concurrently | Aggregate throughput can rise. Per-request latency still depends on load and bottlenecks |
| A high-bandwidth link connects distant servers | The link can carry a high data rate even with a long RTT |
| A large file crosses a slow link | Low effective throughput increases the time needed to complete the transfer |

### Caching and batching

A cache hit can avoid a database call, improving request latency and freeing database capacity so the system can handle more requests. Caching can also reduce upstream traffic.

```text
Without caching: Client -> API -> Database -> API -> Client
Cache hit:       Client -> API -> Cache    -> API -> Client
```

Batching can improve throughput by spreading fixed overhead across multiple operations. A taxi can depart as soon as a passenger arrives, while a bus might wait for more passengers and carry more per trip. Similarly, batching database writes can cut overhead per write while increasing the time individual writes spend waiting for the batch.

### Little's Law

**Little's Law** relates the average number of requests in a stable system to its average request rate and the average time a request spends there.

```text
L = λ × W

L = Average number of requests in the system, including queued requests
λ = Long-run average request rate at the system boundary
W = Average time a request spends within that boundary
```

In a stable system, the long-run arrival and departure rates match. Measure all three quantities over the same system boundary and request population. If the system also processes failed requests, a rate that counts successful completions alone can undercount total departures.

```text
Average requests in flight = 100 requests/s × 0.2 s
                           = 20 requests
```

Little's Law helps estimate in-flight requests, connection demand, and queue sizes. It can inform worker sizing and utilization estimates alongside measurements of service time and resource capacity. At a fixed request rate, a longer average latency means more requests are in flight.

### Bandwidth-delay product

The **bandwidth-delay product (BDP)** estimates how much data a sender needs in flight to sustain the bottleneck bandwidth of a network path over one RTT.

```text
BDP = Bottleneck bandwidth × Round-trip time
    = 100,000,000 bits/s × 0.1 s
    = 10,000,000 bits
    = 1.25 MB
```

On an idealized 100 Mbps path with a 100 ms RTT, a TCP sender needs about **1.25 MB** of unacknowledged data in flight to keep the path fully utilized. A smaller effective send window can limit throughput even when plenty of bandwidth is available. This matters for TCP window and buffer sizing, especially for large transfers over long-distance links.

## Choosing What to Optimize

| Symptom | Where to look | Possible changes |
|---|---|---|
| An individual API request is consistently slow | Time spent in application code, queries, and downstream calls | Profile the request, optimize queries, or cache reusable results |
| The application handles too few concurrent users | Saturation in servers, workers, or shared dependencies | Add capacity at the limiting component and balance load |
| Requests slow down during traffic spikes | Queue depth and tail latency | Autoscale, rate-limit, or apply backpressure |
| Large files transfer slowly | Link capacity and effective throughput | Compress data, use a CDN, or increase capacity |
| Video buffers frequently | Sustained throughput and playback buffer | Use adaptive bitrate, a CDN, or pre-buffering |
| Large API responses arrive slowly | Payload size and transfer time | Paginate, compress, or return fewer fields |
| The database slows under load | Query cost, contention, and available connections | Tune indexes, cache reads, or manage connections |

User-facing APIs need low latency and enough throughput to meet demand. Real-time applications also need predictable response times. Background jobs and data pipelines can often prioritize sustained throughput and resource efficiency, while large transfers benefit from effective network throughput and efficient use of capacity. Choose targets that balance responsiveness, capacity, reliability, and cost.
