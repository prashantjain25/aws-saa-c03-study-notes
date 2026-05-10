# Section 07 — Caching

> **Purpose**: Caching is the most common performance optimization in distributed systems — and the most common source of consistency bugs. A cache is not a faster database; it is a **consistency tradeoff** that accepts stale reads in exchange for reduced latency and load. This section treats caching as an architectural pattern with explicit failure modes, invalidation strategies, and operational implications.
>
> **Official Documentation**: [CloudFront](https://docs.aws.amazon.com/cloudfront/) | [ElastiCache](https://docs.aws.amazon.com/elasticache/) | [Global Accelerator](https://docs.aws.amazon.com/global-accelerator/) | [Lambda@Edge](https://docs.aws.amazon.com/lambda/latest/dg/lambda-edge.html)

---

## 1. Amazon CloudFront: CDN and Edge Caching

### 1.1 What CloudFront Actually Does

CloudFront is a **global content delivery network** that caches content at 450+ edge locations worldwide. But caching is only part of its function:

**CloudFront Capabilities**:
- **Static content caching**: Images, CSS, JS, video files at edge locations
- **Dynamic content acceleration**: TCP connection termination at the edge, persistent connections to origin, route optimization
- **TLS termination**: SSL certificates at the edge, reducing origin load
- **WAF integration**: DDoS protection and application-layer filtering at the edge
- **Origin Shield**: A secondary cache layer between edge locations and origin, reducing origin load
- **Edge compute**: Lambda@Edge and CloudFront Functions for request/response manipulation

### 1.2 CloudFront Caching Architecture

```
User Request ──► Edge Location (nearest to user)
                    │
                    ├── Cache HIT → Serve immediately (sub-10ms)
                    │
                    └── Cache MISS → Origin Shield (if enabled)
                                          │
                                          ├── Cache HIT → Serve + populate edge cache
                                          │
                                          └── Cache MISS → Origin (ALB / S3 / EC2)
                                                              │
                                                              └── Response → Cache at all layers
```

**Cache behaviors** are controlled by **cache policies**:
- **TTL settings**: Minimum, default, maximum TTL
- **Cache keys**: What identifies a unique cacheable object? By default: URL path + query string (selective) + headers (selective) + cookies (selective)
- **Origin request policy**: What to forward to origin (not cached)

> **Architectural Decision**: A common mistake is caching everything with the same TTL. User-specific data (profile pages, shopping carts) should not be cached. Static assets (CSS, JS, images) can cache for days. API responses might cache for seconds or minutes depending on data freshness requirements.

### 1.3 Cache Invalidation

CloudFront provides **cache invalidation** to remove objects before TTL expiration:

```bash
# Invalidate all objects under /images/
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/images/*"
```

**Invalidation limits**:
- 3,000 invalidation paths per month included in price
- Additional paths: $0.005 per path
- Wildcards (`/*`) count as one path but invalidate everything matching
- Invalidation takes 2-5 minutes to propagate globally

> **Operational Reality**: Cache invalidation is an **eventually consistent operation**. During propagation, some edge locations may serve the old version while others serve the new. For zero-downtime deployments, use **cache-busting filenames** (`app.v2.css`) instead of invalidation.

### 1.4 CloudFront Origin Types

| Origin | Use Case | Notes |
|--------|----------|-------|
| **S3** | Static assets, website hosting | Use OAI (Origin Access Identity) or OAC (Origin Access Control) to prevent direct S3 access |
| **ALB / EC2 / Custom HTTP** | Dynamic applications | Use custom headers for origin authentication |
| **API Gateway** | API responses | Cache API responses at edge to reduce Lambda/EC2 load |
| **MediaPackage / MediaStore** | Video streaming | HLS/DASH manifest and segment caching |
| **Another CloudFront distribution** | Multi-tier caching | Origin Shield pattern |

**OAC (Origin Access Control) vs OAI**: OAC is the modern replacement for OAI. It uses IAM policies and signature version 4. OAI used legacy AWS signatures. Always use OAC for new distributions.

### 1.5 CloudFront Signed URLs and Signed Cookies

For restricted content (paid videos, private files), CloudFront supports:
- **Signed URLs**: One URL per user, time-limited access. Best for individual file access.
- **Signed Cookies**: One cookie grants access to multiple files matching a pattern. Best for member areas with many resources.

Both require a **Trusted Key Group** (RSA key pair). The signer application generates the signature; CloudFront verifies it at the edge.

### 1.6 Lambda@Edge vs CloudFront Functions

| Feature | CloudFront Functions | Lambda@Edge |
|---------|---------------------|-------------|
| **Runtime** | JavaScript (lightweight) | Node.js, Python |
| **Scale** | 10M+ requests/sec | 10K+ requests/sec per region |
| **Execution time** | <1ms | 5-30 seconds |
| **Memory** | 2 MB | 128 MB – 10 GB |
| **Use case** | Header manipulation, URL rewrites, simple auth | Complex logic, database access, external API calls |
| **Cost** | Very low | Per-request + compute time |

**Typical Lambda@Edge use case**: Inspect JWT at the edge, validate with Cognito, add claims to headers before forwarding to origin. This offloads authentication from the origin.

### 1.7 Global Accelerator vs CloudFront

| Dimension | CloudFront | Global Accelerator |
|-----------|-----------|-------------------|
| **Primary function** | Content caching at edge | Network acceleration (any TCP/UDP traffic) |
| **Protocol** | HTTP/HTTPS only | Any TCP or UDP |
| **Caching** | Yes | No |
| **Static IP** | No (uses DNS) | Yes (2 Anycast IPs) |
| **Health checks** | Origin health checks | Endpoint health checks with automatic failover |
| **Use case** | Web content, APIs, video | Gaming, IoT, VoIP, non-HTTP protocols, static IP requirements |

> **Key Difference**: CloudFront optimizes content delivery through caching. Global Accelerator optimizes network path without caching — it uses AWS's global backbone to route traffic from the user to the nearest AWS edge, then through private backbone to the application.

---

## 2. Amazon ElastiCache: In-Memory Caching

### 2.1 Why In-Memory Caching?

Database queries are slow (milliseconds). Network hops add latency. In-memory caches are fast (microseconds). The tradeoff is **consistency**: cache data is a snapshot, not live data.

**When to cache**:
- Read-heavy, infrequently changing data (product catalogs, user profiles, config)
- Session state (shopping carts, login state)
- Computed aggregates (dashboard metrics, leaderboards)
- Rate limiting counters

**When NOT to cache**:
- Write-heavy data (cache invalidation dominates)
- Data requiring strong consistency (financial transactions, inventory)
- Small datasets that fit in application memory (adds unnecessary complexity)

### 2.2 Redis vs Memcached

ElastiCache supports two engines with fundamentally different architectures:

| Feature | Redis | Memcached |
|---------|-------|-----------|
| **Data types** | Strings, hashes, lists, sets, sorted sets, streams, JSON | Key-value only (binary-safe strings) |
| **Persistence** | Optional (AOF, RDB snapshots) | None (purely ephemeral) |
| **Replication** | Multi-AZ with automatic failover | None (individual nodes) |
| **Clustering** | Redis Cluster mode (sharded) | Client-side sharding |
| **Complex operations** | Lua scripting, pub/sub, streams, geospatial | Simple get/set/delete/append |
| **Use case** | Session store, real-time analytics, leaderboards, job queues | Simple object caching, fragment caching |

> **Architectural Decision**: Default to Redis. It has replication, failover, and richer data structures. Use Memcached only when you need simple key-value caching at massive scale with no persistence requirements.

### 2.3 Redis Architecture Patterns

**Single-Node with Multi-AZ**:
- One primary, one replica in different AZ.
- Automatic failover if primary fails.
- Read replicas can serve read traffic (offload primary).

**Redis Cluster Mode (Sharded)**:
- Data partitioned across multiple shards (0-500 shards).
- Each shard: 1 primary + 0-5 replicas.
- **Slot-based sharding**: 16,384 hash slots distributed across shards. Keys are hashed to slots.
- **Client routing**: Redis clients must support cluster mode (most modern libraries do).

```
Shard 1 (Slots 0-5461)     Shard 2 (Slots 5462-10922)     Shard 3 (Slots 10923-16383)
├── Primary (AZ-a)          ├── Primary (AZ-b)              ├── Primary (AZ-c)
└── Replica (AZ-b)          └── Replica (AZ-c)              └── Replica (AZ-a)
```

> **Scaling Redis Cluster**: Adding shards requires resharding — redistributing slots. This is an online operation but impacts performance temporarily. Plan shard count based on dataset size and access pattern, not just current needs.

### 2.4 Cache Design Patterns

#### Cache-Aside (Lazy Loading)
```
Application checks cache
    ├── HIT → Return data
    └── MISS → Query database → Store in cache → Return data
```

**Pros**: Simple, only caches data that is actually requested.
**Cons**: Cache stampede on expiry (many requests hit DB simultaneously), stale data until TTL expires.

#### Write-Through
```
Application writes to cache AND database simultaneously
```

**Pros**: Cache always fresh.
**Cons**: Write latency increases (two writes). Cache may contain data never read.

#### Write-Behind (Write-Back)
```
Application writes to cache only. Cache asynchronously writes to database.
```

**Pros**: Fastest writes.
**Cons**: Risk of data loss if cache fails before database write. Complex to implement correctly.

#### Read-Through
```
Application requests from cache. Cache itself fetches from database on miss.
```

**Pros**: Application logic simpler.
**Cons**: Cache library must support it. Tight coupling between cache and DB.

### 2.5 Cache Invalidation Strategies

Cache invalidation is famously one of the "two hard things in computer science." Operational approaches:

| Strategy | When to Use | Risk |
|----------|------------|------|
| **TTL (Time-To-Live)** | Default approach. Set expiration per key. | Stale reads during TTL window. Thundering herd when TTL expires. |
| **Explicit Invalidation** | Write path invalidates cache. | Missed invalidations (bugs, race conditions) lead to stale data. |
| **Event-Driven Invalidation** | DynamoDB Streams / DB triggers → Lambda → Cache invalidation | Eventual consistency delay. Complex pipeline. |
| **Versioned Keys** | `user:123:v1`, `user:123:v2` | No invalidation needed — old versions simply expire. But memory grows with versions. |

> **Thundering Herd Mitigation**: When a popular cache key expires, thousands of requests may hit the database simultaneously. Solutions:
> 1. **Jitter**: Randomize TTLs slightly so keys don't expire simultaneously.
> 2. **Lease token**: One request gets a "lease" to refresh the cache; others wait or serve stale data briefly.
> 3. **Probabilistic early refresh**: Refresh the cache before TTL expires based on access probability.

### 2.6 Cache Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| **Cache node failure** | Cache misses increase, DB load spikes | Redis Multi-AZ automatic failover. Memcached: pre-warm new nodes. |
| **Cache eviction** (memory full) | LRU eviction drops keys. Cache hit rate drops. | Monitor memory usage. Scale vertically (larger node) or horizontally (more shards). |
| **Hot key** | Single key gets massive traffic, overwhelming one shard | Distribute hot keys across multiple keys. Use read replicas for read-heavy keys. |
| **Large value** | Single large value (e.g., 100 MB JSON) slows Redis | Split large objects. Use smaller keys. Consider compression. |
| **Cold start** | New cache empty → all requests hit DB | Pre-warm cache from database before switching traffic. |

---

## 3. Operational Realities of Caching

### 3.1 Cache Hit Rate Monitoring

A cache with < 80% hit rate is often not worth the operational complexity. Monitor:
- `CacheHits` and `CacheMisses`
- `CPUUtilization` (Redis is single-threaded; CPU saturation = shard capacity limit)
- `NetworkBytesIn/Out` (bandwidth saturation)
- `EngineCPUUtilization` (Redis-specific CPU metric)
- `Evictions` (indicates memory pressure)

### 3.2 Security

- **ElastiCache is NOT publicly accessible by default** — it runs in your VPC with security group controls.
- **Redis AUTH**: Password-based authentication. Required for compliance.
- **Encryption in transit**: TLS for Redis connections.
- **Encryption at rest**: KMS-encrypted storage volumes.
- **Security groups**: Restrict access to application subnets only. Never expose Redis to the internet.

> **Common Vulnerability**: Redis without AUTH + security group open to `0.0.0.0/0` = cryptomining infection within hours. Redis is a frequent target because it often has weak security configurations.

---

## 4. Interview Challenges

### Q1: "A product catalog API has 1000 products, accessed millions of times per day. Cache with Redis or CloudFront?"

**Answer**: Both, for different layers.
- **CloudFront**: Cache the API responses at the edge for identical requests. This reduces origin load and improves global latency. TTL: minutes to hours (products don't change frequently).
- **Redis**: Cache the database query results (product details, inventory counts) near the application. TTL: seconds to minutes (inventory changes more frequently than product descriptions).
- **Architecture**: User → CloudFront (edge cache) → ALB → Application → Redis (application cache) → RDS (source of truth).
- **Invalidation**: Product update in admin panel → write to RDS → invalidate Redis key → optionally invalidate CloudFront cache. EventBridge can orchestrate this.

### Q2: "A cached value is showing stale data for 5 minutes after database update. How do you fix it?"

**Answer**: The TTL is too long, or invalidation is not working. Options:
1. **Reduce TTL** to match acceptable staleness window. Tradeoff: more database load.
2. **Implement write-through caching**: Update cache atomically with database write. Tradeoff: write latency.
3. **Event-driven invalidation**: DynamoDB Streams or RDS triggers → Lambda → Redis delete key. Tradeoff: complexity, eventual consistency of invalidation itself.
4. **Versioned cache keys**: Key includes data version. Updates write to new key. Application reads latest version. Old versions expire naturally. Tradeoff: memory overhead.

The right choice depends on the staleness tolerance. Financial data: write-through. Social media feed: shorter TTL or accept staleness.

### Q3: "Memcached or Redis for session storage?"

**Answer**: Redis. Rationale:
- Sessions need persistence (don't want to lose all sessions if a node restarts). Redis offers optional AOF/RDB persistence.
- Sessions benefit from TTL (automatic expiry). Both support this.
- Sessions may need replication for HA. Redis has Multi-AZ failover. Memcached does not.
- Session data might need complex structures (JSON with nested fields). Redis supports hashes and JSON.

Memcached is only appropriate if sessions are truly ephemeral (e.g., anonymous preview sessions that can be lost) and cost is the absolute priority.

---

## 5. Points to Remember

- **CloudFront caches at 450+ edge locations globally** — but cache invalidation takes 2-5 minutes. Use versioned filenames for instant updates.
- **CloudFront OAC replaces OAI** — use OAC for all new S3 origin distributions.
- **Global Accelerator provides static IPs** — use for non-HTTP protocols or when clients need to whitelist IPs. CloudFront uses DNS names.
- **Lambda@Edge has 5-30 second execution limit** — not suitable for database queries or external API calls. Use CloudFront Functions for sub-millisecond operations.
- **Redis is single-threaded** — CPU utilization on the primary node is the bottleneck. Scale horizontally (sharding) when CPU approaches 80%.
- **Memcached is multi-threaded but has no replication** — node failure = complete data loss for that partition.
- **Cache hit rate below 80% often indicates poor cacheability** — re-evaluate whether caching is the right optimization.
- **TTL jitter prevents thundering herd** — don't set all keys to expire at exactly 300 seconds. Add randomness.
- **Redis without AUTH is a security vulnerability** — always enable Redis AUTH and restrict security groups.
- **CloudFront Origin Shield reduces origin load** — enable it when you have many edge locations hitting the same origin.
- **Signed URLs and Signed Cookies require a trusted signer key** — protect this key like any other credential.
- **ElastiCache is VPC-only** — no public endpoint. Access requires VPC connectivity (EC2, Lambda in VPC, VPN, Direct Connect).
- **Cache eviction on memory pressure uses LRU** — frequently accessed keys survive; infrequently accessed keys are dropped first.

---

*Section 07 — Caching | Last Validated: 2026-05-10*
