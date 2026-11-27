---
name: vertical-slice
description: Build an HTTP use case as an AOT-safe vertical slice (Minimal APIs, Clean core). Use when creating, changing or reviewing endpoints, handlers, stores or Program.cs in a factory project.
---

# Vertical slice

Code is organised by feature. Each use case is one slice folder that holds its endpoint, its handler and its data access. The slices sit on top of a pure domain project (see `domain-modeling`). It adapts the rules of the `clean-vsa` skill for the factory.

## Project layout

```
src/
  <App>.Domain/                     # BCL only
  <App>.Api/
    Features/<Area>/<SliceName>/
      <SliceName>.cs                # request, response, Map(), Handle()
      <SliceName>.Store.cs          # data access for this slice only
    Infrastructure/                 # cross-cutting only: JSON context, exception handler, DB setup
    Program.cs
tests/
  <App>.Domain.Tests/               # property-based tests of the domain
  <App>.Api.Tests/                  # HTTP tests against a real database in Docker
  <App>.Architecture.Tests/         # dependency rules
db/
  init.sql                          # schema, with constraints that repeat domain rules
```

`<SliceName>` is verb + noun: `ReceiveStock`, `IssueStock`, `GetOrderById`. No `Services`, `Repositories`, `DTOs`, `Mappers` or `Handlers` folders.

## Slice anatomy

A slice is a `public static class` named like its folder.

```csharp
namespace App.Api.Features.Stock.IssueStock;

public sealed record IssueStockRequest(int Quantity);
public sealed record IssueStockResponse(string Sku, int OnHand);

public static class IssueStock
{
    public static void Map(IEndpointRouteBuilder group) =>
        group.MapPost("/{sku}/issues", Handle).WithName(nameof(IssueStock));

    internal static async Task<Results<Ok<IssueStockResponse>, NotFound>> Handle(
        string sku, IssueStockRequest request, IIssueStockStore store, CancellationToken ct)
    {
        var parsed = Sku.Parse(sku);
        var loaded = await store.FindAsync(parsed, ct);
        if (loaded is null) return TypedResults.NotFound();
        loaded.Item.Issue(request.Quantity);          // the domain enforces the rules
        await store.SaveAsync(loaded, ct);
        return TypedResults.Ok(new IssueStockResponse(loaded.Item.Sku.Value, loaded.Item.OnHand));
    }
}
```

- The handler only orchestrates: parse, load, call the domain, save, map. **No business rule in the handler.**
- Map request to domain and domain to response by hand, inside the slice. No mapping libraries.
- Return `TypedResults` with a `Results<...>` union so the API shape is visible in the signature.
- Split a slice into `.Contracts.cs`, `.Endpoint.cs`, `.Handler.cs`, `.Store.cs` only when the main file passes ~150 lines or the store has more than two queries.

## Data access

- Each slice owns `I<SliceName>Store` and `<SliceName>Store` in `<SliceName>.Store.cs`. The interface has only what this slice needs. No `IRepository<T>`.
- PostgreSQL with `NpgsqlDataSource` and Dapper.AOT (`[module: DapperAot]`), explicit SQL.
- Writes that can race use optimistic concurrency: the store keeps a `version` column next to the domain object and updates `WHERE version = @expected`. Zero rows changed means a `ConcurrencyConflictException`, which the API turns into 409.
- Repeat critical domain rules as database constraints (`CHECK (on_hand >= 0)`) as defense in depth.

## Wiring (explicit, no scanning)

Until a FastSharp release builds with zero trimming warnings, do not reference FastSharp in an AOT app. Use its shape by hand: one route group per area (the "module") and one static class per endpoint.

```csharp
var builder = WebApplication.CreateSlimBuilder(args);
builder.Services.ConfigureHttpJsonOptions(o => o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<DomainExceptionHandler>();
builder.Services.AddSingleton<IIssueStockStore, IssueStockStore>();

var app = builder.Build();
app.UseExceptionHandler();
var stock = app.MapGroup("/stock").WithTags("Stock");
IssueStock.Map(stock);
app.Run();
```

- Every request and response type is listed in `AppJsonContext` (`[JsonSerializable(typeof(...))]`).
- Every store and every `Map()` call is registered by hand in `Program.cs`. One line per slice.
- `DomainException` and its subtypes become `ProblemDetails` in one `IExceptionHandler`. Handlers never catch them.

## AOT rules

- `<PublishAot>true</PublishAot>` in the API project, so `dotnet build` runs the trimming and AOT analyzers on every build.
- `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` for the whole solution. Fix IL2xxx and IL3xxx warnings; never suppress them.
- Minimal APIs only. No MVC, no MediatR, no AutoMapper, no assembly scanning, no `dynamic`, no `Activator`.

## Checks that prove it

1. `dotnet build` is clean (warnings are errors).
2. `dotnet publish -r linux-x64` produces a native binary with zero warnings. Record the binary size.
3. HTTP tests in `<App>.Api.Tests` call every slice against a real PostgreSQL started by Docker Compose, including the error answers (400, 404, 409).
4. Architecture tests: the domain references only the BCL, and no slice uses another slice's namespace.

## Never

- Two slices in one file, or a slice calling another slice.
- Business rules in handlers or stores.
- Hand-written registrations hidden behind reflection.
