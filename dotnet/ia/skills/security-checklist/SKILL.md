---
name: security-checklist
description: >
  Read when the task involves security: authentication, authorization,
  secret management, rate limiting, vulnerability review, or auditing
  before going to production. Covers OWASP Top 10 applied to this stack.
triggers:
  - security
  - authentication
  - authorization
  - JWT
  - secrets
  - OWASP
  - rate limiting
  - audit
  - production
---

# SKILL: Security Checklist

## Quick checklist before opening a PR to main

```
AUTHENTICATION & AUTHORIZATION
[ ] All new endpoints have [Authorize] or [AllowAnonymous] with a comment
[ ] [AllowAnonymous] only on: login, health checks, signature-verified webhooks
[ ] Authorization policies cover the use case correctly

VALIDATION
[ ] FluentValidator for every Command/Query receiving client data
[ ] Maximum lengths defined for all strings and collections
[ ] External IDs validated (do not assume they exist in DB)

SECRETS
[ ] Zero secrets in source code or appsettings.json
[ ] User Secrets for dev, environment variables for prod

LOGGING
[ ] Logs contain no passwords, full tokens, or PII
[ ] 5xx errors are logged at Error level with enough context to diagnose

DATABASE
[ ] Zero queries using user string concatenation
[ ] AsNoTracking() on read-only queries
```

---

## JWT — Secure configuration

```csharp
// src/Api/MyApi/Program.cs
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,

            ValidIssuer = configuration["Jwt:Issuer"],
            ValidAudience = configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(configuration["Jwt:Secret"]!)),

            ClockSkew = TimeSpan.FromSeconds(30)
        };
        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = ctx =>
            {
                ctx.HttpContext.RequestServices
                    .GetRequiredService<ILogger<Program>>()
                    .LogWarning("JWT authentication failed: {Error}", ctx.Exception.Message);
                return Task.CompletedTask;
            }
        };
    });

// Fallback policy: everything requires authentication by default
builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build());
```

---

## Secrets — per-environment management

```csharp
// DEVELOPMENT — User Secrets
// dotnet user-secrets set "Jwt:Secret" "value" --project src/Api/MyApi

// PRODUCTION — Environment variables (__ is the hierarchical separator)
// JWT__SECRET=value

// PRODUCTION ADVANCED — Azure Key Vault with Managed Identity
if (!builder.Environment.IsDevelopment())
{
    builder.Configuration.AddAzureKeyVault(
        new Uri(configuration["KeyVault:Url"]!),
        new DefaultAzureCredential());
}

// ✅ appsettings.json — structure only, values always empty
{
  "Jwt": { "Issuer": "myapi", "Audience": "myapi-clients", "Secret": "" },
  "ConnectionStrings": { "Default": "" }
}
```

---

## Rate Limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    options.AddFixedWindowLimiter("Global", o =>
    {
        o.PermitLimit = 100;
        o.Window = TimeSpan.FromMinutes(1);
        o.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        o.QueueLimit = 10;
    });

    options.AddFixedWindowLimiter("Auth", o =>
    {
        o.PermitLimit = 10;
        o.Window = TimeSpan.FromMinutes(1);
        o.QueueLimit = 0;
    });
});

[HttpPost("login")]
[AllowAnonymous] // public because it IS the login endpoint
[EnableRateLimiting("Auth")]
public async Task<IActionResult> Login(...) { }
```

---

## Safe Logging

```csharp
// ❌ NEVER DO THIS
_logger.LogInformation("Login with password {Password}", password);
_logger.LogDebug("Token: {Token}", fullJwtToken);
_logger.LogError("Error processing card {CardNumber}", cardNumber);

// ✅ CORRECT
_logger.LogInformation("User {UserId} logged in successfully", userId);
_logger.LogDebug("Token issued for {UserId}, prefix: {Prefix}",
    userId, token[..Math.Min(8, token.Length)]);
_logger.LogError("Payment error for order {OrderId}", orderId);

// NEVER in logs (even masked):
// passwords, PINs, full JWT tokens, credit card numbers,
// identity documents, medical data, complete connection strings
```

---

## SQL Injection — Prevention

```csharp
// ✅ EF Core — parameterized automatically (standard approach)
var products = await _context.Products
    .Where(p => p.Name == searchTerm)
    .ToListAsync(ct);

// ✅ If you need raw SQL — ALWAYS with safe interpolation
var products = await _context.Products
    .FromSqlInterpolated($"SELECT * FROM \"Products\" WHERE \"Name\" = {searchTerm}")
    .ToListAsync(ct);

// ❌ VULNERABLE — direct string concatenation
var products = await _context.Products
    .FromSqlRaw($"SELECT * FROM Products WHERE Name = '{searchTerm}'")
    .ToListAsync(ct);
```

---

## HTTP Security Headers

```csharp
// src/Api/MyApi/Program.cs — add before app.MapControllers()
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Append("X-Frame-Options", "DENY");
    context.Response.Headers.Append("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");
    await next();
});
```

---

## OWASP Top 10 — mitigations in this stack

| # | Risk | Mitigation |
|---|---|---|
| A01 | Broken Access Control | `[Authorize]` by default + fallback policy |
| A02 | Cryptographic Failures | Secrets in Key Vault/env vars; HTTPS enforced |
| A03 | Injection | EF Core parameterized; FluentValidation on all inputs |
| A04 | Insecure Design | Result Pattern; validation in Application layer |
| A05 | Security Misconfiguration | Restrictive CORS; security headers |
| A06 | Vulnerable Components | `dotnet list package --vulnerable` in CI |
| A07 | Auth Failures | Full JWT validation; Rate Limiting on auth endpoints |
| A08 | Software Integrity | Verified CI pipeline |
| A09 | Logging Failures | Serilog with CorrelationId; no PII in logs |
| A10 | SSRF | `AllowedHosts` configured; no arbitrary user URLs accepted |

---

## Full checklist for production PR

```
AUTHENTICATION
[ ] JWT validated with issuer, audience, lifetime, and signature
[ ] Rate limiting on authentication endpoints
[ ] Tests verify 401 for unauthenticated clients
[ ] Tests verify 403 for roles without permission

DATA
[ ] FluentValidator for every external Command/Query
[ ] Maximum lengths on all model strings
[ ] No string concatenation in SQL queries

SECRETS
[ ] dotnet user-secrets list shows correct secrets in dev
[ ] Environment variables documented in deployment manifest
[ ] Key Vault configured and reachable from production environment

DEPENDENCIES
[ ] dotnet list package --vulnerable → no critical vulnerabilities

OBSERVABILITY
[ ] Health checks respond correctly
[ ] Error logs include enough context to diagnose
[ ] No log contains sensitive data
```
