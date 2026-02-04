# Caching Strategies

## Table of Contents

1. [Redis Cache Implementation](#redis-cache-implementation)
2. [Cache Patterns](#cache-patterns)
3. [Multi-Level Caching](#multi-level-caching)

---

## Redis Cache Implementation

```python
from redis.asyncio import Redis, ConnectionPool
from abc import ABC, abstractmethod

class CacheService(ABC):
    @abstractmethod
    async def get(self, key: str) -> Optional[str]:
        pass

    @abstractmethod
    async def set(self, key: str, value: str, ttl: int = 300) -> None:
        pass

    @abstractmethod
    async def delete(self, key: str) -> None:
        pass

    @abstractmethod
    async def exists(self, key: str) -> bool:
        pass

class RedisCacheService(CacheService):
    def __init__(self, redis_url: str):
        self._pool = ConnectionPool.from_url(redis_url, max_connections=50, decode_responses=True)
        self._redis = Redis(connection_pool=self._pool)

    async def get(self, key: str) -> Optional[str]:
        return await self._redis.get(key)

    async def set(self, key: str, value: str, ttl: int = 300) -> None:
        await self._redis.setex(key, ttl, value)

    async def delete(self, key: str) -> None:
        await self._redis.delete(key)

    async def exists(self, key: str) -> bool:
        return bool(await self._redis.exists(key))

    async def get_many(self, keys: list[str]) -> dict[str, Optional[str]]:
        values = await self._redis.mget(keys)
        return dict(zip(keys, values))

    async def set_many(self, items: dict[str, str], ttl: int = 300) -> None:
        async with self._redis.pipeline() as pipe:
            for key, value in items.items():
                pipe.setex(key, ttl, value)
            await pipe.execute()

    async def close(self) -> None:
        await self._redis.close()
        await self._pool.disconnect()

# Cached repository decorator
class CachedUserRepository(UserRepository):
    def __init__(self, repo: UserRepository, cache: CacheService):
        self._repo = repo
        self._cache = cache

    async def get_by_id(self, user_id: str) -> Optional[User]:
        cache_key = f"user:{user_id}"
        cached = await self._cache.get(cache_key)

        if cached:
            return User.parse_raw(cached)

        user = await self._repo.get_by_id(user_id)

        if user:
            await self._cache.set(cache_key, user.json(), ttl=300)

        return user

    async def save(self, user: User) -> User:
        saved = await self._repo.save(user)
        await self._cache.delete(f"user:{user.id}")
        return saved
```

---

## Cache Patterns

```python
# 1. Cache-Aside (Lazy Loading)
async def get_user_cache_aside(user_id: str) -> User:
    cached = await cache.get(f"user:{user_id}")
    if cached:
        return User.parse_raw(cached)

    user = await db.get_user(user_id)
    if user:
        await cache.set(f"user:{user_id}", user.json(), ttl=300)
    return user

# 2. Write-Through
async def save_user_write_through(user: User) -> User:
    saved = await db.save_user(user)  # Write to DB first
    await cache.set(f"user:{user.id}", saved.json(), ttl=300)  # Then update cache
    return saved

# 3. Write-Behind (Async)
async def save_user_write_behind(user: User) -> User:
    await cache.set(f"user:{user.id}", user.json(), ttl=300)  # Update cache immediately
    await task_queue.enqueue(save_user_to_db, user)  # Queue DB write
    return user

# 4. Cache Warming
async def warm_cache():
    popular_user_ids = await db.get_popular_user_ids(limit=1000)
    users = await db.get_users_by_ids(popular_user_ids)

    items = {f"user:{user.id}": user.json() for user in users}
    await cache.set_many(items, ttl=3600)

# 5. Cache Stampede Prevention
from asyncio import Lock

class StampedeProtectedCache:
    def __init__(self, cache: CacheService):
        self._cache = cache
        self._locks: dict[str, Lock] = {}

    async def get_or_compute(
        self,
        key: str,
        compute_fn: Callable[[], Awaitable[Any]],
        ttl: int = 300
    ) -> Any:
        cached = await self._cache.get(key)
        if cached:
            return json.loads(cached)

        if key not in self._locks:
            self._locks[key] = Lock()

        async with self._locks[key]:
            # Double-check (another request might have filled it)
            cached = await self._cache.get(key)
            if cached:
                return json.loads(cached)

            value = await compute_fn()
            await self._cache.set(key, json.dumps(value), ttl)
            return value
```

---

## Multi-Level Caching

```python
import time

class MultiLevelCache(Generic[T]):
    """L1: In-memory (LRU), L2: Redis"""

    def __init__(self, redis: CacheService, max_size: int = 1000):
        self._redis = redis
        self._max_size = max_size
        self._memory_cache: dict[str, tuple[T, float]] = {}

    async def get(self, key: str) -> Optional[T]:
        # L1: Check memory
        if key in self._memory_cache:
            value, expires_at = self._memory_cache[key]
            if time.time() < expires_at:
                return value
            else:
                del self._memory_cache[key]

        # L2: Check Redis
        cached = await self._redis.get(key)
        if cached:
            value = json.loads(cached)
            self._set_memory(key, value, ttl=60)  # Promote to L1
            return value

        return None

    async def set(self, key: str, value: T, ttl: int = 300) -> None:
        self._set_memory(key, value, ttl=min(ttl, 60))
        await self._redis.set(key, json.dumps(value), ttl)

    def _set_memory(self, key: str, value: T, ttl: int) -> None:
        # Simple LRU eviction
        if len(self._memory_cache) >= self._max_size:
            oldest_key = next(iter(self._memory_cache))
            del self._memory_cache[oldest_key]

        self._memory_cache[key] = (value, time.time() + ttl)
```
