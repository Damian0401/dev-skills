---
name: setup-integration-tests
description: Sets up a complete xUnit v3 integration test project for a .NET minimal API with PostgreSQL. Use this skill whenever the user wants to add integration tests to a .NET API project, create an IntegrationTests project, set up Testcontainers for PostgreSQL lifecycle management, use Respawn for fast database reset between tests, configure WebApplicationFactory for in-process testing, or write end-to-end HTTP tests against a real database. Triggers on phrases like "setup integration tests", "add integration tests", "create integration test project", "add testcontainers", "setup xunit integration tests", "add respawn", "setup WebApplicationFactory", or when the user wants to test a .NET API against a real database rather than mocking it.
---

# Setup Integration Tests

Creates a complete `{Name}.IntegrationTests` xUnit v3 project that boots the real API in-process via `WebApplicationFactory`, spins up a PostgreSQL Testcontainer, applies EF Core migrations once, and resets data between tests with Respawn — all ready to run with `dotnet test`.

---

## Step 1: Orient

From the solution directory, find the solution file and discover the project layout:

```bash
find . -maxdepth 2 \( -name "*.slnx" -o -name "*.sln" \) | head -1
find . -maxdepth 3 -name "*.Api.csproj" -not -path "*/obj/*" | head -1
find . -maxdepth 3 -name "*.IntegrationTests.csproj" -not -path "*/obj/*" | head -1
```

Extract the **base name** from the solution or API project (e.g. `Dummy.Api.csproj` → `Dummy`, `MyApp.slnx` → `MyApp`). All `{Name}` placeholders below use this.

Also check:
- Does `{Name}.IntegrationTests/` already exist? If yes, skip steps that are already done.
- Does `Program.cs` already contain `public partial class Program`?
- Does `Program.cs` already have `ReferenceHandler.IgnoreCycles`?

---

## Step 2: Read the existing project

Before writing anything, read these files to gather the facts you'll need:

1. **`{Name}.Api/{Name}.Api.csproj`** — find the EF Core version (e.g. `Microsoft.EntityFrameworkCore` `10.0.8`). You'll pin the test project to the same version.

2. **`{Name}.Api/Program.cs`** — find the connection string key name (e.g. `"Default"`, `"DefaultConnection"`). It's the string passed to `GetConnectionString(...)`.

3. **`docker-compose.yml`** (if present) — find the postgres image tag (e.g. `postgres:18`). Use it in the Testcontainer.

4. **`{Name}.Data/Models/*.cs`** or equivalent — list the entity names (e.g. `Author`, `Book`). Each entity gets its own test subdirectory.

5. **Feature handler files** — for each entity, skim the handler `.cs` files to confirm which CRUD endpoints exist (GET all, GET by id, POST, DELETE) and what HTTP status codes they return. The tests assert on these exact codes.

This research phase is worth the time — wrong connection string key or wrong status codes create tests that pass incorrectly.

---

## Step 3: Modify `{Name}.Api/Program.cs`

Two additions are needed. Skip each if already present.

**A — Circular reference fix** (prevents JSON serialization failure when entities have navigation properties that form cycles, e.g. `Author.Books[0].Author...`):

Add the using at the top:
```csharp
using System.Text.Json.Serialization;
```

Add the option registration right after `AddDbContext`:
```csharp
builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.ReferenceHandler = ReferenceHandler.IgnoreCycles);
```

**B — WebApplicationFactory hook** (the test project references `Program` by type; top-level-statement classes are `internal` by default):

Add at the very bottom of the file, after `app.Run()`:
```csharp
public partial class Program { }
```

---

## Step 4: Create the project directory structure

```bash
mkdir -p {Name}.IntegrationTests/Infrastructure
mkdir -p {Name}.IntegrationTests/Helpers
mkdir -p {Name}.IntegrationTests/Tests
# one subdirectory per entity:
# mkdir -p {Name}.IntegrationTests/Tests/Authors
# mkdir -p {Name}.IntegrationTests/Tests/Books
```

---

## Step 5: Create `{Name}.IntegrationTests.csproj`

Use the same `TargetFramework` as the API project. Substitute `{EfCoreVersion}` with the version discovered in Step 2 (e.g. `10.0.8`).

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <IsPackable>false</IsPackable>
    <!-- Required so WebApplicationFactory resolves the API's config at runtime -->
    <PreserveCompilationContext>true</PreserveCompilationContext>
    <!-- Suppress xUnit1051: CancellationToken best-practice hint, not a bug -->
    <NoWarn>$(NoWarn);xUnit1051</NoWarn>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="xunit.v3"                         Version="3.2.2" />
    <PackageReference Include="xunit.runner.visualstudio"        Version="3.1.5">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.NET.Test.Sdk"           Version="18.6.0" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="{EfCoreVersion}" />
    <PackageReference Include="Testcontainers.PostgreSql"        Version="4.12.0" />
    <PackageReference Include="Respawn"                          Version="7.0.0" />
    <PackageReference Include="Npgsql"                           Version="10.0.3" />
    <PackageReference Include="Shouldly"                         Version="4.3.0" />
    <!-- Pin EF Core to match the API version and avoid MSB3277 assembly conflicts -->
    <PackageReference Include="Microsoft.EntityFrameworkCore"             Version="{EfCoreVersion}" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Relational"  Version="{EfCoreVersion}" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\{Name}.Api\{Name}.Api.csproj" />
  </ItemGroup>

</Project>
```

> **Why Shouldly?** FluentAssertions 8.x changed to a commercial license. Shouldly is MIT and has nearly identical syntax (`value.ShouldBe(expected)` instead of `value.Should().Be(expected)`).

---

## Step 6: Create `GlobalUsings.cs`

xUnit v3 does **not** auto-inject `global using Xunit;` even with `ImplicitUsings` enabled. Without this file every test class gets CS0246 errors.

```csharp
global using Xunit;
```

---

## Step 7: Create Infrastructure files

### `Infrastructure/IntegrationTestWebAppFactory.cs`

Replaces the registered `AppDbContext` with one pointing at the Testcontainer. `UseEnvironment("Testing")` prevents the developer exception page from changing 4xx/5xx response formats.

```csharp
using {Name}.Data;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

namespace {Name}.IntegrationTests.Infrastructure;

public class IntegrationTestWebAppFactory(string connectionString)
    : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.ConfigureServices(services =>
        {
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor is not null)
                services.Remove(descriptor);

            services.AddDbContext<AppDbContext>(opt =>
                opt.UseNpgsql(connectionString));
        });
    }
}
```

### `Infrastructure/DatabaseFixture.cs`

Owns the container lifecycle. Starts once per test collection, migrates the schema, initialises Respawn. `ResetDatabaseAsync()` is called before each test.

> **xUnit v3 note:** `IAsyncLifetime` now requires `ValueTask`, not `Task`. Using `Task` compiles but throws CS0738 at runtime.

> **Testcontainers note:** `new PostgreSqlBuilder("postgres:18")` — pass the image as a constructor argument. The parameterless constructor is obsolete in Testcontainers 4.x.

```csharp
using {Name}.Data;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Npgsql;
using Respawn;
using Testcontainers.PostgreSql;

namespace {Name}.IntegrationTests.Infrastructure;

public class DatabaseFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container =
        new PostgreSqlBuilder("{POSTGRES_IMAGE}")   // e.g. "postgres:18"
            .WithDatabase("{name}_tests")
            .WithUsername("test")
            .WithPassword("test")
            .Build();

    private Respawner _respawner = null!;
    public IntegrationTestWebAppFactory Factory { get; private set; } = null!;
    public string ConnectionString => _container.GetConnectionString();

    public async ValueTask InitializeAsync()
    {
        await _container.StartAsync();
        Factory = new IntegrationTestWebAppFactory(ConnectionString);

        // Apply migrations once against the fresh container database
        using var scope = Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();

        // Create Respawn checkpoint — it learns the schema from a live connection
        await using var conn = new NpgsqlConnection(ConnectionString);
        await conn.OpenAsync();
        _respawner = await Respawner.CreateAsync(conn, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            SchemasToInclude = ["public"],
            // Keep migration history so we never need to re-migrate
            TablesToIgnore = ["__EFMigrationsHistory"]
        });
    }

    // Called by TestBase before every test — TRUNCATE + RESTART IDENTITY (fast)
    public async Task ResetDatabaseAsync()
    {
        await using var conn = new NpgsqlConnection(ConnectionString);
        await conn.OpenAsync();
        await _respawner.ResetAsync(conn);
    }

    public async ValueTask DisposeAsync()
    {
        await Factory.DisposeAsync();
        await _container.DisposeAsync();
    }
}
```

Substitute `{POSTGRES_IMAGE}` with the image tag from Step 2 (e.g. `postgres:18`), and `{name}` with the lowercase base name.

### `Infrastructure/TestCollectionDefinition.cs`

Declares the collection so all test classes share the single container instance. Without this, each class spins up its own container.

```csharp
namespace {Name}.IntegrationTests.Infrastructure;

[CollectionDefinition(Name)]
public class IntegrationTestCollection : ICollectionFixture<DatabaseFixture>
{
    public const string Name = "Integration Tests";
}
```

### `Infrastructure/TestBase.cs`

Base class for all test classes. Resets the DB before each test, exposes a pre-configured `HttpClient`, and provides `ExecuteDbAsync` helpers for seeding data directly via EF Core (faster and cleaner than going through HTTP in the Arrange phase).

```csharp
using {Name}.Data;
using Microsoft.Extensions.DependencyInjection;

namespace {Name}.IntegrationTests.Infrastructure;

[Collection(IntegrationTestCollection.Name)]
public abstract class TestBase : IAsyncLifetime
{
    private readonly DatabaseFixture _fixture;

    protected HttpClient Client { get; }
    protected IntegrationTestWebAppFactory Factory => _fixture.Factory;

    protected TestBase(DatabaseFixture fixture)
    {
        _fixture = fixture;
        Client = fixture.Factory.CreateClient();
    }

    public async ValueTask InitializeAsync() => await _fixture.ResetDatabaseAsync();

    public ValueTask DisposeAsync()
    {
        Client.Dispose();
        return ValueTask.CompletedTask;
    }

    protected async Task<T> ExecuteDbAsync<T>(Func<AppDbContext, Task<T>> action)
    {
        using var scope = Factory.Services.CreateScope();
        return await action(scope.ServiceProvider.GetRequiredService<AppDbContext>());
    }

    protected async Task ExecuteDbAsync(Func<AppDbContext, Task> action)
    {
        using var scope = Factory.Services.CreateScope();
        await action(scope.ServiceProvider.GetRequiredService<AppDbContext>());
    }
}
```

---

## Step 8: Create `Helpers/SeedData.cs`

One static method per entity. Using EF Core directly in the Arrange phase keeps test setup fast and decoupled from the endpoints under test.

Create a method for each entity discovered in Step 2. Example for `Author` and `Book`:

```csharp
using {Name}.Data;
using {Name}.Data.Models;

namespace {Name}.IntegrationTests.Helpers;

public static class SeedData
{
    public static async Task<Author> CreateAuthorAsync(AppDbContext db,
        string name = "Test Author", string? bio = null)
    {
        var author = new Author { Name = name, Bio = bio };
        db.Authors.Add(author);
        await db.SaveChangesAsync();
        return author;
    }

    public static async Task<Book> CreateBookAsync(AppDbContext db,
        int authorId, string title = "Test Book", int year = 2024)
    {
        var book = new Book { Title = title, PublishedYear = year, AuthorId = authorId };
        db.Books.Add(book);
        await db.SaveChangesAsync();
        return book;
    }
}
```

---

## Step 9: Create test files

For each entity, create four test classes under `Tests/{Entity}/`. Use private `record` types inside each class for JSON deserialization — no need for shared DTOs.

### Pattern for each test class

```csharp
using System.Net;
using System.Net.Http.Json;
using {Name}.IntegrationTests.Helpers;
using {Name}.IntegrationTests.Infrastructure;
using Shouldly;

namespace {Name}.IntegrationTests.Tests.{Entity}s;

public class Get{Entity}sTests(DatabaseFixture fixture) : TestBase(fixture)
{
    [Fact]
    public async Task GetAll_WhenEmpty_ReturnsEmptyList() { ... }

    [Fact]
    public async Task GetAll_WhenRecordsExist_ReturnsAll() { ... }

    private record {Entity}Dto(int Id, string Name /*, other fields */);
}
```

### Coverage checklist per entity

| Test class | Tests to include |
|---|---|
| `Get{Entity}sTests` | empty list returns 200 + `[]`; seeded records all returned |
| `Get{Entity}ByIdTests` | existing id returns 200 with correct data; missing id returns 404 |
| `Create{Entity}Tests` | valid body returns 201 + `Location` header + persisted in DB; invalid FK returns 404 (if applicable) |
| `Delete{Entity}Tests` | existing id returns 204 + removed from DB; missing id returns 404; cascade behaviour (if applicable) |

Verify status codes against the handlers you read in Step 2 — minimal API handlers using `TypedResults` are explicit about what they return.

### Assertion pattern

```csharp
// HTTP status
response.StatusCode.ShouldBe(HttpStatusCode.Created);

// Response body
var dto = await response.Content.ReadFromJsonAsync<EntityDto>();
dto.ShouldNotBeNull();
dto.Name.ShouldBe("expected");

// DB state (use for verifying deletes / persists, NOT for re-asserting what the HTTP response already told you)
var exists = await ExecuteDbAsync(db => db.Authors.AnyAsync(a => a.Id == seeded.Id));
exists.ShouldBeFalse();
```

---

## Step 10: Update the solution file

### For `.slnx` format (XML)

```xml
<Solution>
  <Project Path="{Name}.Api/{Name}.Api.csproj" />
  <Project Path="{Name}.Data/{Name}.Data.csproj" />
  <Project Path="{Name}.IntegrationTests/{Name}.IntegrationTests.csproj" />
</Solution>
```

### For `.sln` format

```bash
dotnet sln add {Name}.IntegrationTests/{Name}.IntegrationTests.csproj
```

---

## Step 11: Verify the build

```bash
dotnet build {Name}.IntegrationTests/{Name}.IntegrationTests.csproj
```

**Common errors and fixes:**

| Error | Fix |
|---|---|
| `CS0246: IAsyncLifetime not found` | Add `GlobalUsings.cs` with `global using Xunit;` |
| `CS0738: does not implement IAsyncLifetime.InitializeAsync()` | Change return type from `Task` → `ValueTask` |
| `CS1061: MigrateAsync not found` | Add `using Microsoft.EntityFrameworkCore;` to `DatabaseFixture.cs` |
| `CS0618: PostgreSqlBuilder() is obsolete` | Use `new PostgreSqlBuilder("postgres:18")` not `new PostgreSqlBuilder().WithImage(...)` |
| `MSB3277: EF Core assembly conflict` | Ensure EF Core version is pinned in the `.csproj` to match the API project |

Build must succeed with **0 errors** before reporting done.

---

## Step 12: Print summary

End with a table of everything done or skipped:

```
✅ Modified   {Name}.Api/Program.cs (IgnoreCycles + partial Program)
✅ Created    {Name}.IntegrationTests/ project
✅ Created    Infrastructure/ (Factory, Fixture, Collection, Base)
✅ Created    Helpers/SeedData.cs
✅ Created    Tests/Authors/ (4 test classes)
✅ Created    Tests/Books/   (4 test classes)
✅ Updated    {SolutionFile} (added test project)
✅ Verified   dotnet build — 0 errors

To run the tests (Docker must be running):
  dotnet test {Name}.IntegrationTests/
```

Use `⏭ Skipped` for anything that was already in place.
