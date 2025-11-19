# Distributed Systems Patterns

## 📝 Overview

Distributed systems are collections of independent computers that appear to users as a single coherent system. As systems scale, understanding distributed systems patterns becomes critical for Tech Leads.

**Key Challenge**: In distributed systems, things will fail. Your job is to design for failure, not pretend it won't happen.

---

## 🎯 Core Concepts

### The CAP Theorem

**CAP Theorem**: In a distributed system with network partitions, you can only guarantee 2 of 3:

```
C = Consistency: All nodes see the same data at the same time
A = Availability: Every request receives a response (success or failure)
P = Partition Tolerance: System continues despite network failures

Reality: Partitions WILL happen, so choose: CP or AP
```

**Examples**:

```csharp
// CP System (Consistency + Partition Tolerance)
// Sacrifices Availability during network issues
public class BankingService // MUST be consistent
{
    public async Task<bool> TransferMoney(int fromAccount, int toAccount, decimal amount)
    {
        using var transaction = await _dbContext.Database.BeginTransactionAsync();
        try
        {
            var from = await _dbContext.Accounts.FindAsync(fromAccount);
            var to = await _dbContext.Accounts.FindAsync(toAccount);

            if (from == null || to == null)
                return false; // Unavailable if can't reach data

            if (from.Balance < amount)
                return false;

            from.Balance -= amount;
            to.Balance += amount;

            await _dbContext.SaveChangesAsync();
            await transaction.CommitAsync();

            return true; // Consistent: both updated or neither
        }
        catch
        {
            await transaction.RollbackAsync();
            throw; // Unavailable during partition
        }
    }
}

// AP System (Availability + Partition Tolerance)
// Eventually consistent
public class SocialMediaService // Availability more important
{
    public async Task<bool> PostComment(int postId, string comment, string userId)
    {
        // Always succeeds (Available)
        await _messageQueue.PublishAsync(new CommentPosted
        {
            PostId = postId,
            Comment = comment,
            UserId = userId,
            Timestamp = DateTime.UtcNow
        });

        return true; // Available even during partitions

        // Eventually consistent:
        // - Different users might see comment at different times
        // - Acceptable for social media
    }
}
```

**When to Choose**:
- **CP**: Banking, inventory, authentication (correctness > availability)
- **AP**: Social feeds, analytics, caching (availability > consistency)

---

## 🔄 Consistency Patterns

### 1. Strong Consistency

**Definition**: After an update, all subsequent reads see that update.

```csharp
// Example: Distributed Lock for Strong Consistency
public class DistributedLockService
{
    private readonly IDistributedCache _cache;

    public async Task<bool> AcquireLockAsync(string resourceId, string lockId, TimeSpan timeout)
    {
        var key = $"lock:{resourceId}";
        var acquired = await _cache.StringSetAsync(
            key,
            lockId,
            timeout,
            When.NotExists // Only set if doesn't exist
        );

        return acquired;
    }

    public async Task<bool> ReleaseLockAsync(string resourceId, string lockId)
    {
        var key = $"lock:{resourceId}";
        var currentLock = await _cache.StringGetAsync(key);

        // Only release if we own the lock
        if (currentLock == lockId)
        {
            await _cache.KeyDeleteAsync(key);
            return true;
        }

        return false;
    }
}

// Usage: Ensure only one node processes an order
public class OrderProcessor
{
    private readonly DistributedLockService _lockService;

    public async Task ProcessOrderAsync(int orderId)
    {
        var lockId = Guid.NewGuid().ToString();
        var locked = await _lockService.AcquireLockAsync(
            $"order:{orderId}",
            lockId,
            TimeSpan.FromMinutes(5)
        );

        if (!locked)
        {
            // Another node is processing this order
            return;
        }

        try
        {
            // Process order (guaranteed no other node is doing this)
            await ProcessOrder(orderId);
        }
        finally
        {
            await _lockService.ReleaseLockAsync($"order:{orderId}", lockId);
        }
    }
}
```

**Pros**: Simple reasoning, no surprises
**Cons**: Slower, less available during partitions

---

### 2. Eventual Consistency

**Definition**: If no new updates, eventually all reads will return the last updated value.

```csharp
// Example: Event-Driven Eventual Consistency
public class OrderService
{
    private readonly IEventBus _eventBus;

    public async Task<int> CreateOrderAsync(CreateOrderRequest request)
    {
        // 1. Create order immediately (optimistic)
        var order = new Order
        {
            CustomerId = request.CustomerId,
            Status = OrderStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };

        await _repository.SaveAsync(order);

        // 2. Publish event (fire and forget)
        await _eventBus.PublishAsync(new OrderCreatedEvent
        {
            OrderId = order.Id,
            CustomerId = request.CustomerId,
            Items = request.Items
        });

        return order.Id; // Returns immediately

        // Eventually:
        // - Inventory service will reserve stock
        // - Email service will send confirmation
        // - Analytics service will update metrics
        // All asynchronously, eventually consistent
    }
}

// Inventory Service (separate process/service)
public class InventoryEventHandler
{
    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        // This runs eventually, not immediately
        foreach (var item in @event.Items)
        {
            await _inventory.ReserveStockAsync(item.ProductId, item.Quantity);
        }

        // If this fails, retry mechanism ensures it eventually succeeds
    }
}
```

**Pros**: Higher availability, better performance
**Cons**: Temporary inconsistency, complex error handling

---

### 3. Read-Your-Writes Consistency

**Definition**: After you write, YOUR subsequent reads see that write (but others might not yet).

```csharp
// Implementation: Session-based consistency
public class SessionConsistentService
{
    private readonly ICache _cache;
    private readonly IDatabase _database;

    public async Task UpdateUserProfileAsync(string userId, UserProfile profile)
    {
        // Write to database
        await _database.UpdateAsync(userId, profile);

        // Write to session cache
        var sessionKey = $"user:{userId}:session:{GetSessionId()}";
        await _cache.SetAsync(sessionKey, profile, TimeSpan.FromMinutes(10));
    }

    public async Task<UserProfile> GetUserProfileAsync(string userId)
    {
        // Check session cache first (read your own writes)
        var sessionKey = $"user:{userId}:session:{GetSessionId()}";
        var sessionProfile = await _cache.GetAsync<UserProfile>(sessionKey);

        if (sessionProfile != null)
            return sessionProfile; // You see your own writes

        // Otherwise read from database
        return await _database.GetAsync<UserProfile>(userId);
    }
}
```

**Use Case**: User updates their profile and immediately views it - they see their changes even if not globally replicated yet.

---

## 🌐 Scalability Patterns

### 1. Sharding (Horizontal Partitioning)

**Pattern**: Split data across multiple databases by a shard key.

```csharp
// Shard Key: User ID
public class ShardedUserRepository
{
    private readonly IEnumerable<IDatabase> _shards;

    public ShardedUserRepository(IEnumerable<IDatabase> shards)
    {
        _shards = shards;
    }

    private IDatabase GetShard(string userId)
    {
        // Hash user ID to determine shard
        var hash = userId.GetHashCode();
        var shardIndex = Math.Abs(hash) % _shards.Count();
        return _shards.ElementAt(shardIndex);
    }

    public async Task<User> GetUserAsync(string userId)
    {
        var shard = GetShard(userId);
        return await shard.QueryAsync<User>(
            "SELECT * FROM Users WHERE UserId = @UserId",
            new { UserId = userId }
        );
    }

    public async Task SaveUserAsync(User user)
    {
        var shard = GetShard(user.UserId);
        await shard.ExecuteAsync(
            "INSERT INTO Users (UserId, Name, Email) VALUES (@UserId, @Name, @Email)",
            user
        );
    }

    // Challenge: Cross-shard queries
    public async Task<IEnumerable<User>> GetAllUsersAsync()
    {
        // Must query all shards and aggregate
        var tasks = _shards.Select(shard =>
            shard.QueryAsync<User>("SELECT * FROM Users")
        );

        var results = await Task.WhenAll(tasks);
        return results.SelectMany(r => r);
    }
}
```

**Pros**: Linear scalability
**Cons**: Complex queries across shards, resharding is hard

---

### 2. CQRS (Command Query Responsibility Segregation)

**Pattern**: Separate read and write models.

```csharp
// Write Model (Normalized, Transactional)
public class OrderWriteModel
{
    public async Task CreateOrderAsync(CreateOrderCommand command)
    {
        using var transaction = await _dbContext.Database.BeginTransactionAsync();

        var order = new Order
        {
            CustomerId = command.CustomerId,
            OrderDate = DateTime.UtcNow
        };

        await _dbContext.Orders.AddAsync(order);

        foreach (var item in command.Items)
        {
            await _dbContext.OrderItems.AddAsync(new OrderItem
            {
                OrderId = order.Id,
                ProductId = item.ProductId,
                Quantity = item.Quantity
            });
        }

        await _dbContext.SaveChangesAsync();
        await transaction.CommitAsync();

        // Publish event to update read model
        await _eventBus.PublishAsync(new OrderCreatedEvent
        {
            OrderId = order.Id,
            CustomerId = command.CustomerId,
            Items = command.Items
        });
    }
}

// Read Model (Denormalized, Fast Queries)
public class OrderReadModel
{
    // Denormalized view for fast reads
    public class OrderDetails
    {
        public int OrderId { get; set; }
        public string CustomerName { get; set; } // Denormalized!
        public string CustomerEmail { get; set; } // Denormalized!
        public decimal TotalAmount { get; set; } // Pre-calculated!
        public List<OrderItemDetails> Items { get; set; }
    }

    public async Task<OrderDetails> GetOrderDetailsAsync(int orderId)
    {
        // Fast read from denormalized view (no joins needed)
        return await _readDbContext.OrderDetails
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.OrderId == orderId);
    }

    // Event handler keeps read model in sync
    public async Task HandleOrderCreatedAsync(OrderCreatedEvent @event)
    {
        var customer = await _customerService.GetCustomerAsync(@event.CustomerId);

        var orderDetails = new OrderDetails
        {
            OrderId = @event.OrderId,
            CustomerName = customer.Name, // Denormalized
            CustomerEmail = customer.Email, // Denormalized
            TotalAmount = @event.Items.Sum(i => i.Price * i.Quantity),
            Items = @event.Items.Select(i => new OrderItemDetails
            {
                ProductName = GetProductName(i.ProductId), // Denormalized
                Quantity = i.Quantity,
                Price = i.Price
            }).ToList()
        };

        await _readDbContext.OrderDetails.AddAsync(orderDetails);
        await _readDbContext.SaveChangesAsync();
    }
}
```

**Pros**: Optimized for both reads and writes
**Cons**: Eventual consistency, more complex

---

### 3. Load Balancing Strategies

```csharp
// Round Robin Load Balancer
public class RoundRobinLoadBalancer
{
    private readonly List<string> _servers;
    private int _currentIndex = 0;
    private readonly object _lock = new object();

    public string GetNextServer()
    {
        lock (_lock)
        {
            var server = _servers[_currentIndex];
            _currentIndex = (_currentIndex + 1) % _servers.Count;
            return server;
        }
    }
}

// Least Connections Load Balancer
public class LeastConnectionsLoadBalancer
{
    private readonly ConcurrentDictionary<string, int> _connections = new();
    private readonly List<string> _servers;

    public string GetNextServer()
    {
        return _servers
            .OrderBy(server => _connections.GetOrAdd(server, 0))
            .First();
    }

    public void IncrementConnections(string server)
    {
        _connections.AddOrUpdate(server, 1, (k, v) => v + 1);
    }

    public void DecrementConnections(string server)
    {
        _connections.AddOrUpdate(server, 0, (k, v) => Math.Max(0, v - 1));
    }
}

// Weighted Load Balancer
public class WeightedLoadBalancer
{
    private readonly List<(string Server, int Weight)> _servers;

    public string GetNextServer()
    {
        var totalWeight = _servers.Sum(s => s.Weight);
        var random = new Random().Next(totalWeight);

        var cumulative = 0;
        foreach (var (server, weight) in _servers)
        {
            cumulative += weight;
            if (random < cumulative)
                return server;
        }

        return _servers.Last().Server;
    }
}
```

---

## 🔐 Distributed Transactions

### Two-Phase Commit (2PC)

```csharp
// Coordinator
public class TwoPhaseCommitCoordinator
{
    private readonly List<IParticipant> _participants;

    public async Task<bool> ExecuteTransactionAsync(TransactionData data)
    {
        var transactionId = Guid.NewGuid();

        // Phase 1: Prepare
        var prepareResults = await Task.WhenAll(
            _participants.Select(p => p.PrepareAsync(transactionId, data))
        );

        if (prepareResults.All(r => r))
        {
            // Phase 2: Commit
            await Task.WhenAll(
                _participants.Select(p => p.CommitAsync(transactionId))
            );
            return true;
        }
        else
        {
            // Phase 2: Abort
            await Task.WhenAll(
                _participants.Select(p => p.AbortAsync(transactionId))
            );
            return false;
        }
    }
}

// Participant
public interface IParticipant
{
    Task<bool> PrepareAsync(Guid transactionId, TransactionData data);
    Task CommitAsync(Guid transactionId);
    Task AbortAsync(Guid transactionId);
}

public class DatabaseParticipant : IParticipant
{
    private readonly Dictionary<Guid, IDbTransaction> _transactions = new();

    public async Task<bool> PrepareAsync(Guid transactionId, TransactionData data)
    {
        try
        {
            var transaction = await _connection.BeginTransactionAsync();
            _transactions[transactionId] = transaction;

            // Execute changes but don't commit yet
            await ExecuteChangesAsync(data, transaction);

            return true; // Vote: YES
        }
        catch
        {
            return false; // Vote: NO
        }
    }

    public async Task CommitAsync(Guid transactionId)
    {
        if (_transactions.TryGetValue(transactionId, out var transaction))
        {
            await transaction.CommitAsync();
            _transactions.Remove(transactionId);
        }
    }

    public async Task AbortAsync(Guid transactionId)
    {
        if (_transactions.TryGetValue(transactionId, out var transaction))
        {
            await transaction.RollbackAsync();
            _transactions.Remove(transactionId);
        }
    }
}
```

**Pros**: Strong consistency
**Cons**: Blocking, coordinator is single point of failure

---

## 🎤 Common Interview Questions

### Q1: "Explain the CAP theorem and give examples."

**Good Answer**:

"CAP theorem states that in a distributed system with network partitions, you can only guarantee 2 of 3:
- **C**onsistency
- **A**vailability
- **P**artition tolerance

In reality, partitions WILL happen, so you choose between CP or AP.

**CP System (Consistency + Partition Tolerance)**:
Example: Banking system

When network partition occurs → System becomes unavailable
Better to be unavailable than show wrong balance

**AP System (Availability + Partition Tolerance)**:
Example: Social media feed

When partition occurs → System stays available
Users might see slightly stale data, but that's acceptable

**Real Example**:
At my last company, we had:
- **Payment service**: CP (consistency critical)
- **Product catalog**: AP (availability more important)

**Interview Tip**: Always discuss trade-offs and context.
No 'best' choice, only right choice for your use case."

---

### Q2: "How do you handle distributed transactions?"

**Good Answer**:

"I avoid them when possible, but when needed, I use Saga pattern:

**1. Choreography (Event-Driven)**:
Services react to events, no central coordinator

**Example**: Order Processing
```
OrderService creates order → publishes OrderCreated event
↓
InventoryService reserves stock → publishes StockReserved
↓
PaymentService charges card → publishes PaymentSuccess
↓
ShippingService creates label → publishes OrderShipped
```

**Pros**: Loose coupling
**Cons**: Hard to track, eventual consistency

**2. Orchestration (Centralized)**:
Saga coordinator tells each service what to do

**Example**: Same order flow, but coordinator manages:
```
Coordinator calls:
1. InventoryService.Reserve() ✓
2. PaymentService.Charge() ✗ FAIL

Coordinator compensates:
- InventoryService.Release() (rollback)
```

**Pros**: Easier to understand and debug
**Cons**: Coordinator is coupling point

**Compensation**:
Key is compensating transactions for rollback.

**Real Example**:
At my last company, order processing spanned 4 services.

Used Orchestration Saga:
- Clear flow
- Automatic retry and compensation
- 99.9% success rate
- When failures happen, clean rollback

**Alternative**: Two-Phase Commit
Used in legacy system, but:
- Blocking (bad for performance)
- Coordinator SPOF
- Complex for > 3 participants

Prefer Saga for modern distributed systems."

---

## 📚 Quick Reference

### Consistency Models

| Model | Guarantees | Use Case |
|-------|------------|----------|
| **Strong** | All nodes see same data immediately | Banking, inventory |
| **Eventual** | Eventually consistent | Social media, analytics |
| **Read-Your-Writes** | Your reads see your writes | User profiles |
| **Monotonic Reads** | Never see older versions | Comment threads |

### Partition Strategies

| Strategy | Key | Pros | Cons |
|----------|-----|------|------|
| **Hash** | Hash(key) % N | Even distribution | Resharding hard |
| **Range** | Key ranges | Range queries easy | Uneven distribution |
| **Directory** | Lookup table | Flexible | Extra lookup |

---

## 🔗 Related Topics
- [System Design Trade-offs](./trade-offs-guide.md)
- [Microservices Patterns](../architecture/microservices.md)
- [Scalability Patterns](./scalability.md)
- [Architecture Decisions](../technical-leadership/architecture-decisions.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐☆☆ (1-5 stars)
**Priority for Tech Lead**: 🔥 CRITICAL
