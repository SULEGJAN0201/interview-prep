# Dependency Injection and IoC

## 📝 Key Concepts

**Dependency Injection (DI)** is a design pattern where dependencies are provided to a class rather than the class creating them itself. It's a form of **Inversion of Control (IoC)**.

### Benefits
- ✅ Loose coupling between classes
- ✅ Easier unit testing (can mock dependencies)
- ✅ Better code maintainability
- ✅ Follows SOLID principles (especially Dependency Inversion)

## 🔑 Core Principles

### Three Types of DI
1. **Constructor Injection** (Recommended) - Dependencies passed via constructor
2. **Property Injection** - Dependencies set via public properties
3. **Method Injection** - Dependencies passed as method parameters

### Service Lifetimes in .NET Core
1. **Transient** - Created each time they're requested
2. **Scoped** - Created once per request/scope
3. **Singleton** - Created once for the application lifetime

## 💻 Code Examples

### Without DI (Tight Coupling - BAD)
```csharp
public class OrderService
{
    private readonly SqlServerRepository _repository;
    private readonly EmailService _emailService;
    
    public OrderService()
    {
        // Tightly coupled to specific implementations
        _repository = new SqlServerRepository();
        _emailService = new EmailService();
    }
    
    public void ProcessOrder(Order order)
    {
        _repository.Save(order);
        _emailService.SendConfirmation(order.CustomerEmail);
    }
}
```

### With DI (Loose Coupling - GOOD)
```csharp
public interface IOrderRepository
{
    void Save(Order order);
}

public interface IEmailService
{
    void SendConfirmation(string email);
}

public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IEmailService _emailService;
    private readonly ILogger<OrderService> _logger;
    
    // Constructor Injection
    public OrderService(
        IOrderRepository repository,
        IEmailService emailService,
        ILogger<OrderService> logger)
    {
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
        _emailService = emailService ?? throw new ArgumentNullException(nameof(emailService));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }
    
    public void ProcessOrder(Order order)
    {
        _logger.LogInformation("Processing order {OrderId}", order.Id);
        _repository.Save(order);
        _emailService.SendConfirmation(order.CustomerEmail);
    }
}
```

### Registering Services in Program.cs (.NET 6+)
```csharp
var builder = WebApplication.CreateBuilder(args);

// Transient: New instance every time
builder.Services.AddTransient<IEmailService, EmailService>();

// Scoped: One instance per HTTP request
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddScoped<IOrderService, OrderService>();

// Singleton: One instance for application lifetime
builder.Services.AddSingleton<ICacheService, MemoryCacheService>();

var app = builder.Build();
```

### Service Lifetime Examples

#### Transient Example
```csharp
public interface ITransientService
{
    Guid Id { get; }
}

public class TransientService : ITransientService
{
    public Guid Id { get; } = Guid.NewGuid();
}

// Registration
builder.Services.AddTransient<ITransientService, TransientService>();

// Usage: Each injection gets NEW instance
public class MyController : ControllerBase
{
    private readonly ITransientService _service1;
    private readonly ITransientService _service2;
    
    public MyController(ITransientService service1, ITransientService service2)
    {
        _service1 = service1; // Different GUID
        _service2 = service2; // Different GUID
    }
}
```

#### Scoped Example
```csharp
public interface IScopedService
{
    Guid Id { get; }
}

public class ScopedService : IScopedService
{
    public Guid Id { get; } = Guid.NewGuid();
}

// Registration
builder.Services.AddScoped<IScopedService, ScopedService>();

// Usage: Same instance within a request
public class MyController : ControllerBase
{
    private readonly IScopedService _service1;
    private readonly IScopedService _service2;
    
    public MyController(IScopedService service1, IScopedService service2)
    {
        _service1 = service1; // Same GUID within this request
        _service2 = service2; // Same GUID within this request
    }
}
```

#### Singleton Example
```csharp
public interface ISingletonService
{
    Guid Id { get; }
}

public class SingletonService : ISingletonService
{
    public Guid Id { get; } = Guid.NewGuid();
}

// Registration
builder.Services.AddSingleton<ISingletonService, SingletonService>();

// Usage: Same instance across ALL requests
// _service1.Id will be the same for all users, all requests
```

### Using IServiceProvider Directly (Service Locator - Avoid if possible)
```csharp
public class SomeService
{
    private readonly IServiceProvider _serviceProvider;
    
    public SomeService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public void DoWork()
    {
        // Anti-pattern: Hides dependencies
        var repository = _serviceProvider.GetRequiredService<IOrderRepository>();
        repository.Save(new Order());
    }
}

// Better approach: Inject dependencies directly
public class SomeService
{
    private readonly IOrderRepository _repository;
    
    public SomeService(IOrderRepository repository)
    {
        _repository = repository;
    }
    
    public void DoWork()
    {
        _repository.Save(new Order());
    }
}
```

### Factory Pattern with DI
```csharp
public interface INotificationService
{
    void Send(string message);
}

public class EmailNotificationService : INotificationService
{
    public void Send(string message) => Console.WriteLine($"Email: {message}");
}

public class SmsNotificationService : INotificationService
{
    public void Send(string message) => Console.WriteLine($"SMS: {message}");
}

public interface INotificationFactory
{
    INotificationService Create(NotificationType type);
}

public class NotificationFactory : INotificationFactory
{
    private readonly IServiceProvider _serviceProvider;
    
    public NotificationFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public INotificationService Create(NotificationType type)
    {
        return type switch
        {
            NotificationType.Email => _serviceProvider.GetRequiredService<EmailNotificationService>(),
            NotificationType.Sms => _serviceProvider.GetRequiredService<SmsNotificationService>(),
            _ => throw new ArgumentException("Invalid notification type")
        };
    }
}

// Registration
builder.Services.AddTransient<EmailNotificationService>();
builder.Services.AddTransient<SmsNotificationService>();
builder.Services.AddSingleton<INotificationFactory, NotificationFactory>();
```

### Options Pattern
```csharp
// appsettings.json
{
  "EmailSettings": {
    "SmtpServer": "smtp.example.com",
    "Port": 587,
    "FromAddress": "noreply@example.com"
  }
}

// Options class
public class EmailSettings
{
    public string SmtpServer { get; set; }
    public int Port { get; set; }
    public string FromAddress { get; set; }
}

// Registration
builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection("EmailSettings")
);

// Usage with IOptions<T>
public class EmailService : IEmailService
{
    private readonly EmailSettings _settings;
    
    public EmailService(IOptions<EmailSettings> options)
    {
        _settings = options.Value;
    }
    
    public void SendEmail(string to, string subject, string body)
    {
        // Use _settings.SmtpServer, _settings.Port, etc.
    }
}
```

## ⚠️ Common Pitfalls

### 1. Captive Dependencies
```csharp
// BAD: Singleton depends on Scoped service
public class MySingleton
{
    private readonly IScopedService _scopedService;
    
    public MySingleton(IScopedService scopedService) // WRONG!
    {
        _scopedService = scopedService; // This scoped service becomes a singleton!
    }
}

// GOOD: Use IServiceScopeFactory
public class MySingleton
{
    private readonly IServiceScopeFactory _scopeFactory;
    
    public MySingleton(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }
    
    public void DoWork()
    {
        using var scope = _scopeFactory.CreateScope();
        var scopedService = scope.ServiceProvider.GetRequiredService<IScopedService>();
        // Use scopedService
    }
}
```

### 2. Registering the Same Interface Multiple Times
```csharp
// Last registration wins
builder.Services.AddScoped<IEmailService, EmailService>();
builder.Services.AddScoped<IEmailService, MockEmailService>(); // This one is used

// To get all implementations, use IEnumerable<T>
public class NotificationService
{
    private readonly IEnumerable<IEmailService> _emailServices;
    
    public NotificationService(IEnumerable<IEmailService> emailServices)
    {
        _emailServices = emailServices; // Gets ALL registered implementations
    }
}
```

### 3. Forgetting to Register Dependencies
```csharp
// Runtime error: Unable to resolve service for type 'IOrderRepository'
public class OrderService
{
    public OrderService(IOrderRepository repository) // Throws if not registered!
    {
    }
}

// Always register your dependencies
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
```

## 🎯 Best Practices

1. **Prefer Constructor Injection**: Makes dependencies explicit and testable
2. **Use Interfaces**: Depend on abstractions, not concrete implementations
3. **Choose Correct Lifetime**: 
   - Transient: Lightweight, stateless services
   - Scoped: Database contexts, per-request services
   - Singleton: Caching, configuration, expensive to create
4. **Avoid Service Locator**: Don't inject `IServiceProvider` unless absolutely necessary
5. **Validate Dependencies**: Check for null in constructors
6. **Use Options Pattern**: For configuration values
7. **Dispose Resources**: Implement IDisposable when needed (DI container handles disposal)

## 🎤 Common Interview Questions

### Q1: What are the three service lifetimes in .NET Core?
**Answer**: 
- **Transient**: New instance every time requested. Good for lightweight, stateless services.
- **Scoped**: One instance per request/scope. Good for DbContext, per-request services.
- **Singleton**: One instance for application lifetime. Good for caching, configuration.

### Q2: What's the difference between AddScoped and AddTransient?
**Answer**: AddScoped creates one instance per request (HTTP request in web apps), while AddTransient creates a new instance every time it's requested. Scoped is useful for things like DbContext where you want the same instance throughout a request. Transient is for lightweight services.

### Q3: What is a captive dependency and how do you avoid it?
**Answer**: A captive dependency occurs when a longer-lived service (Singleton) captures a shorter-lived service (Scoped/Transient), effectively extending its lifetime. Avoid by:
- Not injecting Scoped/Transient services into Singletons
- Using `IServiceScopeFactory` to create scopes manually when needed
- Keeping dependency lifetimes consistent

### Q4: How do you register multiple implementations of the same interface?
**Answer**: Register them separately, and inject `IEnumerable<TInterface>` to get all implementations:
```csharp
builder.Services.AddScoped<IPaymentProcessor, CreditCardProcessor>();
builder.Services.AddScoped<IPaymentProcessor, PayPalProcessor>();

public class PaymentService(IEnumerable<IPaymentProcessor> processors)
{
    // Can access all processors
}
```

### Q5: When should you use IServiceProvider directly?
**Answer**: Rarely. Main valid use cases:
- Factory patterns where you need to resolve different types at runtime
- When using `IServiceScopeFactory` to create scopes
- In middleware that needs to resolve scoped services
Generally, prefer constructor injection as it makes dependencies explicit.

## 📚 Quick Reference

| Lifetime | Use Case | Example |
|----------|----------|---------|
| Transient | Lightweight, stateless | Email service, validators |
| Scoped | Per-request state | DbContext, repository |
| Singleton | App-wide state, expensive | Cache, configuration, loggers |

### Registration Shortcuts
```csharp
// These are equivalent
builder.Services.AddTransient<IMyService, MyService>();
builder.Services.AddTransient(typeof(IMyService), typeof(MyService));

// Register concrete type only (no interface)
builder.Services.AddTransient<MyConcreteService>();

// Register with factory
builder.Services.AddTransient<IMyService>(sp => 
    new MyService(sp.GetRequiredService<IDependency>())
);
```

## 🔗 Related Topics
- [SOLID Principles](../architecture/solid-principles.md)
- [Unit Testing with Mocking](./unit-testing.md)
- [Repository Pattern](../architecture/repository-pattern.md)

---
**Last Reviewed**: [Add date]
**Confidence Level**: ⭐⭐⭐⭐⭐ (1-5 stars)
