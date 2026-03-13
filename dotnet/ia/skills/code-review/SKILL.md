---
name: code-review
description: >
  Read when the task is to review a PR, audit existing code, or validate
  that an implementation meets project standards. Defines review criteria,
  blocking red flags, and how to give actionable feedback.
triggers:
  - code review
  - review PR
  - review code
  - pull request
  - audit
  - validate implementation
---

# SKILL: Code Review

## Review order — most critical to least critical

```
1. Security       — hardcoded secrets, broken auth, SQL injection
2. Correctness    — wrong logic, async bugs, unhandled edge cases
3. Architecture   — layer violations, incorrect dependencies, cross-module leaks
4. Tests          — coverage, quality, missing scenarios
5. Performance    — N+1 queries, missing pagination, no AsNoTracking
6. Conventions    — naming, structure, documentation
```

---

## 🚨 Red flags — block the PR without exception

```csharp
// HARDCODED SECRET
"Password=RealSecret123!" in any file
"api_key=sk-..." in any file

// BLOCKING ASYNC — causes deadlocks in production
repository.GetAsync(id).Result
service.ProcessAsync().Wait()

// SQL INJECTION
FromSqlRaw($"SELECT * WHERE Name = '{userInput}'")

// BUSINESS LOGIC IN CONTROLLER
[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateRequest req)
{
    if (req.Price > 1000 && req.Category == "Premium") { ... }
    // ↑ this belongs in Domain
}

// DBCONTEXT DIRECTLY IN HANDLER
public class MyHandler(AppDbContext db)
// ↑ use IRepository instead

// SILENT EXCEPTION SWALLOWING
catch (Exception) { }

// ENDPOINT WITH NO AUTH DECISION (neither [Authorize] nor [AllowAnonymous])
[HttpDelete("{id}")]
public async Task<IActionResult> Delete(...)
// ↑ missing explicit decision

// CROSS-MODULE INTERNAL REFERENCE
// A handler in ModuleBModule references ModuleA.Infrastructure directly
// ↑ use ModuleA.Contracts instead
```

---

## Full review checklist

### Security
```
[ ] No hardcoded secrets in any form
[ ] All endpoints have [Authorize] or [AllowAnonymous] with a justifying comment
[ ] Inputs validated with FluentValidation
[ ] No raw SQL with user string concatenation
[ ] Logs contain no sensitive data
```

### Correctness
```
[ ] Happy path works correctly
[ ] Unhappy paths return Result.Failure with the correct error
[ ] Edge cases covered: null, empty, boundary values
[ ] CancellationToken propagated through all I/O
[ ] No .Result / .Wait() / async void
[ ] IDisposable in using / await using
```

### Architecture
```
[ ] Domain does not reference outer layers
[ ] Application does not reference Infrastructure
[ ] Business logic in Domain or Application, not in Controller
[ ] Repository used instead of DbContext directly
[ ] Commands/Queries are immutable records
[ ] Handlers are sealed
[ ] Cross-module communication only through Contracts
[ ] No direct references between module internals
```

### Tests
```
[ ] Unit test for each new Handler (in src/Tests/{Module}.Test/)
[ ] Integration test for each new endpoint (including 401)
[ ] Test names: Method_Scenario_ExpectedResult
[ ] Arrange / Act / Assert structure
[ ] No Thread.Sleep
[ ] No tests that depend on execution order
```

### Performance
```
[ ] AsNoTracking() on read-only queries
[ ] No Include() nested more than 2 levels
[ ] List queries always paginated
[ ] No N+1 — no queries inside loops
```

---

## Comment types for reviews

```markdown
🚨 BLOCKING: This endpoint has no [Authorize] attribute.
Add [Authorize(Policy = "AdminOnly")] or document with
[AllowAnonymous] plus a comment explaining why it is public.

💡 SUGGESTION: Consider AsNoTracking() here — this is a read-only query.
Not a blocker but improves performance under load.

❓ QUESTION: Why FromSqlRaw instead of LINQ here?
If there is a performance reason, add an inline comment to document it.

🔧 NIT: HandleAsync does not follow the convention (should be Handle).
Does not block but is inconsistent with the rest of the codebase.
```

**Actionable feedback — always include the solution:**
```
❌ Vague: "This code is not correct."
✅ Actionable: "🚨 BLOCKING: This query generates an N+1 problem.
Change to:
  await _context.Orders.Include(o => o.Items)
      .Where(o => ids.Contains(o.Id)).ToListAsync(ct);
This reduces from N queries to 2 total."
```

---

## What every new-feature PR must include

Request these before approving if any are missing:

1. **FluentValidator** for the Command/Query receiving external data
2. **Unit test** for the Handler (happy path + at least 1 unhappy path)
3. **Integration test** for the endpoint (including the 401 scenario)
4. **XML documentation** on the Controller method
5. **ProblemDetails** on all error responses
6. **ProducesResponseType** attributes on the Controller action
