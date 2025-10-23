# Async/Await and Asynchronous Programming

## 📝 Key Concepts

**Async/Await** enables writing asynchronous code that looks and behaves like synchronous code, making it easier to write non-blocking operations.

### Why Use Async/Await?
- Improves application responsiveness (UI doesn't freeze)
- Better resource utilization (threads aren't blocked)
- Scalability for web applications (handles more requests)

## 🔑 Core Principles

### 1. Task and Task<T>
- `Task` represents an asynchronous operation that doesn't return a value
- `Task<T>` represents an asynchronous operation that returns a value of type T

### 2. Async Keyword
- Marks a method as asynchronous
- Enables the use of `await` keyword inside the method
- Method can return `Task`, `Task<T>`, or `void` (avoid void except for event handlers)

### 3. Await Keyword
- Suspends execution until the awaited task completes
- Returns control to the caller
- Doesn't block the thread

## 💻 Code Examples

### Basic Async Method
```csharp
public async Task<string> GetDataAsync()
{
    // Simulating an I/O operation
    await Task.Delay(1000);
    return "Data retrieved";
}

// Calling the async method
public async Task ProcessDataAsync()
{
    string result = await GetDataAsync();
    Console.WriteLine(result);
}
```

### Real-World Example: HTTP Call
```csharp
public async Task<User> GetUserByIdAsync(int userId)
{
    using var httpClient = new HttpClient();
    var response = await httpClient.GetAsync($"https://api.example.com/users/{userId}");
    response.EnsureSuccessStatusCode();
    
    var user = await response.Content.ReadAsAsync<User>();
    return user;
}
```

### Multiple Async Operations
```csharp
// BAD: Sequential execution (slow)
public async Task<string> GetDataSlowAsync()
{
    var result1 = await GetData1Async(); // Wait 2 seconds
    var result2 = await GetData2Async(); // Wait 2 seconds
    return result1 + result2; // Total: 4 seconds
}

// GOOD: Parallel execution (fast)
public async Task<string> GetDataFastAsync()
{
    var task1 = GetData1Async(); // Start immediately
    var task2 = GetData2Async(); // Start immediately
    
    await Task.WhenAll(task1, task2); // Wait for both
    
    return task1.Result + task2.Result; // Total: 2 seconds
}
```

### Using Task.WhenAll
```csharp
public async Task<List<User>> GetMultipleUsersAsync(List<int> userIds)
{
    var tasks = userIds.Select(id => GetUserByIdAsync(id));
    var users = await Task.WhenAll(tasks);
    return users.ToList();
}
```

### Exception Handling
```csharp
public async Task<string> GetDataWithErrorHandlingAsync()
{
    try
    {
        var data = await GetDataAsync();
        return data;
    }
    catch (HttpRequestException ex)
    {
        _logger.LogError(ex, "Failed to retrieve data");
        throw; // Re-throw or handle appropriately
    }
}
```

### Cancellation Support
```csharp
public async Task<string> GetDataWithCancellationAsync(CancellationToken cancellationToken)
{
    using var httpClient = new HttpClient();
    var response = await httpClient.GetAsync(
        "https://api.example.com/data",
        cancellationToken
    );
    
    return await response.Content.ReadAsStringAsync(cancellationToken);
}

// Usage
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
try
{
    var data = await GetDataWithCancellationAsync(cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Operation was cancelled");
}
```

## ⚠️ Common Pitfalls

### 1. Async Void (Avoid!)
```csharp
// BAD: Can't catch exceptions, can't await
public async void ProcessDataBad()
{
    await GetDataAsync();
}

// GOOD: Use async Task instead
public async Task ProcessDataGood()
{
    await GetDataAsync();
}

// EXCEPTION: Event handlers must be async void
private async void Button_Click(object sender, EventArgs e)
{
    await ProcessDataAsync();
}
```

### 2. Blocking on Async Code (Deadlock Risk)
```csharp
// BAD: Can cause deadlock in UI/ASP.NET contexts
public string GetDataSync()
{
    return GetDataAsync().Result; // DANGEROUS!
}

public string GetDataSync2()
{
    return GetDataAsync().GetAwaiter().GetResult(); // ALSO DANGEROUS!
}

// GOOD: Use async all the way
public async Task<string> GetDataAsync()
{
    return await GetDataAsync();
}
```

### 3. Not Using ConfigureAwait
```csharp
// For library code, use ConfigureAwait(false) to avoid context capturing
public async Task<string> LibraryMethodAsync()
{
    var data = await GetDataAsync().ConfigureAwait(false);
    return ProcessData(data);
}

// For UI/ASP.NET Core, ConfigureAwait(true) is default and usually correct
```

## 🎯 Best Practices

1. **Async all the way**: Don't mix sync and async code
2. **Avoid async void**: Except for event handlers
3. **Use Task.WhenAll**: For parallel operations
4. **Support cancellation**: Accept CancellationToken parameters
5. **Name methods with Async suffix**: `GetDataAsync()`, not `GetData()`
6. **Don't block on async code**: Never use `.Result` or `.Wait()`
7. **ConfigureAwait in libraries**: Use `ConfigureAwait(false)` in library code
8. **Return Task directly**: When possible, return the task without awaiting

### Return Task Directly (Performance Optimization)
```csharp
// Less efficient - creates state machine
public async Task<string> GetDataAsync()
{
    return await _httpClient.GetStringAsync("url");
}

// More efficient - returns task directly
public Task<string> GetDataAsync()
{
    return _httpClient.GetStringAsync("url");
}

// But use async/await when you need to do work before/after
public async Task<string> GetDataWithLoggingAsync()
{
    _logger.LogInformation("Fetching data...");
    var data = await _httpClient.GetStringAsync("url");
    _logger.LogInformation("Data fetched successfully");
    return data;
}
```

## 🎤 Common Interview Questions

### Q1: What's the difference between Task.Run() and async/await?
**Answer**: `Task.Run()` queues work to run on the thread pool (CPU-bound work), while `async/await` is for I/O-bound operations that don't block threads. Use Task.Run for CPU-intensive work, and async/await for I/O operations like database calls or HTTP requests.

### Q2: What happens when you await a Task?
**Answer**: The method execution is suspended, and control returns to the caller. The thread is freed to do other work. When the task completes, execution resumes after the await statement, potentially on a different thread.

### Q3: Can you explain the difference between Task.WhenAll and Task.WhenAny?
**Answer**: 
- `Task.WhenAll`: Waits for ALL tasks to complete. Returns an array of results. If any task fails, it throws.
- `Task.WhenAny`: Waits for ANY task to complete. Returns the first completed task.

### Q4: What's a SynchronizationContext and why does it matter?
**Answer**: It represents a location where code should execute. In UI applications, it ensures code runs on the UI thread. In ASP.NET Core, there's no SynchronizationContext by default, which improves performance. That's why ConfigureAwait(false) isn't needed in ASP.NET Core controllers.

### Q5: How do you handle exceptions in async code?
**Answer**: Use try-catch around await statements. With Task.WhenAll, all exceptions are captured in an AggregateException. Access individual exceptions via the Exception property or handle them separately.

```csharp
try
{
    await Task.WhenAll(task1, task2, task3);
}
catch (Exception ex)
{
    // Only the first exception is thrown
    // To get all exceptions:
    if (task1.IsFaulted)
        Console.WriteLine(task1.Exception);
}
```

## 📚 Quick Reference

| Scenario | Use |
|----------|-----|
| I/O-bound work (DB, HTTP, File) | `async/await` |
| CPU-bound work | `Task.Run()` |
| Multiple async operations | `Task.WhenAll()` |
| First completed operation | `Task.WhenAny()` |
| Library code | `ConfigureAwait(false)` |
| Need cancellation | `CancellationToken` |

## 🔗 Related Topics
- [Task Parallel Library (TPL)](./task-parallel-library.md)
- [Threading and Synchronization](./threading.md)
- [Performance Optimization](./performance.md)

---
**Last Reviewed**: [Add date]
**Confidence Level**: ⭐⭐⭐⭐⭐ (1-5 stars)
