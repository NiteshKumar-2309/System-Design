# Rate Limiting Cheatsheet

## 📌 Summary of Algorithms

| Algorithm               | Granularity     | Memory Usage | Burst Control | Accuracy | Core Logic                               | Key Limitations                                              | Use Case Suitability                   |
|-------------------------|-----------------|--------------|---------------|----------|----------------------------------------|-------------------------------------------------------------|--------------------------------------|
| Fixed Window Counter    | Minute/hour     | Low          | ❌ No         | Low      | Count requests in fixed time windows   | Boundary burst at window edges; unfair short-term bursts     | Simple use-cases, moderate traffic    |
| Sliding Window Log      | Per request     | High         | ✅ Yes        | High     | Store every request timestamp in a log| High memory and CPU; stores all timestamps                   | Precise limits, low-to-medium scale   |
| Sliding Window Counter  | Time buckets    | Medium       | ⚠️ Partial   | Medium   | Approximate sliding via multiple buckets| Approximation errors; complexity in bucket maintenance      | Balanced trade-off                    |
| Token Bucket            | Per request     | Medium       | ✅ Yes        | High     | Tokens added at fixed rate; consumed per request| Complexity; token synchronization issues; token loss possible| APIs allowing bursts                  |
| Leaky Bucket            | Fixed drain rate| Medium       | ✅ Yes        | Medium   | Requests flow out at fixed rate        | Can introduce delay; no burst allowance                      | Smooth, steady request flow           |

---

## 🔧 Implementation Overview

| Algorithm               | Redis Structure             | Expiry Use     | Key Format               | Atomicity           |
|-------------------------|-----------------------------|----------------|--------------------------|---------------------|
| Fixed Window Counter    | Key per client per window   | Yes            | `clientID:minute`        | Redis INCR/DECR     |
| Sliding Window Log      | Sorted Set of timestamps    | Yes            | `clientID`               | Lua script needed   |
| Sliding Window Counter  | Time buckets (rolling)      | Yes            | `clientID:bucket_time`   | Lua script helpful  |
| Token Bucket            | Last refill + token count   | Yes            | `clientID:bucket`        | Lua script needed   |
| Leaky Bucket            | Queue w/ fixed outflow rate | Yes            | `clientID:leaky`         | Lua optional        |

---

## 🏢 Real-World Adoption & Quick Interview Tips

| Algorithm              | Real-World Example                   | Quick Interview Tip                                               |
|------------------------|------------------------------------|------------------------------------------------------------------|
| Fixed Window Counter   | GitHub’s simple API rate limiting   | Explain the boundary burst problem and how it can cause spikes. |
| Sliding Window Log     | Stripe’s monetary transaction APIs  | Highlight accuracy but note memory tradeoffs with storing timestamps. |
| Sliding Window Counter | Some CDN and proxy caches            | Useful for balancing precision vs resource cost.                 |
| Token Bucket           | AWS API Gateway                     | Emphasize burst allowance and token refill mechanism.            |
| Leaky Bucket           | Google Cloud API controls            | Mention smoothing traffic and fixed output rate.                  |

---

## ✅ Interview-Oriented Questions & Answers

**1. Why not enforce rate limiting on the client side?**  
👉 Because clients can tamper with enforcement. Server-side controls ensure consistency, fairness, and protection from abuse or malicious actors.

**2. How do you handle Redis downtime during rate limiting?**  
👉 Options:
- Let requests through but log the usage for post-recovery adjustments.
- Queue or throttle requests temporarily.
- Build in HA Redis with replicas and failover strategies.

**3. Compare Token Bucket vs Leaky Bucket.**  

|                | Token Bucket                          | Leaky Bucket                          |
|----------------|-------------------------------------|-------------------------------------|
| Purpose        | Allows burst traffic                 | Smooths traffic                      |
| Mechanism      | Tokens added at fixed rate; consumed per request | Requests drained at fixed rate        |
| Bursts         | Allowed if tokens available          | No real burst; steady output enforced |

**4. When is Sliding Window better than Fixed Window?**  
👉 When you want *finer control* over rate limits. Sliding windows reduce the "boundary burst" problem that Fixed Window suffers from.

**5. Can the rate limiter itself become a bottleneck?**  
👉 Yes, especially if:
- Redis is overwhelmed (high QPS).
- Lua scripts aren't optimized.
- Logging adds latency.  
💡 Use caching, circuit breakers, and horizontal scaling of the limiter service.

---

## 🚀 Optimization Tips

- Use Redis TTLs smartly to avoid stale keys
- Lua scripting ensures atomic operations
- Maintain a denylist for blocked clients (avoid extra Redis hits)
- Use exponential backoff for retries
- Monitor Redis usage and evict expired rate keys

---

## 🧠 Python Logic Snippets (Essence)

### Fixed Window Counter
```
redis_key = f"{client_id}:{current_minute}"
if redis.incr(redis_key) > LIMIT:
    reject()
```
### Sliding Window Log (Sorted Set)
```
now = time.time()
redis.zadd(client_id, {now: now})
redis.zremrangebyscore(client_id, 0, now - WINDOW)
if redis.zcard(client_id) > LIMIT:
    reject()
```
### Sliding Window Counter (Time Buckets)
```
current_bucket = int(time.time() / BUCKET_SIZE)
key = f"{client_id}:{current_bucket}"
count = redis.incr(key)
redis.expire(key, BUCKET_SIZE * 2)
if count > LIMIT:
    reject()
```
### Token Bucket
```
refill = (now - last_ts) * rate
tokens = min(capacity, tokens + refill)
if tokens >= 1:
    tokens -= 1
else:
    reject()
```
### Leaky Bucket
```
elapsed = now - last_check
water_level = max(0, water_level - leak_rate * elapsed)
if water_level + 1 > capacity:
    reject()
else:
    water_level += 1
```
