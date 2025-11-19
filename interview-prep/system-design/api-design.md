# RESTful API Design Best Practices

## 📝 Overview

As a Tech Lead, designing good APIs is critical. APIs are **contracts** between systems - poorly designed APIs cause integration pain, poor performance, and maintenance nightmares.

**Key Principle**: APIs should be intuitive, consistent, and follow industry standards.

**Your Role**:
- Define API contracts and standards
- Review API designs for consistency
- Ensure scalability and performance
- Balance flexibility with simplicity

---

## 🎯 REST API Principles

### Resource-Based URLs

```
✅ GOOD: Resource-based (nouns)
GET    /api/products           # Get all products
GET    /api/products/123       # Get specific product
POST   /api/products           # Create product
PUT    /api/products/123       # Update product
DELETE /api/products/123       # Delete product

❌ BAD: Action-based (verbs)
GET    /api/getProducts
GET    /api/getProductById/123
POST   /api/createProduct
POST   /api/updateProduct
POST   /api/deleteProduct
```

**Key**: Use HTTP verbs for actions, URLs for resources

---

## 📐 RESTful API Design Patterns

### 1. Basic CRUD Operations

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;

    // GET api/products
    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetProducts(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20)
    {
        var products = await _productService.GetProductsAsync(page, pageSize);
        return Ok(products);
    }

    // GET api/products/5
    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDto>> GetProduct(int id)
    {
        var product = await _productService.GetProductAsync(id);

        if (product == null)
            return NotFound();

        return Ok(product);
    }

    // POST api/products
    [HttpPost]
    public async Task<ActionResult<ProductDto>> CreateProduct(
        [FromBody] CreateProductRequest request)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        var product = await _productService.CreateProductAsync(request);

        return CreatedAtAction(
            nameof(GetProduct),
            new { id = product.Id },
            product);
    }

    // PUT api/products/5
    [HttpPut("{id}")]
    public async Task<ActionResult> UpdateProduct(
        int id,
        [FromBody] UpdateProductRequest request)
    {
        if (id != request.Id)
            return BadRequest("ID mismatch");

        var exists = await _productService.ProductExistsAsync(id);
        if (!exists)
            return NotFound();

        await _productService.UpdateProductAsync(request);

        return NoContent();
    }

    // DELETE api/products/5
    [HttpDelete("{id}")]
    public async Task<ActionResult> DeleteProduct(int id)
    {
        var exists = await _productService.ProductExistsAsync(id);
        if (!exists)
            return NotFound();

        await _productService.DeleteProductAsync(id);

        return NoContent();
    }
}
```

---

### 2. Nested Resources

```csharp
// Products have reviews (one-to-many relationship)

// GET api/products/5/reviews
[HttpGet("products/{productId}/reviews")]
public async Task<ActionResult<IEnumerable<ReviewDto>>> GetProductReviews(int productId)
{
    var product = await _productService.ProductExistsAsync(productId);
    if (!product)
        return NotFound($"Product {productId} not found");

    var reviews = await _reviewService.GetReviewsByProductAsync(productId);
    return Ok(reviews);
}

// GET api/products/5/reviews/10
[HttpGet("products/{productId}/reviews/{reviewId}")]
public async Task<ActionResult<ReviewDto>> GetProductReview(
    int productId,
    int reviewId)
{
    var review = await _reviewService.GetReviewAsync(reviewId);

    if (review == null || review.ProductId != productId)
        return NotFound();

    return Ok(review);
}

// POST api/products/5/reviews
[HttpPost("products/{productId}/reviews")]
public async Task<ActionResult<ReviewDto>> CreateProductReview(
    int productId,
    [FromBody] CreateReviewRequest request)
{
    var product = await _productService.ProductExistsAsync(productId);
    if (!product)
        return NotFound($"Product {productId} not found");

    request.ProductId = productId;
    var review = await _reviewService.CreateReviewAsync(request);

    return CreatedAtAction(
        nameof(GetProductReview),
        new { productId = productId, reviewId = review.Id },
        review);
}
```

**Best Practice**: Limit nesting to 2 levels

```
✅ GOOD:
/api/products/5/reviews/10

❌ BAD (too deep):
/api/categories/1/products/5/reviews/10/comments/3
```

---

### 3. Filtering, Sorting, Pagination

```csharp
// GET api/products?category=electronics&minPrice=100&maxPrice=500&sort=price&order=asc&page=1&pageSize=20
[HttpGet]
public async Task<ActionResult<PagedResult<ProductDto>>> GetProducts(
    [FromQuery] ProductQueryParameters parameters)
{
    var result = await _productService.GetProductsAsync(parameters);
    return Ok(result);
}

// Query parameters model
public class ProductQueryParameters
{
    // Filtering
    public string Category { get; set; }
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
    public bool? InStock { get; set; }
    public string Search { get; set; }

    // Sorting
    public string SortBy { get; set; } = "name";
    public string Order { get; set; } = "asc"; // asc or desc

    // Pagination
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 20;

    // Validation
    public void Validate()
    {
        if (Page < 1) Page = 1;
        if (PageSize < 1) PageSize = 20;
        if (PageSize > 100) PageSize = 100; // Max limit
    }
}

// Paged result
public class PagedResult<T>
{
    public IEnumerable<T> Data { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalCount { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPrevious => Page > 1;
    public bool HasNext => Page < TotalPages;
}
```

**Pagination Response**:

```json
{
  "data": [
    { "id": 1, "name": "Product 1", "price": 100 },
    { "id": 2, "name": "Product 2", "price": 150 }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 250,
  "totalPages": 13,
  "hasPrevious": false,
  "hasNext": true
}
```

---

### 4. Versioning

**Option 1: URL Path Versioning** (Recommended)

```csharp
// Version 1
[ApiController]
[Route("api/v1/[controller]")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDtoV1>> GetProduct(int id)
    {
        // Old structure
        return Ok(new ProductDtoV1
        {
            Id = 1,
            Name = "Product",
            Price = 100
        });
    }
}

// Version 2 (breaking changes)
[ApiController]
[Route("api/v2/[controller]")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDtoV2>> GetProduct(int id)
    {
        // New structure
        return Ok(new ProductDtoV2
        {
            Id = 1,
            Name = "Product",
            Pricing = new PricingInfo
            {
                Amount = 100,
                Currency = "USD"
            }
        });
    }
}
```

**Option 2: Header Versioning**

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]
    [ApiVersion("1.0")]
    public async Task<ActionResult<ProductDtoV1>> GetProductV1(int id)
    {
        // Version 1 logic
    }

    [HttpGet("{id}")]
    [ApiVersion("2.0")]
    public async Task<ActionResult<ProductDtoV2>> GetProductV2(int id)
    {
        // Version 2 logic
    }
}

// Client sends:
// GET /api/products/5
// Header: api-version: 2.0
```

**Versioning Strategy**:
- Use semantic versioning (v1, v2, v3)
- Support at least 2 versions (current + previous)
- Announce deprecation 6-12 months in advance
- Document breaking changes

---

## 📊 HTTP Status Codes

### Success (2xx)

```csharp
// 200 OK - Successful GET, PUT, DELETE
[HttpGet("{id}")]
public async Task<ActionResult<ProductDto>> GetProduct(int id)
{
    var product = await _service.GetAsync(id);
    return Ok(product); // 200
}

// 201 Created - Successful POST
[HttpPost]
public async Task<ActionResult<ProductDto>> CreateProduct(CreateProductRequest request)
{
    var product = await _service.CreateAsync(request);
    return CreatedAtAction(
        nameof(GetProduct),
        new { id = product.Id },
        product); // 201
}

// 204 No Content - Successful PUT/DELETE with no response body
[HttpPut("{id}")]
public async Task<ActionResult> UpdateProduct(int id, UpdateProductRequest request)
{
    await _service.UpdateAsync(id, request);
    return NoContent(); // 204
}
```

### Client Errors (4xx)

```csharp
// 400 Bad Request - Invalid input
[HttpPost]
public async Task<ActionResult> CreateProduct(CreateProductRequest request)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState); // 400
}

// 401 Unauthorized - Not authenticated
[HttpGet]
[Authorize]
public async Task<ActionResult> GetOrders()
{
    // If user not authenticated, middleware returns 401
}

// 403 Forbidden - Authenticated but not authorized
[HttpGet("{id}")]
[Authorize(Roles = "Admin")]
public async Task<ActionResult> GetOrder(int id)
{
    // If user is not admin, returns 403
}

// 404 Not Found - Resource doesn't exist
[HttpGet("{id}")]
public async Task<ActionResult<ProductDto>> GetProduct(int id)
{
    var product = await _service.GetAsync(id);
    if (product == null)
        return NotFound(); // 404

    return Ok(product);
}

// 409 Conflict - Business rule violation
[HttpPost("{id}/purchase")]
public async Task<ActionResult> PurchaseProduct(int id)
{
    try
    {
        await _service.PurchaseAsync(id);
        return Ok();
    }
    catch (OutOfStockException ex)
    {
        return Conflict(new { message = ex.Message }); // 409
    }
}

// 422 Unprocessable Entity - Validation error (semantic)
[HttpPost]
public async Task<ActionResult> CreateOrder(CreateOrderRequest request)
{
    // Syntactically valid, but semantically invalid
    if (request.Items.Count == 0)
    {
        return UnprocessableEntity(new
        {
            message = "Order must contain at least one item"
        }); // 422
    }
}
```

### Server Errors (5xx)

```csharp
// 500 Internal Server Error - Unexpected error
[HttpGet("{id}")]
public async Task<ActionResult<ProductDto>> GetProduct(int id)
{
    try
    {
        var product = await _service.GetAsync(id);
        return Ok(product);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error fetching product {ProductId}", id);
        return StatusCode(500, new
        {
            message = "An error occurred while processing your request"
        });
    }
}

// 503 Service Unavailable - Service down or overloaded
[HttpGet]
public async Task<ActionResult> HealthCheck()
{
    var isHealthy = await _healthCheckService.IsHealthyAsync();

    if (!isHealthy)
        return StatusCode(503, new { message = "Service temporarily unavailable" });

    return Ok(new { status = "healthy" });
}
```

---

## 🔒 API Security

### 1. Authentication & Authorization

```csharp
// JWT Authentication
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options =>
            {
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer = true,
                    ValidateAudience = true,
                    ValidateLifetime = true,
                    ValidateIssuerSigningKey = true,
                    ValidIssuer = Configuration["Jwt:Issuer"],
                    ValidAudience = Configuration["Jwt:Audience"],
                    IssuerSigningKey = new SymmetricSecurityKey(
                        Encoding.UTF8.GetBytes(Configuration["Jwt:Key"]))
                };
            });

        services.AddAuthorization(options =>
        {
            options.AddPolicy("AdminOnly", policy =>
                policy.RequireRole("Admin"));

            options.AddPolicy("CanEditProduct", policy =>
                policy.RequireClaim("Permission", "Product.Edit"));
        });
    }
}

// Usage in controller
[HttpPut("{id}")]
[Authorize(Policy = "CanEditProduct")]
public async Task<ActionResult> UpdateProduct(int id, UpdateProductRequest request)
{
    await _service.UpdateAsync(id, request);
    return NoContent();
}
```

---

### 2. Rate Limiting

```csharp
// Using AspNetCoreRateLimit
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddMemoryCache();

        services.Configure<IpRateLimitOptions>(options =>
        {
            options.GeneralRules = new List<RateLimitRule>
            {
                new RateLimitRule
                {
                    Endpoint = "*",
                    Limit = 100,
                    Period = "1m" // 100 requests per minute
                },
                new RateLimitRule
                {
                    Endpoint = "POST:/api/*",
                    Limit = 20,
                    Period = "1m" // 20 POST requests per minute
                }
            };
        });

        services.AddSingleton<IIpPolicyStore, MemoryCacheIpPolicyStore>();
        services.AddSingleton<IRateLimitCounterStore, MemoryCacheRateLimitCounterStore>();
        services.AddSingleton<IRateLimitConfiguration, RateLimitConfiguration>();
    }
}

// Custom rate limiting by user
[HttpGet]
public async Task<ActionResult> GetProducts()
{
    var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    var requestCount = await _rateLimiter.IncrementAsync(userId);

    if (requestCount > 1000) // 1000 requests per hour
    {
        return StatusCode(429, new
        {
            message = "Rate limit exceeded. Try again later.",
            retryAfter = 3600 // seconds
        });
    }

    // Process request
}
```

---

### 3. Input Validation

```csharp
// Data annotations
public class CreateProductRequest
{
    [Required(ErrorMessage = "Product name is required")]
    [StringLength(100, MinimumLength = 3,
        ErrorMessage = "Name must be between 3 and 100 characters")]
    public string Name { get; set; }

    [Range(0.01, 1000000,
        ErrorMessage = "Price must be between 0.01 and 1,000,000")]
    public decimal Price { get; set; }

    [Required]
    [Url(ErrorMessage = "Invalid URL format")]
    public string ImageUrl { get; set; }

    [EmailAddress(ErrorMessage = "Invalid email format")]
    public string ContactEmail { get; set; }
}

// FluentValidation (more advanced)
public class CreateProductRequestValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductRequestValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Product name is required")
            .Length(3, 100).WithMessage("Name must be between 3 and 100 characters")
            .Must(BeValidProductName).WithMessage("Product name contains invalid characters");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than 0")
            .LessThanOrEqualTo(1000000).WithMessage("Price cannot exceed 1,000,000");

        RuleFor(x => x.CategoryId)
            .MustAsync(CategoryExists).WithMessage("Invalid category ID");
    }

    private bool BeValidProductName(string name)
    {
        return !name.Contains("<") && !name.Contains(">"); // Prevent XSS
    }

    private async Task<bool> CategoryExists(int categoryId, CancellationToken token)
    {
        return await _categoryRepository.ExistsAsync(categoryId);
    }
}
```

---

## 📄 API Documentation (OpenAPI/Swagger)

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddSwaggerGen(c =>
        {
            c.SwaggerDoc("v1", new OpenApiInfo
            {
                Title = "Product API",
                Version = "v1",
                Description = "API for managing products",
                Contact = new OpenApiContact
                {
                    Name = "Tech Support",
                    Email = "support@example.com"
                }
            });

            // JWT Authentication in Swagger
            c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
            {
                Description = "JWT Authorization header using the Bearer scheme.",
                Name = "Authorization",
                In = ParameterLocation.Header,
                Type = SecuritySchemeType.ApiKey,
                Scheme = "Bearer"
            });

            c.AddSecurityRequirement(new OpenApiSecurityRequirement
            {
                {
                    new OpenApiSecurityScheme
                    {
                        Reference = new OpenApiReference
                        {
                            Type = ReferenceType.SecurityScheme,
                            Id = "Bearer"
                        }
                    },
                    Array.Empty<string>()
                }
            });

            // Include XML comments
            var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
            var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
            c.IncludeXmlComments(xmlPath);
        });
    }
}

// Controller with XML documentation
/// <summary>
/// Gets a product by ID
/// </summary>
/// <param name="id">The product ID</param>
/// <returns>The product details</returns>
/// <response code="200">Returns the product</response>
/// <response code="404">Product not found</response>
[HttpGet("{id}")]
[ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<ActionResult<ProductDto>> GetProduct(int id)
{
    var product = await _service.GetAsync(id);

    if (product == null)
        return NotFound();

    return Ok(product);
}
```

---

## ⚡ Performance Best Practices

### 1. Async/Await

```csharp
// ✅ GOOD: Async all the way
[HttpGet("{id}")]
public async Task<ActionResult<ProductDto>> GetProduct(int id)
{
    var product = await _service.GetProductAsync(id);
    return Ok(product);
}

// ❌ BAD: Blocking on async
[HttpGet("{id}")]
public ActionResult<ProductDto> GetProduct(int id)
{
    var product = _service.GetProductAsync(id).Result; // Deadlock risk!
    return Ok(product);
}
```

---

### 2. Response Compression

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddResponseCompression(options =>
        {
            options.EnableForHttps = true;
            options.Providers.Add<GzipCompressionProvider>();
            options.Providers.Add<BrotliCompressionProvider>();
        });

        services.Configure<GzipCompressionProviderOptions>(options =>
        {
            options.Level = CompressionLevel.Fastest;
        });
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseResponseCompression();
    }
}
```

---

### 3. Caching

```csharp
// Response caching
[HttpGet]
[ResponseCache(Duration = 60)] // Cache for 60 seconds
public async Task<ActionResult<IEnumerable<ProductDto>>> GetProducts()
{
    var products = await _service.GetAllProductsAsync();
    return Ok(products);
}

// Cache-Control header
[HttpGet("{id}")]
public async Task<ActionResult<ProductDto>> GetProduct(int id)
{
    var product = await _service.GetAsync(id);

    Response.Headers["Cache-Control"] = "public, max-age=300"; // 5 minutes
    Response.Headers["ETag"] = GenerateETag(product);

    return Ok(product);
}

// Conditional requests (ETag)
[HttpGet("{id}")]
public async Task<ActionResult<ProductDto>> GetProduct(int id)
{
    var product = await _service.GetAsync(id);
    var etag = GenerateETag(product);

    if (Request.Headers["If-None-Match"] == etag)
        return StatusCode(304); // Not Modified

    Response.Headers["ETag"] = etag;
    return Ok(product);
}
```

---

## 🎤 Common Interview Questions

### Q1: "How do you design a RESTful API?"

**Good Answer**:

"I follow REST principles and industry best practices:

**1. Resource-Based URLs (Nouns, not Verbs)**

```
✅ GET /api/products/123
❌ GET /api/getProduct?id=123
```

**2. HTTP Verbs for Actions**
- GET: Retrieve
- POST: Create
- PUT: Update (full)
- PATCH: Update (partial)
- DELETE: Delete

**3. Consistent Response Structure**

```json
{
  "data": { "id": 123, "name": "Product" },
  "meta": { "timestamp": "2025-01-01T00:00:00Z" }
}
```

**4. Proper HTTP Status Codes**
- 200 OK: Success
- 201 Created: Resource created
- 400 Bad Request: Invalid input
- 404 Not Found: Resource not found
- 500 Internal Server Error: Server error

**5. Versioning**

```
/api/v1/products
/api/v2/products
```

**6. Pagination, Filtering, Sorting**

```
GET /api/products?page=1&pageSize=20&sort=price&order=asc&category=electronics
```

**7. Security**
- JWT authentication
- Rate limiting (100 req/min)
- Input validation
- HTTPS only

**8. Documentation**
- OpenAPI/Swagger
- Clear examples
- Error responses documented

**Real Example**:

At my last company, we redesigned a legacy SOAP API to REST.

**Before** (SOAP):
- Complex XML payloads
- Single endpoint
- No versioning
- Poor documentation

**After** (REST):
- Simple JSON
- Resource-based URLs
- Versioned (v1, v2)
- Swagger docs

**Result**:
- Partner integration time: 2 weeks → 2 days
- API errors: 15% → 2%
- Developer satisfaction: High"

---

### Q2: "How do you handle API versioning?"

**Good Answer**:

"I use **URL path versioning** as it's the most explicit and discoverable:

```
/api/v1/products
/api/v2/products
```

**Why URL Path**:
- Clear and visible
- Easy to test (just change URL)
- Works with all HTTP clients
- Discoverable (you see the version)

**Other Options** (and why I don't prefer them):

**Header Versioning**:
```
GET /api/products
Header: api-version: 2.0
```

Pros: Clean URLs
Cons: Hidden, harder to test in browser

**Query Parameter**:
```
GET /api/products?version=2
```

Cons: Mixes with other query params

---

**Version Strategy**:

**1. Semantic Versioning**: v1, v2, v3 (major versions only)

**2. Support Window**: Support current + previous version (minimum 12 months)

Example:
- Jan 2025: Launch v2, v1 still supported
- Jan 2026: Deprecate v1 (with warnings in response)
- Jul 2026: Remove v1

**3. Breaking vs Non-Breaking Changes**

**Non-Breaking** (same version):
- Adding new endpoint
- Adding optional field
- Relaxing validation

**Breaking** (new version):
- Removing endpoint
- Removing field
- Changing field type
- Stricter validation

**4. Deprecation Strategy**

```csharp
[HttpGet]
[Route("api/v1/products")]
[Obsolete("API v1 is deprecated. Please use v2. v1 will be removed on 2026-07-01")]
public async Task<ActionResult> GetProductsV1()
{
    // Add deprecation header
    Response.Headers["X-API-Deprecated"] = "true";
    Response.Headers["X-API-Sunset"] = "2026-07-01";
    Response.Headers["Link"] = "</api/v2/products>; rel=\"successor-version\"";

    // Old logic
}
```

**Real Example**:

At my previous company, we had no versioning. Every change broke clients.

I implemented versioning:
- v1: Legacy API (18 months support)
- v2: New API with breaking changes
- Email announcements: 12 months, 6 months, 1 month before v1 sunset
- Dashboard showing version usage

**Result**:
- 95% migrated before v1 removal
- Zero breaking changes for clients
- Smooth deprecation"

---

### Q3: "How would you design an API for high performance and scalability?"

**Good Answer**:

"I'd focus on multiple layers of optimization:

**1. Efficient Querying**

```csharp
// ❌ BAD: Returns all fields, always
GET /api/products/123
{
  "id": 123,
  "name": "...",
  "description": "...", // Large field
  "specifications": {...}, // Large nested object
  "reviews": [...] // 1000 reviews!
}

// ✅ GOOD: Field selection
GET /api/products/123?fields=id,name,price
{
  "id": 123,
  "name": "Product",
  "price": 100
}
```

**2. Pagination**

```csharp
// ❌ BAD: Returns 100K products
GET /api/products

// ✅ GOOD: Paginated
GET /api/products?page=1&pageSize=20
{
  "data": [...], // 20 items
  "page": 1,
  "totalPages": 5000,
  "totalCount": 100000
}
```

**3. Caching**

```csharp
[HttpGet("{id}")]
[ResponseCache(Duration = 300)] // Cache for 5 minutes
public async Task<ActionResult<ProductDto>> GetProduct(int id)
{
    // Add ETags for conditional requests
    var product = await _service.GetAsync(id);
    var etag = GenerateETag(product);

    if (Request.Headers["If-None-Match"] == etag)
        return StatusCode(304); // Not Modified (save bandwidth)

    Response.Headers["ETag"] = etag;
    return Ok(product);
}
```

**4. Async/Await**

```csharp
// ✅ GOOD: Non-blocking
public async Task<ActionResult> GetProducts()
{
    var products = await _service.GetProductsAsync();
    return Ok(products);
}
```

**5. Compression**

```csharp
// Gzip/Brotli compression
services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
});

// 500KB JSON → 50KB compressed (10x reduction)
```

**6. Database Optimization**

```csharp
// Use projections (select only needed fields)
var products = await _context.Products
    .Select(p => new ProductListDto
    {
        Id = p.Id,
        Name = p.Name,
        Price = p.Price
        // Don't load description, specs, reviews
    })
    .ToListAsync();
```

**7. Rate Limiting**

```csharp
// Prevent abuse
// 1000 requests per minute per user
services.AddRateLimiting(options =>
{
    options.GlobalRules = new[]
    {
        new RateLimitRule { Limit = 1000, Period = "1m" }
    };
});
```

**8. CDN for Static Responses**

```
Client → CDN (cached) → API Server
       ↓ (90% cache hit)
    Instant response
```

---

**Real Example**:

E-commerce API: 10K requests/sec during Black Friday

**Bottlenecks**:
- API response time: 500ms
- Database CPU: 90%

**Optimizations**:
1. Added Redis caching (product catalog, 5 min TTL)
2. Implemented field selection (?fields=id,name,price)
3. Paginated all list endpoints (max 100 items)
4. Added response compression (Brotli)
5. Optimized DB queries (projections, indexes)

**Result**:
- Response time: 500ms → 50ms (10x faster)
- Handled 50K req/sec (5x capacity)
- Database CPU: 90% → 20%
- Bandwidth: 50% reduction (compression)

**Cost Savings**: $5K/month (reduced server scaling)"

---

## 📚 Quick Reference

### REST API Design Checklist

**URLs**:
- [ ] Resource-based (nouns), not action-based (verbs)
- [ ] Consistent naming (plural: /products, not /product)
- [ ] Nested resources max 2 levels (/products/5/reviews)

**HTTP Methods**:
- [ ] GET for retrieval (idempotent, cacheable)
- [ ] POST for creation
- [ ] PUT for full update (idempotent)
- [ ] PATCH for partial update
- [ ] DELETE for removal (idempotent)

**Status Codes**:
- [ ] 200 OK, 201 Created, 204 No Content
- [ ] 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found
- [ ] 500 Internal Server Error, 503 Service Unavailable

**Features**:
- [ ] Pagination (page, pageSize, total count)
- [ ] Filtering (query parameters)
- [ ] Sorting (sortBy, order)
- [ ] Field selection (?fields=id,name)
- [ ] Versioning (/api/v1/...)

**Security**:
- [ ] Authentication (JWT)
- [ ] Authorization (roles, policies)
- [ ] Rate limiting
- [ ] Input validation
- [ ] HTTPS only

**Performance**:
- [ ] Async/await
- [ ] Response compression (Gzip/Brotli)
- [ ] Caching (ResponseCache, ETags)
- [ ] Efficient queries (projections)

**Documentation**:
- [ ] OpenAPI/Swagger
- [ ] Example requests/responses
- [ ] Error codes explained

---

## 🔗 Related Topics
- [Microservices Architecture](../architecture/microservices.md)
- [Caching Strategies](./caching.md)
- [Distributed Systems](./distributed-systems.md)
- [Authentication & Authorization](../dotnet/authentication.md)

---

**Last Reviewed**: November 19, 2025
**Confidence Level**: ⭐⭐⭐⭐⭐ (1-5 stars)
**Priority for Tech Lead**: 🔥 CRITICAL
