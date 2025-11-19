# Engineering Best Practices & CI/CD

## 📝 Overview

As a Tech Lead, you're responsible for establishing and maintaining engineering best practices across your team. This includes CI/CD pipelines, testing strategies, code quality standards, and automation.

**Key Responsibility**: Create a culture where quality is built-in, not bolted-on.

**Your Role**:
- Define and champion engineering standards
- Build/maintain CI/CD pipelines
- Ensure automated testing at all levels
- Foster code quality and security practices

---

## 🚀 CI/CD Pipeline Best Practices

### The Complete Pipeline

```yaml
# Example: Azure DevOps/GitHub Actions Pipeline
name: CI/CD Pipeline

trigger:
  branches:
    - main
    - develop
  pull_requests:
    - main

stages:
  - stage: Build
    jobs:
      - job: BuildAndTest
        steps:
          - task: DotNetCoreCLI@2
            displayName: 'Restore Dependencies'
            inputs:
              command: 'restore'

          - task: DotNetCoreCLI@2
            displayName: 'Build Solution'
            inputs:
              command: 'build'
              arguments: '--configuration Release --no-restore'

          - task: DotNetCoreCLI@2
            displayName: 'Run Unit Tests'
            inputs:
              command: 'test'
              arguments: '--configuration Release --no-build --collect:"XPlat Code Coverage"'

          - task: PublishCodeCoverageResults@1
            displayName: 'Publish Code Coverage'
            inputs:
              codeCoverageTool: 'Cobertura'
              summaryFileLocation: '$(Agent.TempDirectory)/**/coverage.cobertura.xml'
              failIfCoverageEmpty: true

  - stage: CodeQuality
    jobs:
      - job: StaticAnalysis
        steps:
          - task: SonarCloudPrepare@1
            displayName: 'Prepare SonarCloud'

          - task: DotNetCoreCLI@2
            displayName: 'Build for Analysis'
            inputs:
              command: 'build'

          - task: SonarCloudAnalyze@1
            displayName: 'Run Code Analysis'

          - task: SonarCloudPublish@1
            displayName: 'Publish Analysis Results'

  - stage: SecurityScan
    jobs:
      - job: DependencyScan
        steps:
          - task: dependency-check@6
            displayName: 'OWASP Dependency Check'

          - task: WhiteSource@21
            displayName: 'WhiteSource Security Scan'

  - stage: IntegrationTests
    jobs:
      - job: IntegrationTest
        steps:
          - task: DockerCompose@0
            displayName: 'Start Dependencies (Redis, SQL, RabbitMQ)'
            inputs:
              action: 'Run services'
              dockerComposeFile: 'docker-compose.test.yml'

          - task: DotNetCoreCLI@2
            displayName: 'Run Integration Tests'
            inputs:
              command: 'test'
              projects: '**/*IntegrationTests.csproj'

  - stage: Deploy_Dev
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
    jobs:
      - deployment: DeployToDev
        environment: 'Development'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: 'Deploy to Dev Environment'

  - stage: Deploy_Prod
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployToProduction
        environment: 'Production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: 'Deploy to Production'
                - task: Slack@1
                  displayName: 'Notify Team'
```

### Pipeline Stages Explained

| Stage | Purpose | Fail Fast? | Duration Target |
|-------|---------|------------|-----------------|
| **Build** | Compile code | ✅ Yes (immediate) | < 2 minutes |
| **Unit Tests** | Fast, isolated tests | ✅ Yes (immediate) | < 5 minutes |
| **Code Quality** | Static analysis, linting | ✅ Yes | < 3 minutes |
| **Security Scan** | Vulnerability detection | ⚠️ Warning only | < 5 minutes |
| **Integration Tests** | Test with real dependencies | ✅ Yes | < 10 minutes |
| **Deploy Dev** | Auto-deploy to dev | ✅ Yes | < 5 minutes |
| **Deploy Prod** | Manual approval + deploy | ✅ Yes | < 5 minutes |

**Total Pipeline Time**: < 30 minutes (optimize for fast feedback)

---

## 🧪 Testing Strategy: The Testing Pyramid

### The Right Balance

```
        /\
       /  \
      / E2E\ (5%)     ← Few, expensive, slow
     /______\
    /        \
   /Integration\ (15%) ← Some, test boundaries
  /____________\
 /              \
/   Unit Tests   \ (70%) ← Many, cheap, fast
/__________________\

• Unit Tests: 70% of tests
• Integration Tests: 20% of tests
• E2E/UI Tests: 10% of tests
```

### Unit Tests (70%)

**Goal**: Test single units in isolation

```csharp
// Example: OrderService Unit Test
public class OrderServiceTests
{
    [Fact]
    public async Task ProcessOrder_ValidOrder_CalculatesTotalCorrectly()
    {
        // Arrange
        var mockRepository = new Mock<IOrderRepository>();
        var mockPricingService = new Mock<IPricingService>();

        mockPricingService.Setup(s => s.GetPrice(1))
            .ReturnsAsync(10.00m);

        var service = new OrderService(
            mockRepository.Object,
            mockPricingService.Object
        );

        var order = new Order
        {
            Items = new List<OrderItem>
            {
                new() { ProductId = 1, Quantity = 3 }
            }
        };

        // Act
        await service.ProcessOrderAsync(order);

        // Assert
        Assert.Equal(30.00m, order.Total);
        mockRepository.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.Once);
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public async Task ProcessOrder_InvalidQuantity_ThrowsException(int quantity)
    {
        // Arrange
        var service = new OrderService(null, null);
        var order = new Order
        {
            Items = new List<OrderItem>
            {
                new() { ProductId = 1, Quantity = quantity }
            }
        };

        // Act & Assert
        await Assert.ThrowsAsync<InvalidOperationException>(
            () => service.ProcessOrderAsync(order)
        );
    }
}
```

**Characteristics**:
- Fast (milliseconds)
- No external dependencies
- Use mocks/stubs
- High coverage (80%+ is realistic)

---

### Integration Tests (20%)

**Goal**: Test components working together with real dependencies

```csharp
// Example: Integration Test with Real Database
public class OrderRepositoryIntegrationTests : IClassFixture<DatabaseFixture>
{
    private readonly ApplicationDbContext _context;

    public OrderRepositoryIntegrationTests(DatabaseFixture fixture)
    {
        _context = fixture.Context;
    }

    [Fact]
    public async Task GetOrderWithItems_OrderExists_ReturnsOrderWithItems()
    {
        // Arrange
        var repository = new OrderRepository(_context);

        // Seed test data
        var order = new Order
        {
            CustomerId = "TEST123",
            Items = new List<OrderItem>
            {
                new() { ProductId = 1, Quantity = 2, Price = 10.00m }
            }
        };

        await _context.Orders.AddAsync(order);
        await _context.SaveChangesAsync();

        // Act
        var result = await repository.GetOrderWithItemsAsync(order.Id);

        // Assert
        Assert.NotNull(result);
        Assert.Equal("TEST123", result.CustomerId);
        Assert.Single(result.Items);
        Assert.Equal(2, result.Items.First().Quantity);
    }
}

// Fixture for database setup/cleanup
public class DatabaseFixture : IDisposable
{
    public ApplicationDbContext Context { get; private set; }

    public DatabaseFixture()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=TestDb")
            .Options;

        Context = new ApplicationDbContext(options);
        Context.Database.EnsureCreated();
    }

    public void Dispose()
    {
        Context.Database.EnsureDeleted();
        Context.Dispose();
    }
}
```

**Docker Compose for Integration Tests**:

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      ACCEPT_EULA: Y
      SA_PASSWORD: Test@1234
    ports:
      - "1433:1433"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"
```

**Characteristics**:
- Slower (seconds)
- Real databases, message queues, caches
- Test boundaries between components
- Cover critical paths

---

### E2E Tests (10%)

**Goal**: Test entire user flows

```csharp
// Example: E2E Test with Playwright
public class CheckoutE2ETests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public CheckoutE2ETests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    [Fact]
    public async Task CompleteCheckoutFlow_ValidUser_OrderCreated()
    {
        // Arrange
        await using var playwright = await Playwright.CreateAsync();
        await using var browser = await playwright.Chromium.LaunchAsync();
        var page = await browser.NewPageAsync();

        // Act
        // 1. Navigate to product page
        await page.GotoAsync("http://localhost:5000/products/1");

        // 2. Add to cart
        await page.ClickAsync("button#add-to-cart");
        await page.WaitForSelectorAsync(".cart-count:has-text('1')");

        // 3. Go to checkout
        await page.ClickAsync("a#checkout");

        // 4. Fill shipping info
        await page.FillAsync("#shipping-name", "John Doe");
        await page.FillAsync("#shipping-address", "123 Main St");
        await page.FillAsync("#shipping-city", "Seattle");

        // 5. Fill payment info
        await page.FillAsync("#card-number", "4111111111111111");
        await page.FillAsync("#card-cvv", "123");

        // 6. Submit order
        await page.ClickAsync("button#place-order");

        // Assert
        await page.WaitForSelectorAsync(".order-confirmation");
        var orderNumber = await page.TextContentAsync("#order-number");
        Assert.NotNull(orderNumber);
    }
}
```

**Characteristics**:
- Slowest (minutes)
- Full system + UI
- Cover critical business flows only
- Brittle, expensive to maintain

---

## 📊 Code Quality Standards

### Code Coverage

```csharp
// .runsettings for Code Coverage
<?xml version="1.0" encoding="utf-8"?>
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat Code Coverage">
        <Configuration>
          <Format>cobertura,opencover</Format>
          <Exclude>[*Tests]*,[*TestHelpers]*</Exclude>
          <ExcludeByAttribute>Obsolete,GeneratedCode,ExcludeFromCodeCoverage</ExcludeByAttribute>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

**Coverage Targets**:
| Layer | Target | Why |
|-------|--------|-----|
| **Domain/Core** | 90%+ | Critical business logic |
| **Application/Services** | 80%+ | Orchestration logic |
| **Infrastructure** | 60%+ | Data access, external deps |
| **API/Controllers** | 50%+ | Thin layer, mostly routing |

**Quality Gate**:
```yaml
# Example: Quality Gate Configuration
quality_gate:
  coverage:
    new_code: 80%  # New code must have 80%+ coverage
    overall: 70%   # Overall project coverage

  code_smells:
    max: 0  # No code smells in new code

  duplications:
    max: 3%  # Max 3% code duplication

  security:
    vulnerabilities: 0  # Zero vulnerabilities
    security_hotspots: 0  # All security hotspots reviewed
```

---

### Static Analysis

```xml
<!-- .editorconfig -->
root = true

[*.cs]
# Code Style Rules
csharp_style_var_for_built_in_types = true
csharp_style_var_when_type_is_apparent = true
csharp_prefer_braces = true

# Naming Conventions
dotnet_naming_rule.interfaces_must_start_with_i.severity = error
dotnet_naming_rule.interfaces_must_start_with_i.symbols = interface
dotnet_naming_rule.interfaces_must_start_with_i.style = begins_with_i

# Code Quality Rules
dotnet_diagnostic.CA1031.severity = warning  # Do not catch general exception types
dotnet_diagnostic.CA2007.severity = warning  # Consider calling ConfigureAwait
dotnet_diagnostic.CA1062.severity = warning  # Validate arguments of public methods

# Performance Rules
dotnet_diagnostic.CA1806.severity = error  # Do not ignore method results
dotnet_diagnostic.CA1822.severity = warning  # Mark members as static
dotnet_diagnostic.CA1827.severity = warning  # Do not use Count/LongCount when Any can be used
```

**SonarQube Rules**:

```csharp
// ❌ BAD: Code Smell - Too Complex
public decimal CalculatePrice(Order order)
{
    if (order.Customer.IsPremium)
    {
        if (order.TotalAmount > 1000)
        {
            if (order.Items.Count > 10)
            {
                if (DateTime.Now.Month == 12)
                {
                    return order.TotalAmount * 0.6m; // Cyclomatic complexity = 5
                }
            }
        }
    }
    return order.TotalAmount;
}

// ✅ GOOD: Refactored with Guard Clauses
public decimal CalculatePrice(Order order)
{
    if (!order.Customer.IsPremium) return order.TotalAmount;
    if (order.TotalAmount <= 1000) return order.TotalAmount;
    if (order.Items.Count <= 10) return order.TotalAmount;
    if (DateTime.Now.Month != 12) return order.TotalAmount;

    return order.TotalAmount * 0.6m; // Cyclomatic complexity = 1
}
```

---

## 🔒 Security Practices

### Dependency Scanning

```yaml
# Example: GitHub Dependabot
version: 2
updates:
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10

    # Auto-merge security updates
    labels:
      - "security"
      - "dependencies"

    # Ignore major version updates (need review)
    ignore:
      - dependency-name: "*"
        update-types: ["version-update:semver-major"]
```

### Secret Scanning

```bash
# Pre-commit hook to prevent secrets
#!/bin/bash

# Check for common secret patterns
if git diff --cached --name-only | xargs grep -E "password|api[_-]?key|secret|token" -i; then
    echo "❌ Potential secret detected! Commit blocked."
    echo "   Remove secrets and use environment variables or Key Vault."
    exit 1
fi

# Check for connection strings
if git diff --cached --name-only | xargs grep -E "Server=|Database=|User Id=" -i; then
    echo "❌ Connection string detected! Commit blocked."
    exit 1
fi

echo "✅ No secrets detected"
exit 0
```

---

## 🏗️ Infrastructure as Code

### Environment Configuration

```csharp
// appsettings.json (DEV)
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=MyApp"
  },
  "Redis": {
    "Host": "localhost:6379"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Debug"
    }
  }
}

// appsettings.Production.json
{
  "ConnectionStrings": {
    "DefaultConnection": "" // Injected from Azure Key Vault
  },
  "Redis": {
    "Host": "" // Injected from environment
  },
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  }
}
```

```csharp
// Program.cs - Secure Configuration
public class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);

        // Add Azure Key Vault for production
        if (builder.Environment.IsProduction())
        {
            var keyVaultEndpoint = builder.Configuration["KeyVaultEndpoint"];
            builder.Configuration.AddAzureKeyVault(
                new Uri(keyVaultEndpoint),
                new DefaultAzureCredential()
            );
        }

        builder.Services.AddControllers();

        var app = builder.Build();
        app.Run();
    }
}
```

---

## 📈 Monitoring & Observability

### Structured Logging

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public async Task ProcessOrderAsync(Order order)
    {
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["OrderId"] = order.Id,
            ["CustomerId"] = order.CustomerId,
            ["TotalAmount"] = order.TotalAmount
        }))
        {
            _logger.LogInformation("Processing order {OrderId} for customer {CustomerId}",
                order.Id, order.CustomerId);

            try
            {
                await _repository.AddAsync(order);
                await _unitOfWork.CommitAsync();

                _logger.LogInformation("Order {OrderId} processed successfully", order.Id);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process order {OrderId}", order.Id);
                throw;
            }
        }
    }
}
```

### Application Insights

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddApplicationInsightsTelemetry();

    // Custom telemetry
    services.AddSingleton<ITelemetryInitializer, CustomTelemetryInitializer>();
}

// Custom metrics
public class OrderMetrics
{
    private readonly TelemetryClient _telemetry;

    public void TrackOrderProcessed(Order order)
    {
        _telemetry.TrackEvent("OrderProcessed", new Dictionary<string, string>
        {
            ["OrderId"] = order.Id.ToString(),
            ["TotalAmount"] = order.TotalAmount.ToString()
        });

        _telemetry.TrackMetric("OrderValue", order.TotalAmount);
    }
}
```

---

## 🎤 Common Interview Questions

### Q1: "How do you ensure code quality in your team?"

**Good Answer**:

"I use a multi-layered approach:

**1. Automated Quality Gates**:
- CI/CD pipeline with quality checks:
  * Build succeeds
  * All tests pass (unit + integration)
  * 80%+ code coverage on new code
  * No security vulnerabilities
  * SonarQube quality gate passes

**2. Testing Strategy (70-20-10)**:
- 70% unit tests (fast, cheap)
- 20% integration tests (real dependencies)
- 10% E2E tests (critical flows)

Target: 80% overall coverage, 90% on domain logic

**3. Code Review Standards**:
- All code reviewed before merge
- Checklist: Logic, design, tests, security, performance
- Response SLA: < 4 hours

**4. Static Analysis**:
- .editorconfig for consistent style
- SonarQube for code smells
- Dependency scanning for vulnerabilities

**5. Definition of Done**:
```markdown
A story is done when:
✅ Code written and reviewed
✅ Unit tests written (80%+ coverage)
✅ Integration tests for critical paths
✅ No SonarQube issues
✅ Documentation updated
✅ Deployed to dev and tested
```

**Real Example**:
At my last company, we had 40% bug escape rate to production.

After implementing these practices:
- Bug rate dropped to 8%
- Code coverage increased from 45% to 82%
- Deployment time reduced from 2 hours to 15 minutes
- Team confidence in releases improved dramatically

**Key**: Automate everything possible. Humans are fallible, pipelines are consistent."

---

### Q2: "Describe your ideal CI/CD pipeline."

**Good Answer**:

"My ideal pipeline balances speed, quality, and safety:

**Pipeline Stages** (< 30 minutes total):

1. **Build** (2 min): Compile, restore dependencies
2. **Unit Tests** (5 min): Fast, isolated tests
3. **Code Quality** (3 min): Linting, static analysis
4. **Security Scan** (5 min): Dependency check, SAST
5. **Integration Tests** (10 min): Real dependencies via Docker
6. **Deploy Dev** (5 min): Auto-deploy to dev environment
7. **Deploy Prod** (5 min): Manual approval + deployment

**Key Principles**:

**Fail Fast**:
```
Build fails → Stop immediately (don't waste time on tests)
Unit tests fail → Stop (don't run integration tests)
Security critical → Block deployment
```

**Progressive Deployment**:
```
Commit → Dev (automatic)
       → Staging (automatic after dev tests)
       → Prod (manual approval + canary deployment)
```

**Rollback Strategy**:
- Keep last 5 versions deployable
- Blue-green deployment for zero downtime
- Feature flags for instant kill switch

**Example Pipeline**:

```yaml
stages:
  - build
  - test
  - quality
  - security
  - deploy_dev
  - deploy_staging
  - deploy_prod

build:
  stage: build
  script:
    - dotnet restore
    - dotnet build --no-restore
  artifacts:
    paths:
      - bin/

test:
  stage: test
  needs: [build]
  script:
    - dotnet test --collect:"XPlat Code Coverage"
  coverage: '/\s+Line\s+coverage:\s+(\d+\.\d+)%/'

deploy_prod:
  stage: deploy_prod
  when: manual  # Require approval
  only:
    - main
  script:
    - az webapp deploy --name myapp --resource-group prod
```

**Metrics I Track**:
- Pipeline success rate (target: > 95%)
- Average pipeline duration (target: < 30 min)
- MTTR (Mean Time To Recovery) (target: < 30 min)
- Deployment frequency (target: multiple times/day)

**Real Example**:
At my last company, pipeline took 2 hours. Developers avoided running tests.

I optimized:
- Parallelized test execution
- Cached dependencies
- Moved slow E2E tests to nightly builds
- Result: 15 minutes, developers run tests confidently"

---

### Q3: "How do you balance test coverage with development speed?"

**Good Answer**:

"I don't see it as trade-off - good tests accelerate development:

**Short Term**: Writing tests feels slower
**Long Term**: Tests save massive time

**My Approach**:

**1. Risk-Based Testing**:
Not everything needs 100% coverage.

```
Critical path (payment, auth): 95%+ coverage
Business logic (domain): 85%+ coverage
Infrastructure (repos): 70%+ coverage
Simple DTOs, configs: 50%+ coverage (not worth effort)
```

**2. Write Tests That Matter**:

❌ **BAD** - Testing framework:
```csharp
[Fact]
public void List_Add_IncreasesCount()
{
    var list = new List<int>();
    list.Add(1);
    Assert.Equal(1, list.Count); // Testing .NET, not our code
}
```

✅ **GOOD** - Testing business logic:
```csharp
[Fact]
public void ProcessOrder_InsufficientInventory_ThrowsException()
{
    // Testing OUR business rule
}
```

**3. Test Pyramid** (70-20-10):
- Majority are fast unit tests (milliseconds)
- Some integration tests (seconds)
- Few E2E tests (minutes)

This keeps feedback loop fast.

**4. TDD for Complex Logic**:
For complex algorithms, TDD is FASTER:
- Write test first
- Clarifies requirements
- Catches bugs immediately
- Refactor with confidence

**5. Refactoring Time**:
When I see code that's hard to test, I refactor:

```csharp
// Hard to test (static, DateTime.Now)
public bool IsOrderExpired(Order order)
{
    return DateTime.Now > order.ExpiresAt;
}

// Easy to test (inject time provider)
public bool IsOrderExpired(Order order, ITimeProvider time)
{
    return time.Now > order.ExpiresAt;
}
```

**Real Example**:
Junior dev: "Tests are slowing me down!"

I paired with them for a week.

We had 2 production bugs in untested code that took 8 hours to fix.

Writing tests would've taken 2 hours and caught both.

They became a testing advocate.

**Key**: Tests are not overhead. They're investment that pays back quickly."

---

## 📚 Quick Reference

### CI/CD Pipeline Checklist

**✅ Must Have**:
- [ ] Build on every commit
- [ ] Run unit tests automatically
- [ ] Code coverage reporting (80%+ target)
- [ ] Static code analysis
- [ ] Security vulnerability scanning
- [ ] Auto-deploy to dev environment
- [ ] Manual approval for production

**🎯 Nice to Have**:
- [ ] Integration tests with Docker
- [ ] Performance testing
- [ ] Load testing
- [ ] Smoke tests after deployment
- [ ] Automatic rollback on failure

### Testing Pyramid

| Test Type | % of Tests | Duration | Dependencies | When to Run |
|-----------|-----------|----------|--------------|-------------|
| **Unit** | 70% | ms | None (mocked) | Every commit |
| **Integration** | 20% | sec | Real (Docker) | Every commit |
| **E2E** | 10% | min | Full system | Before deploy |

### Code Quality Metrics

| Metric | Target | Tool |
|--------|--------|------|
| **Code Coverage** | 80%+ new code | Coverlet |
| **Cyclomatic Complexity** | < 10 per method | SonarQube |
| **Code Duplications** | < 3% | SonarQube |
| **Security Vulnerabilities** | 0 critical | OWASP, WhiteSource |
| **Technical Debt** | < 5% | SonarQube |

---

## 🔗 Related Topics
- [Code Review Practices](../technical-leadership/code-review-practices.md)
- [Technical Debt Management](../technical-leadership/technical-debt.md)
- [Clean Architecture](../architecture/clean-architecture.md)
- [Agile Leadership](./agile-leadership.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐⭐⭐ (1-5 stars)
**Priority for Tech Lead**: 🔥 CRITICAL
