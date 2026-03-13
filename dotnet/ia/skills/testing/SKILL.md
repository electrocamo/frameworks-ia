---
name: testing
description: >
  Read when the task is to write, fix, or review tests.
  Covers unit tests with NSubstitute + FluentAssertions,
  integration tests with Testcontainers + WebApplicationFactory,
  and architecture tests with NetArchTest.
triggers:
  - write tests
  - unit test
  - integration test
  - coverage
  - NSubstitute
  - FluentAssertions
  - Testcontainers
  - WebApplicationFactory
---

# SKILL: Testing Strategy

## Pyramid and minimum coverage

```
         ▲
        /E2E\        5%   Postman/Newman in staging
       /──────\
      / Integr \    20%   Testcontainers + WebApplicationFactory
     /──────────\
    / Unit Tests \  75%   xUnit + NSubstitute + FluentAssertions

Minimum coverage required:
  Domain:         90%
  Application:    80%
  Infrastructure: 60%  (covered primarily by integration tests)
  Api:            40%  (covered primarily by integration tests)
```

---

## Unit Tests

### Base structure for a test class

```csharp
public sealed class CreateProductCommandHandlerTests
{
    // Substitutes as readonly fields — xUnit creates a new instance per test method
    private readonly IProductRepository _repository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly CreateProductCommandHandler _sut; // System Under Test

    public CreateProductCommandHandlerTests()
    {
        _repository = Substitute.For<IProductRepository>();
        _unitOfWork = Substitute.For<IUnitOfWork>();
        _sut = new CreateProductCommandHandler(_repository, _unitOfWork);
    }

    [Fact]
    public async Task Handle_WhenCommandIsValid_ReturnsSuccessWithId()
    {
        // Arrange
        var command = new CreateProductCommand("Laptop", "High-end laptop", 1500m, "USD", 10);

        // Act
        var result = await _sut.Handle(command, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeEmpty();
        await _repository.Received(1)
            .AddAsync(Arg.Is<Product>(p => p.Name == "Laptop"), Arg.Any<CancellationToken>());
        await _unitOfWork.Received(1).SaveChangesAsync(Arg.Any<CancellationToken>());
    }

    [Theory]
    [InlineData("")]
    [InlineData(" ")]
    [InlineData(null)]
    public async Task Handle_WhenNameIsInvalid_ReturnsValidationError(string? name)
    {
        // Arrange
        var command = new CreateProductCommand(name!, "Desc", 100m, "USD", 5);

        // Act
        var result = await _sut.Handle(command, CancellationToken.None);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Type.Should().Be(ErrorType.Validation);
        await _repository.DidNotReceive().AddAsync(Arg.Any<Product>(), Arg.Any<CancellationToken>());
        await _unitOfWork.DidNotReceive().SaveChangesAsync(Arg.Any<CancellationToken>());
    }
}
```

### NSubstitute — essential patterns

```csharp
// Configure return value
_repository.GetByIdAsync(productId, Arg.Any<CancellationToken>())
    .Returns(product);

// Return null (resource not found)
_repository.GetByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
    .Returns((Product?)null);

// Verify called exactly once
await _repository.Received(1).AddAsync(Arg.Any<Product>(), Arg.Any<CancellationToken>());

// Verify NOT called
await _repository.DidNotReceive().AddAsync(Arg.Any<Product>(), Arg.Any<CancellationToken>());

// Verify with argument condition
await _repository.Received(1)
    .AddAsync(Arg.Is<Product>(p => p.Name == "Laptop" && p.Status == ProductStatus.Active),
              Arg.Any<CancellationToken>());

// Simulate exception
_repository.GetByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
    .Throws(new TimeoutException());

// Mock TimeProvider (.NET 8) — never use DateTime.Now or Thread.Sleep in tests
var timeProvider = Substitute.For<TimeProvider>();
timeProvider.GetUtcNow().Returns(new DateTimeOffset(2025, 1, 15, 10, 0, 0, TimeSpan.Zero));
```

### FluentAssertions — essential patterns

```csharp
// Result Pattern
result.IsSuccess.Should().BeTrue();
result.IsFailure.Should().BeTrue();
result.Error.Code.Should().Be("Product.NotFound");
result.Error.Type.Should().Be(ErrorType.NotFound);

// Values
result.Value.Should().NotBeEmpty();
result.Value.Should().Be(expectedValue);

// Strings
product.Name.Should().Be("Laptop");
product.Name.Should().NotBeNullOrWhiteSpace();

// Collections
products.Should().HaveCount(3);
products.Should().ContainSingle(p => p.Name == "Laptop");
products.Should().BeInAscendingOrder(p => p.Name);
products.Should().NotContainNulls();

// Types
result.Value.Should().BeOfType<ProductResponse>();
```

---

## Integration Tests

### Setup with Testcontainers + WebApplicationFactory

```csharp
// tests/MyApi.IntegrationTests/Common/IntegrationTestBase.cs
public sealed class IntegrationTestBase : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .WithDatabase("myapi_test")
        .WithUsername("test")
        .WithPassword("test")
        .Build();

    public async Task InitializeAsync()
    {
        await _db.StartAsync();
        using var scope = Services.CreateScope();
        await scope.ServiceProvider
            .GetRequiredService<AppDbContext>()
            .Database.MigrateAsync();
    }

    public new async Task DisposeAsync()
    {
        await _db.StopAsync();
        await base.DisposeAsync();
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(o =>
                o.UseNpgsql(_db.GetConnectionString()));
        });
    }

    public HttpClient CreateAuthenticatedClient(string role = "User")
    {
        var client = CreateClient();
        client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", JwtTestHelper.GenerateToken(role));
        return client;
    }

    public async Task ResetDatabaseAsync()
    {
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.ExecuteSqlRawAsync(
            "TRUNCATE TABLE \"Products\", \"Orders\" RESTART IDENTITY CASCADE");
    }
}
```

### Integration test — standard structure

```csharp
// IClassFixture: Docker container starts ONCE for the entire class
// IAsyncLifetime: cleans data after each test method
[Trait("Category", "Products")]
public sealed class ProductsApiTests : IClassFixture<IntegrationTestBase>, IAsyncLifetime
{
    private readonly IntegrationTestBase _fixture;
    private readonly HttpClient _client;

    public ProductsApiTests(IntegrationTestBase fixture)
    {
        _fixture = fixture;
        _client = fixture.CreateAuthenticatedClient();
    }

    public Task InitializeAsync() => Task.CompletedTask;
    public Task DisposeAsync() => _fixture.ResetDatabaseAsync();

    [Fact]
    public async Task POST_Products_WhenValid_Returns201WithLocation()
    {
        // Arrange
        var request = new { Name = "Test Product", Description = "Desc",
                            Price = 99.99, Currency = "USD", StockQuantity = 5 };
        // Act
        var response = await _client.PostAsJsonAsync("/api/v1/products", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
    }

    [Fact]
    public async Task GET_Products_WhenNotAuthenticated_Returns401()
    {
        var response = await _fixture.CreateClient().GetAsync("/api/v1/products");
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task GET_Products_ById_WhenNotExists_Returns404WithProblemDetails()
    {
        var response = await _client.GetAsync($"/api/v1/products/{Guid.NewGuid()}");

        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
        var problem = await response.Content.ReadFromJsonAsync<ProblemDetails>();
        problem!.Status.Should().Be(404);
    }
}
```

---

## Architecture Tests

```csharp
// tests/MyApi.ArchTests/ArchitectureTests.cs
public sealed class ArchitectureTests
{
    private static readonly Assembly Domain = typeof(Product).Assembly;
    private static readonly Assembly Application = typeof(CreateProductCommand).Assembly;
    private static readonly Assembly Infrastructure = typeof(AppDbContext).Assembly;
    private static readonly Assembly Api = typeof(ProductsController).Assembly;

    [Fact]
    public void Domain_ShouldNotHaveDependency_OnApplication()
        => Types.InAssembly(Domain).ShouldNot()
            .HaveDependencyOn("MyApi.Application").GetResult()
            .IsSuccessful.Should().BeTrue();

    [Fact]
    public void Domain_ShouldNotHaveDependency_OnInfrastructure()
        => Types.InAssembly(Domain).ShouldNot()
            .HaveDependencyOn("MyApi.Infrastructure").GetResult()
            .IsSuccessful.Should().BeTrue();

    [Fact]
    public void Application_ShouldNotHaveDependency_OnInfrastructure()
        => Types.InAssembly(Application).ShouldNot()
            .HaveDependencyOn("MyApi.Infrastructure").GetResult()
            .IsSuccessful.Should().BeTrue();

    [Fact]
    public void Handlers_ShouldBeSealed()
        => Types.InAssembly(Application).That()
            .HaveNameEndingWith("Handler").Should().BeSealed()
            .GetResult().IsSuccessful.Should().BeTrue();

    [Fact]
    public void Controllers_ShouldNotReferenceInfrastructure()
        => Types.InAssembly(Api).That()
            .Inherit(typeof(ControllerBase))
            .ShouldNot().HaveDependencyOn("MyApi.Infrastructure")
            .GetResult().IsSuccessful.Should().BeTrue();
}
```

---

## What to test and what not to

| ✅ Always test | ❌ Never test |
|---|---|
| Domain logic (entities, Value Objects) | Getters/setters with no logic |
| Entity factory methods | Auto-generated code (EF migrations) |
| Handlers — happy path AND unhappy paths | Obvious DI registrations |
| Validators — critical business rules | Constants and simple enums |
| Pipeline behaviors | ASP.NET framework code |
| Repositories (in integration tests) | DTOs with no transformation |

---

## Coverage commands

```bash
dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage

# Requires: dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator \
  -reports:"coverage/**/coverage.cobertura.xml" \
  -targetdir:"coverage/report" \
  -reporttypes:"Html;TextSummary"

cat coverage/report/Summary.txt
```
