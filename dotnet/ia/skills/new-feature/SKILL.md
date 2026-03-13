---
name: new-feature
description: >
  Read when the task is to add a new endpoint, use case, or complete
  feature end-to-end. Defines the correct implementation order layer by
  layer, which files to create, and in what order.
triggers:
  - new endpoint
  - new feature
  - new use case
  - add functionality
  - implement CRUD
  - create command
  - create query
---

# SKILL: Adding a New Feature

## Implementation order — always inside-out

```
Domain → Application → Infrastructure → Api → Tests
```

**Never start with the Controller.** The domain defines the contract;
the outer layers adapt to it.

---

## Step 1 — Domain

```csharp
// src/MyApi.Domain/Entities/Order.cs
public sealed class Order : Entity
{
    private Order() { }

    public string CustomerId { get; private set; } = default!;
    public Money Total { get; private set; } = default!;
    public OrderStatus Status { get; private set; }

    // Factory method — returns Result<T>, never throws for business errors
    public static Result<Order> Create(string customerId, IReadOnlyList<OrderItem> items)
    {
        if (string.IsNullOrWhiteSpace(customerId))
            return OrderErrors.InvalidCustomer;

        if (items is null || items.Count == 0)
            return OrderErrors.EmptyOrder;

        return new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Pending,
            Total = items.Aggregate(Money.Zero, (acc, i) => acc + i.Price)
        };
    }

    // Business logic belongs here, not in handlers
    public Result Confirm()
    {
        if (Status != OrderStatus.Pending)
            return OrderErrors.InvalidStatusTransition(Status, OrderStatus.Confirmed);

        Status = OrderStatus.Confirmed;
        return Result.Success();
    }
}
```

```csharp
// src/MyApi.Domain/Errors/OrderErrors.cs
public static class OrderErrors
{
    public static readonly Error InvalidCustomer =
        Error.Validation("Order.InvalidCustomer", "Customer ID cannot be empty.");

    public static readonly Error EmptyOrder =
        Error.Validation("Order.EmptyOrder", "Order must have at least one item.");

    public static Error NotFound(Guid id) =>
        Error.NotFound("Order.NotFound", $"Order '{id}' was not found.");

    public static Error InvalidStatusTransition(OrderStatus from, OrderStatus to) =>
        Error.Conflict("Order.InvalidStatus", $"Cannot transition from '{from}' to '{to}'.");
}
```

```csharp
// src/MyApi.Domain/Repositories/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<PagedResult<Order>> GetPagedAsync(int page, int pageSize, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    void Update(Order order);
}
```

---

## Step 2 — Application

### Command + Handler + Validator

```csharp
// src/MyApi.Application/Orders/Commands/CreateOrder/CreateOrderCommand.cs
public sealed record CreateOrderCommand(
    string CustomerId,
    IReadOnlyList<OrderItemDto> Items
) : IRequest<Result<Guid>>;

public sealed record OrderItemDto(Guid ProductId, int Quantity, decimal Price, string Currency);
```

```csharp
// src/MyApi.Application/Orders/Commands/CreateOrder/CreateOrderCommandHandler.cs
public sealed class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, Result<Guid>>
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _unitOfWork;

    public CreateOrderCommandHandler(IOrderRepository orders, IUnitOfWork unitOfWork)
    {
        _orders = orders;
        _unitOfWork = unitOfWork;
    }

    public async Task<Result<Guid>> Handle(CreateOrderCommand command, CancellationToken ct)
    {
        var items = command.Items
            .Select(i => OrderItem.Create(i.ProductId, i.Quantity, Money.Create(i.Price, i.Currency)))
            .ToList();

        var orderResult = Order.Create(command.CustomerId, items);
        if (orderResult.IsFailure) return orderResult.Error;

        await _orders.AddAsync(orderResult.Value, ct);
        await _unitOfWork.SaveChangesAsync(ct);

        return orderResult.Value.Id;
    }
}
```

```csharp
// src/MyApi.Application/Orders/Commands/CreateOrder/CreateOrderCommandValidator.cs
public sealed class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();

        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("Order must have at least one item.")
            .Must(i => i.Count <= 50).WithMessage("Maximum 50 items per order.");

        RuleForEach(x => x.Items).ChildRules(item =>
        {
            item.RuleFor(i => i.ProductId).NotEmpty();
            item.RuleFor(i => i.Quantity).GreaterThan(0);
            item.RuleFor(i => i.Price).GreaterThan(0);
            item.RuleFor(i => i.Currency).NotEmpty().Length(3);
        });
    }
}
```

### Query + Handler + Response DTO

```csharp
// src/MyApi.Application/Orders/Queries/GetOrderById/GetOrderByIdQuery.cs
public sealed record GetOrderByIdQuery(Guid Id) : IRequest<Result<OrderResponse>>;

// src/MyApi.Application/Orders/Queries/GetOrderById/OrderResponse.cs
public sealed record OrderResponse(
    Guid Id, string CustomerId, decimal Total,
    string Currency, string Status, DateTime CreatedAt);
```

```csharp
// src/MyApi.Application/Orders/Queries/GetOrderById/GetOrderByIdQueryHandler.cs
public sealed class GetOrderByIdQueryHandler : IRequestHandler<GetOrderByIdQuery, Result<OrderResponse>>
{
    private readonly IOrderRepository _orders;
    public GetOrderByIdQueryHandler(IOrderRepository orders) => _orders = orders;

    public async Task<Result<OrderResponse>> Handle(GetOrderByIdQuery query, CancellationToken ct)
    {
        var order = await _orders.GetByIdAsync(query.Id, ct);
        return order is null
            ? OrderErrors.NotFound(query.Id)
            : order.ToResponse(); // extension method co-located with the query
    }
}
```

---

## Step 3 — Infrastructure

```csharp
// src/MyApi.Infrastructure/Repositories/OrderRepository.cs
public sealed class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;
    public OrderRepository(AppDbContext context) => _context = context;

    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => await _context.Orders
            .AsNoTracking()
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct = default)
        => await _context.Orders.AddAsync(order, ct);

    public void Update(Order order) => _context.Orders.Update(order);
}
```

```csharp
// src/MyApi.Infrastructure/Persistence/Configurations/OrderConfiguration.cs
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");
        builder.HasKey(o => o.Id);
        builder.Property(o => o.CustomerId).HasMaxLength(100).IsRequired();
        builder.Property(o => o.Status).HasConversion<string>().HasMaxLength(20);
        builder.OwnsOne(o => o.Total, money =>
        {
            money.Property(m => m.Amount).HasColumnName("TotalAmount").HasPrecision(18, 2);
            money.Property(m => m.Currency).HasColumnName("TotalCurrency").HasMaxLength(3);
        });
    }
}
```

Add to `AppDbContext`: `public DbSet<Order> Orders => Set<Order>();`

Then generate the migration — see `ia/skills/ef-migrations/SKILL.md`.

Register in `src/MyApi.Infrastructure/DependencyInjection.cs`:

```csharp
services.AddScoped<IOrderRepository, OrderRepository>();
```

---

## Step 4 — Api

```csharp
// src/MyApi.Api/Controllers/OrdersController.cs
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/orders")]
[Authorize]
public sealed class OrdersController : ControllerBase
{
    private readonly ISender _sender;
    public OrdersController(ISender sender) => _sender = sender;

    /// <summary>Creates a new order.</summary>
    [HttpPost]
    [ProducesResponseType(typeof(Guid), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status422UnprocessableEntity)]
    public async Task<IActionResult> Create([FromBody] CreateOrderRequest request, CancellationToken ct)
    {
        var result = await _sender.Send(request.ToCommand(), ct);
        return result.IsSuccess
            ? CreatedAtAction(nameof(GetById), new { id = result.Value }, result.Value)
            : result.Error.ToProblemResult(this);
    }

    /// <summary>Gets an order by its ID.</summary>
    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(OrderResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(Guid id, CancellationToken ct)
    {
        var result = await _sender.Send(new GetOrderByIdQuery(id), ct);
        return result.IsSuccess ? Ok(result.Value) : result.Error.ToProblemResult(this);
    }
}
```

---

## Step 5 — Required Tests

See `ia/skills/testing/SKILL.md` for complete examples.

Minimum required per feature:

- Unit test for the Handler — happy path + at least 1 unhappy path
- Integration test for the endpoint — including the `401 Unauthorized` test
