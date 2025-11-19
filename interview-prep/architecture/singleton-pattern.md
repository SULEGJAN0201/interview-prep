# Singleton Pattern

## 📝 Overview

The Singleton Pattern ensures a class has only one instance and provides a global point of access to it.

**Problem It Solves**: When you need exactly one instance of a class (e.g., configuration manager, logging service, cache manager).

**Warning**: Often considered an anti-pattern in modern applications. Use with caution!

---

## 🎯 Implementation Variations

### 1. Thread-Safe Lazy Initialization (Recommended)

```csharp
public sealed class ConfigurationManager
{
    private static readonly Lazy<ConfigurationManager> _instance =
        new Lazy<ConfigurationManager>(() => new ConfigurationManager());

    private readonly Dictionary<string, string> _config;

    // Private constructor prevents instantiation from outside
    private ConfigurationManager()
    {
        _config = LoadConfiguration();
    }

    public static ConfigurationManager Instance => _instance.Value;

    public string GetSetting(string key)
    {
        return _config.TryGetValue(key, out var value) ? value : null;
    }

    private Dictionary<string, string> LoadConfiguration()
    {
        // Load from file, database, etc.
        return new Dictionary<string, string>
        {
            ["ApiKey"] = "secret-key",
            ["DatabaseConnection"] = "Server=localhost;Database=MyDb"
        };
    }
}

// Usage
var apiKey = ConfigurationManager.Instance.GetSetting("ApiKey");
```

**Pros**: Thread-safe, lazy initialization, simple
**Cons**: Lazy<T> overhead (minimal)

---

### 2. Eager Initialization

```csharp
public sealed class Logger
{
    // Eager initialization (created at startup)
    private static readonly Logger _instance = new Logger();

    private Logger()
    {
        // Initialize logger
    }

    public static Logger Instance => _instance;

    public void Log(string message)
    {
        Console.WriteLine($"[{DateTime.Now}] {message}");
    }
}

// Usage
Logger.Instance.Log("Application started");
```

**Pros**: Simple, thread-safe, no lazy overhead
**Cons**: Created even if never used

---

### 3. Double-Check Locking (Legacy, avoid if possible)

```csharp
public sealed class CacheManager
{
    private static CacheManager _instance;
    private static readonly object _lock = new object();

    private CacheManager() { }

    public static CacheManager Instance
    {
        get
        {
            if (_instance == null) // First check (no lock)
            {
                lock (_lock)
                {
                    if (_instance == null) // Second check (with lock)
                    {
                        _instance = new CacheManager();
                    }
                }
            }
            return _instance;
        }
    }
}
```

**Note**: Use Lazy<T> instead in modern C#. This is legacy.

---

## ❌ Why Singleton is Often Considered an Anti-Pattern

### Problem 1: Global State

```csharp
// ❌ BAD: Singleton creates hidden global state
public class OrderService
{
    public void ProcessOrder(Order order)
    {
        // Hidden dependency on singleton!
        Logger.Instance.Log("Processing order");
        ConfigManager.Instance.GetSetting("ApiKey");

        // Hard to test - can't mock the singleton
        // Hard to see dependencies
    }
}
```

### Problem 2: Hard to Test

```csharp
// Can't test this without the real singleton
[Fact]
public void ProcessOrder_ShouldLogMessage()
{
    var service = new OrderService();
    service.ProcessOrder(new Order());

    // How do I verify Logger.Instance was called?
    // Can't inject a mock!
}
```

### Problem 3: Tight Coupling

```csharp
// OrderService is tightly coupled to concrete Logger class
// Can't swap implementations
// Can't use different logger for testing
```

---

## ✅ Better Alternative: Dependency Injection

```csharp
// Instead of Singleton, use DI with Singleton lifetime

// Interface
public interface ILogger
{
    void Log(string message);
}

// Implementation
public class ConsoleLogger : ILogger
{
    public void Log(string message)
    {
        Console.WriteLine($"[{DateTime.Now}] {message}");
    }
}

// Register as singleton in DI container
public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<ILogger, ConsoleLogger>();
    // DI container ensures single instance
}

// Usage (injected, not static)
public class OrderService
{
    private readonly ILogger _logger;

    public OrderService(ILogger logger) // Injected!
    {
        _logger = logger;
    }

    public void ProcessOrder(Order order)
    {
        _logger.Log("Processing order");
        // Easy to test - can inject mock logger
        // Dependencies are explicit
    }
}

// Testing
[Fact]
public void ProcessOrder_ShouldLogMessage()
{
    var mockLogger = new Mock<ILogger>();
    var service = new OrderService(mockLogger.Object);

    service.ProcessOrder(new Order());

    mockLogger.Verify(l => l.Log("Processing order"), Times.Once);
}
```

**Benefits**:
- Still a single instance (via DI)
- But testable and flexible
- Explicit dependencies
- Can swap implementations

---

## 🎯 When Singleton IS Appropriate

### Use Case 1: Hardware Access (Truly Single Resource)

```csharp
// Printer manager - only one physical printer
public sealed class PrinterManager
{
    private static readonly Lazy<PrinterManager> _instance =
        new Lazy<PrinterManager>(() => new PrinterManager());

    private PrinterManager()
    {
        // Initialize printer connection
    }

    public static PrinterManager Instance => _instance.Value;

    public void Print(string document)
    {
        // Send to physical printer
    }
}
```

### Use Case 2: Static Configuration (Read-Only)

```csharp
// Application-wide settings that never change
public sealed class AppConstants
{
    private static readonly Lazy<AppConstants> _instance =
        new Lazy<AppConstants>(() => new AppConstants());

    private AppConstants()
    {
        MaxRetries = 3;
        TimeoutSeconds = 30;
        ApiVersion = "v1";
    }

    public static AppConstants Instance => _instance.Value;

    public int MaxRetries { get; }
    public int TimeoutSeconds { get; }
    public string ApiVersion { get; }
}
```

**Key**: Read-only and truly global

---

## 🎤 Common Interview Questions

### Q: "What is the Singleton Pattern and when would you use it?"

**Good Answer**:

"Singleton ensures a class has only one instance with global access.

**Implementation (Modern C#)**:
```csharp
public sealed class ConfigManager
{
    private static readonly Lazy<ConfigManager> _instance =
        new Lazy<ConfigManager>(() => new ConfigManager());

    private ConfigManager() { }

    public static ConfigManager Instance => _instance.Value;
}
```

**When to Use**:
- Truly single resource (hardware, license manager)
- Read-only global configuration
- Performance-critical (avoid repeated initialization)

**When NOT to Use** (Most of the time!):
- Can use Dependency Injection instead
- Creates testability problems
- Hidden dependencies
- Global mutable state (very bad!)

**Modern Approach**:
Instead of Singleton, use DI with singleton lifetime:

```csharp
services.AddSingleton<IConfigManager, ConfigManager>();
```

Benefits:
- Still single instance
- But testable
- Explicit dependencies
- Can swap implementations

**Real Example**:
At my last company, we had a 'singleton' configuration manager.

Problems:
- Hard to test
- Couldn't have different config for tests
- Hidden dependency in 50+ classes

Refactored to:
- IConfigurationManager interface
- Registered as singleton in DI
- Injected where needed

Result:
- Tests could use TestConfigManager
- Dependencies explicit
- Same performance (still single instance)
- Much more maintainable"

---

### Q: "What are the problems with the Singleton Pattern?"

**Good Answer**:

"Singleton has several issues that make it an anti-pattern in modern development:

**1. Global State**
- Tight coupling throughout codebase
- Hidden dependencies (can't see from constructor)
- Hard to reason about when state can change

**2. Testability**
- Can't mock for tests
- Can't inject different implementation
- Tests affect each other (shared state)

**3. Violates Single Responsibility Principle**
- Class responsible for its logic AND ensuring single instance

**4. Concurrency Issues**
- Mutable singleton = race conditions
- Need careful thread-safety

**5. Hard to Extend**
- Sealed, static - can't inherit or replace

**Example of Problems**:
```csharp
// Service tightly coupled to Logger singleton
public class OrderService
{
    public void Process()
    {
        Logger.Instance.Log("test"); // Hidden dependency!
    }
}

// In test - can't control Logger
[Fact]
public void Test()
{
    var service = new OrderService();
    service.Process(); // Uses real Logger!
}
```

**Solution**:
Use DI with singleton lifetime:
```csharp
services.AddSingleton<ILogger, ConsoleLogger>();
```

Get single instance benefit without singleton problems."

---

## 📚 Quick Reference

### Singleton Implementations

| Implementation | Pros | Cons | Use When |
|----------------|------|------|----------|
| **Lazy<T>** | Thread-safe, lazy, simple | Slight overhead | Most cases (if using singleton) |
| **Eager** | Simple, fast | Created even if unused | Always need instance |
| **Double-check** | Lazy, thread-safe | Complex, legacy | Never (use Lazy<T>) |

### Decision Tree

```
Need single instance?
  ├─ Truly global resource (hardware)? → Singleton OK
  ├─ Just need one instance? → Use DI with singleton lifetime
  ├─ For testing? → Definitely use DI
  └─ Stateful? → Avoid singleton! (race conditions)
```

---

## 🔗 Related Topics
- [Dependency Injection](../dotnet/dependency-injection.md)
- [Factory Pattern](./factory-pattern.md)
- [SOLID Principles](./solid-principles.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐⭐⭐ (1-5 stars)
**Priority for Tech Lead**: ⚡ MEDIUM (Know it, but prefer DI)
