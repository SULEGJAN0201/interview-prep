# System Design Trade-offs: Comprehensive Guide

## 📝 Overview

Every system design decision involves trade-offs. There's no "perfect" architecture - only the right balance for your specific requirements, constraints, and context.

**First Law of Software Architecture**: Everything in software architecture is a trade-off.

---

## 🎯 The 15 Critical Trade-offs

### 1. Performance vs. Scalability

**The Trade-off**:
- Optimizing for single-request performance ≠ Optimizing for handling many requests

```
Performance: How fast can we process ONE request?
Scalability: How many requests can we handle?
```

**Examples**:

```csharp
// Optimized for Performance (Single Request)
public async Task<User> GetUserAsync(int id)
{
    // In-memory cache, super fast
    return _memoryCache.GetOrCreate(id, async entry =>
    {
        entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1);
        return await _dbContext.Users.FindAsync(id);
    });
}
// Fast for one request, but doesn't scale - all servers have own cache

// Optimized for Scalability (Many Requests)
public async Task<User> GetUserAsync(int id)
{
    // Distributed cache (Redis), slightly slower per request
    var cached = await _distributedCache.GetStringAsync($"user:{id}");
    if (cached != null)
        return JsonSerializer.Deserialize<User>(cached);

    var user = await _dbContext.Users.FindAsync(id);
    await _distributedCache.SetStringAsync($"user:{id}",
        JsonSerializer.Serialize(user),
        new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
        });

    return user;
}
// Slightly slower per request, but scales across servers
```

**When to Choose**:
- **Performance**: Low traffic, complex computation, real-time requirements
- **Scalability**: High traffic, need to add servers easily

---

### 2. Consistency vs. Availability (CAP Theorem)

**The Trade-off**:
In a distributed system with Partitions (network failures), you must choose between Consistency and Availability.

```
CAP Theorem: Pick 2
- Consistency: Everyone sees the same data
- Availability: System always responds
- Partition Tolerance: Works despite network issues
```

**Example: Banking System**

```csharp
// CP System (Consistency + Partition Tolerance)
// Sacrifices Availability during network issues
public async Task<bool> TransferMoney(int from, int to, decimal amount)
{
    using var transaction = await _dbContext.Database.BeginTransactionAsync();
    try
    {
        var fromAccount = await _dbContext.Accounts
            .FirstOrDefaultAsync(a => a.Id == from);
        var toAccount = await _dbContext.Accounts
            .FirstOrDefaultAsync(a => a.Id == to);

        if (fromAccount == null || toAccount == null)
            return false; // Not available if can't reach all data

        if (fromAccount.Balance < amount)
            return false;

        fromAccount.Balance -= amount;
        toAccount.Balance += amount;

        await _dbContext.SaveChangesAsync();
        await transaction.CommitAsync();

        return true; // Consistent - both updated or neither
    }
    catch
    {
        await transaction.RollbackAsync();
        throw; // Unavailable during issues
    }
}

// AP System (Availability + Partition Tolerance)
// Eventual consistency
public async Task<bool> PostComment(int postId, string comment)
{
    // Always succeeds (Available)
    await _messageQueue.PublishAsync(new CommentCreatedEvent
    {
        PostId = postId,
        Comment = comment,
        Timestamp = DateTime.UtcNow
    });

    return true; // Always available

    // Eventually consistent - comment will appear soon
    // Different users might see different states briefly
}
```

**When to Choose**:
- **CP (Consistency)**: Financial transactions, inventory management, any scenario where correctness > availability
- **AP (Availability)**: Social media, analytics, search results, where slight staleness is acceptable

---

### 3. Latency vs. Throughput

**The Trade-off**:
- **Latency**: Time to process one request
- **Throughput**: Number of requests processed per second

```
You can improve throughput by batching, but it increases latency
```

**Example**:

```csharp
// Low Latency (Individual Processing)
public async Task ProcessOrderAsync(Order order)
{
    await _database.SaveAsync(order); // Immediate save
    await _emailService.SendConfirmationAsync(order); // Immediate email
}
// Latency: 50ms per order
// Throughput: 20 orders/second

// High Throughput (Batching)
private List<Order> _batch = new List<Order>();
private readonly object _lock = new object();

public Task ProcessOrderAsync(Order order)
{
    lock (_lock)
    {
        _batch.Add(order);
    }
    return Task.CompletedTask; // Returns immediately
}

// Background worker
private async Task BatchProcessor()
{
    while (true)
    {
        await Task.Delay(1000); // Wait for batch

        List<Order> toProcess;
        lock (_lock)
        {
            toProcess = _batch.ToList();
            _batch.Clear();
        }

        if (toProcess.Any())
        {
            await _database.SaveManyAsync(toProcess); // Bulk save
            await _emailService.SendBulkAsync(toProcess); // Bulk email
        }
    }
}
// Latency: Up to 1 second per order
// Throughput: 1000+ orders/second
```

**When to Choose**:
- **Low Latency**: User-facing requests, real-time systems, interactive applications
- **High Throughput**: Background processing, analytics, batch operations

---

### 4. Read Optimization vs. Write Optimization

**The Trade-off**:
Optimizing for reads often makes writes slower, and vice versa.

**Example: Database Design**

```sql
-- Write-Optimized (Normalized)
-- Minimize data duplication, fast writes
CREATE TABLE Users (
    UserId INT PRIMARY KEY,
    UserName VARCHAR(100),
    Email VARCHAR(100)
);

CREATE TABLE Orders (
    OrderId INT PRIMARY KEY,
    UserId INT FOREIGN KEY REFERENCES Users(UserId),
    OrderDate DATETIME,
    TotalAmount DECIMAL
);

-- Fast writes (insert once)
-- Slow reads (join needed)
SELECT u.UserName, o.OrderId, o.TotalAmount
FROM Users u
INNER JOIN Orders o ON u.UserId = o.UserId
WHERE u.UserId = 123;

-- Read-Optimized (Denormalized)
-- Duplicate data, fast reads
CREATE TABLE OrdersWithUserInfo (
    OrderId INT PRIMARY KEY,
    UserId INT,
    UserName VARCHAR(100), -- Duplicated!
    Email VARCHAR(100), -- Duplicated!
    OrderDate DATETIME,
    TotalAmount DECIMAL
);

-- Slow writes (must update user info in all orders when changed)
-- Fast reads (no join needed)
SELECT UserName, OrderId, TotalAmount
FROM OrdersWithUserInfo
WHERE UserId = 123;
```

**CQRS Pattern (Compromise)**:

```csharp
// Separate read and write models

// Write Model - Normalized
public class WriteOrderService
{
    public async Task CreateOrderAsync(Order order)
    {
        await _writeDb.Orders.AddAsync(order);
        await _writeDb.SaveChangesAsync();

        // Publish event for read model
        await _eventBus.PublishAsync(new OrderCreatedEvent(order));
    }
}

// Read Model - Denormalized
public class ReadOrderService
{
    public async Task<OrderDetails> GetOrderDetailsAsync(int orderId)
    {
        // Fast read from denormalized view
        return await _readDb.OrderDetails.FindAsync(orderId);
    }
}

// Event handler keeps read model in sync
public class OrderCreatedHandler
{
    public async Task Handle(OrderCreatedEvent evt)
    {
        var orderDetails = new OrderDetails
        {
            OrderId = evt.OrderId,
            UserName = evt.UserName, // Denormalized
            ProductName = evt.ProductName, // Denormalized
            // ... all data in one place for fast reads
        };

        await _readDb.OrderDetails.AddAsync(orderDetails);
        await _readDb.SaveChangesAsync();
    }
}
```

**When to Choose**:
- **Read-Optimized**: Read-heavy workloads (90% reads), analytics, reporting
- **Write-Optimized**: Write-heavy workloads, transactional systems
- **CQRS**: Large scale with different read/write patterns

---

### 5. Monolith vs. Microservices

**The Trade-off**:

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Development** | Faster initially | Slower initially |
| **Deployment** | Simple, one unit | Complex, many units |
| **Scaling** | Scale entire app | Scale services independently |
| **Technology** | One stack | Different stacks possible |
| **Testing** | Easier (integrated) | Harder (distributed) |
| **Debugging** | Easier (one codebase) | Harder (trace across services) |
| **Team Size** | Works for small teams | Better for large teams |
| **Failure Isolation** | One failure affects all | Failures isolated |

**Modular Monolith (Middle Ground)**:

```csharp
// Best of both worlds for many teams

// Monolith deployment, but modular code
namespace ECommerce.Catalog
{
    public interface ICatalogService { }
    internal class CatalogService : ICatalogService { } // Internal!
}

namespace ECommerce.Orders
{
    public class OrderService
    {
        private readonly ICatalogService _catalog; // Interface only!

        public OrderService(ICatalogService catalog)
        {
            _catalog = catalog;
        }
    }
}

// Benefits:
// ✅ Simple deployment (monolith)
// ✅ Clear boundaries (can extract later)
// ✅ Fast development
// ✅ Easy to refactor to microservices when needed
```

**Decision Matrix**:

```
Choose Monolith when:
- Team < 10 people
- Unclear domain boundaries
- Need to move fast initially
- Limited DevOps maturity

Choose Microservices when:
- Team > 20 people
- Clear domain boundaries
- Independent scaling needs
- Mature DevOps practices

Choose Modular Monolith when:
- Medium team (5-20 people)
- Want to start fast but plan to scale
- Domain boundaries emerging
- Growing DevOps capabilities
```

---

### 6. Normalization vs. Denormalization

**The Trade-off**:
- **Normalized**: No duplication, data integrity, slower reads
- **Denormalized**: Duplication for speed, faster reads, risk of inconsistency

**Example**:

```sql
-- Normalized (3NF)
Users: UserId, UserName, Email
Products: ProductId, ProductName, Price
Orders: OrderId, UserId, ProductId, Quantity

-- Pros: Update user once, reflects everywhere
-- Cons: 2 joins for order details

-- Denormalized
Orders: OrderId, UserId, UserName, Email, ProductId, ProductName, Price, Quantity

-- Pros: No joins, super fast reads
-- Cons: If user changes email, must update all their orders
```

**When to Denormalize**:
1. **Read-heavy with rare updates**: User profile names in social feeds
2. **Historical records**: Order snapshot at time of purchase
3. **Performance critical**: Dashboards, reports
4. **Acceptable staleness**: Analytics, caching

**When to Normalize**:
1. **Frequent updates**: Current prices, inventory
2. **Data integrity critical**: Financial data
3. **Storage constrained**: Large duplication would be costly

---

### 7. Strong Consistency vs. Eventual Consistency

**The Trade-off**:

```
Strong Consistency: Everyone sees the same data immediately
- Slower, less available, simpler to reason about

Eventual Consistency: Data becomes consistent "eventually"
- Faster, more available, harder to reason about
```

**Example**:

```csharp
// Strong Consistency (Synchronous)
public async Task CreateUserAsync(User user)
{
    await _database.SaveAsync(user); // Wait for write
    await _cache.InvalidateAsync(user.Id); // Wait for cache clear
    await _searchIndex.UpdateAsync(user); // Wait for index update

    // All systems consistent before returning
    // Slower, but consistent
}

// Eventual Consistency (Event-Driven)
public async Task CreateUserAsync(User user)
{
    await _database.SaveAsync(user); // Only wait for database

    // Fire and forget
    await _eventBus.PublishAsync(new UserCreatedEvent(user));

    // Returns immediately
    // Cache and search will be updated "eventually" by event handlers
}

// Event handlers
public class CacheInvalidationHandler
{
    public async Task Handle(UserCreatedEvent evt)
    {
        await _cache.InvalidateAsync(evt.UserId);
    }
}

public class SearchIndexHandler
{
    public async Task Handle(UserCreatedEvent evt)
    {
        await _searchIndex.UpdateAsync(evt.User);
    }
}
```

**When to Choose**:
- **Strong Consistency**: Financial transactions, inventory, authentication
- **Eventual Consistency**: Social feeds, analytics, search results, notifications

---

### 8. SQL vs. NoSQL

**The Trade-off**:

| Aspect | SQL | NoSQL |
|--------|-----|-------|
| **Schema** | Rigid, predefined | Flexible, schemaless |
| **Transactions** | ACID guarantees | Often eventual consistency |
| **Scalability** | Vertical (bigger servers) | Horizontal (more servers) |
| **Queries** | Complex joins, aggregations | Simple lookups, limited joins |
| **Consistency** | Strong | Often eventual |
| **Use Case** | Relational data, complex queries | High scale, flexible data |

**Decision Framework**:

```markdown
## Choose SQL (PostgreSQL, MySQL) when:
- ✅ Data is relational (users → orders → products)
- ✅ Need complex queries and joins
- ✅ ACID transactions required
- ✅ Data integrity is critical
- ✅ Schema is stable
- ✅ Example: E-commerce, banking, ERP

## Choose Document DB (MongoDB, Cosmos) when:
- ✅ Flexible, evolving schema
- ✅ Hierarchical data (user profile with nested objects)
- ✅ Horizontal scaling needed
- ✅ Each document is self-contained
- ✅ Example: Content management, user profiles, catalogs

## Choose Key-Value (Redis, DynamoDB) when:
- ✅ Simple lookups by key
- ✅ Caching layer
- ✅ Session storage
- ✅ Massive scale
- ✅ Example: Cache, sessions, real-time leaderboards

## Choose Wide-Column (Cassandra) when:
- ✅ Time-series data
- ✅ Massive write throughput
- ✅ High availability required
- ✅ Example: IoT data, logs, analytics

## Choose Graph DB (Neo4j) when:
- ✅ Relationship queries
- ✅ Social networks
- ✅ Recommendation engines
- ✅ Example: Social graphs, fraud detection
```

---

### 9. Synchronous vs. Asynchronous Communication

**The Trade-off**:

```
Synchronous (HTTP/REST):
- Simple, immediate response
- Tight coupling, cascading failures

Asynchronous (Message Queues/Events):
- Loose coupling, resilient
- Complex, eventual consistency
```

**Example**:

```csharp
// Synchronous
public async Task ProcessOrderAsync(Order order)
{
    // Wait for each service
    var inventory = await _inventoryService.ReserveAsync(order);
    var payment = await _paymentService.ChargeAsync(order);
    var shipping = await _shippingService.CreateLabelAsync(order);

    // If any service is down, entire request fails
    // User sees error
}

// Asynchronous (Event-Driven)
public async Task ProcessOrderAsync(Order order)
{
    // Save order
    await _repository.SaveAsync(order);

    // Publish event
    await _eventBus.PublishAsync(new OrderPlacedEvent(order));

    // Return immediately
    return new OrderResponse { OrderId = order.Id, Status = "Processing" };

    // Background handlers will:
    // - Reserve inventory
    // - Charge payment
    // - Create shipping label
    // - Update order status

    // If a service is down, message stays in queue and retries
    // User gets immediate response
}

// Saga pattern for distributed transactions
public class OrderSaga
{
    public async Task Handle(OrderPlacedEvent evt)
    {
        try
        {
            await _inventoryService.ReserveAsync(evt.Order);
            await _paymentService.ChargeAsync(evt.Order);
            await _shippingService.CreateLabelAsync(evt.Order);

            await _eventBus.PublishAsync(new OrderCompletedEvent(evt.Order));
        }
        catch (Exception)
        {
            // Compensating actions
            await _inventoryService.ReleaseAsync(evt.Order);
            await _paymentService.RefundAsync(evt.Order);

            await _eventBus.PublishAsync(new OrderFailedEvent(evt.Order));
        }
    }
}
```

**When to Choose**:
- **Synchronous**: Simple workflows, need immediate response, user waiting
- **Asynchronous**: Long-running operations, multiple steps, high resilience needed

---

### 10. Vertical Scaling vs. Horizontal Scaling

**The Trade-off**:

```
Vertical Scaling (Scale Up):
- Add more power to one server (bigger CPU, more RAM)
- Simple, no code changes
- Limited by hardware limits
- Single point of failure

Horizontal Scaling (Scale Out):
- Add more servers
- Unlimited scaling potential
- Requires stateless design
- Distributed complexity
```

**Example**:

```csharp
// Design for Horizontal Scaling

// ❌ BAD: Stateful (doesn't scale horizontally)
public class OrderController
{
    private static Dictionary<int, Order> _orders = new(); // In-memory state!

    [HttpPost]
    public IActionResult CreateOrder(Order order)
    {
        _orders[order.Id] = order;
        return Ok();
    }

    [HttpGet]
    public IActionResult GetOrder(int id)
    {
        return Ok(_orders[id]); // Only works on same server!
    }
}

// ✅ GOOD: Stateless (scales horizontally)
public class OrderController
{
    private readonly IOrderRepository _repository; // External state

    [HttpPost]
    public async Task<IActionResult> CreateOrder(Order order)
    {
        await _repository.SaveAsync(order); // Shared database
        return Ok();
    }

    [HttpGet]
    public async Task<IActionResult> GetOrder(int id)
    {
        var order = await _repository.GetAsync(id);
        return Ok(order); // Works from any server
    }
}

// Session state handling
public void ConfigureServices(IServiceCollection services)
{
    // ❌ BAD: In-memory session (doesn't scale)
    services.AddSession();

    // ✅ GOOD: Distributed session (scales)
    services.AddStackExchangeRedisCache(options =>
    {
        options.Configuration = "redis:6379";
    });
    services.AddSession(options =>
    {
        options.Cookie.Name = ".MyApp.Session";
    });
}
```

---

## 🎤 Common Interview Questions

### Q1: "Design a URL shortener. Discuss trade-offs."

**Good Answer**:

```markdown
## Requirements Clarification
- 100M URLs/month
- 10:1 read:write ratio
- 100ms latency requirement

## Key Trade-offs

### 1. ID Generation
**Option A**: Auto-increment database ID
- Pro: Simple, guaranteed unique
- Con: Single point of failure, harder to scale

**Option B**: Distributed ID generator (Snowflake)
- Pro: Scalable, no single point of failure
- Con: More complex

**Option C**: Hash-based (MD5 first 6 chars)
- Pro: Stateless, scales easily
- Con: Collision handling needed

**Decision**: Snowflake-style distributed IDs
**Trade-off**: Accept complexity for scalability

### 2. Database Choice
**Option A**: SQL (PostgreSQL)
- Pro: ACID, simple
- Con: Vertical scaling limits

**Option B**: NoSQL (DynamoDB)
- Pro: Unlimited horizontal scaling
- Con: Eventual consistency

**Decision**: Start with PostgreSQL + read replicas
**Trade-off**: Accept scaling limits initially for simplicity
**Mitigation**: Can migrate to NoSQL if we hit limits

### 3. Caching Strategy
**Consistency vs. Performance**
- Cache hot URLs (80/20 rule)
- TTL: 1 hour
- **Trade-off**: Slightly stale click counts acceptable for performance

### 4. Availability vs. Consistency
**Decision**: Favor Availability
- Use eventual consistency for click counts
- Critical that URL redirects work (availability)
- **Trade-off**: Click counts might be slightly off
```

---

## 📚 Quick Reference Table

| Trade-off | Choose A When... | Choose B When... |
|-----------|------------------|------------------|
| **Consistency vs. Availability** | Financial transactions, inventory | Social media, analytics |
| **SQL vs. NoSQL** | Complex relationships, ACID needed | Massive scale, flexible schema |
| **Monolith vs. Microservices** | Small team, unclear boundaries | Large team, clear boundaries |
| **Sync vs. Async** | Simple flow, immediate response | Complex flow, resilience needed |
| **Normalized vs. Denormalized** | Frequent updates, data integrity | Read-heavy, historical records |
| **Vertical vs. Horizontal Scaling** | Simple app, limited scale | High scale, stateless design |
| **Latency vs. Throughput** | User-facing, real-time | Background processing, batch |
| **Strong vs. Eventual Consistency** | Banking, auth | Social feeds, search |

---

## 🔗 Related Topics
- [Architecture Decisions](../technical-leadership/architecture-decisions.md)
- [Scalability Patterns](./scalability.md)
- [Caching Strategies](./caching.md)
- [System Design Questions](../interview-questions/system-design-questions.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐☆☆ (1-5 stars)
**Priority for Tech Lead**: 🔥 CRITICAL
