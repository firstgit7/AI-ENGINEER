# System Design Advanced Patterns & Distributed Systems Concepts

---

## Distributed Systems Fundamentals

### The 8 Fallacies of Distributed Computing

Assumptions that everyone makes but are false:

1. **The network is reliable** â†’ Networks fail. Plan for it.
2. **Latency is zero** â†’ Network latency exists. Plan for timeouts.
3. **Bandwidth is infinite** â†’ It's limited. Compress data.
4. **The network is secure** â†’ It's not. Use encryption/TLS.
5. **Topology doesn't change** â†’ Servers go up/down. Use service discovery.
6. **There is one administrator** â†’ Multiple teams. Need clear ownership.
7. **Transport cost is zero** â†’ Infrastructure costs money. Minimize transfers.
8. **The network is homogeneous** â†’ Different hardware/OS. Be version-agnostic.

**Application**: These aren't true; systems must handle them.

---

## Consistency Models Explained

### Immediate/Strong Consistency

**Definition**: All replicas see the same data instantly.

**Implementation**: 
- All writes must be serialized
- Quorum-based writes (write to all/majority)
- Synchronous replication

**Trade-off**: High latency for writes

**Use Cases**: Banking, payments (where correctness > speed)

### Eventual Consistency

**Definition**: Replicas converge to same state over time.

**Implementation**:
- Asynchronous replication
- Conflict resolution strategy
- Accept stale reads temporarily

**Trade-off**: Temporary inconsistency, higher availability

**Use Cases**: Social media feeds, product listings

### Causal Consistency

**Definition**: Causally-related operations are seen in order.

**Implementation**:
- Vector clocks or logical clocks
- Track operation dependencies

**Trade-off**: More complex than eventual, less strict than strong

**Use Cases**: Group chat, collaborative editing

### Session Consistency

**Definition**: Client sees their own writes immediately.

**Implementation**:
- Route client to same server
- Or: client keeps local version until server confirms

**Trade-off**: Works for single client, but multi-device problematic

**Use Cases**: Most web applications

---

## Failover & High Availability Patterns

### Master-Slave Replication

**Architecture**:
```
Master (Read/Write) 
  â†“
Slave 1 (Read-only)
Slave 2 (Read-only)
Slave 3 (Read-only)
```

**Advantages**:
- Simple failover: Promote a slave to master
- Read scaling: Distribute reads across slaves
- Backups: Use a slave for backup without affecting master

**Disadvantages**:
- Single point of failure (master)
- Replication lag: Slaves might be stale
- Split-brain: If master fails and not detected properly

**Failover Strategy**:
1. Detect master failure (heartbeat, health checks)
2. Verify no writes in progress
3. Promote best slave (most recent data)
4. Update clients to new master
5. Old master re-joins as slave

### Master-Master Replication

**Architecture**:
```
Master 1 â† â†’ Master 2 â† â†’ Master 3
```

**Advantages**:
- No single point of failure
- Can write to any master
- Geographic distribution possible

**Disadvantages**:
- Complex conflict resolution
- Circular replication possible
- Higher write latency

**Conflict Resolution**:
- Last-write-wins: Simple but might lose data
- Version vectors: Track causality
- Application-level: Custom logic

---

## Distributed Consensus & Agreement

### The Consensus Problem

How do multiple servers agree on a value when some may fail?

### Raft Algorithm

**Overview**: Easier to understand than Paxos

**States**:
- **Follower**: Passive state, listens to leader
- **Candidate**: Becomes when leader election starts
- **Leader**: Chosen by election, handles client requests

**Election Process**:
1. Followers become candidates when heartbeat expires
2. Candidate requests votes from other servers
3. Majority voting wins leadership
4. Leader sends heartbeats to prevent new elections

**Use Cases**: Etcd, Consul, RaftDB

### Paxos Algorithm

**Overview**: More complex but very powerful

**Roles**:
- **Proposer**: Proposes values
- **Acceptor**: Votes on values
- **Learner**: Learns agreed values

**Phases**:
1. Prepare: Get promises to accept
2. Accept: Send accepted value
3. Decide: Chosen value learned by all

---

## Distributed Tracing

### Purpose

Understand request flow across microservices:
- Where does latency come from?
- Which service failed?
- How do multiple services interact?

### Key Components

**Trace**: Complete journey of one request
- Multiple spans
- Each service adds a span

**Span**: Operation within one service
- Start/end time
- Operation name
- Service name
- Error status

**Trace ID**: Unique identifier following request
- Passed through all service calls
- Used to correlate logs

### Example

Request to order service:
1. API Gateway: Span 1 (authenticate user)
2. Order Service: Span 2 (create order)
3. Payment Service: Span 3 (process payment)
4. Inventory Service: Span 4 (reduce stock)

All connected by same Trace ID.

**Tools**: Jaeger, Zipkin, AWS X-Ray, DataDog

---

## Service Discovery Patterns

### Client-Side Discovery

```
Service Registry:
  service1: [ip1:port1, ip2:port2]
  service2: [ip3:port3]

Client asks registry, then connects directly
```

**Advantages**: Simple, direct connection

**Disadvantages**: Client complexity, every client needs registry access

### Server-Side Discovery

```
Service Registry:
  service1: [ip1:port1, ip2:port2]

Load Balancer queries registry, routes to instance
Client â†’ Load Balancer â†’ Service Instance
```

**Advantages**: Client simplicity, centralized routing

**Disadvantages**: Additional hop, load balancer becomes bottleneck

### Self-Registration

```
Service Instance registers itself:
1. Start up
2. Query registry for service info
3. Register with registry
4. On shutdown, deregister
```

**Tools**: Consul, Eureka

### Third-Party Registration

```
External component (orchestrator) registers services:
- Container orchestrator (Kubernetes)
- Deployment framework
- Watches services, updates registry
```

**Tools**: Kubernetes, Docker Swarm

---

## Load Balancing Advanced Strategies

### Consistent Hashing

**Problem**: Regular hashing with N servers means all keys remapped when server added/removed

**Solution**: Arrange servers in a ring
- Hash key and server ID same way
- Find nearest server clockwise on ring
- Adding server only affects nearby keys

**Benefit**: Minimal remapping on server change

### Weighted Load Balancing

Servers have different capacities:
```
Server A: 4 cores â†’ weight 4
Server B: 2 cores â†’ weight 2
Server C: 1 core â†’ weight 1
```

Distribute requests proportionally.

### Least Loaded

Route to server with fewest active connections.

**Advantage**: Adaptive to actual load
**Disadvantage**: Requires state tracking, more overhead

### Geographic/Latency-Based

Route to nearest data center:
```
Client in NY â†’ Route to US-East DC
Client in London â†’ Route to EU-West DC
```

---

## Data Consistency Strategies

### Two-Phase Commit (2PC)

**Ensure ACID across multiple databases**:

Phase 1 (Prepare):
- Coordinator asks all participants to prepare
- Participants lock resources, check if transaction possible
- Respond "yes I can commit" or "no I can't"

Phase 2 (Commit):
- If all said yes: Coordinator tells all to commit
- If any said no: Coordinator tells all to rollback

**Problem**: If coordinator fails, participants locked indefinitely

### Saga Pattern (Distributed Transactions)

**For eventual consistency scenarios**:

Instead of one transaction:
- Transaction 1: Create order
- If success â†’ Transaction 2: Debit payment
- If success â†’ Transaction 3: Reduce inventory
- If fail â†’ Compensating transactions (reverse steps)

**Choreography vs Orchestration**:
- Choreography: Services react to events (loosely coupled)
- Orchestration: Central coordinator tells services what to do (tightly coupled)

---

## Circuit Breaker Pattern Deep Dive

### States

```
Closed (Normal)
  â†“ (failures exceed threshold)
Open (Failing fast)
  â†“ (after timeout)
Half-Open (Testing)
  â†“ (success) â†™ (failure)
Closed          Open
```

### Configuration

```
Failure threshold: If 50% of requests fail
Timeout: Try new request after 30 seconds
Success threshold: 2 successful requests to close
```

### Implementation Example

```
try {
  response = serviceCall()
} catch (Exception e) {
  if (circuitBreaker.isOpen()) {
    throw new CircuitBreakerException()
  } else if (circuitBreaker.isHalfOpen()) {
    // Test request
    circuitBreaker.recordResult(success/failure)
  } else {
    // Normal operation
    circuitBreaker.recordFailure()
  }
}
```

---

## Rate Limiting Advanced Techniques

### Distributed Rate Limiting

**Problem**: Multiple servers, need global limit

**Solution 1: Centralized Counter**
```
All servers â†’ Redis Counter
Increment and check against limit
```
**Disadvantage**: Single point of failure, network overhead

**Solution 2: Local + Sync**
```
Each server has local counter
Periodically sync with others
Each has quota of global limit
```
**Advantage**: Resilient, lower latency
**Disadvantage**: Possible bursts, eventual consistency

### Per-User Rate Limiting

```
GET /api/users?limit=5
User Rate Limit: 100 requests/hour
Global Rate Limit: 1M requests/hour

Check both limits, use minimum
```

### Backpressure

When system overloaded, signal clients to slow down:
```
HTTP 429 Too Many Requests
Retry-After: 60 seconds
```

---

## Distributed Caching Challenges

### Cache Coherency

Multiple servers with cached data. When to invalidate?

**Strategy 1: Time-based (TTL)**
```
Set TTL on cache entry
After X seconds, entry expires
Simple but might serve stale data
```

**Strategy 2: Event-based**
```
When data changes, publish invalidation event
Cache listeners remove entry
Accurate but requires infrastructure
```

**Strategy 3: Write-through**
```
Write to cache AND database simultaneously
Always consistent, but slower
```

### Cache Stampede

Many requests hit cache just as it expires:
```
All requests miss cache
All hit database simultaneously
Database overwhelms
```

**Solutions**:
- Probabilistic early expiration (refresh before expiration)
- Distributed locking (first request refreshes, others wait)
- Use "lock" value in cache

### Cold Start Problem

Fresh cache is empty. All requests miss.

**Solutions**:
- Pre-warm cache with popular data on startup
- Gradual cache warming
- Expect cache misses initially

---

## Monitoring & Alerting Patterns

### Key Metrics to Track

**System Metrics**:
- CPU usage
- Memory usage
- Disk I/O
- Network bandwidth

**Application Metrics**:
- Request latency (p50, p95, p99)
- Request rate (QPS)
- Error rate
- Business metrics

**Database Metrics**:
- Connection pool usage
- Query latency
- Slow queries
- Replication lag

### Red Flags (Alert When)

```
CPU > 80% for 5 minutes
Memory > 85% of available
Error rate > 1%
Request latency p99 > 500ms
Database replication lag > 10 seconds
Queue depth > 10,000 messages
```

### Alerting Best Practices

âœ… Alert on actionable items only
âœ… Set appropriate thresholds (avoid noise)
âœ… Include runbook link in alert
âœ… Escalation policy (who to notify)
âœ… Throttle alerts (don't spam for same issue)
âœ… Page on-call only for P1 issues

---

## Performance Optimization Techniques

### Batching

Instead of processing one item:
```
Process 1000 items in batch
Reduce overhead (network, DB calls)

Trade-off: Latency increase (waiting for batch)
```

### Caching Strategies

1. **Cache-Aside**
   - Check cache â†’ miss â†’ fetch from DB â†’ update cache
   - Pros: Only caches accessed data
   - Cons: Cache miss latency

2. **Write-Through**
   - Write to cache AND DB together
   - Pros: Cache always consistent
   - Cons: Write latency increases

3. **Write-Behind**
   - Write to cache, return immediately
   - Async write to DB later
   - Pros: Fast writes
   - Cons: Data loss if cache fails before DB write

### Denormalization

Store pre-computed results instead of joining:
```
Instead of joining 3 tables at query time
Pre-compute and store result
Trade-off: Write complexity, storage overhead â†’ faster reads
```

### Database Indexing

Create indexes on frequently queried columns:
```
Index on user_id: Fast lookups by user
Index on created_at: Fast sorting by date
Trade-off: Slower writes, more storage
```

---

## Security Best Practices in System Design

### API Security

1. **Authentication**
   - OAuth 2.0 for third-party apps
   - API keys for internal services
   - JWT tokens with expiration

2. **Authorization**
   - RBAC: Role-Based Access Control
   - ABAC: Attribute-Based Access Control
   - Principle of least privilege

3. **Rate Limiting**
   - Per API key
   - Per IP address
   - Per user

### Data Protection

1. **In Transit**
   - TLS 1.2+ (HTTPS)
   - Certificate pinning for mobile

2. **At Rest**
   - Encryption for sensitive fields
   - Key management (rotate keys)
   - Secure key storage

3. **In Database**
   - Row-level encryption
   - Column-level encryption
   - Field-level encryption for PII

### Network Security

- VPC/Network segmentation
- Security groups (firewall rules)
- DDoS protection
- Web Application Firewall (WAF)
- Regular security audits

---

## Technology Selection Criteria

### SQL Database Selection

Choose PostgreSQL over MySQL when:
- Need advanced features (JSONB, arrays, custom types)
- Complex queries required
- Large dataset scale
- Window functions needed

Choose MySQL when:
- Simple, predictable workload
- Read-heavy with occasional writes
- LAMP stack compatibility
- Lower operational complexity

### NoSQL Selection

| Type | Use Case |
|------|----------|
| Document (MongoDB) | Flexible schema, JSON-like |
| Key-Value (Redis) | Session, cache, leaderboards |
| Wide-Column (Cassandra) | Time-series, massive scale |
| Graph (Neo4j) | Relationships, patterns |
| Search (Elasticsearch) | Full-text search, analytics |

### Message Queue Selection

| System | Best For |
|--------|----------|
| Kafka | High-throughput, event streaming |
| RabbitMQ | Task queues, request-response |
| Redis | Simple pub-sub, low latency |
| AWS SNS/SQS | Cloud-native, managed |

---

## Key Takeaways

1. **System design is about trade-offs**: Understand what you're optimizing for
2. **Scalability isn't just databases**: Consider every component
3. **Reliability requires redundancy**: No single points of failure
4. **Monitoring is essential**: Can't optimize what you don't measure
5. **Security is not optional**: Build it in from the start
6. **Simple is better than complex**: Proven technologies often beat new ones
7. **Understand your constraints**: Cost, latency, consistency, availability
8. **Communicate clearly**: Diagrams and explanations matter

---

**Remember**: There's no one-size-fits-all solution. The best architecture depends on your specific requirements, constraints, and trade-offs.
