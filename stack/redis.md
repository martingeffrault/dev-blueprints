# Redis (2025)

> **Last updated**: January 2026
> **Versions covered**: Redis 7.x
> **Purpose**: In-memory data store for caching, sessions, queues, and real-time data

---

## Philosophy (2025-2026)

Redis is the **blazing-fast in-memory data store** used for caching, session management, rate limiting, queues, and real-time leaderboards.

**Key philosophical shifts:**
- **Cache invalidation strategy** — TTL vs event-based
- **Redis Stack** — JSON, Search, Graph, TimeSeries modules
- **Cluster mode default** — For production high availability
- **Connection pooling** — Essential for performance
- **Persistence options** — RDB snapshots vs AOF append-only
- **Security required** — Authentication, ACLs, TLS

---

## TL;DR

- Always use connection pooling
- Set appropriate TTLs on all cached data
- Never use `KEYS *` in production — use SCAN
- Enable authentication (requirepass or ACLs)
- Use pipelines for multiple operations
- Monitor memory usage and eviction policies
- Don't store large blobs — keep values small
- Use appropriate data structures for the job

---

## Best Practices

### Connection Setup (Node.js/ioredis)

```typescript
// src/lib/redis.ts
import Redis from 'ioredis';

// Single instance
const redis = new Redis({
  host: process.env.REDIS_HOST ?? 'localhost',
  port: Number(process.env.REDIS_PORT ?? 6379),
  password: process.env.REDIS_PASSWORD,
  db: 0,
  retryDelayOnFailover: 100,
  maxRetriesPerRequest: 3,
  lazyConnect: true,
  // Connection pooling is built into ioredis
});

// Cluster mode
const cluster = new Redis.Cluster([
  { host: 'redis-node-1', port: 6379 },
  { host: 'redis-node-2', port: 6379 },
  { host: 'redis-node-3', port: 6379 },
], {
  redisOptions: {
    password: process.env.REDIS_PASSWORD,
  },
  scaleReads: 'slave',
});

export { redis, cluster };
```

### Basic Operations

```typescript
import { redis } from './lib/redis';

// String operations
await redis.set('user:123:name', 'John Doe');
await redis.set('session:abc', 'data', 'EX', 3600);  // 1 hour TTL
const name = await redis.get('user:123:name');

// Check existence
const exists = await redis.exists('user:123:name');

// Delete
await redis.del('user:123:name');

// Increment/Decrement
await redis.incr('page:views');
await redis.incrby('user:123:points', 10);
await redis.decr('inventory:item:5');

// Set with options
await redis.set('key', 'value', 'EX', 60, 'NX');  // Only if not exists
await redis.set('key', 'value', 'EX', 60, 'XX');  // Only if exists
```

### Caching Pattern

```typescript
// src/services/cache.ts
import { redis } from '../lib/redis';

interface CacheOptions {
  ttl?: number;  // seconds
}

export async function cached<T>(
  key: string,
  fetcher: () => Promise<T>,
  options: CacheOptions = {}
): Promise<T> {
  const { ttl = 3600 } = options;

  // Try cache first
  const cached = await redis.get(key);
  if (cached) {
    return JSON.parse(cached);
  }

  // Fetch and cache
  const data = await fetcher();
  await redis.set(key, JSON.stringify(data), 'EX', ttl);

  return data;
}

// Usage
const user = await cached(
  `user:${userId}`,
  () => db.user.findUnique({ where: { id: userId } }),
  { ttl: 300 }  // 5 minutes
);

// Cache invalidation
export async function invalidate(pattern: string): Promise<void> {
  // Use SCAN for pattern matching (never KEYS in production)
  let cursor = '0';
  do {
    const [nextCursor, keys] = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
    cursor = nextCursor;
    if (keys.length > 0) {
      await redis.del(...keys);
    }
  } while (cursor !== '0');
}

// Usage
await invalidate('user:123:*');
```

### Hash Operations (Objects)

```typescript
// Store user as hash
await redis.hset('user:123', {
  name: 'John Doe',
  email: 'john@example.com',
  role: 'admin',
  createdAt: Date.now().toString(),
});

// Get single field
const email = await redis.hget('user:123', 'email');

// Get multiple fields
const [name, role] = await redis.hmget('user:123', 'name', 'role');

// Get all fields
const user = await redis.hgetall('user:123');
// { name: 'John Doe', email: 'john@example.com', ... }

// Update single field
await redis.hset('user:123', 'role', 'superadmin');

// Increment hash field
await redis.hincrby('user:123', 'loginCount', 1);

// Delete field
await redis.hdel('user:123', 'temporaryField');

// Set TTL on hash
await redis.expire('user:123', 3600);
```

### Lists (Queues)

```typescript
// Queue pattern (FIFO)
await redis.rpush('queue:emails', JSON.stringify({ to: 'user@example.com', subject: 'Hello' }));
await redis.rpush('queue:emails', JSON.stringify({ to: 'other@example.com', subject: 'Hi' }));

// Pop from queue (blocking)
const [, item] = await redis.blpop('queue:emails', 5);  // 5 second timeout
const email = JSON.parse(item);

// Stack pattern (LIFO)
await redis.lpush('stack:undo', JSON.stringify(action));
const lastAction = await redis.lpop('stack:undo');

// Get list range
const recentItems = await redis.lrange('queue:emails', 0, 9);  // First 10

// List length
const queueLength = await redis.llen('queue:emails');
```

### Sets (Unique Collections)

```typescript
// Add to set
await redis.sadd('user:123:following', 'user:456', 'user:789');

// Check membership
const isFollowing = await redis.sismember('user:123:following', 'user:456');

// Get all members
const following = await redis.smembers('user:123:following');

// Set operations
const mutualFollowers = await redis.sinter('user:123:followers', 'user:456:followers');
const allFollowers = await redis.sunion('user:123:followers', 'user:456:followers');

// Random member
const randomUser = await redis.srandmember('active:users');

// Remove member
await redis.srem('user:123:following', 'user:789');

// Count members
const followerCount = await redis.scard('user:123:followers');
```

### Sorted Sets (Leaderboards)

```typescript
// Add scores
await redis.zadd('leaderboard:weekly', 1500, 'user:123');
await redis.zadd('leaderboard:weekly', 2000, 'user:456');
await redis.zadd('leaderboard:weekly', 1750, 'user:789');

// Get top 10
const topPlayers = await redis.zrevrange('leaderboard:weekly', 0, 9, 'WITHSCORES');
// ['user:456', '2000', 'user:789', '1750', 'user:123', '1500']

// Get rank (0-indexed)
const rank = await redis.zrevrank('leaderboard:weekly', 'user:123');

// Get score
const score = await redis.zscore('leaderboard:weekly', 'user:123');

// Increment score
await redis.zincrby('leaderboard:weekly', 100, 'user:123');

// Get players in score range
const midTier = await redis.zrangebyscore('leaderboard:weekly', 1000, 2000);

// Remove old entries
await redis.zremrangebyrank('leaderboard:weekly', 0, -101);  // Keep top 100
```

### Rate Limiting

```typescript
// Sliding window rate limiter
export async function rateLimit(
  key: string,
  limit: number,
  windowSeconds: number
): Promise<{ allowed: boolean; remaining: number }> {
  const now = Date.now();
  const windowStart = now - (windowSeconds * 1000);

  const multi = redis.multi();

  // Remove old entries
  multi.zremrangebyscore(key, 0, windowStart);

  // Add current request
  multi.zadd(key, now, `${now}-${Math.random()}`);

  // Count requests in window
  multi.zcard(key);

  // Set TTL
  multi.expire(key, windowSeconds);

  const results = await multi.exec();
  const count = results![2][1] as number;

  return {
    allowed: count <= limit,
    remaining: Math.max(0, limit - count),
  };
}

// Usage
const { allowed, remaining } = await rateLimit(`ratelimit:${userId}`, 100, 60);
if (!allowed) {
  throw new Error('Rate limit exceeded');
}
```

### Pipelines (Batch Operations)

```typescript
// Pipeline for multiple operations
const pipeline = redis.pipeline();

pipeline.get('user:123:name');
pipeline.hgetall('user:123:profile');
pipeline.smembers('user:123:roles');
pipeline.zrevrange('user:123:activity', 0, 9);

const results = await pipeline.exec();
// [[null, 'John'], [null, {...}], [null, ['admin']], [null, [...]]]

// Transaction (atomic)
const multi = redis.multi();

multi.decrby('inventory:item:5', 1);
multi.rpush('orders:pending', orderId);
multi.hincrby('stats:daily', 'orders', 1);

await multi.exec();
```

### Pub/Sub

```typescript
// Publisher
await redis.publish('notifications', JSON.stringify({
  type: 'new_message',
  userId: '123',
  data: { messageId: 'abc' },
}));

// Subscriber (separate connection)
const subscriber = new Redis();

subscriber.subscribe('notifications', (err, count) => {
  console.log(`Subscribed to ${count} channels`);
});

subscriber.on('message', (channel, message) => {
  const data = JSON.parse(message);
  console.log(`Received on ${channel}:`, data);
});

// Pattern subscribe
subscriber.psubscribe('user:*:events');
subscriber.on('pmessage', (pattern, channel, message) => {
  console.log(`Pattern ${pattern}, channel ${channel}:`, message);
});
```

### Session Store (Express)

```typescript
import session from 'express-session';
import RedisStore from 'connect-redis';
import { redis } from './lib/redis';

app.use(session({
  store: new RedisStore({ client: redis }),
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 1000 * 60 * 60 * 24,  // 1 day
  },
}));
```

---

## Anti-Patterns

### ❌ Using KEYS Command in Production

**Why it's bad**: Blocks server, scans entire keyspace.

```typescript
// ❌ DON'T — Blocks the entire Redis instance
const keys = await redis.keys('user:*');

// ✅ DO — Use SCAN with cursor
async function scanKeys(pattern: string): Promise<string[]> {
  const keys: string[] = [];
  let cursor = '0';

  do {
    const [nextCursor, batch] = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
    cursor = nextCursor;
    keys.push(...batch);
  } while (cursor !== '0');

  return keys;
}
```

### ❌ No Connection Pooling

**Why it's bad**: Connection overhead per request.

```typescript
// ❌ DON'T — New connection per request
async function getUser(id: string) {
  const redis = new Redis();  // Creates new connection!
  const user = await redis.get(`user:${id}`);
  await redis.quit();
  return user;
}

// ✅ DO — Reuse connection pool (ioredis handles this)
import { redis } from './lib/redis';

async function getUser(id: string) {
  return redis.get(`user:${id}`);
}
```

### ❌ Storing Large Values

**Why it's bad**: Blocks network, memory issues.

```typescript
// ❌ DON'T — Store large blobs
await redis.set('file', largeBuffer);  // Several MB

// ✅ DO — Store references, keep data in object storage
await redis.set('file:123:url', 's3://bucket/file.pdf');
await redis.set('file:123:meta', JSON.stringify({ size: 1024, type: 'pdf' }));
```

### ❌ No TTL on Cached Data

**Why it's bad**: Memory grows unbounded, stale data.

```typescript
// ❌ DON'T — No expiration
await redis.set('cache:user:123', JSON.stringify(user));

// ✅ DO — Always set TTL
await redis.set('cache:user:123', JSON.stringify(user), 'EX', 3600);
```

### ❌ No Authentication

**Why it's bad**: Anyone can access your data.

```typescript
// ❌ DON'T — No password in production
const redis = new Redis({ host: 'redis.example.com' });

// ✅ DO — Enable authentication
const redis = new Redis({
  host: 'redis.example.com',
  password: process.env.REDIS_PASSWORD,
  tls: process.env.NODE_ENV === 'production' ? {} : undefined,
});
```

---

## 2025-2026 Changelog

| Feature | Date | Description |
|---------|------|-------------|
| Redis 7.0 | 2022 | Functions, ACLv2, Sharded Pub/Sub |
| Redis 7.2 | 2023 | Triggers, improved cluster |
| Redis Stack | 2024+ | JSON, Search, Graph, TimeSeries |
| Valkey | 2024 | Open source Redis fork |

---

## Quick Reference

| Data Type | Use Case | Key Commands |
|-----------|----------|--------------|
| String | Cache, counters | GET, SET, INCR |
| Hash | Objects, profiles | HGET, HSET, HGETALL |
| List | Queues, feeds | LPUSH, RPOP, LRANGE |
| Set | Tags, unique items | SADD, SISMEMBER, SINTER |
| Sorted Set | Leaderboards, ranges | ZADD, ZRANGE, ZRANK |
| Stream | Event logs, messaging | XADD, XREAD, XGROUP |

| Command | Purpose |
|---------|---------|
| `SET key value EX 60` | Set with 60s TTL |
| `GET key` | Get value |
| `DEL key` | Delete key |
| `EXISTS key` | Check existence |
| `EXPIRE key 60` | Set TTL |
| `TTL key` | Get remaining TTL |
| `SCAN cursor MATCH pattern` | Safe iteration |
| `PIPELINE` | Batch commands |
| `MULTI/EXEC` | Transaction |

---

## Resources

- [Official Redis Documentation](https://redis.io/docs/)
- [Redis Best Practices](https://redis.io/faq/doc/1mebipyp1e/performance-tuning-best-practices)
- [Redis Performance Tuning](https://www.percona.com/blog/redis-performance-best-practices/)
- [Redis in Docker](https://hub.docker.com/_/redis)
- [ioredis Documentation](https://github.com/redis/ioredis)
- [Redis Monitoring](https://www.groundcover.com/blog/monitor-redis)
