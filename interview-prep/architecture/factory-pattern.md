# Factory Pattern

## 📝 Overview

The Factory Pattern is a creational design pattern that provides an interface for creating objects without specifying their exact classes. It encapsulates object creation logic.

**Problem It Solves**: When you need to create different types of objects but want to keep the creation logic separate from the business logic.

---

## 🎯 Types of Factory Patterns

### 1. Simple Factory (Not a formal pattern, but useful)

```csharp
// Simple Factory
public class PaymentProcessorFactory
{
    public static IPaymentProcessor Create(PaymentMethod method)
    {
        return method switch
        {
            PaymentMethod.CreditCard => new CreditCardProcessor(),
            PaymentMethod.PayPal => new PayPalProcessor(),
            PaymentMethod.BankTransfer => new BankTransferProcessor(),
            _ => throw new NotSupportedException($"Payment method {method} not supported")
        };
    }
}

// Usage
var processor = PaymentProcessorFactory.Create(PaymentMethod.CreditCard);
var result = await processor.ProcessPaymentAsync(amount);
```

---

### 2. Factory Method Pattern

**Definition**: Define an interface for creating an object, but let subclasses decide which class to instantiate.

```csharp
// Product interface
public interface INotification
{
    Task SendAsync(string message, string recipient);
}

// Concrete products
public class EmailNotification : INotification
{
    public async Task SendAsync(string message, string recipient)
    {
        Console.WriteLine($"Sending email to {recipient}: {message}");
        await Task.CompletedTask;
    }
}

public class SmsNotification : INotification
{
    public async Task SendAsync(string message, string recipient)
    {
        Console.WriteLine($"Sending SMS to {recipient}: {message}");
        await Task.CompletedTask;
    }
}

public class PushNotification : INotification
{
    public async Task SendAsync(string message, string recipient)
    {
        Console.WriteLine($"Sending push notification to {recipient}: {message}");
        await Task.CompletedTask;
    }
}

// Creator (abstract factory)
public abstract class NotificationFactory
{
    public abstract INotification CreateNotification();

    // Template method using the factory method
    public async Task NotifyAsync(string message, string recipient)
    {
        var notification = CreateNotification();
        await notification.SendAsync(message, recipient);
    }
}

// Concrete creators
public class EmailNotificationFactory : NotificationFactory
{
    public override INotification CreateNotification()
    {
        return new EmailNotification();
    }
}

public class SmsNotificationFactory : NotificationFactory
{
    public override INotification CreateNotification()
    {
        return new SmsNotification();
    }
}

// Usage
NotificationFactory factory = new EmailNotificationFactory();
await factory.NotifyAsync("Your order shipped!", "user@example.com");
```

---

### 3. Abstract Factory Pattern

**Definition**: Provide an interface for creating families of related objects without specifying their concrete classes.

```csharp
// Abstract products
public interface IButton
{
    void Render();
}

public interface ICheckbox
{
    void Render();
}

// Concrete products for Windows
public class WindowsButton : IButton
{
    public void Render() => Console.WriteLine("Rendering Windows button");
}

public class WindowsCheckbox : ICheckbox
{
    public void Render() => Console.WriteLine("Rendering Windows checkbox");
}

// Concrete products for Mac
public class MacButton : IButton
{
    public void Render() => Console.WriteLine("Rendering Mac button");
}

public class MacCheckbox : ICheckbox
{
    public void Render() => Console.WriteLine("Rendering Mac checkbox");
}

// Abstract factory
public interface IUIFactory
{
    IButton CreateButton();
    ICheckbox CreateCheckbox();
}

// Concrete factories
public class WindowsUIFactory : IUIFactory
{
    public IButton CreateButton() => new WindowsButton();
    public ICheckbox CreateCheckbox() => new WindowsCheckbox();
}

public class MacUIFactory : IUIFactory
{
    public IButton CreateButton() => new MacButton();
    public ICheckbox CreateCheckbox() => new MacCheckbox();
}

// Usage
public class Application
{
    private readonly IUIFactory _uiFactory;
    private IButton _button;
    private ICheckbox _checkbox;

    public Application(IUIFactory uiFactory)
    {
        _uiFactory = uiFactory;
    }

    public void CreateUI()
    {
        _button = _uiFactory.CreateButton();
        _checkbox = _uiFactory.CreateCheckbox();
    }

    public void Render()
    {
        _button.Render();
        _checkbox.Render();
    }
}

// Client code
var factory = Environment.OSVersion.Platform == PlatformID.Win32NT
    ? (IUIFactory)new WindowsUIFactory()
    : new MacUIFactory();

var app = new Application(factory);
app.CreateUI();
app.Render();
```

---

## 💡 Real-World Example: Multi-Cloud Storage

```csharp
// Abstract product
public interface ICloudStorage
{
    Task UploadAsync(string fileName, Stream content);
    Task<Stream> DownloadAsync(string fileName);
    Task DeleteAsync(string fileName);
}

// Concrete products
public class AWSStorage : ICloudStorage
{
    private readonly IAmazonS3 _s3Client;

    public AWSStorage(IAmazonS3 s3Client)
    {
        _s3Client = s3Client;
    }

    public async Task UploadAsync(string fileName, Stream content)
    {
        await _s3Client.PutObjectAsync(new PutObjectRequest
        {
            BucketName = "my-bucket",
            Key = fileName,
            InputStream = content
        });
    }

    // Download and Delete implementations...
}

public class AzureStorage : ICloudStorage
{
    private readonly BlobServiceClient _blobClient;

    public AzureStorage(BlobServiceClient blobClient)
    {
        _blobClient = blobClient;
    }

    public async Task UploadAsync(string fileName, Stream content)
    {
        var containerClient = _blobClient.GetBlobContainerClient("my-container");
        var blobClient = containerClient.GetBlobClient(fileName);
        await blobClient.UploadAsync(content);
    }

    // Download and Delete implementations...
}

// Factory
public interface ICloudStorageFactory
{
    ICloudStorage CreateStorage();
}

public class AWSStorageFactory : ICloudStorageFactory
{
    private readonly IAmazonS3 _s3Client;

    public AWSStorageFactory(IAmazonS3 s3Client)
    {
        _s3Client = s3Client;
    }

    public ICloudStorage CreateStorage() => new AWSStorage(_s3Client);
}

public class AzureStorageFactory : ICloudStorageFactory
{
    private readonly BlobServiceClient _blobClient;

    public AzureStorageFactory(BlobServiceClient blobClient)
    {
        _blobClient = blobClient;
    }

    public ICloudStorage CreateStorage() => new AzureStorage(_blobClient);
}

// Dependency Injection setup
public void ConfigureServices(IServiceCollection services)
{
    var cloudProvider = Configuration["CloudProvider"];

    if (cloudProvider == "AWS")
    {
        services.AddSingleton<IAmazonS3, AmazonS3Client>();
        services.AddSingleton<ICloudStorageFactory, AWSStorageFactory>();
    }
    else if (cloudProvider == "Azure")
    {
        services.AddSingleton(new BlobServiceClient(Configuration["AzureConnectionString"]));
        services.AddSingleton<ICloudStorageFactory, AzureStorageFactory>();
    }

    services.AddScoped(sp =>
    {
        var factory = sp.GetRequiredService<ICloudStorageFactory>();
        return factory.CreateStorage();
    });
}

// Usage in business logic
public class DocumentService
{
    private readonly ICloudStorage _storage;

    public DocumentService(ICloudStorage storage)
    {
        _storage = storage; // Don't care if it's AWS or Azure!
    }

    public async Task SaveDocumentAsync(string fileName, Stream content)
    {
        await _storage.UploadAsync(fileName, content);
    }
}
```

**Benefits**:
- Switch from AWS to Azure by changing configuration
- Business logic doesn't know about cloud provider
- Easy to test with mock storage

---

## 🎤 Common Interview Questions

### Q: "When would you use the Factory Pattern?"

**Good Answer**:

"I use Factory Pattern when:

**1. Object Creation is Complex**
- Multiple steps to create an object
- Conditional logic based on type
- External dependencies to inject

**2. Need to Decouple Creation from Usage**
- Business logic shouldn't know about concrete classes
- Want to swap implementations easily

**3. Family of Related Objects**
- Abstract Factory for creating related objects
- Example: UI components for different platforms

**Example**:
At my last company, we supported multiple payment processors (Stripe, PayPal, Braintree).

Used Factory Pattern:
```csharp
var processor = _paymentFactory.Create(customer.PreferredMethod);
await processor.ProcessAsync(order.Total);
```

Benefits:
- Add new payment processor without changing business logic
- Easy to test with mock processor
- Configuration-driven (switch providers via config)

**When NOT to Use**:
- Simple object creation (just use `new`)
- Single implementation
- Adds unnecessary complexity"

---

## 📚 Quick Reference

| Pattern Type | When to Use | Example |
|--------------|-------------|---------|
| **Simple Factory** | Basic object creation | Payment processor selection |
| **Factory Method** | Subclasses decide which class to instantiate | Notification sender (email, SMS) |
| **Abstract Factory** | Create families of related objects | UI components (Windows, Mac) |

---

## 🔗 Related Topics
- [Dependency Injection](../dotnet/dependency-injection.md)
- [Singleton Pattern](./singleton-pattern.md)
- [SOLID Principles](./solid-principles.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐⭐☆ (1-5 stars)
**Priority for Tech Lead**: ⚡ HIGH
