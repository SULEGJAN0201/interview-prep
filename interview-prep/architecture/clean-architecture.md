# Clean Architecture

## 📝 Overview

Clean Architecture (by Robert C. Martin) is an architectural pattern that emphasizes **separation of concerns** and **independence** of frameworks, UI, databases, and external agencies.

**Core Principle**: Business logic should not depend on external concerns. Dependencies point inward toward business rules.

---

## 🎯 The Clean Architecture Layers

```
┌─────────────────────────────────────────┐
│        Frameworks & Drivers             │  ← External Layer
│    (Web, DB, UI, External Services)    │
├─────────────────────────────────────────┤
│     Interface Adapters                  │  ← Controllers, Presenters
│  (Controllers, Gateways, Presenters)   │     Repositories
├─────────────────────────────────────────┤
│       Application Business Rules        │  ← Use Cases
│          (Use Cases)                    │
├─────────────────────────────────────────┤
│      Enterprise Business Rules          │  ← Entities (Core)
│          (Entities)                     │
└─────────────────────────────────────────┘

**Dependency Rule**: Dependencies can only point INWARD
Outer layers can depend on inner layers, but NOT vice versa
```

---

## 🏗️ Layer Breakdown

### Layer 1: Entities (Enterprise Business Rules)

**What**: Core business objects and rules that are universal to your enterprise

```csharp
// Domain/Entities/Order.cs
namespace CleanArchitecture.Domain.Entities
{
    public class Order
    {
        public int Id { get; private set; }
        public string CustomerId { get; private set; }
        public DateTime OrderDate { get; private set; }
        public OrderStatus Status { get; private set; }
        private List<OrderItem> _items = new();
        public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

        public decimal TotalAmount => _items.Sum(i => i.Subtotal);

        // Business rules live HERE, not in services
        public void AddItem(Product product, int quantity)
        {
            if (quantity <= 0)
                throw new DomainException("Quantity must be positive");

            if (Status != OrderStatus.Draft)
                throw new DomainException("Cannot modify non-draft order");

            var existingItem = _items.FirstOrDefault(i => i.ProductId == product.Id);
            if (existingItem != null)
            {
                existingItem.IncreaseQuantity(quantity);
            }
            else
            {
                _items.Add(new OrderItem(product.Id, product.Price, quantity));
            }
        }

        public void Submit()
        {
            if (!_items.Any())
                throw new DomainException("Cannot submit empty order");

            if (Status != OrderStatus.Draft)
                throw new DomainException("Order already submitted");

            Status = OrderStatus.Submitted;
            OrderDate = DateTime.UtcNow;
        }

        public void Cancel()
        {
            if (Status == OrderStatus.Shipped || Status == OrderStatus.Delivered)
                throw new DomainException($"Cannot cancel {Status} order");

            Status = OrderStatus.Cancelled;
        }
    }

    public enum OrderStatus
    {
        Draft,
        Submitted,
        Processing,
        Shipped,
        Delivered,
        Cancelled
    }
}
```

**Key Characteristics**:
- No dependencies on outer layers
- Pure business logic
- Framework-agnostic
- Highly testable

---

### Layer 2: Use Cases (Application Business Rules)

**What**: Application-specific business rules, orchestrating the flow of data

```csharp
// Application/UseCases/Orders/CreateOrder/CreateOrderUseCase.cs
namespace CleanArchitecture.Application.UseCases.Orders
{
    public interface ICreateOrderUseCase
    {
        Task<CreateOrderResponse> ExecuteAsync(CreateOrderRequest request);
    }

    public class CreateOrderUseCase : ICreateOrderUseCase
    {
        private readonly IOrderRepository _orderRepository;
        private readonly IProductRepository _productRepository;
        private readonly IUnitOfWork _unitOfWork;
        private readonly IEmailService _emailService;

        public CreateOrderUseCase(
            IOrderRepository orderRepository,
            IProductRepository productRepository,
            IUnitOfWork unitOfWork,
            IEmailService emailService)
        {
            _orderRepository = orderRepository;
            _productRepository = productRepository;
            _unitOfWork = unitOfWork;
            _emailService = emailService;
        }

        public async Task<CreateOrderResponse> ExecuteAsync(CreateOrderRequest request)
        {
            // 1. Validate input
            if (string.IsNullOrEmpty(request.CustomerId))
                throw new ValidationException("Customer ID is required");

            // 2. Get products
            var products = await _productRepository.GetByIdsAsync(
                request.Items.Select(i => i.ProductId).ToList()
            );

            // 3. Create order (business logic in entity)
            var order = new Order(request.CustomerId);
            foreach (var item in request.Items)
            {
                var product = products.FirstOrDefault(p => p.Id == item.ProductId);
                if (product == null)
                    throw new NotFoundException($"Product {item.ProductId} not found");

                order.AddItem(product, item.Quantity);
            }

            order.Submit();

            // 4. Save
            await _orderRepository.AddAsync(order);
            await _unitOfWork.CommitAsync();

            // 5. Send notification (side effect)
            await _emailService.SendOrderConfirmationAsync(order.Id);

            // 6. Return response
            return new CreateOrderResponse
            {
                OrderId = order.Id,
                TotalAmount = order.TotalAmount,
                Status = order.Status.ToString()
            };
        }
    }

    // Request/Response DTOs
    public record CreateOrderRequest
    {
        public string CustomerId { get; init; }
        public List<OrderItemRequest> Items { get; init; }
    }

    public record OrderItemRequest
    {
        public int ProductId { get; init; }
        public int Quantity { get; init; }
    }

    public record CreateOrderResponse
    {
        public int OrderId { get; init; }
        public decimal TotalAmount { get; init; }
        public string Status { get; init; }
    }
}
```

**Key Characteristics**:
- Orchestrates business logic
- Uses entities to enforce rules
- Depends on interfaces, not implementations
- Framework-agnostic

---

### Layer 3: Interface Adapters

**What**: Converts data between use cases and external layers

#### Controllers (Web Layer)

```csharp
// Presentation/Controllers/OrdersController.cs
namespace CleanArchitecture.Presentation.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class OrdersController : ControllerBase
    {
        private readonly ICreateOrderUseCase _createOrderUseCase;
        private readonly IGetOrderUseCase _getOrderUseCase;

        public OrdersController(
            ICreateOrderUseCase createOrderUseCase,
            IGetOrderUseCase getOrderUseCase)
        {
            _createOrderUseCase = createOrderUseCase;
            _getOrderUseCase = getOrderUseCase;
        }

        [HttpPost]
        public async Task<ActionResult<OrderDto>> CreateOrder(CreateOrderDto dto)
        {
            try
            {
                // Convert DTO to Use Case Request
                var request = new CreateOrderRequest
                {
                    CustomerId = dto.CustomerId,
                    Items = dto.Items.Select(i => new OrderItemRequest
                    {
                        ProductId = i.ProductId,
                        Quantity = i.Quantity
                    }).ToList()
                };

                // Execute use case
                var response = await _createOrderUseCase.ExecuteAsync(request);

                // Convert response to DTO
                var orderDto = new OrderDto
                {
                    Id = response.OrderId,
                    TotalAmount = response.TotalAmount,
                    Status = response.Status
                };

                return CreatedAtAction(nameof(GetOrder), new { id = orderDto.Id }, orderDto);
            }
            catch (ValidationException ex)
            {
                return BadRequest(new { error = ex.Message });
            }
            catch (Exception ex)
            {
                return StatusCode(500, new { error = "An error occurred" });
            }
        }

        [HttpGet("{id}")]
        public async Task<ActionResult<OrderDto>> GetOrder(int id)
        {
            var response = await _getOrderUseCase.ExecuteAsync(id);
            if (response == null)
                return NotFound();

            return Ok(response);
        }
    }

    // Web DTOs (different from Use Case DTOs if needed)
    public record CreateOrderDto
    {
        public string CustomerId { get; init; }
        public List<OrderItemDto> Items { get; init; }
    }

    public record OrderItemDto
    {
        public int ProductId { get; init; }
        public int Quantity { get; init; }
    }

    public record OrderDto
    {
        public int Id { get; init; }
        public decimal TotalAmount { get; init; }
        public string Status { get; init; }
    }
}
```

#### Repositories (Data Layer)

```csharp
// Infrastructure/Persistence/Repositories/OrderRepository.cs
namespace CleanArchitecture.Infrastructure.Persistence.Repositories
{
    public class OrderRepository : IOrderRepository
    {
        private readonly ApplicationDbContext _context;

        public OrderRepository(ApplicationDbContext context)
        {
            _context = context;
        }

        public async Task<Order> GetByIdAsync(int id)
        {
            // Convert EF entity to Domain entity
            var dbOrder = await _context.Orders
                .Include(o => o.Items)
                .FirstOrDefaultAsync(o => o.Id == id);

            if (dbOrder == null)
                return null;

            return MapToDomain(dbOrder);
        }

        public async Task AddAsync(Order order)
        {
            var dbOrder = MapToDb(order);
            await _context.Orders.AddAsync(dbOrder);
        }

        // Mapping between domain and persistence models
        private Order MapToDomain(DbOrder dbOrder)
        {
            // Map EF entity to domain entity
            // This isolates domain from persistence concerns
        }

        private DbOrder MapToDb(Order order)
        {
            // Map domain entity to EF entity
        }
    }
}
```

---

### Layer 4: Frameworks & Drivers

**What**: External tools and frameworks (Web, DB, External Services)

```csharp
// Infrastructure/ExternalServices/EmailService.cs
namespace CleanArchitecture.Infrastructure.ExternalServices
{
    public class EmailService : IEmailService
    {
        private readonly IConfiguration _configuration;
        private readonly SmtpClient _smtpClient;

        public EmailService(IConfiguration configuration)
        {
            _configuration = configuration;
            _smtpClient = new SmtpClient(_configuration["Email:Server"]);
        }

        public async Task SendOrderConfirmationAsync(int orderId)
        {
            // External service implementation
            // Domain doesn't know about SMTP, SendGrid, etc.
        }
    }
}
```

---

## 🎯 Folder Structure

```
CleanArchitecture/
├── src/
│   ├── Domain/                      # Layer 1: Entities
│   │   ├── Entities/
│   │   │   ├── Order.cs
│   │   │   ├── Product.cs
│   │   │   └── Customer.cs
│   │   ├── ValueObjects/
│   │   │   ├── Money.cs
│   │   │   └── Address.cs
│   │   ├── Exceptions/
│   │   │   └── DomainException.cs
│   │   └── Interfaces/              # Interfaces for outer layers
│   │       ├── IOrderRepository.cs
│   │       └── IEmailService.cs
│   │
│   ├── Application/                 # Layer 2: Use Cases
│   │   ├── UseCases/
│   │   │   ├── Orders/
│   │   │   │   ├── CreateOrder/
│   │   │   │   │   ├── CreateOrderUseCase.cs
│   │   │   │   │   ├── CreateOrderRequest.cs
│   │   │   │   │   └── CreateOrderResponse.cs
│   │   │   │   └── GetOrder/
│   │   │   └── Products/
│   │   ├── Common/
│   │   │   ├── Interfaces/
│   │   │   └── Exceptions/
│   │   └── DependencyInjection.cs
│   │
│   ├── Infrastructure/              # Layer 3 & 4: Adapters & Drivers
│   │   ├── Persistence/
│   │   │   ├── ApplicationDbContext.cs
│   │   │   ├── Repositories/
│   │   │   │   └── OrderRepository.cs
│   │   │   └── Configurations/
│   │   ├── ExternalServices/
│   │   │   ├── EmailService.cs
│   │   │   └── PaymentService.cs
│   │   └── DependencyInjection.cs
│   │
│   └── Presentation/                # Layer 3: Controllers/Presenters
│       ├── Controllers/
│       │   └── OrdersController.cs
│       ├── DTOs/
│       ├── Program.cs
│       └── Startup.cs
│
└── tests/
    ├── Domain.Tests/
    ├── Application.Tests/
    └── Presentation.Tests/
```

---

## 💡 Benefits of Clean Architecture

### 1. **Testability**

```csharp
// Easy to test use cases without database or web framework
public class CreateOrderUseCaseTests
{
    [Fact]
    public async Task CreateOrder_ValidRequest_ReturnsOrderId()
    {
        // Arrange - Mock dependencies
        var mockOrderRepo = new Mock<IOrderRepository>();
        var mockProductRepo = new Mock<IProductRepository>();
        var mockUnitOfWork = new Mock<IUnitOfWork>();
        var mockEmailService = new Mock<IEmailService>();

        var useCase = new CreateOrderUseCase(
            mockOrderRepo.Object,
            mockProductRepo.Object,
            mockUnitOfWork.Object,
            mockEmailService.Object
        );

        var request = new CreateOrderRequest
        {
            CustomerId = "CUST123",
            Items = new List<OrderItemRequest>
            {
                new() { ProductId = 1, Quantity = 2 }
            }
        };

        // Act
        var response = await useCase.ExecuteAsync(request);

        // Assert
        Assert.NotNull(response);
        Assert.True(response.OrderId > 0);
        mockOrderRepo.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.Once);
    }
}
```

### 2. **Framework Independence**

Can swap out:
- Web framework (ASP.NET → Minimal APIs → gRPC)
- Database (SQL Server → PostgreSQL → MongoDB)
- Email service (SMTP → SendGrid → AWS SES)

**Without changing business logic!**

### 3. **Screaming Architecture**

Folder structure screams what the app DOES, not what it USES:

```
✅ Good (Clean Architecture):
/UseCases/Orders/CreateOrder
/UseCases/Orders/CancelOrder
/UseCases/Products/AddProduct

❌ Bad (Framework-first):
/Controllers
/Models
/Views
```

---

## 🎤 Common Interview Questions

### Q1: "What is Clean Architecture and why is it important?"

**Good Answer**:

"Clean Architecture is a pattern that keeps business logic independent of frameworks, databases, and UIs.

**Core Idea**: Dependencies point inward toward business rules.

**Layers**:
1. **Entities**: Core business rules (Order, Product)
2. **Use Cases**: Application logic (CreateOrder, CancelOrder)
3. **Interface Adapters**: Controllers, Repositories
4. **Frameworks**: Web, DB, External services

**Benefits**:
- **Testability**: Can test business logic without database or web framework
- **Flexibility**: Can swap frameworks/databases without changing business logic
- **Maintainability**: Changes in one layer don't ripple to others

**Example**:
At my last company, we migrated from SQL Server to PostgreSQL. Because we used Clean Architecture, we only changed the repository implementations. Business logic remained untouched. Migration took 2 days instead of 2 months.

**Trade-off**: More upfront structure and code, but pays off in maintainability for long-lived projects."

---

### Q2: "How does Clean Architecture differ from Layered Architecture?"

**Good Answer**:

"Key difference is **dependency direction**:

**Layered Architecture**:
```
UI → Business Logic → Data Access
(Each layer depends on layer below)
```

**Clean Architecture**:
```
UI → Use Cases ← Repositories
     ↓
   Entities (center)
(All dependencies point inward)
```

**Problem with Layered**:
Business logic depends on data access layer. Changing database affects business rules.

**Clean Architecture Solution**:
Business logic defines interfaces (IOrderRepository). Infrastructure implements them. Business logic doesn't know about database.

**Example**:
```csharp
// Layered (bad)
public class OrderService
{
    private SqlOrderRepository _repo; // Depends on concrete implementation!
}

// Clean (good)
public class CreateOrderUseCase
{
    private IOrderRepository _repo; // Depends on interface!
}
```

**Trade-off**: Clean Architecture requires more abstractions (interfaces), but provides better decoupling."

---

## 📚 Quick Reference

### Clean Architecture Checklist

**Domain Layer**:
- [ ] No dependencies on other layers
- [ ] Business rules in entities, not services
- [ ] Value objects for concepts
- [ ] Domain exceptions

**Application Layer**:
- [ ] Use cases orchestrate, don't contain business logic
- [ ] Depends on Domain interfaces
- [ ] Request/Response DTOs
- [ ] No framework dependencies

**Infrastructure Layer**:
- [ ] Implements Domain interfaces
- [ ] Can depend on Domain and Application
- [ ] Database, external services, file system

**Presentation Layer**:
- [ ] Converts web DTOs to use case requests
- [ ] Handles HTTP concerns (status codes, routing)
- [ ] Thin controllers, logic in use cases

---

## 🔗 Related Topics
- [SOLID Principles](./solid-principles.md)
- [Dependency Injection](../dotnet/dependency-injection.md)
- [Repository Pattern](./repository-pattern.md)
- [Microservices Patterns](./microservices.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐☆☆ (1-5 stars)
**Priority for Tech Lead**: 🔥 HIGH
