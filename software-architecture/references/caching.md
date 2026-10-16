# Caching Strategies

Pseudocode conventions are in [SKILL.md](../SKILL.md#pseudocode-conventions). Redis/Valkey is used as the example distributed cache; the patterns apply to Memcached, cloud caches, CDN/edge caches, and in-process caches.

## Table of Contents

1. [Before You Cache](#before-you-cache)
2. [Cache Port and Adapter](#cache-port-and-adapter)
3. [Cached Repository Decorator](#cached-repository-decorator)
4. [Cache Patterns](#cache-patterns)
5. [Stampede Protection](#stampede-protection)
6. [Multi-Level Caching](#multi-level-caching)
7. [Keys, TTLs, and Invalidation](#keys-ttls-and-invalidation)
8. [HTTP and Client-Side Caching](#http-and-client-side-caching)

---

## Before You Cache

Caching trades **freshness and complexity** for **latency and load**. First check: is the slow thing a missing index, an N+1, or an oversized payload? Fix that first. Then decide explicitly:
- How stale may this data be? (that's your TTL)
- What happens on a cache outage? (the system must still work, just slower)
- Who invalidates it, and when?

---

## Cache Port and Adapter

```
interface Cache {
  get(key: string) -> bytes?
  getMany(keys: List<string>) -> Map<string, bytes?>
  set(key: string, value: bytes, ttl: Duration) -> void
  setMany(entries: Map<string, bytes>, ttl: Duration) -> void
  delete(key: string) -> void
  setIfAbsent(key: string, value: bytes, ttl: Duration) -> bool    // for locks / dedupe
}

class RedisCache(client: RedisClient) implements Cache {     // client from a pooled connection
  get(key)               { return client.GET(key) }
  getMany(keys)          { return zip(keys, client.MGET(keys)) }
  set(key, value, ttl)   { client.SET(key, value, EX = ttl.seconds) }
  setMany(entries, ttl)  { pipeline { for (k, v) in entries: SET(k, v, EX = ttl.seconds) } }
  delete(key)            { client.DEL(key) }
  setIfAbsent(k, v, ttl) { return client.SET(k, v, NX, EX = ttl.seconds) == OK }
}
```

**Cache failures must not fail the request.** Wrap calls so a cache timeout or connection error is logged/metered and treated as a miss. Set short timeouts on cache operations (tens of milliseconds).

**Serialisation**: use an explicit, versioned format (JSON, MessagePack, Protobuf). Include a schema version in the key or payload so a deploy that changes the shape doesn't crash on old entries.

---

## Cached Repository Decorator

Adds caching without touching use cases or the real repository:

```
class CachedUserRepository(inner: UserRepository, cache: Cache, ttl: Duration) implements UserRepository {
  getById(id) -> User? {
    key = "user:v1:" + id
    hit = safely { cache.get(key) }
    if hit is present: return deserialize(hit)

    user = inner.getById(id)
    if user is present: safely { cache.set(key, serialize(user), ttl) }
    return user
  }

  save(user) {
    inner.save(user)
    safely { cache.delete("user:v1:" + user.id) }   // delete-on-write is safer than set-on-write
  }
}
```

---

## Cache Patterns

| Pattern | How | Use when | Risk |
|---|---|---|---|
| **Cache-aside (lazy)** | Read: cache → miss → DB → fill. Write: DB → delete key. | Default for most reads | First request after expiry is slow; brief staleness |
| **Read-through** | The cache library loads from the DB on a miss | You have a caching library/loader abstraction | Same as cache-aside |
| **Write-through** | Write DB, then update cache synchronously | Read-heavy data that must be fresh right after writes | Race: concurrent writers can leave the older value in cache — prefer delete-on-write |
| **Write-behind** | Write cache, queue the DB write | Very high write rates where some loss is acceptable | **Data loss** if the cache dies before flush; rarely appropriate for business data |
| **Refresh-ahead / warming** | Refresh hot keys before expiry, or preload at startup | Predictable hot set (home page, config, top products) | Wasted work on keys nobody reads |

```
// Cache-aside
getProduct(id) {
  cached = cache.get("product:v1:" + id)
  if cached is present: return deserialize(cached)
  product = db.getProduct(id)
  if product is present: cache.set("product:v1:" + id, serialize(product), ttl = 10 minutes)
  return product
}

// Warming
warmPopularProducts() {
  ids = analytics.topProductIds(limit = 1000)
  cache.setMany({ "product:v1:" + p.id: serialize(p) for p in db.getProducts(ids) }, ttl = 1 hour)
}
```

---

## Stampede Protection

When a hot key expires, hundreds of concurrent requests can all miss and hammer the database. Mitigations, from simplest:

1. **TTL jitter** — `ttl = base × random(0.9, 1.1)` so keys don't expire together.
2. **Request coalescing (single-flight)** — within one process, concurrent misses for the same key share one load.
3. **Distributed lock** — across instances, only the lock holder recomputes; others briefly wait or serve stale.
4. **Stale-while-revalidate** — store a soft expiry inside the value; serve stale and refresh in the background.

```
// Single-flight within a process: one in-flight load per key, removed when done
class SingleFlight {
  private inFlight: ConcurrentMap<string, Future<Value>>

  do(key, load: () -> Value) -> Value {
    (future, isLeader) = inFlight.getOrCreate(key, () -> Future.start(load))
    try { return await future }
    finally { if isLeader: inFlight.remove(key) }   // don't let the map grow forever
  }
}

getOrCompute(key, compute, ttl) -> Value {
  hit = cache.get(key)
  if hit is present: return deserialize(hit)
  return singleFlight.do(key, () -> {
    hit = cache.get(key)                         // double-check after winning
    if hit is present: return deserialize(hit)
    value = compute()
    cache.set(key, serialize(value), ttl × random(0.9, 1.1))
    return value
  })
}

// Distributed variant: lock via SET NX with a short TTL and a unique token
if cache.setIfAbsent("lock:" + key, token, ttl = 10 seconds):
  try { recompute and fill } finally { deleteIfValueEquals("lock:" + key, token) }  // atomic compare-and-delete (Lua script)
else:
  wait briefly and re-read, or serve stale
```

Built-ins: Go `golang.org/x/sync/singleflight`, Caffeine/Guava `LoadingCache` (Java), `HybridCache`/`IMemoryCache.GetOrCreate` (.NET), `dataloader` (JS), Rails `race_condition_ttl`.

---

## Multi-Level Caching

```
L1: in-process LRU (microseconds, per instance, small, short TTL)
L2: distributed cache (sub-millisecond to ms, shared, larger, longer TTL)
L3: database / origin
```

```
class TwoLevelCache(l1: InMemoryLru, l2: Cache) {
  get(key) -> Value? {
    v = l1.get(key)
    if v is present: return v
    raw = l2.get(key)
    if raw is present:
      v = deserialize(raw)
      l1.set(key, v, ttl = 30 seconds)        // promote with a shorter TTL
      return v
    return none
  }
  set(key, value, ttl) {
    l2.set(key, serialize(value), ttl)
    l1.set(key, value, min(ttl, 30 seconds))
  }
  invalidate(key) {
    l2.delete(key)
    l1.delete(key)
    broadcast("invalidate", key)              // pub/sub so other instances drop their L1 copy
  }
}
```

Use a real bounded LRU/LFU from the ecosystem (Caffeine, `MemoryCache`, `lru-cache`, `cachetools`, `ristretto`/`otter`, `moka`, Cachex) rather than a hand-rolled map. Keep L1 TTLs short since it can't be invalidated reliably without pub/sub.

---

## Keys, TTLs, and Invalidation

- **Key format**: `<service>:<entity>:<version>:<id>[:<variant>]` — e.g. `orders:order:v2:123`. Bump the version when the cached shape changes.
- **Never cache per-user data under a shared key**; include tenant/user ID in the key for anything personalised or permission-filtered.
- **TTLs on everything** — even "invalidated on write" entries — as a safety net against missed invalidations.
- **Invalidate on events**: when a service publishes `ProductUpdated`, consumers delete their cached copies.
- **Negative caching**: cache "not found" briefly (seconds) to protect against repeated lookups for missing keys.
- **Measure**: hit rate, latency, evictions, memory. A cache with a low hit rate is pure overhead.

---

## HTTP and Client-Side Caching

Often the cheapest cache is the one in front of your service:

- `Cache-Control: public, max-age=60, stale-while-revalidate=300` for shared, non-personal responses (CDN-friendly).
- `Cache-Control: private, no-store` for personal or sensitive data.
- `ETag` / `If-None-Match` → `304 Not Modified` saves bandwidth even when content must be revalidated.
- CDN/edge caching for static assets with content-hashed filenames and long `max-age`, `immutable`.
- Client apps: see [client-applications.md](client-applications.md#data-and-caching).
