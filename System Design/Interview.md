# System Design Interview Preparation: Cheat Sheet & Interview Tips

---

## Quick Reference: The 60-Minute Framework

### Timeline Breakdown (60-Minute Interview)

| Phase | Time | Focus |
|-------|------|-------|
| **Clarify Requirements** | 5 min | Ask detailed questions, understand constraints |
| **Capacity Estimation** | 5 min | Calculate QPS, storage, bandwidth |
| **High-Level Design** | 10 min | Draw architecture, identify components |
| **Detailed Design** | 25 min | Deep dive into 1-2 critical components |
| **Optimization & Trade-offs** | 10 min | Address bottlenecks, discuss alternatives |
| **Buffer & Follow-up** | 5 min | Answer clarifications, showcase thinking |

---

## Essential Questions to Ask

### Understanding the Problem

**Functional Requirements (What should the system do?):**
- What are the core features?
- What's the main use case?
- Who are the primary users?
- What's the scope? (MVP vs. full product)
- What operations are critical?

**Non-Functional Requirements (What quality matters?):**
- How many users/DAU?
- Global or regional?
- What's the latency requirement?
- Is consistency critical?
- What's the availability requirement (uptime %)?
- Is cost a constraint?

**Scope Clarifications:**
- Do I need to support mobile, web, or both?
- What about real-time features?
- Do we need to handle file uploads?
- What about notifications/alerts?

---

## Capacity Estimation Quick Formulas

### QPS (Queries Per Second) Calculation

```
QPS = (Daily Active Users Ã— Average Operations per User per Day) / (24 Ã— 3600)

Example:
100M DAU Ã— 100 operations/day Ã· 86,400 = 115,740 QPS
```

### Storage Calculation

```
Storage = Users Ã— Data per User Ã— Retention Period

Example:
100M users Ã— 1KB per user Ã— 365 days = 36.5 TB
```

### Bandwidth Calculation

```
Bandwidth = (Total Data per Second Ã— 8 bits per byte) / (1 Gbps)

Example:
1 TB/day Ã· 86,400 = 11.6 MB/s â‰ˆ 93 Gbps
```

### Memory/Cache Calculation

```
Cache Size = (Working Set Size) / (Cache Hit Rate)

Example:
If 20% of 1TB needs caching with 80% hit rate:
(1TB Ã— 0.2) / 0.8 = 250 GB cache needed
```

---

## Component Selection Matrix

### Choosing Databases

| Use Case | Recommended | Why |
|----------|-----------|-----|
| Banking/Financial | PostgreSQL | ACID transactions, consistency critical |
| Social Media Feed | DynamoDB/MongoDB | Fast writes, scalable, eventual consistency OK |
| Search Functionality | Elasticsearch | Full-text search, analytics |
| Session Storage | Redis | Fast reads/writes, expiration support |
| Time Series Data | InfluxDB/TimescaleDB | Optimized for metrics, timestamps |
| Analytics | BigQuery/Redshift | Massive data volumes, analytical queries |
| Document Storage | MongoDB | Flexible schema, JSON documents |
| Graph Relationships | Neo4j | Entity relationships, pattern matching |

### Choosing Message Queues

| Scenario | Choose | Reason |
|----------|--------|--------|
| Ultra-high throughput (1M+/sec) | Kafka | Optimized for volume, persistence |
| Low-latency messaging | RabbitMQ | Direct delivery, priority queues |
| Task Queues | RabbitMQ/Celery | Job processing, retries |
| Event Sourcing | Kafka | Event store, replay capability |
| Publish-Subscribe | Kafka/Redis | Fan-out to many subscribers |
| Request-Response RPC | gRPC | Synchronous, performance |

### API Choices

| Scenario | Choose | Reason |
|----------|--------|--------|
| Public API | REST | Browser compatible, cacheable |
| Microservice Communication | gRPC | Performance, HTTP/2, streaming |
| Simple CRUD | REST | Straightforward mapping to resources |
| Real-time Data | WebSocket | Bidirectional, low-latency |
| File Transfer | REST with multipart | Browser support |
| Streaming | gRPC / WebSocket | Efficient bidirectional flow |

---

## Architecture Patterns Cheat Sheet

### Handling Scalability

**Problem**: System bottleneck at database

**Solution:**
```
1. Add Cache Layer (Redis)
   - Cache-aside pattern for reads
   - Cache-through for consistency

2. Database Optimization
   - Indexing on frequently queried columns
   - Query optimization (explain analyze)
   - Denormalization for read-heavy workloads

3. Sharding (for writes)
   - Hash-based sharding by user_id
   - Range-based sharding by date
   - Geographic sharding for multi-region
```

**Problem**: Database write bottleneck

**Solution:**
```
1. Write Sharding
   - Distribute writes across multiple shards
   - Use consistent hashing for distribution

2. Async Processing
   - Write to message queue immediately
   - Process asynchronously with workers
   - Use eventual consistency

3. Database Optimization
   - Batch writes together
   - Use bulk insert operations
   - Optimize indexes for writes (minimal indexes)
```

**Problem**: Server CPU/Memory bottleneck

**Solution:**
```
1. Horizontal Scaling
   - Add more servers
   - Use load balancer
   - Session management (sticky sessions or store in cache)

2. Code Optimization
   - Optimize hot paths
   - Profile to find bottlenecks
   - Use connection pooling

3. Caching
   - Cache expensive computations
   - Cache database queries
   - Cache API responses
```

**Problem**: Network/Bandwidth bottleneck

**Solution:**
```
1. CDN for Static Content
   - Serve images, CSS, JS from CDN
   - Reduce main server load

2. Data Compression
   - GZIP compression for responses
   - Protocol buffers instead of JSON
   - Reduce payload size

3. Request Optimization
   - Batch multiple requests
   - Reduce number of round-trips
   - Use persistent connections
```

### Handling Reliability

**Problem**: Service failure causes cascading failures

**Solution:**
```
1. Circuit Breaker Pattern
   - Detect failures (error rate threshold)
   - Open: Stop calling failing service
   - Half-open: Test if service recovered
   - Closed: Resume normal operation

2. Bulkhead Pattern
   - Isolate critical components
   - Separate thread pools for different operations
   - Prevent one failure from affecting others

3. Retry with Backoff
   - Exponential backoff: 1s, 2s, 4s, 8s...
   - Jitter to prevent thundering herd
   - Max retry limit to fail fast
```

**Problem**: Data loss from hardware failure

**Solution:**
```
1. Replication
   - Master-slave for high availability
   - Replicate to multiple data centers
   - Asynchronous replication for performance

2. Backup Strategy
   - Regular snapshots (daily/weekly)
   - Point-in-time recovery capability
   - Test recovery procedures

3. Redundancy
   - Replicate critical data (3 copies minimum)
   - Different storage media/locations
   - RAID for physical servers
```

**Problem**: Inconsistent state across replicas

**Solution:**
```
1. Consistency Levels
   - Strong consistency for critical ops
   - Eventual consistency for reads
   - Causal consistency for related ops

2. Conflict Resolution
   - Last-write-wins (simple but lossy)
   - Vector clocks (complex but accurate)
   - Application-level resolution (most flexible)

3. Quorum Reads/Writes
   - Write to majority of replicas
   - Read from multiple replicas
   - Ensures consistency guarantees
```

---

## Data Flow Patterns

### Request-Response (Synchronous)

```
Client â†’ Load Balancer â†’ API Server â†’ Database â†’ Cache â†’ API Server â†’ Client

Best for: Real-time data, immediate feedback
Trade-off: Higher latency, tightly coupled
```

### Async with Message Queue

```
Client â†’ API Server â†’ Message Queue â†’ Worker â†’ Database
          â†“ (return immediately to client)

Best for: Long-running operations, decoupling
Trade-off: Eventual consistency, complex debugging
```

### Event Streaming

```
Event Source â†’ Event Bus (Kafka) â†’ Multiple Consumers
                                   â†’ Consumer 1 (Stream Processor)
                                   â†’ Consumer 2 (Analytics)
                                   â†’ Consumer 3 (Notification)

Best for: Real-time analytics, event sourcing
Trade-off: Eventual consistency, higher complexity
```

### CQRS (Command Query Responsibility Segregation)

```
Commands (Writes):          Queries (Reads):
Client â†’ Write Model    â†â†’  Client â†’ Read Model
         â†“                            â†‘
    Event Bus                     Event Bus
         â†“                            â†‘
    Event Store              Materialized Views

Best for: Complex domains, independent scaling
Trade-off: Eventual consistency, multiple data stores
```

---

## Common Scenarios & Solutions

### High-Traffic Spike Handling

**Scenario**: Flash sale causes 100x normal traffic

**Solution:**
```
1. Pre-warm caches
2. Increase database connection pool
3. Auto-scale servers based on load
4. Implement rate limiting (token bucket)
5. Queue excess requests
6. Degrade non-critical features
7. Show "sold out" early to cap demand
```

### Real-time Notifications

**Scenario**: Notify millions of users instantly

**Solution:**
```
1. Use WebSocket for persistent connections
2. Publish events to message queue
3. Consumer broadcasts to subscribed clients
4. For offline users: store in notification queue
5. Deliver on next login or via push notification
```

### Video Streaming at Scale

**Scenario**: Billions of video streams concurrently

**Solution:**
```
1. Use CDN for geographic distribution
2. Adaptive bitrate streaming (multiple qualities)
3. Cache popular content at edge
4. Segment videos (m3u8/DASH format)
5. Store video chunks in object storage
6. Origin server handles cache misses
```

### Search with Millions of Documents

**Scenario**: Fast search across massive dataset

**Solution:**
```
1. Use Elasticsearch for full-text search
2. Shard index by document ID or date
3. Cache popular searches
4. Implement autocomplete with trie/suggestions
5. Async indexing for new documents
6. Relevance scoring with ML models
```

---

## Red Flags to Avoid

âŒ **Single point of failure** â†’ Add redundancy
âŒ **No caching strategy** â†’ Leads to database overload
âŒ **Synchronous everything** â†’ Causes cascading failures
âŒ **No monitoring/alerting** â†’ Can't detect issues
âŒ **No rate limiting** â†’ DDoS vulnerability
âŒ **Strong consistency everywhere** â†’ Performance killer
âŒ **Monolithic architecture** â†’ Can't scale independently
âŒ **No database indexing** â†’ Slow queries
âŒ **Ignoring network latency** â†’ Unrealistic design
âŒ **No graceful degradation** â†’ Complete failures

---

## Interview Red Lines (Practice These!)

### Question: "Design a system that stores 1 billion records"

**Good Answer:**
```
- Database sharding by hash of ID
- Horizontal scaling across shards
- Caching popular records
- Indexes on frequently queried columns
- Monitoring and alerting
```

**Bad Answer:**
```
- "Just use a big database server"
- No mention of partitioning
- Single point of failure
- No consideration of consistency
```

### Question: "How do you handle database failures?"

**Good Answer:**
```
- Master-slave replication for failover
- Automated detection and promotion
- Data backups for recovery
- Multi-region replication
- Health checks and monitoring
```

**Bad Answer:**
```
- "It won't fail"
- "Just use backups" (without failover plan)
- No mention of RPO/RTO
```

### Question: "What's your caching strategy?"

**Good Answer:**
```
- Client-side caching for static assets
- CDN for geographic distribution
- Application-level cache (Redis) for hot data
- Cache invalidation strategy (TTL + events)
- Cache hit rate monitoring
```

**Bad Answer:**
```
- "Cache everything"
- "No caching needed"
- No consideration of cache invalidation
```

---

## Design Principles to Remember

### KISS (Keep It Simple, Stupid)
- Don't over-engineer initially
- Add complexity only when metrics justify it
- Proven, simple solutions > Complex new tech

### DRY (Don't Repeat Yourself)
- Reusable components
- Shared infrastructure
- Avoid duplicating logic

### SOLID Principles
- **S**ingle Responsibility: One job per service
- **O**pen/Closed: Open for extension, closed for modification
- **L**iskov Substitution: Substitutable implementations
- **I**nterface Segregation: Small, focused interfaces
- **D**ependency Inversion: Depend on abstractions, not concretions

### CAP Theorem Trade-offs
- Choose two of: Consistency, Availability, Partition Tolerance
- Understand your use case requirements
- Be explicit about trade-offs

### Cost-Performance Trade-off
- Not everything needs to be optimized
- Optimize where it matters most
- Monitor to find actual bottlenecks
- Consider operational overhead

---

## Sample Estimations (Reference)

### User Base Scenarios

| Type | Scale | Example |
|------|-------|---------|
| Startup | 100K DAU | Local service |
| Growth | 1M DAU | Regional app |
| Scale | 100M DAU | National service |
| Massive | 1B+ DAU | Global platform |

### Storage Rules of Thumb

| Type | Size/User |
|------|-----------|
| User profile | 1 KB |
| Single message | 1-5 KB |
| Photo | 1-3 MB |
| Video (1 hour) | 500 MB - 1 GB |
| User activity log | 10 KB/day |

### Database Limits (Rough)

| Metric | Limit |
|--------|-------|
| Single MySQL server | 10M-100M rows |
| Single Redis instance | 10-100 GB |
| Elasticsearch shard | 30-50 GB |
| Single MongoDB collection | 10B documents |

### Network Throughput (Rough)

| Connection | Bandwidth |
|-----------|-----------|
| 1Gbps network | 125 MB/s |
| Typical internet | 5-10 Mbps |
| 4G mobile | 5-20 Mbps |
| 5G mobile | 100+ Mbps |

---

## Presentation Tips

### During Explanation

âœ… **Think out loud**: Explain your reasoning
âœ… **Draw diagrams**: Visualize the architecture
âœ… **Ask clarifying questions**: Don't assume
âœ… **State assumptions**: Make constraints explicit
âœ… **Discuss trade-offs**: Show understanding
âœ… **Mention alternatives**: Show you've considered options
âœ… **Listen to feedback**: Adapt based on hints

### Pacing Techniques

- Spend 70% time on architecture, 30% on details
- Prioritize impact: scale problem, then optimize
- Be ready to go deeper or higher level
- Prepare 2-3 minute explanations for each component
- Have backup topics in case you finish early

### Handling Challenges

**When challenged:** "That's a great point. Let me think about that..."
**When unsure:** "In my experience, X is common, but let me consider..."
**When corrected:** "I appreciate that feedback. So the better approach is..."
**When running low on time:** "Let me prioritize and focus on..."

---

## Post-Interview Checklist

After your interview, self-evaluate:

âœ… Did I ask clarifying questions?
âœ… Did I estimate capacity correctly?
âœ… Did I address scalability concerns?
âœ… Did I discuss reliability and fault tolerance?
âœ… Did I mention monitoring and alerting?
âœ… Did I discuss trade-offs explicitly?
âœ… Did I consider operational complexity?
âœ… Did I draw and explain diagrams clearly?
âœ… Did I handle feedback well?
âœ… Did I show system design thinking?

---

## Resources for Practice

### Websites
- SystemDesignPrimer.com
- DesignGurus.io
- HighScalability.com
- Educative.io/system-design

### Books
- System Design Interview (Alex Xu)
- Designing Data-Intensive Applications (Martin Kleppmann)

### Practice Problems (In Difficulty Order)

**Beginner:**
1. Design URL Shortener
2. Design Web Crawler
3. Design Parking Lot

**Intermediate:**
4. Design Twitter Feed
5. Design Instagram
6. Design WhatsApp

**Advanced:**
7. Design YouTube
8. Design Uber
9. Design Stock Exchange
10. Design Distributed Database

---

## Final Thoughts

System design is about **problem-solving**, not memorization. Each interview is unique. The goal is to demonstrate:

1. **Communication**: Clearly explain your thinking
2. **Analysis**: Ask the right questions
3. **Knowledge**: Know when to use which tools
4. **Judgment**: Make sound architectural decisions
5. **Flexibility**: Adapt to feedback and constraints

Practice these frameworks, and you'll handle any system design interview!

---

**Good luck! ðŸš€**
