---
name: architecture-patterns
description: >
  Read when you need to implement an architectural pattern used in this project:
  Result Pattern, Value Objects, Options Pattern, Pipeline Behaviors,
  Repository Pattern, Unit of Work, or Saga Pattern.
  Includes complete, ready-to-use implementations.
triggers:
  - result pattern
  - CQRS
  - repository pattern
  - value object
  - options pattern
  - pipeline behavior
  - saga pattern
  - architectural pattern
  - unit of work
---

# SKILL: Architectural Patterns

## Result Pattern

The most critical pattern in the project. Every method that can fail for business
reasons returns `Result<T>` — it never throws exceptions for normal control flow.

```csharp
// src/CommonUtils/Result.cs (or in a shared Domain base if preferred)
public sealed class Result<TValue>
{
    private readonly TValue? _value;
    private Result(TValue value)  { IsSuccess = true; _value = value; Error = Error.None; }
    private Result(Error error)   { IsSuccess = false; Error = error; _value = default; }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error Error { get; }
    public TValue Value => IsSuccess
        ? _value!
        : throw new InvalidOperationException("Cannot read Value from a failed Result.");

    public static Result<TValue> Success(TValue value) => new(value);
    public static Result<TValue> Failure(Error error) => new(error);
    public static implicit operator Result<TValue>(TValue value) => Success(value);
    public static implicit operator Result<TValue>(Error error) => Failure(error);
}

public sealed class Result
{
    private Result(bool isSuccess, Error error) { IsSuccess = isSuccess; Error = error; }
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error Error { get; }
    public static Result Success() => new(true, Error.None);
    public static Result Failure(Error error) => new(false, error);
    public static implicit operator Result(Error error) => Failure(error);
}

public sealed record Error(string Code, string Description, ErrorType Type)
{
    public static readonly Error None = new(string.Empty, string.Empty, ErrorType.None);
    public static Error NotFound(string code, string desc)   => new(code, desc, ErrorType.NotFound);
    public static Error Validation(string code, string desc) => new(code, desc, ErrorType.Validation);
    public static Error Conflict(string code, string desc)   => new(code, desc, ErrorType.Conflict);
    public static Error Failure(string code, string desc)    => new(code, desc, ErrorType.Failure);
}

public enum ErrorType { None, NotFound, Validation, Conflict, Failure }
```

**Full flow Domain → Controller:**
```csharp
// Domain — src/Modules/ModuleAModule/ModuleA.Domain/Entities/Order.cs
public Result Confirm()
{
    if (Status != OrderStatus.Pending)
        return OrderErrors.InvalidStatusTransition(Status, OrderStatus.Confirmed);
    Status = OrderStatus.Confirmed;
    return Result.Success();
}

// Handler — src/Modules/ModuleAModule/ModuleA.Application/Orders/Commands/ConfirmOrder/...
var order = await _orders.GetByIdAsync(command.Id, ct);
if (order is null) return OrderErrors.NotFound(command.Id);
var result = order.Confirm();
if (result.IsFailure) return result.Error;
await _unitOfWork.SaveChangesAsync(ct);
return Result.Success();

// Controller — src/Modules/ModuleAModule/ModuleA.Api/Controllers/OrdersController.cs
return result.IsSuccess ? NoContent() : result.Error.ToProblemResult(this);
```

---

## Value Objects

For domain attributes with their own semantics (money, email, coordinates, etc.).

```csharp
// src/Modules/ModuleAModule/ModuleA.Domain/ValueObjects/Money.cs
public sealed record Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    private Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    public static readonly Money Zero = new(0, "USD");

    public static Result<Money> Create(decimal amount, string currency)
    {
        if (amount < 0)
            return Error.Validation("Money.Negative", "Amount cannot be negative.");
        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
            return Error.Validation("Money.InvalidCurrency", "Invalid ISO 4217 currency code.");
        return new Money(amount, currency);
    }

    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new InvalidOperationException($"Cannot add {a.Currency} and {b.Currency}");
        return new Money(a.Amount + b.Amount, a.Currency);
    }

    public static Money operator *(Money m, int qty) => new(m.Amount * qty, m.Currency);
}
```

---

## Options Pattern

For typed, startup-validatable configuration.

```csharp
// src/Api/MyApi/Options/JwtSettings.cs
public sealed class JwtSettings
{
    public const string SectionName = "Jwt";

    [Required] public string Secret { get; init; } = default!;
    [Required] public string Issuer { get; init; } = default!;
    [Required] public string Audience { get; init; } = default!;
    [Range(5, 1440)] public int ExpirationMinutes { get; init; } = 60;
}

// Registration in Program.cs — fails at startup if configuration is invalid
builder.Services
    .AddOptions<JwtSettings>()
    .BindConfiguration(JwtSettings.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();

// Injection into services
public class JwtTokenService(IOptions<JwtSettings> options)
{
    private readonly JwtSettings _settings = options.Value;
}
```

---

## Pipeline Behaviors (MediatR)

Execute automatically around every Handler. Registration order = execution order.

```csharp
// Execution order: LoggingBehavior → ValidationBehavior → Handler

// src/Modules/ModuleAModule/ModuleA.Application/Common/Behaviors/LoggingBehavior.cs
public sealed class LoggingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;
    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
        => _logger = logger;

    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var name = typeof(TRequest).Name;
        _logger.LogInformation("Executing {Request}", name);
        var sw = Stopwatch.StartNew();
        var response = await next();
        _logger.LogInformation("{Request} completed in {Ms}ms", name, sw.ElapsedMilliseconds);
        return response;
    }
}

// src/Modules/ModuleAModule/ModuleA.Application/Common/Behaviors/ValidationBehavior.cs
public sealed class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!_validators.Any()) return await next();

        var failures = _validators
            .Select(v => v.Validate(new ValidationContext<TRequest>(request)))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count != 0) throw new ValidationException(failures);
        return await next();
    }
}

// Registration in the module's DependencyInjection.cs (order matters)
services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(DependencyInjection).Assembly);
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
});
```

---

## Repository Pattern + Unit of Work

```csharp
// INTERFACE in Domain — no EF Core references allowed here
// src/Modules/ModuleAModule/ModuleA.Domain/Repositories/IProductRepository.cs
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<PagedResult<Product>> GetPagedAsync(int page, int pageSize, CancellationToken ct = default);
    Task AddAsync(Product product, CancellationToken ct = default);
    void Update(Product product);
    void Delete(Product product);
}

// IMPLEMENTATION in Infrastructure
// src/Modules/ModuleAModule/ModuleA.Infrastructure/Repositories/ProductRepository.cs
public sealed class ProductRepository : IProductRepository
{
    private readonly ModuleADbContext _context;
    public ProductRepository(ModuleADbContext context) => _context = context;

    public async Task<Product?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => await _context.Products.AsNoTracking().FirstOrDefaultAsync(p => p.Id == id, ct);

    public async Task<PagedResult<Product>> GetPagedAsync(
        int page, int pageSize, CancellationToken ct = default)
    {
        var query = _context.Products.AsNoTracking();
        var total = await query.CountAsync(ct);
        var items = await query
            .OrderBy(p => p.Name)
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync(ct);
        return new PagedResult<Product>(items, page, pageSize, total);
    }

    public async Task AddAsync(Product product, CancellationToken ct = default)
        => await _context.Products.AddAsync(product, ct);

    public void Update(Product product) => _context.Products.Update(product);
    public void Delete(Product product) => _context.Products.Remove(product);
}

// Unit of Work interface — in Domain or CommonUtils
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct = default);
    Task BeginTransactionAsync(CancellationToken ct = default);
    Task CommitTransactionAsync(CancellationToken ct = default);
    Task RollbackTransactionAsync(CancellationToken ct = default);
}
```

---

## Saga Pattern

Use when an operation has multiple steps that must be compensated on failure.

```csharp
// Example: checkout = create order + charge payment + reduce inventory
// If any step fails, all previous steps are compensated.
// src/Modules/ModuleAModule/ModuleA.Application/Checkout/CheckoutSaga.cs
public sealed class CheckoutSaga
{
    private readonly IOrderRepository _orders;
    private readonly IPaymentService _payments;
    private readonly IInventoryService _inventory;
    private readonly IUnitOfWork _unitOfWork;

    public async Task<Result<Guid>> ExecuteAsync(CheckoutCommand command, CancellationToken ct)
    {
        var orderResult = Order.Create(command.CustomerId, command.Items);
        if (orderResult.IsFailure) return orderResult.Error;
        await _orders.AddAsync(orderResult.Value, ct);

        var paymentResult = await _payments.ChargeAsync(
            command.Payment, orderResult.Value.Total, ct);

        if (paymentResult.IsFailure)
        {
            orderResult.Value.Cancel("Payment declined");
            await _unitOfWork.SaveChangesAsync(ct);
            return paymentResult.Error;
        }

        var inventoryResult = await _inventory.ReserveAsync(command.Items, ct);
        if (inventoryResult.IsFailure)
        {
            await _payments.RefundAsync(paymentResult.Value.PaymentId, ct);
            orderResult.Value.Cancel("Insufficient stock");
            await _unitOfWork.SaveChangesAsync(ct);
            return inventoryResult.Error;
        }

        orderResult.Value.Confirm();
        await _unitOfWork.SaveChangesAsync(ct);
        return orderResult.Value.Id;
    }
}
```
