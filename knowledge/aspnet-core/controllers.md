# ASP.NET Core Controllers

> Official Documentation: https://learn.microsoft.com/aspnet/core/web-api/

## Overview

ASP.NET Core controllers handle incoming HTTP requests and return responses. The Web API controller model provides attribute routing, automatic model validation, and content negotiation out of the box. Controllers inherit from `ControllerBase` for API scenarios or `Controller` for MVC with view support.

---

## 1. Controller Base Classes

### ControllerBase vs Controller

| Base Class | Namespace | Use Case | View Support |
|---|---|---|---|
| `ControllerBase` | Microsoft.AspNetCore.Mvc | Web APIs | No |
| `Controller` | Microsoft.AspNetCore.Mvc | MVC + Views | Yes (View(), PartialView()) |

```csharp
using Microsoft.AspNetCore.Mvc;

namespace MyApp.Controllers;

// API controller - use ControllerBase
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;

    public ProductsController(IProductService productService)
    {
        _productService = productService;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetAll()
    {
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }
}

// MVC controller - use Controller (includes view support)
public class HomeController : Controller
{
    public IActionResult Index()
    {
        return View(); // Returns a Razor view
    }
}
```

---

## 2. The [ApiController] Attribute

The `[ApiController]` attribute applies several opinionated behaviors that streamline API development.

### Behaviors Enabled by [ApiController]

| Behavior | Description |
|---|---|
| Automatic 400 responses | Returns `ValidationProblemDetails` when `ModelState` is invalid |
| Binding source inference | `[FromBody]` inferred for complex types, `[FromQuery]` for simple types |
| Multipart/form-data inference | `[FromForm]` inferred for `IFormFile` parameters |
| Problem details for errors | Error responses use RFC 7807 `ProblemDetails` format |
| Attribute routing required | Convention-based routing is not available |

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    // [ApiController] automatically returns 400 if CreateOrderDto fails validation
    // No need for manual ModelState.IsValid checks
    [HttpPost]
    public async Task<ActionResult<OrderDto>> Create(CreateOrderDto dto)
    {
        // If we reach here, model validation has already passed
        var order = await _orderService.CreateAsync(dto);
        return CreatedAtAction(nameof(GetById), new { id = order.Id }, order);
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<OrderDto>> GetById(int id)
    {
        var order = await _orderService.GetByIdAsync(id);
        return order is null ? NotFound() : Ok(order);
    }
}
```

### Customizing ProblemDetails Response

```csharp
builder.Services.AddControllers()
    .ConfigureApiBehaviorOptions(options =>
    {
        options.InvalidModelStateResponseFactory = context =>
        {
            var problemDetails = new ValidationProblemDetails(context.ModelState)
            {
                Type = "https://myapi.com/validation-error",
                Title = "Validation Error",
                Status = StatusCodes.Status422UnprocessableEntity,
                Detail = "One or more validation errors occurred.",
                Instance = context.HttpContext.Request.Path
            };

            problemDetails.Extensions.Add("traceId", context.HttpContext.TraceIdentifier);

            return new UnprocessableEntityObjectResult(problemDetails)
            {
                ContentTypes = { "application/problem+json" }
            };
        };
    });
```

---

## 3. Routing

### Route Templates

```csharp
[ApiController]
[Route("api/v{version:int}/[controller]")]  // api/v1/products
public class ProductsController : ControllerBase
{
    // GET api/v1/products
    [HttpGet]
    public ActionResult<IEnumerable<ProductDto>> GetAll() => Ok(_products);

    // GET api/v1/products/5
    [HttpGet("{id:int}")]
    public ActionResult<ProductDto> GetById(int id) => Ok(_product);

    // GET api/v1/products/search?q=laptop&minPrice=500
    [HttpGet("search")]
    public ActionResult<IEnumerable<ProductDto>> Search(
        [FromQuery] string q,
        [FromQuery] decimal? minPrice) => Ok(_results);

    // GET api/v1/products/category/electronics
    [HttpGet("category/{categoryName}")]
    public ActionResult<IEnumerable<ProductDto>> GetByCategory(string categoryName) => Ok(_results);

    // POST api/v1/products
    [HttpPost]
    public ActionResult<ProductDto> Create(CreateProductDto dto) => CreatedAtAction(nameof(GetById), new { id = 1 }, _product);

    // PUT api/v1/products/5
    [HttpPut("{id:int}")]
    public IActionResult Update(int id, UpdateProductDto dto) => NoContent();

    // PATCH api/v1/products/5
    [HttpPatch("{id:int}")]
    public IActionResult Patch(int id, JsonPatchDocument<UpdateProductDto> patchDoc) => NoContent();

    // DELETE api/v1/products/5
    [HttpDelete("{id:int}")]
    public IActionResult Delete(int id) => NoContent();
}
```

### Route Constraints

| Constraint | Example | Matches |
|---|---|---|
| `int` | `{id:int}` | 123 |
| `long` | `{id:long}` | 9223372036854775807 |
| `guid` | `{id:guid}` | `CD2C1638-...` |
| `bool` | `{active:bool}` | true, false |
| `decimal` | `{price:decimal}` | 49.99 |
| `alpha` | `{name:alpha}` | Only letters |
| `minlength(n)` | `{name:minlength(3)}` | At least 3 chars |
| `maxlength(n)` | `{name:maxlength(50)}` | At most 50 chars |
| `length(n)` | `{code:length(6)}` | Exactly 6 chars |
| `min(n)` | `{age:min(18)}` | >= 18 |
| `max(n)` | `{age:max(120)}` | <= 120 |
| `range(m,n)` | `{age:range(18,120)}` | 18 to 120 |
| `regex(expr)` | `{code:regex(^[A-Z]{{3}}$)}` | 3 uppercase letters |
| `required` | `{name:required}` | Must have value |

```csharp
// Multiple constraints
[HttpGet("{id:int:min(1)}")]
public ActionResult<ProductDto> GetById(int id) => Ok(_product);

// Catch-all parameter
[HttpGet("files/{**path}")]
public IActionResult GetFile(string path) => PhysicalFile(path, "application/octet-stream");

// Optional parameter
[HttpGet("page/{pageNumber:int?}")]
public ActionResult<PagedResult<ProductDto>> GetPaged(int pageNumber = 1) => Ok(_results);
```

---

## 4. Model Binding

### Binding Source Attributes

| Attribute | Source | Default For |
|---|---|---|
| `[FromBody]` | Request body | Complex types (with [ApiController]) |
| `[FromQuery]` | Query string | Simple types, arrays |
| `[FromRoute]` | Route data | Route parameters |
| `[FromHeader]` | HTTP headers | N/A |
| `[FromForm]` | Form data | IFormFile, IFormFileCollection |
| `[FromServices]` | DI container | N/A |
| `[AsParameters]` | Multiple sources | Minimal API parameter objects |

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    // [FromBody] - inferred for CreateUserDto by [ApiController]
    [HttpPost]
    public async Task<ActionResult<UserDto>> Create(CreateUserDto dto)
    {
        var user = await _userService.CreateAsync(dto);
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }

    // [FromRoute] - inferred for 'id' from route template
    [HttpGet("{id:int}")]
    public async Task<ActionResult<UserDto>> GetById(int id)
    {
        var user = await _userService.GetByIdAsync(id);
        return user is null ? NotFound() : Ok(user);
    }

    // [FromQuery] - explicit for search parameters
    [HttpGet]
    public async Task<ActionResult<PagedResult<UserDto>>> Search(
        [FromQuery] string? name,
        [FromQuery] string? email,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20)
    {
        var result = await _userService.SearchAsync(name, email, page, pageSize);
        return Ok(result);
    }

    // [FromHeader] - custom header binding
    [HttpGet("me")]
    public async Task<ActionResult<UserDto>> GetCurrent(
        [FromHeader(Name = "X-Correlation-Id")] string? correlationId)
    {
        var user = await _userService.GetCurrentAsync();
        return Ok(user);
    }

    // [FromForm] - file upload
    [HttpPost("avatar")]
    public async Task<IActionResult> UploadAvatar(
        [FromForm] IFormFile file,
        [FromForm] string description)
    {
        if (file.Length == 0) return BadRequest("File is empty");
        await _userService.SaveAvatarAsync(file, description);
        return NoContent();
    }

    // [FromServices] - inject service directly into action
    [HttpGet("statistics")]
    public async Task<ActionResult<StatsDto>> GetStats(
        [FromServices] IStatisticsService statsService)
    {
        var stats = await statsService.GetUserStatsAsync();
        return Ok(stats);
    }
}
```

### Complex Model Binding

```csharp
public record SearchFilter
{
    [FromQuery(Name = "q")]
    public string? Query { get; init; }

    [FromQuery(Name = "sort")]
    public string SortBy { get; init; } = "name";

    [FromQuery(Name = "dir")]
    public string SortDirection { get; init; } = "asc";

    [FromQuery(Name = "page")]
    public int Page { get; init; } = 1;

    [FromQuery(Name = "size")]
    public int PageSize { get; init; } = 20;
}

[HttpGet("search")]
public async Task<ActionResult<PagedResult<ProductDto>>> Search(
    [FromQuery] SearchFilter filter)
{
    var results = await _productService.SearchAsync(filter);
    return Ok(results);
}
```

---

## 5. Action Results

### Common Action Result Helper Methods

| Method | Status Code | Use Case |
|---|---|---|
| `Ok(value)` | 200 | Successful GET |
| `Created(uri, value)` | 201 | Successful POST |
| `CreatedAtAction(action, routeValues, value)` | 201 | POST with location header |
| `CreatedAtRoute(routeName, routeValues, value)` | 201 | POST with named route |
| `Accepted()` | 202 | Async processing started |
| `NoContent()` | 204 | Successful PUT/DELETE |
| `BadRequest(error)` | 400 | Validation failure |
| `Unauthorized()` | 401 | Authentication required |
| `Forbid()` | 403 | Insufficient permissions |
| `NotFound()` | 404 | Resource not found |
| `Conflict(error)` | 409 | Resource conflict |
| `UnprocessableEntity(error)` | 422 | Semantic validation failure |
| `StatusCode(code)` | Any | Custom status code |
| `File(bytes, contentType)` | 200 | File download |
| `PhysicalFile(path, contentType)` | 200 | File from disk |

### IActionResult vs ActionResult<T>

```csharp
// IActionResult - untyped, no Swagger type inference
[HttpGet("{id}")]
public async Task<IActionResult> GetByIdUntyped(int id)
{
    var product = await _productService.GetByIdAsync(id);
    if (product is null) return NotFound();
    return Ok(product);
}

// ActionResult<T> - typed, Swagger infers response type automatically
[HttpGet("{id}")]
public async Task<ActionResult<ProductDto>> GetByIdTyped(int id)
{
    var product = await _productService.GetByIdAsync(id);
    if (product is null) return NotFound();
    return product; // Implicit conversion to Ok(product)
}

// Multiple response types with [ProducesResponseType]
[HttpPost]
[ProducesResponseType<ProductDto>(StatusCodes.Status201Created)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
[ProducesResponseType(StatusCodes.Status409Conflict)]
public async Task<ActionResult<ProductDto>> Create(CreateProductDto dto)
{
    if (await _productService.ExistsBySkuAsync(dto.Sku))
        return Conflict(new { message = $"Product with SKU '{dto.Sku}' already exists" });

    var product = await _productService.CreateAsync(dto);
    return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
}
```

### TypedResults (Minimal API Style in Controllers)

```csharp
// .NET 8+ allows TypedResults for better OpenAPI metadata
[HttpGet("{id}")]
public async Task<Results<Ok<ProductDto>, NotFound>> GetById(int id)
{
    var product = await _productService.GetByIdAsync(id);
    if (product is null) return TypedResults.NotFound();
    return TypedResults.Ok(product);
}

[HttpPost]
public async Task<Results<Created<ProductDto>, ValidationProblem, Conflict>> Create(
    CreateProductDto dto)
{
    if (await _productService.ExistsBySkuAsync(dto.Sku))
        return TypedResults.Conflict();

    var product = await _productService.CreateAsync(dto);
    return TypedResults.Created($"/api/products/{product.Id}", product);
}
```

---

## 6. Model Validation

### Data Annotations

```csharp
using System.ComponentModel.DataAnnotations;

public record CreateProductDto
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(200, MinimumLength = 2, ErrorMessage = "Name must be 2-200 characters")]
    public required string Name { get; init; }

    [StringLength(2000, ErrorMessage = "Description cannot exceed 2000 characters")]
    public string? Description { get; init; }

    [Required]
    [Range(0.01, 999999.99, ErrorMessage = "Price must be between 0.01 and 999,999.99")]
    public decimal Price { get; init; }

    [Required]
    [RegularExpression(@"^[A-Z]{2,4}-\d{4,8}$", ErrorMessage = "SKU must match format: XX-0000")]
    public required string Sku { get; init; }

    [Required]
    [Range(0, int.MaxValue, ErrorMessage = "Stock cannot be negative")]
    public int StockQuantity { get; init; }

    [Url(ErrorMessage = "Invalid URL format")]
    public string? ImageUrl { get; init; }

    [EmailAddress(ErrorMessage = "Invalid email format")]
    public string? SupplierEmail { get; init; }
}
```

### Custom Validation Attribute

```csharp
public class FutureDateAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext context)
    {
        if (value is DateTime dateTime && dateTime <= DateTime.UtcNow)
        {
            return new ValidationResult(
                ErrorMessage ?? "Date must be in the future",
                new[] { context.MemberName! });
        }

        return ValidationResult.Success;
    }
}

public class DateRangeAttribute : ValidationAttribute
{
    private readonly string _startDateProperty;

    public DateRangeAttribute(string startDateProperty)
    {
        _startDateProperty = startDateProperty;
    }

    protected override ValidationResult? IsValid(object? value, ValidationContext context)
    {
        var startDateProperty = context.ObjectType.GetProperty(_startDateProperty);
        if (startDateProperty is null)
            return new ValidationResult($"Unknown property: {_startDateProperty}");

        var startDate = (DateTime?)startDateProperty.GetValue(context.ObjectInstance);
        var endDate = (DateTime?)value;

        if (startDate.HasValue && endDate.HasValue && endDate <= startDate)
        {
            return new ValidationResult(
                ErrorMessage ?? "End date must be after start date");
        }

        return ValidationResult.Success;
    }
}

public record CreateEventDto
{
    [Required]
    public required string Title { get; init; }

    [Required]
    [FutureDate(ErrorMessage = "Start date must be in the future")]
    public DateTime StartDate { get; init; }

    [Required]
    [DateRange(nameof(StartDate), ErrorMessage = "End date must be after start date")]
    public DateTime EndDate { get; init; }
}
```

### IValidatableObject for Cross-Property Validation

```csharp
public record CreateOrderDto : IValidatableObject
{
    [Required]
    public required string CustomerId { get; init; }

    [Required]
    [MinLength(1, ErrorMessage = "At least one item is required")]
    public required List<OrderItemDto> Items { get; init; }

    public string? CouponCode { get; init; }
    public decimal? DiscountAmount { get; init; }

    public IEnumerable<ValidationResult> Validate(ValidationContext context)
    {
        if (CouponCode is not null && DiscountAmount is not null)
        {
            yield return new ValidationResult(
                "Cannot use both coupon code and discount amount",
                new[] { nameof(CouponCode), nameof(DiscountAmount) });
        }

        if (Items.Any(i => i.Quantity <= 0))
        {
            yield return new ValidationResult(
                "All items must have a positive quantity",
                new[] { nameof(Items) });
        }
    }
}
```

---

## 7. Action Filters

### Built-in Filter Types

| Filter Type | Interface | Purpose |
|---|---|---|
| Authorization | `IAuthorizationFilter` | Access control |
| Resource | `IResourceFilter` | Caching, short-circuiting |
| Action | `IActionFilter` | Before/after action execution |
| Exception | `IExceptionFilter` | Global exception handling |
| Result | `IResultFilter` | Before/after result execution |

### Custom Action Filter

```csharp
public class ValidateModelAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            context.Result = new BadRequestObjectResult(context.ModelState);
        }
    }
}

// Logging filter
public class LogExecutionTimeFilter : IAsyncActionFilter
{
    private readonly ILogger<LogExecutionTimeFilter> _logger;

    public LogExecutionTimeFilter(ILogger<LogExecutionTimeFilter> logger)
    {
        _logger = logger;
    }

    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)
    {
        var stopwatch = Stopwatch.StartNew();
        var actionName = context.ActionDescriptor.DisplayName;

        _logger.LogInformation("Executing {Action}", actionName);

        var resultContext = await next();

        stopwatch.Stop();
        var statusCode = (resultContext.Result as ObjectResult)?.StatusCode ?? 200;

        _logger.LogInformation(
            "Executed {Action} in {Elapsed}ms with status {StatusCode}",
            actionName, stopwatch.ElapsedMilliseconds, statusCode);
    }
}

// Register globally or per-controller
builder.Services.AddControllers(options =>
{
    options.Filters.Add<LogExecutionTimeFilter>();
});

// Per controller or action
[ServiceFilter(typeof(LogExecutionTimeFilter))]
[Route("api/[controller]")]
public class ProductsController : ControllerBase { }
```

### Exception Filter

```csharp
public class GlobalExceptionFilter : IExceptionFilter
{
    private readonly ILogger<GlobalExceptionFilter> _logger;
    private readonly IHostEnvironment _env;

    public GlobalExceptionFilter(ILogger<GlobalExceptionFilter> logger, IHostEnvironment env)
    {
        _logger = logger;
        _env = env;
    }

    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "Unhandled exception occurred");

        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "An error occurred while processing your request",
            Type = "https://tools.ietf.org/html/rfc7231#section-6.6.1"
        };

        if (_env.IsDevelopment())
        {
            problemDetails.Detail = context.Exception.ToString();
        }

        context.Result = new ObjectResult(problemDetails)
        {
            StatusCode = StatusCodes.Status500InternalServerError
        };

        context.ExceptionHandled = true;
    }
}
```

---

## 8. API Versioning

```csharp
// Install: Asp.Versioning.Mvc and Asp.Versioning.Mvc.ApiExplorer

builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-Api-Version"),
        new QueryStringApiVersionReader("api-version"));
})
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// URL segment versioning (preferred)
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public ActionResult<IEnumerable<ProductV1Dto>> GetAll()
        => Ok(new List<ProductV1Dto>());
}

[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("2.0")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public ActionResult<IEnumerable<ProductV2Dto>> GetAll()
        => Ok(new List<ProductV2Dto>());

    // Map deprecated endpoint
    [HttpGet]
    [ApiVersion("1.0", Deprecated = true)]
    [MapToApiVersion("1.0")]
    public ActionResult<IEnumerable<ProductV1Dto>> GetAllV1()
        => Ok(new List<ProductV1Dto>());
}
```

---

## 9. Content Negotiation

```csharp
builder.Services.AddControllers(options =>
{
    options.RespectBrowserAcceptHeader = true;
    options.ReturnHttpNotAcceptable = true; // Returns 406 for unsupported formats
})
.AddXmlSerializerFormatters()     // Add XML support
.AddJsonOptions(options =>
{
    options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    options.JsonSerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
    options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
    options.JsonSerializerOptions.ReferenceHandler = ReferenceHandler.IgnoreCycles;
});

// Custom output formatter
public class CsvOutputFormatter : TextOutputFormatter
{
    public CsvOutputFormatter()
    {
        SupportedMediaTypes.Add("text/csv");
        SupportedEncodings.Add(Encoding.UTF8);
    }

    protected override bool CanWriteType(Type? type)
    {
        return typeof(IEnumerable).IsAssignableFrom(type);
    }

    public override async Task WriteResponseBodyAsync(
        OutputFormatterWriteContext context,
        Encoding selectedEncoding)
    {
        var response = context.HttpContext.Response;
        var items = (IEnumerable<object>)context.Object!;

        foreach (var item in items)
        {
            var properties = item.GetType().GetProperties();
            var values = properties.Select(p => p.GetValue(item)?.ToString() ?? "");
            await response.WriteAsync(string.Join(",", values) + "\n");
        }
    }
}
```

---

## 10. Async Controller Actions

```csharp
[ApiController]
[Route("api/[controller]")]
public class ReportsController : ControllerBase
{
    private readonly IReportService _reportService;

    public ReportsController(IReportService reportService)
    {
        _reportService = reportService;
    }

    // CancellationToken is automatically bound from the request
    [HttpGet("{id}")]
    public async Task<ActionResult<ReportDto>> GetById(
        int id,
        CancellationToken cancellationToken)
    {
        var report = await _reportService.GetByIdAsync(id, cancellationToken);
        return report is null ? NotFound() : Ok(report);
    }

    // Long-running operation with accepted pattern
    [HttpPost("generate")]
    public async Task<IActionResult> GenerateReport(
        GenerateReportRequest request,
        CancellationToken cancellationToken)
    {
        var jobId = await _reportService.QueueGenerationAsync(request, cancellationToken);

        return AcceptedAtAction(
            nameof(GetJobStatus),
            new { jobId },
            new { jobId, status = "Processing" });
    }

    [HttpGet("jobs/{jobId}")]
    public async Task<ActionResult<JobStatusDto>> GetJobStatus(
        string jobId,
        CancellationToken cancellationToken)
    {
        var status = await _reportService.GetJobStatusAsync(jobId, cancellationToken);
        return status is null ? NotFound() : Ok(status);
    }

    // Streaming large responses
    [HttpGet("export")]
    public async Task ExportAll(CancellationToken cancellationToken)
    {
        Response.ContentType = "application/json";
        Response.Headers.ContentDisposition = "attachment; filename=reports.json";

        await using var writer = new Utf8JsonWriter(Response.BodyWriter);
        writer.WriteStartArray();

        await foreach (var report in _reportService.StreamAllAsync(cancellationToken))
        {
            JsonSerializer.Serialize(writer, report);
        }

        writer.WriteEndArray();
        await writer.FlushAsync(cancellationToken);
    }
}
```

---

## 11. Full CRUD Controller Example

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Authorization;

namespace MyApp.Controllers;

[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class CustomersController : ControllerBase
{
    private readonly ICustomerService _customerService;
    private readonly ILogger<CustomersController> _logger;

    public CustomersController(
        ICustomerService customerService,
        ILogger<CustomersController> logger)
    {
        _customerService = customerService;
        _logger = logger;
    }

    /// <summary>
    /// Get all customers with optional filtering and pagination.
    /// </summary>
    [HttpGet]
    [ProducesResponseType<PagedResult<CustomerDto>>(StatusCodes.Status200OK)]
    public async Task<ActionResult<PagedResult<CustomerDto>>> GetAll(
        [FromQuery] string? search,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        CancellationToken cancellationToken = default)
    {
        var result = await _customerService.GetAllAsync(
            search, page, pageSize, cancellationToken);
        return Ok(result);
    }

    /// <summary>
    /// Get a customer by ID.
    /// </summary>
    [HttpGet("{id:int}")]
    [ProducesResponseType<CustomerDto>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<CustomerDto>> GetById(
        int id,
        CancellationToken cancellationToken)
    {
        var customer = await _customerService.GetByIdAsync(id, cancellationToken);
        return customer is null ? NotFound() : Ok(customer);
    }

    /// <summary>
    /// Create a new customer.
    /// </summary>
    [HttpPost]
    [Authorize(Policy = "RequireManagerRole")]
    [ProducesResponseType<CustomerDto>(StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<ActionResult<CustomerDto>> Create(
        CreateCustomerDto dto,
        CancellationToken cancellationToken)
    {
        if (await _customerService.ExistsByEmailAsync(dto.Email, cancellationToken))
        {
            return Conflict(new ProblemDetails
            {
                Title = "Duplicate Email",
                Detail = $"A customer with email '{dto.Email}' already exists",
                Status = StatusCodes.Status409Conflict
            });
        }

        var customer = await _customerService.CreateAsync(dto, cancellationToken);
        _logger.LogInformation("Customer {Id} created", customer.Id);
        return CreatedAtAction(nameof(GetById), new { id = customer.Id }, customer);
    }

    /// <summary>
    /// Update an existing customer.
    /// </summary>
    [HttpPut("{id:int}")]
    [Authorize(Policy = "RequireManagerRole")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Update(
        int id,
        UpdateCustomerDto dto,
        CancellationToken cancellationToken)
    {
        var exists = await _customerService.ExistsAsync(id, cancellationToken);
        if (!exists) return NotFound();

        await _customerService.UpdateAsync(id, dto, cancellationToken);
        return NoContent();
    }

    /// <summary>
    /// Delete a customer.
    /// </summary>
    [HttpDelete("{id:int}")]
    [Authorize(Roles = "Admin")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(
        int id,
        CancellationToken cancellationToken)
    {
        var exists = await _customerService.ExistsAsync(id, cancellationToken);
        if (!exists) return NotFound();

        await _customerService.DeleteAsync(id, cancellationToken);
        _logger.LogInformation("Customer {Id} deleted", customer.Id);
        return NoContent();
    }
}
```

---

## Best Practices

1. **Always use `[ApiController]`** - It provides automatic model validation, binding source inference, and ProblemDetails responses.
2. **Prefer `ActionResult<T>` over `IActionResult`** - Enables better Swagger documentation and type safety.
3. **Accept `CancellationToken`** - Every async action should accept and pass `CancellationToken` for graceful request cancellation.
4. **Keep controllers thin** - Delegate business logic to services; controllers should only handle HTTP concerns.
5. **Use `CreatedAtAction` for POST** - Returns a 201 with a Location header pointing to the new resource.
6. **Document response types** - Use `[ProducesResponseType]` for accurate OpenAPI documentation.
7. **Use route constraints** - Validate route parameters at the routing level with `{id:int:min(1)}`.
8. **Return ProblemDetails for errors** - Follow RFC 7807 for consistent error responses.

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---|---|---|
| Forgetting `[ApiController]` | No automatic validation, no binding inference | Always add to API controllers |
| Using `Controller` for APIs | Unnecessary view dependencies loaded | Use `ControllerBase` for Web APIs |
| Blocking async calls | `Task.Result` or `.Wait()` causes deadlocks | Always use `await` |
| Missing `CancellationToken` | Requests cannot be cancelled by client | Add `CancellationToken` to every async action |
| Fat controllers | Business logic mixed with HTTP concerns | Extract to service layer |
| Inconsistent error formats | Mix of string errors and ProblemDetails | Standardize on ProblemDetails |
| Ignoring route conflicts | Ambiguous routes cause runtime errors | Use unique, specific route templates |
| Not setting `[Produces]` | Swagger shows incorrect content types | Add `[Produces("application/json")]` |
