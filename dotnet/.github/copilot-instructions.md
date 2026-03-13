# Copilot Instructions — ASP.NET Core Professional Framework

This file complements `AGENTS.md`. When an agent builds the project, these rules
apply to every line of code generated. When a developer writes code manually,
Copilot applies these same rules in every inline suggestion.

---

## Primary Rule

Follow the phases and patterns defined in `AGENTS.md`. If there is a conflict
between this file and `AGENTS.md`, **`AGENTS.md` takes precedence**.

---

## Mandatory Stack

| Concern | Library / Pattern |
|---|---|
| Framework | ASP.NET Core .NET 10 LTS |
| Architecture | Modular Monolith — Clean Architecture per module |
| Mediator | MediatR |
| Validation | FluentValidation |
| ORM | Entity Framework Core 8 |
| Database | PostgreSQL 16 |
| Error handling | Result Pattern (never throw for business flow) |
| HTTP errors | ProblemDetails — RFC 7807 |
| Logging | Serilog with structured logging |
| Unit tests | xUnit + NSubstitute + FluentAssertions |
| Integration tests | Testcontainers + WebApplicationFactory |
| Architecture tests | NetArchTest |

---

## Project Layout

Every module lives under `src/Modules/{Module}Module/` with up to six layers:

```
src/
  Api/MyApi/                              ← Host: Program.cs, global middleware, DI root
  CommonUtils/                            ← Shared cross-cutting helpers
  Modules/{Module}Module/
    {Module}.Api/                          ← Controllers / endpoints for this module
    {Module}.Application/                  ← Commands, Queries, Handlers, Validators (MediatR)
    {Module}.Contracts/                    ← DTOs and interfaces shared across modules
    {Module}.DbMigrator/                   ← EF Core migrations for this module
    {Module}.Domain/                       ← Entities, Value Objects, domain errors
    {Module}.Infrastructure/               ← EF DbContext, repositories, external services
  Tests/{Module}.Test/                     ← Unit + integration tests per module
```

Lightweight modules that own no data only need: `Api`, `Application`, `Contracts`.

---

## TypeScript — Strict Rules (C# equivalent)

```csharp
// ✅ Correct — sealed handlers and validators (not designed for inheritance)
public sealed class CreateProductCommandHandler : IRequestHandler<...>
public sealed class CreateProductCommandValidator : AbstractValidator<...>

// ✅ Correct — records for immutable data contracts
public sealed record CreateProductCommand(string Name, decimal Price) : IRequest<Result<Guid>>;
public sealed record ProductResponse(Guid Id, string Name, decimal Price);

// ✅ Correct — Result<T> for operations that can fail
public static Result<Product> Create(string name, Money price)
{
    if (string.IsNullOrWhiteSpace(name))
        return ProductErrors.InvalidName;
    return new Product { Name = name, Price = price };
}

// ❌ Never — throwing for normal business flow
public static Product Create(string name, Money price)
{
    if (string.IsNullOrWhiteSpace(name))
        throw new ArgumentException("Name is required"); // use Result instead
}

// ❌ Never — .Result or .Wait() in async code
var product = _repository.GetByIdAsync(id).Result;   // deadlock risk
_unitOfWork.SaveChangesAsync().Wait();                // deadlock risk
```

---

## Domain Layer Rules

```csharp
// ✅ Entities with private setters and factory methods
// src/Modules/ModuleAModule/ModuleA.Domain/Entities/Product.cs
public sealed class Product : Entity
{
    private Product() { }  // EF Core constructor

    public string Name { get; private set; } = default!;
    public Money Price { get; private set; } = default!;
    public ProductStatus Status { get; private set; }

    public static Result<Product> Create(string name, Money price)
    {
        if (string.IsNullOrWhiteSpace(name))
            return ProductErrors.InvalidName;
        if (price.Amount <= 0)
            return ProductErrors.InvalidPrice;

        return new Product
        {
            Id = Guid.NewGuid(),
            Name = name.Trim(),
            Price = price,
            Status = ProductStatus.Active,
        };
    }

    public Result Deactivate()
    {
        if (Status == ProductStatus.Inactive)
            return ProductErrors.AlreadyInactive;
        Status = ProductStatus.Inactive;
        return Result.Success();
    }
}

// ✅ Domain errors as static members — never magic strings inline
// src/Modules/ModuleAModule/ModuleA.Domain/Errors/ProductErrors.cs
public static class ProductErrors
{
    public static readonly Error InvalidName =
        Error.Validation("Product.InvalidName", "Product name cannot be empty.");

    public static readonly Error InvalidPrice =
        Error.Validation("Product.InvalidPrice", "Product price must be greater than zero.");

    public static Error NotFound(Guid id) =>
        Error.NotFound("Product.NotFound", $"Product '{id}' was not found.");

    public static readonly Error AlreadyInactive =
        Error.Conflict("Product.AlreadyInactive", "Product is already inactive.");
}

// ✅ Value Objects with private constructors and validation
// src/Modules/ModuleAModule/ModuleA.Domain/ValueObjects/Money.cs
public sealed record Money
{
    private Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    public decimal Amount { get; }
    public string Currency { get; }
    public static readonly Money Zero = new(0, "USD");

    public static Result<Money> Create(decimal amount, string currency)
    {
        if (amount < 0) return Error.Validation("Money.Negative", "Amount cannot be negative.");
        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
            return Error.Validation("Money.InvalidCurrency", "Invalid ISO 4217 currency code.");
        return new Money(amount, currency);
    }
}
```

---

## Application Layer Rules

```csharp
// ✅ Commands and queries as immutable records
// src/Modules/ModuleAModule/ModuleA.Application/Products/Commands/CreateProduct/CreateProductCommand.cs
public sealed record CreateProductCommand(
    string Name,
    string Description,
    decimal Price,
    string Currency,
    int StockQuantity
) : IRequest<Result<Guid>>;

// ✅ Handler: use repository interface, never DbContext directly
// src/Modules/ModuleAModule/ModuleA.Application/Products/Commands/CreateProduct/CreateProductCommandHandler.cs
public sealed class CreateProductCommandHandler
    : IRequestHandler<CreateProductCommand, Result<Guid>>
{
    private readonly IProductRepository _products;
    private readonly IUnitOfWork _unitOfWork;

    public CreateProductCommandHandler(
        IProductRepository products,
        IUnitOfWork unitOfWork)
    {
        _products = products;
        _unitOfWork = unitOfWork;
    }

    public async Task<Result<Guid>> Handle(
        CreateProductCommand command,
        CancellationToken ct)
    {
        var priceResult = Money.Create(command.Price, command.Currency);
        if (priceResult.IsFailure) return priceResult.Error;

        var productResult = Product.Create(command.Name, priceResult.Value);
        if (productResult.IsFailure) return productResult.Error;

        await _products.AddAsync(productResult.Value, ct);
        await _unitOfWork.SaveChangesAsync(ct);

        return productResult.Value.Id;
    }
}

// ✅ Validator — covers all inputs before they reach the handler
// src/Modules/ModuleAModule/ModuleA.Application/Products/Commands/CreateProduct/CreateProductCommandValidator.cs
public sealed class CreateProductCommandValidator
    : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required.")
            .MaximumLength(200).WithMessage("Name cannot exceed 200 characters.");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than zero.");

        RuleFor(x => x.Currency)
            .NotEmpty()
            .Length(3).WithMessage("Currency must be a 3-letter ISO 4217 code.");

        RuleFor(x => x.StockQuantity)
            .GreaterThanOrEqualTo(0).WithMessage("Stock quantity cannot be negative.");
    }
}
```

---

## Infrastructure Layer Rules

```csharp
// ✅ Repository implementation — AsNoTracking on reads
// src/Modules/ModuleAModule/ModuleA.Infrastructure/Repositories/ProductRepository.cs
public sealed class ProductRepository : IProductRepository
{
    private readonly ModuleADbContext _context;

    public ProductRepository(ModuleADbContext context) => _context = context;

    public async Task<Product?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => await _context.Products
            .AsNoTracking()
            .FirstOrDefaultAsync(p => p.Id == id, ct);

    public async Task<IReadOnlyList<Product>> GetAllAsync(CancellationToken ct = default)
        => await _context.Products
            .AsNoTracking()
            .OrderBy(p => p.Name)
            .ToListAsync(ct);

    public async Task AddAsync(Product product, CancellationToken ct = default)
        => await _context.Products.AddAsync(product, ct);

    public void Update(Product product) => _context.Products.Update(product);
    public void Delete(Product product) => _context.Products.Remove(product);
}

// ✅ Fluent API configuration — keep separate from entity class
// src/Modules/ModuleAModule/ModuleA.Infrastructure/Persistence/Configurations/ProductConfiguration.cs
public sealed class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");
        builder.HasKey(p => p.Id);

        builder.Property(p => p.Name)
            .HasMaxLength(200)
            .IsRequired();

        builder.Property(p => p.Status)
            .HasConversion<string>()
            .HasMaxLength(20);

        builder.OwnsOne(p => p.Price, price =>
        {
            price.Property(m => m.Amount)
                .HasColumnName("PriceAmount")
                .HasPrecision(18, 2)
                .IsRequired();
            price.Property(m => m.Currency)
                .HasColumnName("PriceCurrency")
                .HasMaxLength(3)
                .IsRequired();
        });
    }
}
```

---

## Api Layer Rules

```csharp
// ✅ Controller — thin, no business logic, always uses ISender
// src/Modules/ModuleAModule/ModuleA.Api/Controllers/ProductsController.cs
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/products")]
[Authorize]
public sealed class ProductsController : ControllerBase
{
    private readonly ISender _sender;

    public ProductsController(ISender sender) => _sender = sender;

    /// <summary>Creates a new product.</summary>
    /// <response code="201">Product created successfully.</response>
    /// <response code="400">Validation error in the request body.</response>
    /// <response code="409">A conflict with existing data was detected.</response>
    [HttpPost]
    [ProducesResponseType(typeof(Guid), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status409Conflict)]
    public async Task<IActionResult> Create(
        [FromBody] CreateProductRequest request,
        CancellationToken ct)
    {
        var result = await _sender.Send(request.ToCommand(), ct);
        return result.IsSuccess
            ? CreatedAtAction(nameof(GetById), new { id = result.Value }, result.Value)
            : result.Error.ToProblemResult(this);
    }

    /// <summary>Gets a product by its unique identifier.</summary>
    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(ProductResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(Guid id, CancellationToken ct)
    {
        var result = await _sender.Send(new GetProductByIdQuery(id), ct);
        return result.IsSuccess ? Ok(result.Value) : result.Error.ToProblemResult(this);
    }
}

// ❌ Never — business logic in controller
[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateProductRequest req)
{
    if (req.Price > 1000) { ... } // belongs in Domain
    var product = new Product { Name = req.Name }; // belongs in factory method
}
```

---

## Contracts Layer Rules

```csharp
// ✅ Contracts expose DTOs and interfaces for cross-module communication
// src/Modules/ModuleAModule/ModuleA.Contracts/DTOs/ProductResponse.cs
public sealed record ProductResponse(Guid Id, string Name, decimal Price, string Currency);

// ✅ Interface that other modules can consume
// src/Modules/ModuleAModule/ModuleA.Contracts/IModuleAService.cs
public interface IModuleAService
{
    Task<ProductResponse?> GetProductAsync(Guid productId, CancellationToken ct = default);
}
```

---

## Testing Rules

```csharp
// ✅ Unit test structure — Arrange / Act / Assert, one concept per test
// src/Tests/ModuleA.Test/Application/CreateProductCommandHandlerTests.cs
public sealed class CreateProductCommandHandlerTests
{
    private readonly IProductRepository _repository = Substitute.For<IProductRepository>();
    private readonly IUnitOfWork _unitOfWork = Substitute.For<IUnitOfWork>();
    private readonly CreateProductCommandHandler _sut;

    public CreateProductCommandHandlerTests()
        => _sut = new CreateProductCommandHandler(_repository, _unitOfWork);

    [Fact]
    public async Task Handle_WhenCommandIsValid_ReturnsSuccessWithNewId()
    {
        var command = new CreateProductCommand("Laptop", "High-end laptop", 1500m, "USD", 10);

        var result = await _sut.Handle(command, CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeEmpty();
        await _repository.Received(1).AddAsync(
            Arg.Is<Product>(p => p.Name == "Laptop"),
            Arg.Any<CancellationToken>());
    }

    [Theory]
    [InlineData("")]
    [InlineData(null)]
    [InlineData("   ")]
    public async Task Handle_WhenNameIsEmpty_ReturnsValidationError(string? name)
    {
        var command = new CreateProductCommand(name!, "Desc", 100m, "USD", 5);

        var result = await _sut.Handle(command, CancellationToken.None);

        result.IsFailure.Should().BeTrue();
        result.Error.Type.Should().Be(ErrorType.Validation);
        await _repository.DidNotReceive().AddAsync(
            Arg.Any<Product>(), Arg.Any<CancellationToken>());
        await _unitOfWork.DidNotReceive().SaveChangesAsync(Arg.Any<CancellationToken>());
    }
}
```

---

## Logging Rules

```csharp
// ✅ Structured logging — use message templates, never string interpolation
_logger.LogInformation("Product {ProductId} created by user {UserId}", productId, userId);
_logger.LogWarning("Inventory low for product {ProductId}: {Remaining} units", id, remaining);
_logger.LogError(ex, "Failed to charge payment for order {OrderId}", orderId);

// ❌ Never — string interpolation loses structured data
_logger.LogInformation($"Product {productId} created");

// ❌ Never — logging sensitive data
_logger.LogDebug("Login attempt: user={Email} password={Password}", email, password);
_logger.LogInformation("Token issued: {Token}", fullJwtToken);

// ✅ Partial sensitive data only when absolutely necessary for debugging
_logger.LogDebug("Token issued for {UserId}, prefix: {TokenPrefix}",
    userId, token[..Math.Min(8, token.Length)]);
```

---

## Absolute Prohibitions

The agent and Copilot must never suggest:

- ❌ `.Result` or `.Wait()` on async methods — use `await`
- ❌ `async void` — use `async Task`
- ❌ Business logic in Controllers — belongs in Domain or Application
- ❌ `DbContext` injected directly into Handlers — use `IRepository`
- ❌ `throw` for normal business error flow — use `Result.Failure(error)`
- ❌ `FromSqlRaw` with string concatenation — use `FromSqlInterpolated` or LINQ
- ❌ Hardcoded secrets, connection strings, or API keys anywhere in code
- ❌ `Console.WriteLine` or `Debug.WriteLine` — use `ILogger<T>`
- ❌ Nullable reference type warnings suppressed with `!` without a comment
- ❌ Empty `catch` blocks that swallow exceptions silently
- ❌ Direct references between module internals — use Contracts
