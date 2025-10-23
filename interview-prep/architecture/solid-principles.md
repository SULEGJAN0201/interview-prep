# SOLID Principles

## 📝 Overview

**SOLID** is an acronym for five design principles that make software designs more understandable, flexible, and maintainable.

- **S** - Single Responsibility Principle
- **O** - Open/Closed Principle
- **L** - Liskov Substitution Principle
- **I** - Interface Segregation Principle
- **D** - Dependency Inversion Principle

## 🔑 1. Single Responsibility Principle (SRP)

**"A class should have only one reason to change."**

Each class should have only one responsibility or job.

### ❌ Violating SRP
```csharp
// BAD: This class has multiple responsibilities
public class UserService
{
    public void CreateUser(User user)
    {
        // Validate user
        if (string.IsNullOrEmpty(user.Email))
            throw new Exception("Email is required");
        
        // Save to database
        using var connection = new SqlConnection("...");
        connection.Execute("INSERT INTO Users...", user);
        
        // Send email
        var client = new SmtpClient();
        client.Send(user.Email, "Welcome!", "Welcome to our app");
        
        // Log activity
        File.AppendAllText("log.txt", $"User {user.Email} created");
    }
}
```

### ✅ Following SRP
```csharp
// GOOD: Each class has single responsibility
public class UserValidator
{
    public void Validate(User user)
    {
        if (string.IsNullOrEmpty(user.Email))
            throw new ValidationException("Email is required");
        
        if (string.IsNullOrEmpty(user.Name))
            throw new ValidationException("Name is required");
    }
}

public class UserRepository
{
    private readonly string _connectionString;
    
    public UserRepository(string connectionString)
    {
        _connectionString = connectionString;
    }
    
    public void Save(User user)
    {
        using var connection = new SqlConnection(_connectionString);
        connection.Execute("INSERT INTO Users (Name, Email) VALUES (@Name, @Email)", user);
    }
}

public class EmailService
{
    private readonly SmtpClient _smtpClient;
    
    public EmailService(SmtpClient smtpClient)
    {
        _smtpClient = smtpClient;
    }
    
    public void SendWelcomeEmail(string email)
    {
        _smtpClient.Send(email, "Welcome!", "Welcome to our app");
    }
}

public class Logger
{
    public void Log(string message)
    {
        File.AppendAllText("log.txt", $"{DateTime.Now}: {message}");
    }
}

// Orchestrator class
public class UserService
{
    private readonly UserValidator _validator;
    private readonly UserRepository _repository;
    private readonly EmailService _emailService;
    private readonly Logger _logger;
    
    public UserService(
        UserValidator validator,
        UserRepository repository,
        EmailService emailService,
        Logger logger)
    {
        _validator = validator;
        _repository = repository;
        _emailService = emailService;
        _logger = logger;
    }
    
    public void CreateUser(User user)
    {
        _validator.Validate(user);
        _repository.Save(user);
        _emailService.SendWelcomeEmail(user.Email);
        _logger.Log($"User {user.Email} created");
    }
}
```

## 🔑 2. Open/Closed Principle (OCP)

**"Software entities should be open for extension but closed for modification."**

You should be able to add new functionality without changing existing code.

### ❌ Violating OCP
```csharp
// BAD: Must modify class to add new discount types
public class DiscountCalculator
{
    public decimal Calculate(decimal amount, string discountType)
    {
        if (discountType == "Seasonal")
            return amount * 0.9m;
        else if (discountType == "VIP")
            return amount * 0.8m;
        else if (discountType == "Employee")
            return amount * 0.7m;
        
        return amount;
    }
}
```

### ✅ Following OCP
```csharp
// GOOD: Can add new discount types without modifying existing code
public interface IDiscountStrategy
{
    decimal ApplyDiscount(decimal amount);
}

public class SeasonalDiscount : IDiscountStrategy
{
    public decimal ApplyDiscount(decimal amount) => amount * 0.9m;
}

public class VipDiscount : IDiscountStrategy
{
    public decimal ApplyDiscount(decimal amount) => amount * 0.8m;
}

public class EmployeeDiscount : IDiscountStrategy
{
    public decimal ApplyDiscount(decimal amount) => amount * 0.7m;
}

public class DiscountCalculator
{
    public decimal Calculate(decimal amount, IDiscountStrategy discountStrategy)
    {
        return discountStrategy.ApplyDiscount(amount);
    }
}

// Usage
var calculator = new DiscountCalculator();
var discountedAmount = calculator.Calculate(100, new VipDiscount());

// Add new discount type without changing existing code
public class StudentDiscount : IDiscountStrategy
{
    public decimal ApplyDiscount(decimal amount) => amount * 0.85m;
}
```

## 🔑 3. Liskov Substitution Principle (LSP)

**"Objects of a superclass should be replaceable with objects of a subclass without breaking the application."**

Derived classes must be substitutable for their base classes.

### ❌ Violating LSP
```csharp
// BAD: Square violates LSP
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    
    public int GetArea() => Width * Height;
}

public class Square : Rectangle
{
    private int _side;
    
    public override int Width
    {
        get => _side;
        set { _side = value; }
    }
    
    public override int Height
    {
        get => _side;
        set { _side = value; }
    }
}

// This breaks LSP
Rectangle rect = new Square();
rect.Width = 5;
rect.Height = 10;
Console.WriteLine(rect.GetArea()); // Expected: 50, Actual: 100 (broken!)
```

### ✅ Following LSP
```csharp
// GOOD: Use interface or abstract class
public interface IShape
{
    int GetArea();
}

public class Rectangle : IShape
{
    public int Width { get; set; }
    public int Height { get; set; }
    
    public int GetArea() => Width * Height;
}

public class Square : IShape
{
    public int Side { get; set; }
    
    public int GetArea() => Side * Side;
}

// Now both can be used interchangeably through IShape
IShape shape1 = new Rectangle { Width = 5, Height = 10 };
IShape shape2 = new Square { Side = 5 };
Console.WriteLine(shape1.GetArea()); // 50
Console.WriteLine(shape2.GetArea()); // 25
```

## 🔑 4. Interface Segregation Principle (ISP)

**"Clients should not be forced to depend on interfaces they don't use."**

Many specific interfaces are better than one general-purpose interface.

### ❌ Violating ISP
```csharp
// BAD: Fat interface forces classes to implement methods they don't need
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
    void GetPaid();
}

public class HumanWorker : IWorker
{
    public void Work() { /* ... */ }
    public void Eat() { /* ... */ }
    public void Sleep() { /* ... */ }
    public void GetPaid() { /* ... */ }
}

public class RobotWorker : IWorker
{
    public void Work() { /* ... */ }
    public void Eat() { /* Robots don't eat! */ throw new NotImplementedException(); }
    public void Sleep() { /* Robots don't sleep! */ throw new NotImplementedException(); }
    public void GetPaid() { /* Robots don't get paid! */ throw new NotImplementedException(); }
}
```

### ✅ Following ISP
```csharp
// GOOD: Segregate into smaller, specific interfaces
public interface IWorkable
{
    void Work();
}

public interface IFeedable
{
    void Eat();
}

public interface ISleepable
{
    void Sleep();
}

public interface IPayable
{
    void GetPaid();
}

public class HumanWorker : IWorkable, IFeedable, ISleepable, IPayable
{
    public void Work() { /* ... */ }
    public void Eat() { /* ... */ }
    public void Sleep() { /* ... */ }
    public void GetPaid() { /* ... */ }
}

public class RobotWorker : IWorkable
{
    public void Work() { /* ... */ }
    // No need to implement Eat, Sleep, GetPaid
}

// Usage
public class WorkManager
{
    public void Manage(IWorkable worker)
    {
        worker.Work(); // Works for both Human and Robot
    }
}
```

## 🔑 5. Dependency Inversion Principle (DIP)

**"High-level modules should not depend on low-level modules. Both should depend on abstractions."**

Depend on abstractions, not on concrete implementations.

### ❌ Violating DIP
```csharp
// BAD: High-level class depends on concrete low-level classes
public class EmailNotification
{
    public void Send(string message)
    {
        Console.WriteLine($"Email sent: {message}");
    }
}

public class OrderService
{
    private readonly EmailNotification _notification;
    
    public OrderService()
    {
        _notification = new EmailNotification(); // Tight coupling!
    }
    
    public void ProcessOrder(Order order)
    {
        // Process order
        _notification.Send("Order processed");
    }
}
```

### ✅ Following DIP
```csharp
// GOOD: Both depend on abstraction
public interface INotificationService
{
    void Send(string message);
}

public class EmailNotification : INotificationService
{
    public void Send(string message)
    {
        Console.WriteLine($"Email sent: {message}");
    }
}

public class SmsNotification : INotificationService
{
    public void Send(string message)
    {
        Console.WriteLine($"SMS sent: {message}");
    }
}

public class OrderService
{
    private readonly INotificationService _notification;
    
    // Dependency injected through constructor
    public OrderService(INotificationService notification)
    {
        _notification = notification;
    }
    
    public void ProcessOrder(Order order)
    {
        // Process order
        _notification.Send("Order processed");
    }
}

// Usage with DI container
services.AddScoped<INotificationService, EmailNotification>();
// Easy to switch to SMS without changing OrderService
// services.AddScoped<INotificationService, SmsNotification>();
```

## 🎯 Real-World Example: All SOLID Principles Together

```csharp
// Interfaces (ISP + DIP)
public interface IOrderRepository
{
    void Save(Order order);
}

public interface IPaymentProcessor
{
    bool ProcessPayment(decimal amount);
}

public interface INotificationService
{
    void Notify(string message);
}

public interface IOrderValidator
{
    void Validate(Order order);
}

// Implementations (SRP)
public class SqlOrderRepository : IOrderRepository
{
    public void Save(Order order)
    {
        // Save to SQL database
    }
}

public class CreditCardPaymentProcessor : IPaymentProcessor
{
    public bool ProcessPayment(decimal amount)
    {
        // Process credit card payment
        return true;
    }
}

public class EmailNotificationService : INotificationService
{
    public void Notify(string message)
    {
        // Send email
    }
}

public class OrderValidator : IOrderValidator
{
    public void Validate(Order order)
    {
        if (order.Amount <= 0)
            throw new ValidationException("Invalid amount");
    }
}

// High-level service (OCP + DIP)
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IPaymentProcessor _paymentProcessor;
    private readonly INotificationService _notificationService;
    private readonly IOrderValidator _validator;
    
    public OrderService(
        IOrderRepository repository,
        IPaymentProcessor paymentProcessor,
        INotificationService notificationService,
        IOrderValidator validator)
    {
        _repository = repository;
        _paymentProcessor = paymentProcessor;
        _notificationService = notificationService;
        _validator = validator;
    }
    
    public void ProcessOrder(Order order)
    {
        _validator.Validate(order);
        
        if (_paymentProcessor.ProcessPayment(order.Amount))
        {
            _repository.Save(order);
            _notificationService.Notify($"Order {order.Id} processed successfully");
        }
    }
}
```

## 🎤 Common Interview Questions

### Q1: Explain the Single Responsibility Principle with an example
**Answer**: SRP states that a class should have only one reason to change. For example, a `UserService` class that validates users, saves them to a database, sends emails, and logs activities violates SRP because it has multiple responsibilities. We should split it into `UserValidator`, `UserRepository`, `EmailService`, and `Logger` classes, each with a single responsibility.

### Q2: What's the difference between Open/Closed Principle and extensibility?
**Answer**: OCP means your code should be open for extension (add new features) but closed for modification (don't change existing code). You achieve this through abstractions like interfaces and abstract classes. For example, using a `IDiscountStrategy` interface allows adding new discount types without modifying the discount calculator class.

### Q3: How does Dependency Inversion relate to Dependency Injection?
**Answer**: Dependency Inversion is the principle (depend on abstractions), while Dependency Injection is the technique to implement it (inject dependencies through constructors/properties). DI helps achieve DIP by providing dependencies from outside rather than creating them inside the class.

### Q4: Can you give an example where LSP is violated?
**Answer**: The classic example is Rectangle-Square problem. If Square inherits from Rectangle and overrides Width/Height to maintain equal sides, it violates LSP because code expecting a Rectangle (where width and height can be different) will break with a Square. The solution is to use separate classes with a common interface.

### Q5: Why is Interface Segregation important?
**Answer**: ISP prevents forcing clients to depend on methods they don't use. Large interfaces lead to classes implementing dummy methods with `NotImplementedException`. By segregating interfaces, classes only implement what they need, leading to cleaner, more maintainable code. For example, separating `IWorkable`, `IFeedable`, and `ISleepable` instead of one large `IWorker` interface.

## 📚 Quick Reference

| Principle | Question to Ask | Solution |
|-----------|----------------|----------|
| **SRP** | Does this class have multiple reasons to change? | Split into smaller classes |
| **OCP** | Will I need to modify this class to add new features? | Use abstractions (interfaces/abstract classes) |
| **LSP** | Can I replace the base class with derived class without issues? | Ensure derived classes don't break base class behavior |
| **ISP** | Are there methods clients don't use? | Create smaller, focused interfaces |
| **DIP** | Am I depending on concrete classes? | Depend on interfaces, inject dependencies |

## 🔗 Related Topics
- [Dependency Injection](../dotnet/dependency-injection.md)
- [Design Patterns](./design-patterns.md)
- [Clean Architecture](./clean-architecture.md)

---
**Last Reviewed**: [Add date]
**Confidence Level**: ⭐⭐⭐⭐⭐ (1-5 stars)
