# Microservices Patterns

## 📝 Overview

Microservices architecture is an approach where a large application is built as a suite of small, independently deployable services, each running in its own process and communicating via lightweight mechanisms (HTTP, messaging).

**Key Principle**: Build small services around business capabilities that can be deployed independently.

---

## 🎯 Microservices vs Monolith

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Deployment** | Single unit | Multiple independent services |
| **Scaling** | Scale entire app | Scale services independently |
| **Technology** | One tech stack | Different stacks per service |
| **Development** | One codebase | Multiple codebases |
| **Testing** | Easier (integrated) | Harder (distributed) |
| **Debugging** | Easier (single process) | Harder (trace across services) |
| **Latency** | Low (in-process calls) | Higher (network calls) |
| **Complexity** | Lower initially | Higher from day one |
| **Team Size** | Works for small teams | Better for large teams |
| **Failure Isolation** | One bug affects all | Failures isolated |

---

## 🏗️ Core Microservices Patterns

### 1. Service per Business Capability

**Pattern**: Each microservice owns a specific business capability

```
✅ Good (Business Capabilities):
- OrderService: Order lifecycle
- PaymentService: Payment processing
- InventoryService: Stock management
- NotificationService: Email/SMS notifications

❌ Bad (Technical Layers):
- DatabaseService
- UIService
- LogicService
```

**C# Example**:

```csharp
// OrderService/Controllers/OrdersController.cs
namespace OrderService.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class OrdersController : ControllerBase
    {
        private readonly IOrderRepository _repository;
        private readonly IMessageBus _messageBus;

        [HttpPost]
        public async Task<ActionResult<OrderDto>> CreateOrder(CreateOrderRequest request)
        {
            // 1. Create order
            var order = new Order
            {
                CustomerId = request.CustomerId,
                Items = request.Items,
                Status = OrderStatus.Created
            };

            await _repository.SaveAsync(order);

            // 2. Publish event for other services
            await _messageBus.PublishAsync(new OrderCreatedEvent
            {
                OrderId = order.Id,
                CustomerId = order.CustomerId,
                TotalAmount = order.TotalAmount,
                Items = order.Items
            });

            return Created($"/api/orders/{order.Id}", OrderDto.From(order));
        }
    }
}

// InventoryService/Handlers/OrderCreatedHandler.cs
namespace InventoryService.Handlers
{
    public class OrderCreatedHandler : IEventHandler<OrderCreatedEvent>
    {
        private readonly IInventoryRepository _repository;

        public async Task HandleAsync(OrderCreatedEvent @event)
        {
            // Reserve inventory when order created
            foreach (var item in @event.Items)
            {
                await _repository.ReserveStockAsync(item.ProductId, item.Quantity);
            }
        }
    }
}
```

---

### 2. Database per Service

**Pattern**: Each microservice has its own database (or schema)

**Why**: Ensures loose coupling, allows independent scaling and technology choices

```
OrderService → OrderDB (SQL Server)
ProductCatalog → CatalogDB (PostgreSQL)
UserProfile → ProfileDB (MongoDB)
```

**C# Example**:

```csharp
// OrderService has its own DbContext
public class OrderDbContext : DbContext
{
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }

    // Only Order-related tables
}

// ProductService has completely separate DbContext
public class ProductDbContext : DbContext
{
    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }

    // Only Product-related tables
}
```

**Challenge**: How to query across services?

**Solutions**:
1. **API Composition**: Call multiple services and combine results
2. **CQRS with Read Models**: Denormalized read-only views
3. **Event Sourcing**: Rebuild state from events

---

### 3. API Gateway Pattern

**Pattern**: Single entry point for clients, routes to appropriate microservices

```
Mobile App  ┐
Web App     ├──→ API Gateway ──→ OrderService
Partner API ┘            ├──→ ProductService
                         ├──→ UserService
                         └──→ PaymentService
```

**Benefits**:
- Single endpoint for clients
- Authentication/authorization in one place
- Request routing and composition
- Rate limiting, caching

**C# Example with Ocelot**:

```json
// ocelot.json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/orders/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "order-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/orders/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    },
    {
      "DownstreamPathTemplate": "/api/products/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "product-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/products/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://api.mycompany.com"
  }
}
```

```csharp
// Program.cs
public class Program
{
    public static void Main(string[] args)
    {
        new WebHostBuilder()
            .UseKestrel()
            .UseContentRoot(Directory.GetCurrentDirectory())
            .ConfigureAppConfiguration((hostingContext, config) =>
            {
                config.AddJsonFile("ocelot.json");
            })
            .ConfigureServices(s => {
                s.AddOcelot();
            })
            .Configure(app => {
                app.UseOcelot().Wait();
            })
            .Build()
            .Run();
    }
}
```

---

### 4. Service Discovery Pattern

**Pattern**: Services register themselves, clients discover service locations dynamically

**Why**: Service instances are dynamic (IP addresses change, scaling up/down)

**Solutions**:
- **Client-Side Discovery**: Client queries service registry (Consul, Eureka)
- **Server-Side Discovery**: Load balancer queries registry (Kubernetes, AWS ELB)

**C# Example with Consul**:

```csharp
// Service Registration
public class ConsulServiceRegistration : IHostedService
{
    private readonly IConsulClient _consulClient;
    private readonly IConfiguration _configuration;

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        var registration = new AgentServiceRegistration
        {
            ID = $"order-service-{Guid.NewGuid()}",
            Name = "order-service",
            Address = _configuration["ServiceAddress"],
            Port = int.Parse(_configuration["ServicePort"]),
            Check = new AgentServiceCheck
            {
                HTTP = $"http://{_configuration["ServiceAddress"]}:{_configuration["ServicePort"]}/health",
                Interval = TimeSpan.FromSeconds(10),
                Timeout = TimeSpan.FromSeconds(5)
            }
        };

        await _consulClient.Agent.ServiceRegister(registration, cancellationToken);
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        await _consulClient.Agent.ServiceDeregister(serviceId, cancellationToken);
    }
}

// Service Discovery
public class OrderServiceClient
{
    private readonly IConsulClient _consulClient;
    private readonly HttpClient _httpClient;

    public async Task<Order> GetOrderAsync(int orderId)
    {
        // Discover service
        var services = await _consulClient.Agent.Services();
        var orderService = services.Response.Values
            .FirstOrDefault(s => s.Service == "order-service");

        if (orderService == null)
            throw new ServiceNotFoundException("order-service");

        // Call service
        var url = $"http://{orderService.Address}:{orderService.Port}/api/orders/{orderId}";
        var response = await _httpClient.GetAsync(url);

        return await response.Content.ReadAsAsync<Order>();
    }
}
```

---

### 5. Circuit Breaker Pattern

**Pattern**: Prevent cascading failures by stopping calls to failing services

**States**:
- **Closed**: Requests flow normally
- **Open**: Requests fail immediately (service is failing)
- **Half-Open**: Test if service recovered

**C# Example with Polly**:

```csharp
public class ResilientHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly IAsyncPolicy<HttpResponseMessage> _policy;

    public ResilientHttpClient(HttpClient httpClient)
    {
        _httpClient = httpClient;

        // Circuit Breaker + Retry Policy
        var circuitBreakerPolicy = Policy<HttpResponseMessage>
            .Handle<HttpRequestException>()
            .OrResult(r => !r.IsSuccessStatusCode)
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 3,  // Open after 3 failures
                durationOfBreak: TimeSpan.FromSeconds(30), // Stay open for 30s
                onBreak: (result, duration) =>
                {
                    Console.WriteLine($"Circuit breaker opened for {duration.TotalSeconds}s");
                },
                onReset: () =>
                {
                    Console.WriteLine("Circuit breaker reset");
                }
            );

        var retryPolicy = Policy<HttpResponseMessage>
            .Handle<HttpRequestException>()
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)),
                onRetry: (result, timespan, retryCount, context) =>
                {
                    Console.WriteLine($"Retry {retryCount} after {timespan.TotalSeconds}s");
                }
            );

        _policy = Policy.WrapAsync(retryPolicy, circuitBreakerPolicy);
    }

    public async Task<T> GetAsync<T>(string url)
    {
        var response = await _policy.ExecuteAsync(() => _httpClient.GetAsync(url));
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsAsync<T>();
    }
}

// Usage
public class OrderService
{
    private readonly ResilientHttpClient _productClient;

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        // Even if ProductService is down, circuit breaker prevents cascading failure
        try
        {
            var product = await _productClient.GetAsync<Product>(
                $"http://product-service/api/products/{request.ProductId}"
            );
        }
        catch (BrokenCircuitException)
        {
            // Circuit is open, fail fast
            throw new ServiceUnavailableException("Product service is currently unavailable");
        }
    }
}
```

---

### 6. Saga Pattern (Distributed Transactions)

**Pattern**: Manage transactions across multiple services using events

**Problem**: Traditional ACID transactions don't work across microservices

**Solutions**:

#### A. Choreography Saga (Event-Driven)

```csharp
// OrderService
public async Task CreateOrderAsync(CreateOrderRequest request)
{
    var order = new Order { Status = OrderStatus.Pending };
    await _repository.SaveAsync(order);

    // Publish event
    await _eventBus.PublishAsync(new OrderCreatedEvent
    {
        OrderId = order.Id,
        CustomerId = request.CustomerId,
        Items = request.Items
    });
}

// InventoryService
public class OrderCreatedHandler : IEventHandler<OrderCreatedEvent>
{
    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        var success = await _inventory.ReserveStockAsync(@event.Items);

        if (success)
        {
            await _eventBus.PublishAsync(new InventoryReservedEvent
            {
                OrderId = @event.OrderId
            });
        }
        else
        {
            await _eventBus.PublishAsync(new InventoryReservationFailedEvent
            {
                OrderId = @event.OrderId,
                Reason = "Insufficient stock"
            });
        }
    }
}

// PaymentService
public class InventoryReservedHandler : IEventHandler<InventoryReservedEvent>
{
    public async Task HandleAsync(InventoryReservedEvent @event)
    {
        var success = await _payment.ChargeAsync(@event.OrderId);

        if (success)
        {
            await _eventBus.PublishAsync(new PaymentSuccessEvent
            {
                OrderId = @event.OrderId
            });
        }
        else
        {
            // Compensating transaction
            await _eventBus.PublishAsync(new PaymentFailedEvent
            {
                OrderId = @event.OrderId
            });
        }
    }
}

// OrderService - Compensating transaction handler
public class PaymentFailedHandler : IEventHandler<PaymentFailedEvent>
{
    public async Task HandleAsync(PaymentFailedEvent @event)
    {
        // Rollback: Cancel order
        var order = await _repository.GetByIdAsync(@event.OrderId);
        order.Status = OrderStatus.Cancelled;
        await _repository.SaveAsync(order);

        // Tell inventory to release stock
        await _eventBus.PublishAsync(new ReleaseInventoryEvent
        {
            OrderId = @event.OrderId
        });
    }
}
```

#### B. Orchestration Saga (Centralized)

```csharp
// OrderSagaOrchestrator
public class OrderSagaOrchestrator
{
    private readonly IOrderRepository _orderRepo;
    private readonly IInventoryServiceClient _inventoryClient;
    private readonly IPaymentServiceClient _paymentClient;
    private readonly IShippingServiceClient _shippingClient;

    public async Task<SagaResult> ExecuteOrderSagaAsync(CreateOrderRequest request)
    {
        var sagaState = new OrderSagaState();

        try
        {
            // Step 1: Create Order
            var order = await _orderRepo.CreateAsync(request);
            sagaState.OrderId = order.Id;

            // Step 2: Reserve Inventory
            sagaState.InventoryReserved = await _inventoryClient.ReserveAsync(order.Items);
            if (!sagaState.InventoryReserved)
                throw new SagaException("Inventory reservation failed");

            // Step 3: Process Payment
            sagaState.PaymentProcessed = await _paymentClient.ChargeAsync(order.TotalAmount);
            if (!sagaState.PaymentProcessed)
                throw new SagaException("Payment failed");

            // Step 4: Create Shipment
            sagaState.ShipmentCreated = await _shippingClient.CreateShipmentAsync(order);
            if (!sagaState.ShipmentCreated)
                throw new SagaException("Shipment creation failed");

            // Success!
            return SagaResult.Success(order.Id);
        }
        catch (SagaException ex)
        {
            // Compensate (rollback)
            await CompensateAsync(sagaState);
            return SagaResult.Failure(ex.Message);
        }
    }

    private async Task CompensateAsync(OrderSagaState state)
    {
        // Rollback in reverse order
        if (state.ShipmentCreated)
            await _shippingClient.CancelShipmentAsync(state.OrderId);

        if (state.PaymentProcessed)
            await _paymentClient.RefundAsync(state.OrderId);

        if (state.InventoryReserved)
            await _inventoryClient.ReleaseAsync(state.OrderId);

        if (state.OrderId > 0)
            await _orderRepo.CancelAsync(state.OrderId);
    }
}
```

---

### 7. Event Sourcing

**Pattern**: Store all changes as a sequence of events instead of current state

```csharp
// Events
public record OrderCreatedEvent(int OrderId, string CustomerId, decimal TotalAmount);
public record OrderPaidEvent(int OrderId, decimal Amount, string TransactionId);
public record OrderShippedEvent(int OrderId, string TrackingNumber);
public record OrderCancelledEvent(int OrderId, string Reason);

// Aggregate
public class Order
{
    public int Id { get; private set; }
    public string CustomerId { get; private set; }
    public decimal TotalAmount { get; private set; }
    public OrderStatus Status { get; private set; }

    private List<object> _events = new();

    // Build state from events
    public void ApplyEvent(object @event)
    {
        switch (@event)
        {
            case OrderCreatedEvent e:
                Id = e.OrderId;
                CustomerId = e.CustomerId;
                TotalAmount = e.TotalAmount;
                Status = OrderStatus.Created;
                break;

            case OrderPaidEvent e:
                Status = OrderStatus.Paid;
                break;

            case OrderShippedEvent e:
                Status = OrderStatus.Shipped;
                break;

            case OrderCancelledEvent e:
                Status = OrderStatus.Cancelled;
                break;
        }
    }

    // Command handlers emit events
    public void Create(string customerId, decimal totalAmount)
    {
        var @event = new OrderCreatedEvent(Id, customerId, totalAmount);
        ApplyEvent(@event);
        _events.Add(@event);
    }

    public void MarkAsPaid(decimal amount, string transactionId)
    {
        if (Status != OrderStatus.Created)
            throw new InvalidOperationException("Order must be in Created status");

        var @event = new OrderPaidEvent(Id, amount, transactionId);
        ApplyEvent(@event);
        _events.Add(@event);
    }

    public IReadOnlyList<object> GetUncommittedEvents() => _events.AsReadOnly();
    public void ClearUncommittedEvents() => _events.Clear();
}

// Event Store
public class EventStore
{
    public async Task SaveEventsAsync(int aggregateId, IEnumerable<object> events)
    {
        foreach (var @event in events)
        {
            await _dbContext.Events.AddAsync(new EventRecord
            {
                AggregateId = aggregateId,
                EventType = @event.GetType().Name,
                EventData = JsonSerializer.Serialize(@event),
                Timestamp = DateTime.UtcNow
            });
        }
        await _dbContext.SaveChangesAsync();
    }

    public async Task<Order> LoadAggregateAsync(int aggregateId)
    {
        var eventRecords = await _dbContext.Events
            .Where(e => e.AggregateId == aggregateId)
            .OrderBy(e => e.Timestamp)
            .ToListAsync();

        var order = new Order();
        foreach (var record in eventRecords)
        {
            var @event = DeserializeEvent(record.EventType, record.EventData);
            order.ApplyEvent(@event);
        }

        return order;
    }
}
```

---

## 🎤 Common Interview Questions

### Q1: "When would you choose microservices over a monolith?"

**Good Answer**:

"I use decision criteria based on team, scale, and complexity:

**Choose Microservices when**:
1. **Large team** (>15-20 engineers): Multiple teams need autonomy
2. **Different scaling needs**: Some components need independent scaling
3. **Clear business domains**: Well-understood bounded contexts
4. **Mature DevOps**: Strong CI/CD, monitoring, containerization
5. **Polyglot requirements**: Different services need different tech stacks

**Choose Monolith when**:
1. **Small team** (<10 engineers): Coordination overhead too high for microservices
2. **Unclear domain boundaries**: Still figuring out business model
3. **Tight timeline**: Need to move fast initially
4. **Limited DevOps maturity**: Not ready for distributed systems complexity

**My Recommendation**: Start with **Modular Monolith**
- Clear module boundaries (like microservices)
- Single deployment (like monolith)
- Easy to extract to microservices later

**Real Example**:
At my last company, we started monolith with 8 engineers. At 25 engineers, we extracted payment and notification services. This hybrid approach let us scale gradually without big-bang rewrite."

---

### Q2: "How do you handle distributed transactions in microservices?"

**Good Answer**:

"Two main patterns: **Saga Pattern** with choreography or orchestration.

**Choreography (Event-Driven)**:
- Services publish events when they complete their work
- Other services listen and react
- No central coordinator
- **Pro**: Loose coupling
- **Con**: Hard to track overall flow

**Orchestration (Centralized)**:
- Saga coordinator tells each service what to do
- Handles rollback if steps fail
- **Pro**: Easier to understand and debug
- **Con**: Coordinator is single point of failure

**Example (Order Processing)**:
```
1. Create Order → Success
2. Reserve Inventory → Success
3. Charge Payment → FAIL
4. Compensate: Release Inventory + Cancel Order
```

**Key Principle**: Design for failure from day one
- Each step must be idempotent
- Compensating transactions for rollback
- Event sourcing for auditability

**Trade-off**: Eventual consistency instead of strong consistency. Acceptable for most business scenarios (orders, bookings) but not for financial transactions."

---

## 📚 Quick Reference

### Microservices Decision Checklist

**✅ Good Candidates for Microservices**:
- Large, complex applications
- Multiple development teams
- Different scaling requirements per component
- Need for technology diversity
- Frequent, independent deployments

**❌ Poor Candidates for Microservices**:
- Simple CRUD applications
- Small teams (<5 engineers)
- Unclear business boundaries
- Tight coupling between components
- No DevOps/automation infrastructure

---

## 🔗 Related Topics
- [Clean Architecture](./clean-architecture.md)
- [System Design Trade-offs](../system-design/trade-offs-guide.md)
- [Distributed Systems](../system-design/distributed-systems.md)
- [Architecture Decisions](../technical-leadership/architecture-decisions.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐☆☆ (1-5 stars)
**Priority for Tech Lead**: 🔥 CRITICAL
