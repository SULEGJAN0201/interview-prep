# Caching Strategies & Implementation

## 📝 Overview

**Caching** is storing frequently accessed data in a fast-access layer to reduce latency, database load, and costs.

**Key Trade-off**: Speed vs. Data Freshness

**As a Tech Lead**: You need to know when to cache, what to cache, and how to handle cache invalidation (the hard problem).

**Phil Karlton's Famous Quote**: "There are only two hard things in Computer Science: cache invalidation and naming things."

---

## 🎯 Why Cache?

### The Performance Problem

```csharp
// ❌ WITHOUT CACHING: Slow, expensive
public async Task<Product> GetProductAsync(int id)
{
    // Every request hits the database
    return await _dbContext.Products
        .Include(p => p.Category)
        .Include(p => p.Reviews)
        .FirstOrDefaultAsync(p => p.Id == id);

    // 50ms database query
    // × 1000 requests/sec = 50,000ms = 50 seconds of DB time!
    // Database is bottleneck
}

// ✅ WITH CACHING: Fast, efficient
public async Task<Product> GetProductAsync(int id)
{
    var cacheKey = $"product:{id}";

    // Try cache first (< 1ms)
    var cachedProduct = await _cache.GetAsync<Product>(cacheKey);
    if (cachedProduct != null)
        return cachedProduct;

    // Cache miss: fetch from DB
    var product = await _dbContext.Products
        .Include(p => p.Category)
        .Include(p => p.Reviews)
        .FirstOrDefaultAsync(p => p.Id == id);

    // Store in cache for 5 minutes
    await _cache.SetAsync(cacheKey, product, TimeSpan.FromMinutes(5));

    return product;
}
```

**Impact**:
- Latency: 50ms → 1ms (50x faster!)
- Database load: 1000 queries/sec → 10 queries/sec (100x reduction)
- Cost: Reduce database scaling needs

---

## 🗂️ Caching Layers

### The Caching Hierarchy

```
┌─────────────────────────────────┐
│   Browser Cache (HTTP Cache)    │ ← Fastest, client-side
├─────────────────────────────────┤
│   CDN (CloudFlare, CloudFront)  │ ← Static assets, edge locations
├─────────────────────────────────┤
│   Application Memory (IMemoryCache) │ ← In-process, local
├─────────────────────────────────┤
│   Distributed Cache (Redis)      │ ← Shared across servers
├─────────────────────────────────┤
│   Database Query Cache           │ ← Database-level
├─────────────────────────────────┤
│   Database (Source of Truth)     │ ← Slowest, persistent
└─────────────────────────────────┘
```

### Layer Comparison

| Layer | Latency | Shared? | Use Case |
|-------|---------|---------|----------|
| **Browser** | ~0ms | No (per user) | Static assets, API responses |
| **CDN** | 10-50ms | Yes (global) | Images, JS, CSS, static HTML |
| **Memory Cache** | <1ms | No (per server) | Reference data, sessions |
| **Redis** | 1-5ms | Yes (multi-server) | User sessions, API responses |
| **Database** | 10-100ms | Yes | Source of truth |

---

## 📦 Caching Patterns

### 1. Cache-Aside (Lazy Loading)

**Most Common Pattern**

```csharp
public class ProductService
{
    private readonly IDistributedCache _cache;
    private readonly IProductRepository _repository;

    public async Task<Product> GetProductAsync(int id)
    {
        var cacheKey = $"product:{id}";

        // 1. Try cache first
        var cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null)
        {
            return JsonSerializer.Deserialize<Product>(cached);
        }

        // 2. Cache miss: fetch from database
        var product = await _repository.GetByIdAsync(id);
        if (product == null)
            return null;

        // 3. Store in cache for future requests
        var serialized = JsonSerializer.Serialize(product);
        await _cache.SetStringAsync(
            cacheKey,
            serialized,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
            }
        );

        return product;
    }
}
```

**Pros**:
- Simple to implement
- Cache only what's requested (efficient)
- Resilient (cache failure doesn't break app)

**Cons**:
- First request is always slow (cache miss)
- Cache stampede risk (many requests miss cache simultaneously)

**When to Use**: Read-heavy data, acceptable to have some cache misses

---

### 2. Read-Through Cache

**Cache handles database access**

```csharp
public class ReadThroughCache<T>
{
    private readonly IDistributedCache _cache;
    private readonly Func<string, Task<T>> _loadFunction;

    public ReadThroughCache(
        IDistributedCache cache,
        Func<string, Task<T>> loadFunction)
    {
        _cache = cache;
        _loadFunction = loadFunction;
    }

    public async Task<T> GetAsync(string key)
    {
        // Try cache
        var cached = await _cache.GetStringAsync(key);
        if (cached != null)
        {
            return JsonSerializer.Deserialize<T>(cached);
        }

        // Cache miss: load from source (database)
        var value = await _loadFunction(key);

        // Store in cache
        var serialized = JsonSerializer.Serialize(value);
        await _cache.SetStringAsync(key, serialized);

        return value;
    }
}

// Usage
var cache = new ReadThroughCache<Product>(
    distributedCache,
    async (id) => await _repository.GetByIdAsync(int.Parse(id))
);

var product = await cache.GetAsync("123");
```

**Pros**:
- Cleaner separation (cache logic separate from business logic)
- Consistent caching behavior

**Cons**:
- More abstraction
- Tight coupling between cache and data source

**When to Use**: Want to centralize caching logic, multiple services use same data

---

### 3. Write-Through Cache

**Write to cache AND database simultaneously**

```csharp
public class ProductService
{
    private readonly IDistributedCache _cache;
    private readonly IProductRepository _repository;

    public async Task UpdateProductAsync(Product product)
    {
        // 1. Update database
        await _repository.UpdateAsync(product);

        // 2. Update cache immediately
        var cacheKey = $"product:{product.Id}";
        var serialized = JsonSerializer.Serialize(product);
        await _cache.SetStringAsync(
            cacheKey,
            serialized,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
            }
        );
    }
}
```

**Pros**:
- Cache always fresh
- No stale data
- Read performance consistent

**Cons**:
- Write latency increased (two operations)
- If cache write fails, cache becomes stale

**When to Use**: Data must be always fresh, write frequency is low

---

### 4. Write-Behind (Write-Back) Cache

**Write to cache first, database later (async)**

```csharp
public class WriteBehindCache
{
    private readonly IDistributedCache _cache;
    private readonly IBackgroundTaskQueue _taskQueue;

    public async Task UpdateProductAsync(Product product)
    {
        // 1. Write to cache immediately (fast)
        var cacheKey = $"product:{product.Id}";
        var serialized = JsonSerializer.Serialize(product);
        await _cache.SetStringAsync(cacheKey, serialized);

        // 2. Queue database write (async, later)
        await _taskQueue.QueueBackgroundWorkItemAsync(async token =>
        {
            await _repository.UpdateAsync(product);
        });
    }
}

// Background worker
public class DatabaseSyncWorker : BackgroundService
{
    private readonly IBackgroundTaskQueue _taskQueue;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var workItem = await _taskQueue.DequeueAsync(stoppingToken);
            await workItem(stoppingToken);
        }
    }
}
```

**Pros**:
- Ultra-fast writes
- Reduced database load
- Can batch database writes

**Cons**:
- Risk of data loss (if cache fails before DB write)
- Complexity (background workers, retry logic)

**When to Use**: Write-heavy workloads, eventual consistency is acceptable (e.g., view counts, analytics)

---

### 5. Cache-Aside with Refresh-Ahead

**Proactively refresh cache before expiration**

```csharp
public class RefreshAheadCache
{
    private readonly IDistributedCache _cache;
    private readonly IProductRepository _repository;

    public async Task<Product> GetProductAsync(int id)
    {
        var cacheKey = $"product:{id}";

        // Try cache
        var cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null)
        {
            var product = JsonSerializer.Deserialize<Product>(cached);

            // Check if cache is about to expire (e.g., < 1 minute left)
            var metadata = await _cache.GetAsync($"{cacheKey}:metadata");
            if (metadata != null)
            {
                var expiration = JsonSerializer.Deserialize<DateTime>(metadata);
                if (expiration < DateTime.UtcNow.AddMinutes(1))
                {
                    // Refresh cache in background
                    _ = Task.Run(async () =>
                    {
                        var fresh = await _repository.GetByIdAsync(id);
                        await SetCacheAsync(cacheKey, fresh);
                    });
                }
            }

            return product;
        }

        // Cache miss: load and cache
        var productFromDb = await _repository.GetByIdAsync(id);
        await SetCacheAsync(cacheKey, productFromDb);
        return productFromDb;
    }

    private async Task SetCacheAsync(string key, Product product)
    {
        var serialized = JsonSerializer.Serialize(product);
        await _cache.SetStringAsync(key, serialized,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
            });

        // Store expiration metadata
        var expiration = DateTime.UtcNow.AddMinutes(5);
        await _cache.SetStringAsync($"{key}:metadata",
            JsonSerializer.Serialize(expiration));
    }
}
```

**Pros**:
- Avoids cache misses for popular items
- Consistent low latency

**Cons**:
- Complexity
- May refresh items that won't be accessed again

**When to Use**: Hot data (frequently accessed), latency-critical

---

## ⚡ Redis Implementation (C#)

### Setup

```csharp
// Startup.cs / Program.cs
public void ConfigureServices(IServiceCollection services)
{
    // Add Redis distributed cache
    services.AddStackExchangeRedisCache(options =>
    {
        options.Configuration = configuration["Redis:ConnectionString"];
        options.InstanceName = "MyApp:";
    });

    services.AddScoped<IProductService, ProductService>();
}
```

### Basic Operations

```csharp
public class ProductService
{
    private readonly IDistributedCache _cache;

    // GET
    public async Task<Product> GetAsync(int id)
    {
        var key = $"product:{id}";
        var cached = await _cache.GetStringAsync(key);

        if (cached != null)
            return JsonSerializer.Deserialize<Product>(cached);

        return null;
    }

    // SET with expiration
    public async Task SetAsync(Product product)
    {
        var key = $"product:{product.Id}";
        var serialized = JsonSerializer.Serialize(product);

        await _cache.SetStringAsync(key, serialized,
            new DistributedCacheEntryOptions
            {
                // Absolute expiration (5 minutes from now)
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5),

                // OR Sliding expiration (reset on access)
                // SlidingExpiration = TimeSpan.FromMinutes(5)
            });
    }

    // DELETE
    public async Task RemoveAsync(int id)
    {
        var key = $"product:{id}";
        await _cache.RemoveAsync(key);
    }
}
```

### Advanced: Complex Objects

```csharp
public class CacheService<T>
{
    private readonly IDistributedCache _cache;

    public async Task<T> GetAsync(string key)
    {
        var bytes = await _cache.GetAsync(key);
        if (bytes == null) return default;

        return JsonSerializer.Deserialize<T>(bytes);
    }

    public async Task SetAsync(string key, T value, TimeSpan? expiration = null)
    {
        var serialized = JsonSerializer.Serialize(value);
        var bytes = Encoding.UTF8.GetBytes(serialized);

        var options = new DistributedCacheEntryOptions();
        if (expiration.HasValue)
        {
            options.AbsoluteExpirationRelativeToNow = expiration.Value;
        }

        await _cache.SetAsync(key, bytes, options);
    }

    public async Task RemoveAsync(string key)
    {
        await _cache.RemoveAsync(key);
    }
}

// Usage
var cacheService = new CacheService<Product>(_cache);

await cacheService.SetAsync("product:123", product, TimeSpan.FromMinutes(5));
var cached = await cacheService.GetAsync("product:123");
```

---

## 🧹 Cache Invalidation Strategies

### 1. Time-Based Expiration (TTL)

**Simplest approach**: Cache expires after fixed time

```csharp
// Absolute expiration (expires at specific time)
await _cache.SetStringAsync(key, value,
    new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
    });

// Sliding expiration (resets on access)
await _cache.SetStringAsync(key, value,
    new DistributedCacheEntryOptions
    {
        SlidingExpiration = TimeSpan.FromMinutes(5)
        // Expires 5 minutes after LAST access
    });
```

**When to Use**:
- Data changes infrequently
- Eventual consistency is acceptable
- Simple scenario

**Pros**: Simple, automatic
**Cons**: May serve stale data until expiration

---

### 2. Manual Invalidation

**Explicitly remove from cache when data changes**

```csharp
public class ProductService
{
    private readonly IDistributedCache _cache;
    private readonly IProductRepository _repository;

    public async Task UpdateProductAsync(Product product)
    {
        // 1. Update database
        await _repository.UpdateAsync(product);

        // 2. Invalidate cache
        var cacheKey = $"product:{product.Id}";
        await _cache.RemoveAsync(cacheKey);

        // Next read will fetch fresh data from database
    }

    public async Task DeleteProductAsync(int id)
    {
        await _repository.DeleteAsync(id);

        // Invalidate all related caches
        await _cache.RemoveAsync($"product:{id}");
        await _cache.RemoveAsync($"products:all");
        await _cache.RemoveAsync($"products:category:{product.CategoryId}");
    }
}
```

**When to Use**: Data must be immediately consistent

**Pros**: Guarantees fresh data
**Cons**: Must remember to invalidate everywhere (easy to forget!)

---

### 3. Event-Based Invalidation

**Use events/messages to invalidate cache across services**

```csharp
// ProductService (publishes events)
public class ProductService
{
    private readonly IMessageBus _messageBus;

    public async Task UpdateProductAsync(Product product)
    {
        await _repository.UpdateAsync(product);

        // Publish event
        await _messageBus.PublishAsync(new ProductUpdatedEvent
        {
            ProductId = product.Id,
            CategoryId = product.CategoryId
        });
    }
}

// CacheInvalidationService (listens to events)
public class CacheInvalidationService : IEventHandler<ProductUpdatedEvent>
{
    private readonly IDistributedCache _cache;

    public async Task HandleAsync(ProductUpdatedEvent evt)
    {
        // Invalidate related caches
        await _cache.RemoveAsync($"product:{evt.ProductId}");
        await _cache.RemoveAsync($"products:category:{evt.CategoryId}");
        await _cache.RemoveAsync("products:all");
    }
}
```

**When to Use**: Microservices architecture, multiple services share cache

**Pros**: Decoupled, scalable
**Cons**: Eventual consistency (small delay), complexity

---

### 4. Cache Tagging

**Tag cache entries, invalidate by tag**

```csharp
public class TaggedCache
{
    private readonly IDistributedCache _cache;

    public async Task SetAsync<T>(string key, T value, string[] tags, TimeSpan expiration)
    {
        // Store value
        await SetCacheAsync(key, value, expiration);

        // Store tags
        foreach (var tag in tags)
        {
            var tagKey = $"tag:{tag}";
            var keys = await GetTagKeysAsync(tagKey) ?? new HashSet<string>();
            keys.Add(key);
            await SetCacheAsync(tagKey, keys, expiration);
        }
    }

    public async Task InvalidateByTagAsync(string tag)
    {
        var tagKey = $"tag:{tag}";
        var keys = await GetTagKeysAsync(tagKey);

        if (keys != null)
        {
            foreach (var key in keys)
            {
                await _cache.RemoveAsync(key);
            }
            await _cache.RemoveAsync(tagKey);
        }
    }

    private async Task<HashSet<string>> GetTagKeysAsync(string tagKey)
    {
        var cached = await _cache.GetStringAsync(tagKey);
        return cached != null
            ? JsonSerializer.Deserialize<HashSet<string>>(cached)
            : null;
    }

    private async Task SetCacheAsync<T>(string key, T value, TimeSpan expiration)
    {
        var serialized = JsonSerializer.Serialize(value);
        await _cache.SetStringAsync(key, serialized,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = expiration
            });
    }
}

// Usage
var cache = new TaggedCache(_distributedCache);

// Cache product with tags
await cache.SetAsync(
    key: "product:123",
    value: product,
    tags: new[] { "product", "category:electronics", "brand:apple" },
    expiration: TimeSpan.FromMinutes(5)
);

// Invalidate all products in "category:electronics"
await cache.InvalidateByTagAsync("category:electronics");
```

**When to Use**: Complex invalidation logic, related entities

**Pros**: Flexible, precise invalidation
**Cons**: Overhead (storing tags), complexity

---

## ⚠️ Common Caching Pitfalls

### 1. Cache Stampede

**Problem**: Many requests hit cache miss simultaneously, all query database

```csharp
// ❌ BAD: Cache stampede
public async Task<Product> GetProductAsync(int id)
{
    var cached = await _cache.GetAsync(key);
    if (cached != null) return cached;

    // If 1000 requests hit this simultaneously:
    // → 1000 database queries!
    var product = await _repository.GetByIdAsync(id);
    await _cache.SetAsync(key, product);
    return product;
}

// ✅ GOOD: Use locking to prevent stampede
private static readonly SemaphoreSlim _semaphore = new SemaphoreSlim(1, 1);

public async Task<Product> GetProductAsync(int id)
{
    var cached = await _cache.GetAsync(key);
    if (cached != null) return cached;

    // Only one request fetches from database
    await _semaphore.WaitAsync();
    try
    {
        // Double-check cache (another thread may have filled it)
        cached = await _cache.GetAsync(key);
        if (cached != null) return cached;

        // Fetch from database (only once)
        var product = await _repository.GetByIdAsync(id);
        await _cache.SetAsync(key, product, TimeSpan.FromMinutes(5));
        return product;
    }
    finally
    {
        _semaphore.Release();
    }
}
```

---

### 2. Stale Data

**Problem**: Cache has old data, database has updated data

```csharp
// ✅ Solution 1: Short TTL
await _cache.SetAsync(key, value, TimeSpan.FromMinutes(1)); // Refresh frequently

// ✅ Solution 2: Manual invalidation on updates
public async Task UpdateProductAsync(Product product)
{
    await _repository.UpdateAsync(product);
    await _cache.RemoveAsync($"product:{product.Id}"); // Invalidate
}

// ✅ Solution 3: Cache versioning
public async Task SetAsync(string key, object value)
{
    var versionedKey = $"{key}:v{_cacheVersion}";
    await _cache.SetAsync(versionedKey, value);
}

// Increment version to invalidate all cache
public void InvalidateAllCache()
{
    _cacheVersion++;
    // All old versioned keys are now orphaned
}
```

---

### 3. Cache Penetration

**Problem**: Requests for non-existent data bypass cache, hit database repeatedly

```csharp
// ❌ BAD: Non-existent product queried 1000 times
public async Task<Product> GetProductAsync(int id)
{
    var cached = await _cache.GetAsync(key);
    if (cached != null) return cached;

    var product = await _repository.GetByIdAsync(id);
    // If product is null, cache is not set
    // Next request hits database again!
    if (product != null)
    {
        await _cache.SetAsync(key, product);
    }

    return product;
}

// ✅ GOOD: Cache null results (with short TTL)
public async Task<Product> GetProductAsync(int id)
{
    var cached = await _cache.GetAsync(key);
    if (cached != null)
    {
        return cached == "NULL" ? null : JsonSerializer.Deserialize<Product>(cached);
    }

    var product = await _repository.GetByIdAsync(id);

    if (product == null)
    {
        // Cache null result for 1 minute
        await _cache.SetStringAsync(key, "NULL", TimeSpan.FromMinutes(1));
        return null;
    }

    await _cache.SetAsync(key, product, TimeSpan.FromMinutes(5));
    return product;
}
```

---

## 🎤 Common Interview Questions

### Q1: "Explain different caching strategies and when to use each."

**Good Answer**:

"There are several caching patterns, each with trade-offs:

**1. Cache-Aside (Lazy Loading)** - Most Common

Application checks cache first, loads from DB on miss.

```csharp
var cached = await _cache.GetAsync(key);
if (cached == null)
{
    cached = await _db.GetAsync(id);
    await _cache.SetAsync(key, cached);
}
return cached;
```

**Use when**: Read-heavy workloads, acceptable to have occasional cache misses
**Pros**: Simple, efficient
**Cons**: First request is slow (cold start)

---

**2. Write-Through**

Write to cache AND database simultaneously.

**Use when**: Data must be always fresh, write frequency is low
**Pros**: Cache always consistent
**Cons**: Slower writes (two operations)

---

**3. Write-Behind**

Write to cache first, database later (async).

**Use when**: Write-heavy, eventual consistency OK (e.g., view counts)
**Pros**: Ultra-fast writes
**Cons**: Risk of data loss if cache fails

---

**Real Example**:

At my last company, we had product catalog API (10K requests/sec, 95% reads).

**Problem**: Database was bottleneck (50ms queries)

**Solution**: Cache-Aside with Redis
- Cache product data for 5 minutes
- Invalidate cache on product updates

**Result**:
- Latency: 50ms → 2ms (25x improvement)
- Database load: 10K queries/sec → 100 queries/sec (100x reduction)
- Handled Black Friday traffic (100K requests/sec) without scaling database"

---

### Q2: "What is cache invalidation and why is it hard?"

**Good Answer**:

"Cache invalidation is removing stale data from cache. It's famously one of the hardest problems in computer science.

**Why It's Hard**:

**1. Consistency Challenge**

Database is updated, but cache still has old data.

Example:
```
Time 0: Product price = $100 (in DB and cache)
Time 1: Update DB: price = $80
Time 2: User requests product → Gets $100 from cache (stale!)
```

**2. Distributed Systems**

In microservices, multiple services may cache the same data.

When one service updates data, how do others know to invalidate their cache?

**3. Cascading Invalidation**

One entity change may invalidate many cache entries.

Example: Update category name → Invalidate:
- Category cache
- All products in that category
- Category list cache
- Search results cache

Easy to miss one!

---

**Strategies**:

**1. Time-Based (TTL)** - Simplest

```csharp
await _cache.SetAsync(key, value, TimeSpan.FromMinutes(5));
// Auto-expires after 5 minutes
```

**Pros**: Automatic, simple
**Cons**: May serve stale data until expiration

**When**: Eventual consistency OK

---

**2. Manual Invalidation**

```csharp
public async Task UpdateProductAsync(Product product)
{
    await _db.UpdateAsync(product);
    await _cache.RemoveAsync($"product:{product.Id}");
}
```

**Pros**: Immediate consistency
**Cons**: Must remember to invalidate (easy to forget!)

**When**: Strong consistency required

---

**3. Event-Based**

Publish event when data changes, listeners invalidate cache.

```csharp
// Service A
await _bus.PublishAsync(new ProductUpdatedEvent { Id = 123 });

// Service B (listener)
public async Task Handle(ProductUpdatedEvent evt)
{
    await _cache.RemoveAsync($"product:{evt.Id}");
}
```

**Pros**: Decoupled, works across services
**Cons**: Eventual consistency, complexity

**When**: Microservices architecture

---

**Real Example**:

At my previous company, we had a bug where product updates didn't invalidate search results cache.

Users saw old product names in search for up to 10 minutes (cache TTL).

Fixed by:
1. Publishing `ProductUpdatedEvent` on updates
2. Search service listening and invalidating affected search result caches
3. Reduced TTL to 2 minutes as safety net

Result: Fresh data within seconds, not minutes."

---

### Q3: "How would you implement caching for a high-traffic API?"

**Good Answer**:

"I'd use a layered caching strategy:

**Layer 1: HTTP Caching (CDN)**

For static/public endpoints:

```csharp
[HttpGet("products/{id}")]
[ResponseCache(Duration = 300)] // 5 minutes
public async Task<ActionResult<Product>> GetProduct(int id)
{
    var product = await _productService.GetAsync(id);
    return Ok(product);
}
```

**Benefits**:
- Offload from servers entirely
- Low latency (edge locations)
- Scales infinitely

**Use for**: Public product pages, images, CSS/JS

---

**Layer 2: Distributed Cache (Redis)**

For dynamic/personalized data:

```csharp
public async Task<Product> GetProductAsync(int id)
{
    var cacheKey = $"product:{id}";

    // Check Redis
    var cached = await _redis.GetAsync(cacheKey);
    if (cached != null) return cached;

    // Fetch from database
    var product = await _db.GetProductAsync(id);

    // Cache for 5 minutes
    await _redis.SetAsync(cacheKey, product, TimeSpan.FromMinutes(5));

    return product;
}
```

**Benefits**:
- Shared across servers
- Fast (1-5ms)
- Handles dynamic data

**Use for**: Product details, user sessions, API responses

---

**Layer 3: Application Memory Cache**

For reference data:

```csharp
public async Task<Category> GetCategoryAsync(int id)
{
    // Check in-memory cache first
    if (_memoryCache.TryGetValue($"category:{id}", out Category cached))
        return cached;

    // Fetch from database
    var category = await _db.GetCategoryAsync(id);

    // Cache for 1 hour (categories rarely change)
    _memoryCache.Set($"category:{id}", category, TimeSpan.FromHours(1));

    return category;
}
```

**Benefits**:
- Ultra-fast (<1ms)
- No network call

**Use for**: Categories, config, reference data (changes infrequently)

---

**Monitoring**:

```csharp
public class CacheMetrics
{
    private readonly IMetrics _metrics;

    public async Task<T> GetWithMetricsAsync<T>(string key, Func<Task<T>> loadFunc)
    {
        var cached = await _cache.GetAsync<T>(key);

        if (cached != null)
        {
            _metrics.Increment("cache.hit");
            return cached;
        }

        _metrics.Increment("cache.miss");

        var value = await loadFunc();
        await _cache.SetAsync(key, value);

        return value;
    }
}

// Track:
// - Cache hit rate (target: >90%)
// - Cache response time (target: <5ms)
// - Cache memory usage
```

---

**Real Example**:

Previous company: E-commerce site, 50K requests/sec during Black Friday.

**Initial**: No caching, database crashed at 5K requests/sec

**After caching**:
- CDN: Product images, static pages (99% hit rate)
- Redis: Product data (95% hit rate)
- Memory: Categories, config (100% hit rate)

**Result**: Handled 50K req/sec, database saw only 500 req/sec (100x reduction)

Cost savings: $10K/month in database scaling avoided"

---

## 📚 Quick Reference

### When to Cache

**✅ Cache**:
- Read-heavy data (95%+ reads)
- Expensive queries (>50ms)
- Reference data (categories, countries)
- API responses
- Session data

**❌ Don't Cache**:
- Write-heavy data (frequent updates)
- User-specific sensitive data
- Real-time data (stock prices)
- Data that must be immediately consistent

### Cache Patterns Quick Guide

| Pattern | Consistency | Complexity | Use Case |
|---------|------------|------------|----------|
| **Cache-Aside** | Eventual | Low | Most common, read-heavy |
| **Read-Through** | Eventual | Medium | Centralized cache logic |
| **Write-Through** | Strong | Medium | Data must be fresh |
| **Write-Behind** | Eventual | High | Write-heavy, high throughput |

### Redis Commands (C#)

```csharp
// Get
var value = await _cache.GetStringAsync(key);

// Set with expiration
await _cache.SetStringAsync(key, value,
    new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
    });

// Delete
await _cache.RemoveAsync(key);

// Check existence
var exists = await _cache.GetAsync(key) != null;
```

---

## 🔗 Related Topics
- [Distributed Systems](./distributed-systems.md)
- [System Design Trade-offs](./trade-offs-guide.md)
- [Microservices Architecture](../architecture/microservices.md)
- [Performance Optimization](../dotnet/performance.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐⭐⭐ (1-5 stars)
**Priority for Tech Lead**: 🔥 CRITICAL
