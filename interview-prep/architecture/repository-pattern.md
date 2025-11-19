# Repository Pattern & Unit of Work

## 📝 Overview

The **Repository Pattern** provides an abstraction layer between the business logic and data access logic. It encapsulates the logic required to access data sources.

The **Unit of Work Pattern** maintains a list of objects affected by a business transaction and coordinates the writing out of changes.

**Together**: They provide clean separation, testability, and transaction management.

---

## 🎯 Repository Pattern

### Problem It Solves

Without Repository:
```csharp
// ❌ BAD: Business logic directly coupled to EF Core
public class OrderService
{
    private readonly ApplicationDbContext _context;

    public async Task<Order> GetOrderAsync(int id)
    {
        return await _context.Orders
            .Include(o => o.Items)
            .Include(o => o.Customer)
            .FirstOrDefaultAsync(o => o.Id == id);
    }

    public async Task CreateOrderAsync(Order order)
    {
        _context.Orders.Add(order);
        await _context.SaveChangesAsync();
    }
}

// Problems:
// - Business logic knows about EF Core
// - Hard to test (need real database)
// - Can't switch to different data source
// - Scattered data access code
```

With Repository:
```csharp
// ✅ GOOD: Business logic depends on interface
public class OrderService
{
    private readonly IOrderRepository _repository;

    public async Task<Order> GetOrderAsync(int id)
    {
        return await _repository.GetByIdAsync(id);
    }

    public async Task CreateOrderAsync(Order order)
    {
        await _repository.AddAsync(order);
    }
}

// Benefits:
// - Business logic doesn't know about data access details
// - Easy to mock for testing
// - Can swap implementations
// - Centralized data access logic
```

---

## 💻 Repository Implementation

### Generic Repository Interface

```csharp
// Core/Interfaces/IRepository.cs
public interface IRepository<T> where T : class
{
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate);
    Task AddAsync(T entity);
    Task AddRangeAsync(IEnumerable<T> entities);
    void Update(T entity);
    void Remove(T entity);
    void RemoveRange(IEnumerable<T> entities);
}
```

### Generic Repository Implementation

```csharp
// Infrastructure/Repositories/Repository.cs
public class Repository<T> : IRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context;

    public Repository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<T> GetByIdAsync(int id)
    {
        return await _context.Set<T>().FindAsync(id);
    }

    public async Task<IEnumerable<T>> GetAllAsync()
    {
        return await _context.Set<T>().ToListAsync();
    }

    public async Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate)
    {
        return await _context.Set<T>().Where(predicate).ToListAsync();
    }

    public async Task AddAsync(T entity)
    {
        await _context.Set<T>().AddAsync(entity);
    }

    public async Task AddRangeAsync(IEnumerable<T> entities)
    {
        await _context.Set<T>().AddRangeAsync(entities);
    }

    public void Update(T entity)
    {
        _context.Set<T>().Update(entity);
    }

    public void Remove(T entity)
    {
        _context.Set<T>().Remove(entity);
    }

    public void RemoveRange(IEnumerable<T> entities)
    {
        _context.Set<T>().RemoveRange(entities);
    }
}
```

### Specific Repository with Custom Methods

```csharp
// Core/Interfaces/IOrderRepository.cs
public interface IOrderRepository : IRepository<Order>
{
    Task<Order> GetOrderWithItemsAsync(int id);
    Task<IEnumerable<Order>> GetOrdersByCustomerAsync(string customerId);
    Task<IEnumerable<Order>> GetPendingOrdersAsync();
    Task<decimal> GetTotalRevenueAsync(DateTime from, DateTime to);
}

// Infrastructure/Repositories/OrderRepository.cs
public class OrderRepository : Repository<Order>, IOrderRepository
{
    public OrderRepository(ApplicationDbContext context) : base(context)
    {
    }

    public async Task<Order> GetOrderWithItemsAsync(int id)
    {
        return await _context.Orders
            .Include(o => o.Items)
                .ThenInclude(i => i.Product)
            .Include(o => o.Customer)
            .FirstOrDefaultAsync(o => o.Id == id);
    }

    public async Task<IEnumerable<Order>> GetOrdersByCustomerAsync(string customerId)
    {
        return await _context.Orders
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.OrderDate)
            .ToListAsync();
    }

    public async Task<IEnumerable<Order>> GetPendingOrdersAsync()
    {
        return await _context.Orders
            .Where(o => o.Status == OrderStatus.Pending)
            .OrderBy(o => o.OrderDate)
            .ToListAsync();
    }

    public async Task<decimal> GetTotalRevenueAsync(DateTime from, DateTime to)
    {
        return await _context.Orders
            .Where(o => o.OrderDate >= from && o.OrderDate <= to)
            .Where(o => o.Status == OrderStatus.Completed)
            .SumAsync(o => o.TotalAmount);
    }
}
```

---

## 🔄 Unit of Work Pattern

### Problem It Solves

```csharp
// ❌ BAD: Multiple SaveChanges calls
public class OrderService
{
    private readonly IOrderRepository _orderRepo;
    private readonly IInventoryRepository _inventoryRepo;

    public async Task ProcessOrderAsync(Order order)
    {
        await _orderRepo.AddAsync(order);
        await _orderRepo.SaveChangesAsync(); // ← First save

        foreach (var item in order.Items)
        {
            await _inventoryRepo.ReduceStockAsync(item.ProductId, item.Quantity);
            await _inventoryRepo.SaveChangesAsync(); // ← Multiple saves!
        }

        // If this fails, we have inconsistent state!
    }
}
```

### Unit of Work Interface

```csharp
// Core/Interfaces/IUnitOfWork.cs
public interface IUnitOfWork : IDisposable
{
    IOrderRepository Orders { get; }
    IProductRepository Products { get; }
    ICustomerRepository Customers { get; }
    IInventoryRepository Inventory { get; }

    Task<int> CommitAsync();
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}
```

### Unit of Work Implementation

```csharp
// Infrastructure/Data/UnitOfWork.cs
public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private IDbContextTransaction _transaction;

    // Lazy-loaded repositories
    private IOrderRepository _orders;
    private IProductRepository _products;
    private ICustomerRepository _customers;
    private IInventoryRepository _inventory;

    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
    }

    public IOrderRepository Orders =>
        _orders ??= new OrderRepository(_context);

    public IProductRepository Products =>
        _products ??= new ProductRepository(_context);

    public ICustomerRepository Customers =>
        _customers ??= new CustomerRepository(_context);

    public IInventoryRepository Inventory =>
        _inventory ??= new InventoryRepository(_context);

    public async Task<int> CommitAsync()
    {
        return await _context.SaveChangesAsync();
    }

    public async Task BeginTransactionAsync()
    {
        _transaction = await _context.Database.BeginTransactionAsync();
    }

    public async Task CommitTransactionAsync()
    {
        try
        {
            await _context.SaveChangesAsync();
            await _transaction.CommitAsync();
        }
        catch
        {
            await RollbackTransactionAsync();
            throw;
        }
        finally
        {
            await _transaction.DisposeAsync();
            _transaction = null;
        }
    }

    public async Task RollbackTransactionAsync()
    {
        if (_transaction != null)
        {
            await _transaction.RollbackAsync();
            await _transaction.DisposeAsync();
            _transaction = null;
        }
    }

    public void Dispose()
    {
        _transaction?.Dispose();
        _context.Dispose();
    }
}
```

### Usage in Service

```csharp
// ✅ GOOD: Single transaction, all or nothing
public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;

    public OrderService(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    public async Task ProcessOrderAsync(CreateOrderRequest request)
    {
        await _unitOfWork.BeginTransactionAsync();

        try
        {
            // 1. Create order
            var order = new Order
            {
                CustomerId = request.CustomerId,
                OrderDate = DateTime.UtcNow,
                Status = OrderStatus.Pending
            };

            foreach (var item in request.Items)
            {
                var product = await _unitOfWork.Products.GetByIdAsync(item.ProductId);
                order.AddItem(product, item.Quantity);
            }

            await _unitOfWork.Orders.AddAsync(order);

            // 2. Reduce inventory
            foreach (var item in order.Items)
            {
                await _unitOfWork.Inventory.ReduceStockAsync(
                    item.ProductId,
                    item.Quantity
                );
            }

            // 3. Record revenue
            var revenue = new RevenueRecord
            {
                OrderId = order.Id,
                Amount = order.TotalAmount,
                Date = DateTime.UtcNow
            };
            await _unitOfWork.Revenue.AddAsync(revenue);

            // 4. Commit ALL changes in single transaction
            await _unitOfWork.CommitTransactionAsync();

            return order.Id;
        }
        catch (Exception)
        {
            await _unitOfWork.RollbackTransactionAsync();
            throw;
        }
    }
}
```

---

## 🧪 Testing with Repository & Unit of Work

### Easy to Mock

```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task ProcessOrder_ValidOrder_SuccessfullyCreates()
    {
        // Arrange
        var mockUnitOfWork = new Mock<IUnitOfWork>();
        var mockOrderRepo = new Mock<IOrderRepository>();
        var mockProductRepo = new Mock<IProductRepository>();
        var mockInventoryRepo = new Mock<IInventoryRepository>();

        mockUnitOfWork.Setup(u => u.Orders).Returns(mockOrderRepo.Object);
        mockUnitOfWork.Setup(u => u.Products).Returns(mockProductRepo.Object);
        mockUnitOfWork.Setup(u => u.Inventory).Returns(mockInventoryRepo.Object);

        mockProductRepo.Setup(r => r.GetByIdAsync(1))
            .ReturnsAsync(new Product { Id = 1, Name = "Test Product", Price = 10.00m });

        var service = new OrderService(mockUnitOfWork.Object);

        var request = new CreateOrderRequest
        {
            CustomerId = "CUST123",
            Items = new List<OrderItemRequest>
            {
                new() { ProductId = 1, Quantity = 2 }
            }
        };

        // Act
        var orderId = await service.ProcessOrderAsync(request);

        // Assert
        Assert.True(orderId > 0);
        mockOrderRepo.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.Once);
        mockUnitOfWork.Verify(u => u.CommitTransactionAsync(), Times.Once);
        mockInventoryRepo.Verify(r => r.ReduceStockAsync(1, 2), Times.Once);
    }

    [Fact]
    public async Task ProcessOrder_InsufficientInventory_RollsBack()
    {
        // Arrange
        var mockUnitOfWork = new Mock<IUnitOfWork>();
        var mockInventoryRepo = new Mock<IInventoryRepository>();

        mockInventoryRepo.Setup(r => r.ReduceStockAsync(It.IsAny<int>(), It.IsAny<int>()))
            .ThrowsAsync(new InsufficientStockException());

        mockUnitOfWork.Setup(u => u.Inventory).Returns(mockInventoryRepo.Object);

        var service = new OrderService(mockUnitOfWork.Object);

        // Act & Assert
        await Assert.ThrowsAsync<InsufficientStockException>(
            () => service.ProcessOrderAsync(new CreateOrderRequest())
        );

        mockUnitOfWork.Verify(u => u.RollbackTransactionAsync(), Times.Once);
        mockUnitOfWork.Verify(u => u.CommitTransactionAsync(), Times.Never);
    }
}
```

---

## 🎤 Common Interview Questions

### Q1: "Explain the Repository Pattern and when to use it."

**Good Answer**:

"The Repository Pattern abstracts data access logic from business logic.

**Key Benefits**:
1. **Testability**: Easy to mock repositories for unit tests
2. **Maintainability**: Centralized data access code
3. **Flexibility**: Can swap data sources (SQL → NoSQL → API)
4. **Separation of Concerns**: Business logic doesn't know about EF Core, Dapper, etc.

**Example**:
```csharp
// Business logic depends on interface, not implementation
public class OrderService
{
    private readonly IOrderRepository _repository;
    // Doesn't know if it's SQL, NoSQL, or mock
}
```

**When NOT to use**:
- Very simple CRUD apps (may be overkill)
- When ORM already provides good abstraction
- Team prefers direct DbContext access

**My Approach**:
For complex domains, I use repository to encapsulate complex queries. For simple entities, sometimes direct DbContext is fine.

**Trade-off**: Extra abstraction layer vs. testability and flexibility."

---

### Q2: "What is Unit of Work and how does it relate to Repository?"

**Good Answer**:

"Unit of Work maintains a list of changes and commits them in a single transaction.

**Problem Without It**:
```csharp
// ❌ Multiple SaveChanges = Multiple transactions
await orderRepo.AddAsync(order);
await orderRepo.SaveChangesAsync();

await inventoryRepo.ReduceStockAsync(...);
await inventoryRepo.SaveChangesAsync();

// If second fails, first is already committed!
```

**Solution With Unit of Work**:
```csharp
// ✅ Single transaction, all or nothing
await unitOfWork.Orders.AddAsync(order);
await unitOfWork.Inventory.ReduceStockAsync(...);
await unitOfWork.CommitAsync(); // Single commit
```

**Key Benefits**:
1. **Atomicity**: All changes succeed or all fail
2. **Performance**: Single database round-trip
3. **Consistency**: No partial updates

**Relationship to Repository**:
- **Repository**: Abstracts data access per entity
- **Unit of Work**: Coordinates multiple repositories in a transaction

**Example**:
At my last company, order processing involved 5 tables. Unit of Work ensured all updates were atomic. Reduced data inconsistency bugs by 90%."

---

## 📚 Quick Reference

### Repository Pattern Checklist

**✅ Use Repository when**:
- Complex business domain
- Need high testability
- Might change data sources
- Want centralized data access logic

**❌ Skip Repository when**:
- Simple CRUD operations
- Small application
- Direct ORM usage is preferred by team

### Unit of Work Checklist

**✅ Use Unit of Work when**:
- Operations span multiple repositories
- Need transactional consistency
- Want to batch database operations

**❌ Skip Unit of Work when**:
- Single repository operations
- Don't need transactions
- Using framework with built-in unit of work (EF Core DbContext already IS a unit of work)

---

## 🔗 Related Topics
- [Clean Architecture](./clean-architecture.md)
- [Dependency Injection](../dotnet/dependency-injection.md)
- [SOLID Principles](./solid-principles.md)
- [Entity Framework Core](../dotnet/entity-framework.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐⭐☆ (1-5 stars)
**Priority for Tech Lead**: ⚡ HIGH
