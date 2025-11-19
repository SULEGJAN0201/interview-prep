# Code Review Best Practices

## 📝 Overview

As a Tech Lead, code review is one of your most important responsibilities. It's not just about finding bugs - it's about **knowledge sharing**, **maintaining standards**, **mentoring**, and **building team culture**.

**Key Insight**: Good code reviews improve code quality AND team growth.

---

## 🎯 Goals of Code Review

1. **Catch bugs before production** (20%)
2. **Maintain code quality and standards** (30%)
3. **Share knowledge across team** (30%)
4. **Mentor and teach** (20%)

**NOT**: Finding every possible flaw or showing how smart you are

---

## 💻 Code Review Checklist

### 1. Functionality & Logic

- [ ] Does the code do what it's supposed to do?
- [ ] Are there any obvious logic errors?
- [ ] Are edge cases handled?
- [ ] Are there any off-by-one errors?
- [ ] Is error handling appropriate?

```csharp
// ❌ BAD: Missing edge case
public decimal CalculateDiscount(decimal price, int quantity)
{
    return price * quantity * 0.1m;
    // What if quantity is negative? What if price is zero?
}

// ✅ GOOD: Edge cases handled
public decimal CalculateDiscount(decimal price, int quantity)
{
    if (price < 0)
        throw new ArgumentException("Price cannot be negative", nameof(price));

    if (quantity <= 0)
        throw new ArgumentException("Quantity must be positive", nameof(quantity));

    return price * quantity * 0.1m;
}
```

---

### 2. Code Design & Architecture

- [ ] Does it follow SOLID principles?
- [ ] Is it in the right layer? (Domain/Application/Infrastructure)
- [ ] Are abstractions appropriate?
- [ ] Is there duplication that should be extracted?
- [ ] Are dependencies injected, not created?

```csharp
// ❌ BAD: Violates SRP, hard-coded dependency
public class OrderService
{
    public void ProcessOrder(Order order)
    {
        // Validation
        if (order.Items.Count == 0)
            throw new Exception("No items");

        // Database access
        using var connection = new SqlConnection("...");
        connection.Execute("INSERT INTO Orders...");

        // Email
        var smtp = new SmtpClient();
        smtp.Send("order@example.com", "Order placed");

        // Logging
        File.AppendAllText("log.txt", "Order processed");
    }
}

// ✅ GOOD: Single responsibility, injected dependencies
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IEmailService _emailService;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        IOrderRepository repository,
        IEmailService emailService,
        ILogger<OrderService> logger)
    {
        _repository = repository;
        _emailService = emailService;
        _logger = logger;
    }

    public async Task ProcessOrderAsync(Order order)
    {
        // Validation in separate validator
        // Repository handles data access
        // Email service handles notifications
        // Logger handles logging
    }
}
```

**Review Comment Example**:
```
❌ "This is wrong."

✅ "This class is doing too much (validation, DB, email, logging).
Consider:
1. Move validation to OrderValidator
2. Use IOrderRepository for DB access
3. Use IEmailService for notifications

This will make it easier to test and change individual concerns.

See our architecture guide: [link]"
```

---

### 3. Readability & Maintainability

- [ ] Are names clear and descriptive?
- [ ] Is the code self-documenting?
- [ ] Are comments necessary and helpful?
- [ ] Is complexity appropriate?
- [ ] Can this be simplified?

```csharp
// ❌ BAD: Unclear names, complex logic
public List<int> GetData(List<Order> o, int t)
{
    var r = new List<int>();
    foreach (var x in o)
    {
        if (x.Date > DateTime.Now.AddDays(-t) && x.Status == 2)
            r.Add(x.Id);
    }
    return r;
}

// ✅ GOOD: Clear names, extracted logic
public List<int> GetRecentCompletedOrderIds(List<Order> orders, int daysBack)
{
    var cutoffDate = DateTime.Now.AddDays(-daysBack);

    return orders
        .Where(order => IsRecentAndCompleted(order, cutoffDate))
        .Select(order => order.Id)
        .ToList();
}

private bool IsRecentAndCompleted(Order order, DateTime cutoffDate)
{
    return order.Date > cutoffDate &&
           order.Status == OrderStatus.Completed;
}
```

**Review Comment Example**:
```
❌ "Bad variable names."

✅ "Consider more descriptive names:
- 'o' → 'orders'
- 't' → 'daysBack'
- 'r' → 'completedOrderIds'

Also, Status == 2 is a magic number. Use OrderStatus.Completed enum instead.

This makes the code self-documenting and easier for the next developer."
```

---

### 4. Testing

- [ ] Are there tests for new functionality?
- [ ] Do tests cover edge cases?
- [ ] Are tests meaningful (not just for coverage)?
- [ ] Are test names descriptive?
- [ ] Is test setup clear?

```csharp
// ❌ BAD: Vague test name, unclear purpose
[Fact]
public void Test1()
{
    var service = new OrderService();
    var result = service.Process(new Order());
    Assert.True(result);
}

// ✅ GOOD: Descriptive name, clear AAA pattern
[Fact]
public void ProcessOrder_WhenInventoryInsufficient_ThrowsInsufficientStockException()
{
    // Arrange
    var mockInventory = new Mock<IInventoryRepository>();
    mockInventory.Setup(i => i.GetStock(It.IsAny<int>()))
        .ReturnsAsync(0); // No stock available

    var service = new OrderService(mockInventory.Object);
    var order = new Order
    {
        Items = new List<OrderItem>
        {
            new() { ProductId = 1, Quantity = 5 }
        }
    };

    // Act & Assert
    await Assert.ThrowsAsync<InsufficientStockException>(
        () => service.ProcessOrderAsync(order)
    );
}
```

---

### 5. Performance

- [ ] Are there any obvious performance issues?
- [ ] Are N+1 query problems avoided?
- [ ] Is caching used appropriately?
- [ ] Are large collections handled efficiently?
- [ ] Are async/await used correctly?

```csharp
// ❌ BAD: N+1 query problem
public async Task<List<OrderDto>> GetOrdersAsync()
{
    var orders = await _context.Orders.ToListAsync();

    foreach (var order in orders)
    {
        order.Customer = await _context.Customers
            .FirstOrDefaultAsync(c => c.Id == order.CustomerId);
        // N+1: One query for orders + N queries for customers!
    }

    return orders;
}

// ✅ GOOD: Single query with Include
public async Task<List<OrderDto>> GetOrdersAsync()
{
    return await _context.Orders
        .Include(o => o.Customer) // Eager load in single query
        .ToListAsync();
}
```

**Review Comment Example**:
```
❌ "This is slow."

✅ "This will cause N+1 queries: one for orders, then one per order for customers.

For 100 orders, that's 101 database calls!

Use .Include(o => o.Customer) to eager load in a single query.

See: [link to EF Core performance guide]"
```

---

### 6. Security

- [ ] Is input validated?
- [ ] Are SQL injection vulnerabilities avoided?
- [ ] Is authentication/authorization checked?
- [ ] Are secrets hard-coded? (they shouldn't be!)
- [ ] Is sensitive data logged?

```csharp
// ❌ BAD: SQL injection vulnerability!
public async Task<User> GetUserAsync(string username)
{
    var sql = $"SELECT * FROM Users WHERE Username = '{username}'";
    // If username = "admin' OR '1'='1", you're hacked!
    return await _connection.QueryFirstAsync<User>(sql);
}

// ✅ GOOD: Parameterized query
public async Task<User> GetUserAsync(string username)
{
    var sql = "SELECT * FROM Users WHERE Username = @Username";
    return await _connection.QueryFirstAsync<User>(sql, new { Username = username });
}
```

---

## 🎯 How to Give Effective Feedback

### Use the SBI Model

**S**ituation: Where in the code
**B**ehavior: What you observed
**I**mpact: Why it matters

```markdown
❌ BAD COMMENT:
"This method is terrible."

✅ GOOD COMMENT:
**Situation**: In OrderService.ProcessOrder (line 45)
**Behavior**: The method is 150 lines long and handles validation, DB access, email, and logging
**Impact**: This makes it hard to test, modify, and violates Single Responsibility Principle

**Suggestion**: Consider extracting into smaller, focused methods:
1. ValidateOrder()
2. SaveOrder()
3. SendNotification()

This will make each piece testable in isolation.
```

---

### Positive First, Then Constructive

```markdown
❌ BAD:
"Your code is inefficient and doesn't follow our standards."

✅ GOOD:
"Great work on implementing the new payment flow! The error handling is thorough.

I have a few suggestions:

1. **Performance**: The query on line 67 could be optimized with Include()
   to avoid N+1 queries. This will be important as we scale.

2. **Standards**: Can you extract the validation logic to PaymentValidator?
   This keeps our service classes focused (see architecture guide).

Happy to pair on these if helpful!"
```

---

### Be Specific and Actionable

```markdown
❌ VAGUE:
"The naming could be better."

✅ SPECIFIC:
"Consider renaming:
- 'ProcessData()' → 'CalculateMonthlyRevenue()' (describes what it does)
- 'temp' → 'unprocessedOrders' (more descriptive)
- 'x' → 'order' (clear in forEach context)

Descriptive names make the code self-documenting."
```

---

### Ask Questions, Don't Command

```markdown
❌ COMMANDING:
"Change this to use async/await."

✅ QUESTIONING:
"Have you considered using async/await here? Since we're calling
the database, async would prevent thread blocking.

If you're intentionally using sync, can you share the reasoning?"
```

---

## 🔄 Code Review Process

### For Reviewers

#### 1. **Respond Quickly**

**SLA**: Aim for < 4 hours for critical PRs, < 24 hours for others

**Why**: Blocked developers lose context and momentum

```markdown
If you can't review in time:
"I see this PR. Won't be able to review until tomorrow. @teammate can you cover?"
```

#### 2. **Review in Layers**

Don't try to catch everything at once:

**First Pass** (5 min):
- High-level design
- Is this the right approach?
- Major architectural issues

**Second Pass** (15 min):
- Logic and correctness
- Tests
- Edge cases

**Third Pass** (10 min):
- Code quality
- Naming
- Comments

#### 3. **Distinguish Must-Fix from Nice-to-Have**

```markdown
❌ Everything looks critical:
"Fix the naming. Also add null check. Also extract this method."

✅ Clear priorities:
**🚨 Must Fix (Blocking)**:
- SQL injection vulnerability on line 45 must be fixed

**⚠️ Should Fix (Important)**:
- Missing null check on line 67 could cause crashes

**💡 Nice to Have (Optional)**:
- Consider extracting this 20-line method for readability
```

#### 4. **Approve with Minor Comments**

Don't block PRs for trivial issues:

```markdown
✅ "LGTM! 🚀

Minor suggestion (don't block on this):
- Line 34: Consider extracting to a constant

Feel free to merge after addressing the null check issue."
```

---

### For PR Authors

#### 1. **Keep PRs Small**

**Ideal**: < 400 lines of changes
**Max**: < 800 lines

**Why**: Reviewers find 95% of bugs in small PRs, only 50% in large PRs

```markdown
✅ Good PR:
"Add payment validation logic"
- 3 files changed
- 250 lines

❌ Bad PR:
"Implement entire checkout flow"
- 25 files changed
- 2,500 lines
```

**Tip**: Break large features into multiple PRs

#### 2. **Write Descriptive PR Descriptions**

```markdown
❌ BAD:
"Fixed bug"

✅ GOOD:
## Summary
Fix NullReferenceException when processing orders with deleted products

## Problem
Orders with deleted products caused crashes in production (see bug #456)

## Solution
- Added null check in OrderService.ProcessOrder()
- Return clear error message instead of crashing
- Added test for deleted product scenario

## Testing
- Unit tests: OrderServiceTests.ProcessOrder_DeletedProduct_ReturnsError
- Manual: Tested with order 12345 (has deleted product)
```

#### 3. **Respond to All Comments**

Even if you disagree:

```markdown
Reviewer: "Consider extracting this to a method"

✅ Good Response:
"Good point! I've extracted it to CalculateSubtotal().
See commit abc123."

OR

"I considered that, but this logic is only used here and
is simple enough (4 lines). Extracting might add complexity
without benefit. WDYT?"
```

#### 4. **Use Draft PRs for Early Feedback**

```markdown
# [DRAFT] Implement new payment flow

I'm implementing the payment flow from design doc X.

Current status:
- ✅ Basic flow implemented
- ⏳ Working on error handling
- ❌ Tests not done yet

**Question**: Is this the right approach? See PaymentService.cs

Not ready to merge, just want early architecture feedback.
```

---

## 🎤 Common Interview Questions

### Q1: "How do you conduct effective code reviews?"

**Good Answer**:

"I use a structured approach:

**1. Review Layers**:
- First: Architecture and design (right approach?)
- Second: Logic and correctness
- Third: Code quality and style

**2. Feedback Framework (SBI)**:
- **Situation**: Where in the code
- **Behavior**: What I observed
- **Impact**: Why it matters + Suggestion

**Example**:
```
Instead of: 'This is wrong'

I say: 'In OrderService.ProcessOrder (line 45), this method is
150 lines and handles validation, DB, and email. This makes it
hard to test. Consider extracting into OrderValidator, OrderRepository,
and EmailService.'
```

**3. Balance**:
- Start with positive feedback
- Distinguish blocking issues from suggestions
- Ask questions, don't command

**4. Speed**: Aim for <4 hours response time

**5. Education**: Code reviews are teaching moments
- Link to docs
- Explain WHY, not just WHAT
- Pair on complex issues

**Result**: At my last company, this approach:
- Reduced review time from 3 days to <1 day
- Improved code quality (bug rate down 40%)
- Team reported reviews felt collaborative, not adversarial"

---

### Q2: "How do you handle disagreements in code reviews?"

**Good Answer**:

"I use a structured escalation approach:

**Level 1: Discuss in PR Comments**
- Ask questions to understand their reasoning
- Share my concerns with data/examples
- Stay curious, not combative

**Example**:
```
Them: 'Let's use MongoDB for this'
Me: 'Can you share your reasoning? My concern is we don't have MongoDB
expertise and this data is relational. Have you considered PostgreSQL?'
```

**Level 2: Synchronous Discussion**
If async discussion goes back-and-forth 3+ times:
- Jump on a call
- Whiteboard the options
- Usually resolve in 15 minutes vs. days of comments

**Level 3: Document Trade-offs**
If we still disagree:
- Create an Architecture Decision Record (ADR)
- List both approaches with pros/cons
- Make a decision (my call as tech lead or escalate)
- Document WHY for future reference

**Level 4: Escalate**
For major decisions:
- Get input from architect/engineering manager
- Present both sides fairly
- Accept the decision and move forward

**Important Principles**:
1. **Disagree and commit**: Once decision is made, fully support it
2. **No ego**: Be willing to be wrong
3. **Data over opinion**: Use benchmarks, not hunches

**Real Example**:
Engineer wanted to rewrite service in Go. I wanted to stay with C#.

We:
- Listed pros/cons
- Did a POC (2 days each)
- Evaluated learning curve for team
- Decided C# was better for our context (team expertise, timeline)
- Documented decision in ADR

He was disappointed but respected the process and became a strong contributor."

---

## 📚 Quick Reference

### Code Review Checklist Summary

**Functionality**:
- [ ] Works as intended
- [ ] Edge cases handled
- [ ] Error handling appropriate

**Design**:
- [ ] SOLID principles followed
- [ ] Right architectural layer
- [ ] Dependencies injected

**Readability**:
- [ ] Clear naming
- [ ] Appropriate complexity
- [ ] Useful comments

**Testing**:
- [ ] Tests included
- [ ] Edge cases covered
- [ ] Descriptive test names

**Performance**:
- [ ] No N+1 queries
- [ ] Async/await used correctly
- [ ] Appropriate caching

**Security**:
- [ ] Input validated
- [ ] No SQL injection
- [ ] Auth/authz checked
- [ ] No secrets hardcoded

---

## 🔗 Related Topics
- [Team Mentoring](../leadership/team-mentoring.md)
- [SOLID Principles](../architecture/solid-principles.md)
- [Architecture Decisions](./architecture-decisions.md)
- [Tech Lead Behavioral Questions](../interview-questions/tech-lead-behavioral.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐⭐☆ (1-5 stars)
**Priority for Tech Lead**: 🔥 CRITICAL
