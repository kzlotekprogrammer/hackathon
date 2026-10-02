# Szkielet startowy

Kod budowany **raz**, na początku hackathonu, zanim powstanie pierwsza funkcja. Po nim każdy wycinek dokłada tylko pliki z [`02-code-patterns.md`](02-code-patterns.md).

Kolejność: projekty → konfiguracja buildu → `Domain` (wspólne) → `Persistence` → `Infrastructure` → konto i sesja → `Program.cs` → testy szkieletu → pierwsza migracja. Szkielet jest gotowy, gdy `dotnet build` i `dotnet test` są zielone, a rejestracja, logowanie i wylogowanie działają w Scalar.

## Projekty

```
backend/
├── .editorconfig                 ← plik gotowy w tym zestawie
├── Directory.Build.props         ← plik gotowy w tym zestawie
├── docker-compose.yml            ← plik gotowy w tym zestawie
├── dotnet-tools.json             ← plik gotowy w tym zestawie (dotnet-ef)
├── Hackathon.slnx
├── src/Hackathon.Api/Hackathon.Api.csproj
└── tests/Hackathon.Tests/Hackathon.Tests.csproj
```

```bash
cd backend
dotnet new sln -n Hackathon
dotnet new web -n Hackathon.Api -o src/Hackathon.Api
dotnet new classlib -n Hackathon.Tests -o tests/Hackathon.Tests
dotnet sln add src/Hackathon.Api tests/Hackathon.Tests
dotnet tool restore
```

Zawartość `.csproj` nadpisujesz poniższą — nie polegasz na tym, co dał szablon. Pliki przykładowe szablonu (`WeatherForecast`, `Class1`) usuwasz.

`src/Hackathon.Api/Hackathon.Api.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="10.0.12" />
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="10.0.11" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.11">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Relational" Version="10.0.11" />
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.0.3" />
    <PackageReference Include="Scalar.AspNetCore" Version="2.17.3" />
  </ItemGroup>

</Project>
```

`tests/Hackathon.Tests/Hackathon.Tests.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="coverlet.collector" Version="6.0.4" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="10.0.11" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.14.1" />
    <PackageReference Include="NetArchTest.Rules" Version="1.3.2" />
    <PackageReference Include="Testcontainers.PostgreSql" Version="4.15.0" />
    <PackageReference Include="xunit" Version="2.9.3" />
    <PackageReference Include="xunit.runner.visualstudio" Version="3.1.4" />
  </ItemGroup>

  <ItemGroup>
    <Using Include="Xunit" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\Hackathon.Api\Hackathon.Api.csproj" />
  </ItemGroup>

</Project>
```

`appsettings.json` — connection string zgodny z `docker-compose.yml`:

```json
{
  "ConnectionStrings": {
    "Hackathon": "Host=localhost;Port=5432;Database=hackathon;Username=hackathon;Password=hackathon"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

`GlobalUsings.cs` — w API i w testach:

```csharp
global using Hackathon.Api.Domain;
global using Hackathon.Api.Infrastructure;
global using Hackathon.Api.Persistence;
global using Hackathon.Api.Queries;
```

`Queries/` jest pusty, dopóki nie powstanie pierwsze zapytanie — `global using` pustego namespace'u nie kompiluje się, więc tę linię dopisujesz razem z pierwszym `…Query`.

## Domain — wspólne

`Domain/DomainException.cs`

```csharp
namespace Hackathon.Api.Domain;

/// <summary>
/// Thrown when a request would leave the model in a state the product does not allow.
/// </summary>
public class DomainException : Exception
{
    public DomainException(string message, int statusCode = StatusCodes.Status400BadRequest)
        : base(message)
    {
        StatusCode = statusCode;
    }

    public int StatusCode { get; }
}
```

`Domain/NamedEnum.cs`

```csharp
using System.Globalization;

namespace Hackathon.Api.Domain;

/// <summary>
/// Parses an enum from its name in the code — <c>ACTIVE</c> — never from an ordinal.
/// <see cref="Enum.TryParse{TEnum}(string, bool, out TEnum)"/> accepts <c>"1"</c> as a defined
/// member; that is not a value the API should take.
/// </summary>
public static class NamedEnum
{
    public static bool TryParse<TEnum>(string? value, out TEnum parsed)
        where TEnum : struct, Enum
    {
        parsed = default;

        if (string.IsNullOrWhiteSpace(value))
        {
            return false;
        }

        string trimmed = value.Trim();
        if (long.TryParse(trimmed, NumberStyles.Integer, CultureInfo.InvariantCulture, out _))
        {
            return false;
        }

        return Enum.TryParse(trimmed, ignoreCase: true, out parsed) && Enum.IsDefined(parsed);
    }
}
```

`Domain/Workspace/IWorkspaceOwned.cs`

```csharp
namespace Hackathon.Api.Domain;

/// <summary>
/// Marks an entity that belongs to a single <see cref="Workspace"/>. Every entity implementing it
/// gets the workspace isolation filter automatically.
/// </summary>
public interface IWorkspaceOwned
{
    Guid WorkspaceId { get; }
}
```

`Domain/Workspace/Workspace.cs`

```csharp
namespace Hackathon.Api.Domain;

/// <summary>
/// Technical container and isolation boundary for one user's data. Invisible in the interface.
/// </summary>
public sealed class Workspace
{
    private Workspace()
    {
    }

    public Guid Id { get; private set; }

    public Guid OwnerUserId { get; private set; }

    public DateTimeOffset CreatedAt { get; private set; }

    public static Workspace CreateFor(Guid ownerUserId, DateTimeOffset createdAt)
    {
        if (ownerUserId == Guid.Empty)
        {
            throw new InvalidOperationException("The owner user id of a new workspace is empty.");
        }

        return new Workspace
        {
            Id = Guid.CreateVersion7(),
            OwnerUserId = ownerUserId,
            CreatedAt = createdAt,
        };
    }
}
```

`Domain/Identity/User.cs`

```csharp
using System.Net.Mail;

namespace Hackathon.Api.Domain;

/// <summary>An account. Lives outside any workspace: sign-in finds it before a workspace is known.</summary>
public sealed class User
{
    public const int EmailMaxLength = 254;

    private User()
    {
    }

    public Guid Id { get; private set; }

    /// <summary>Trimmed and lower-cased, so one address cannot register twice in different case.</summary>
    public string Email { get; private set; } = null!;

    public string PasswordHash { get; private set; } = null!;

    public DateTimeOffset CreatedAt { get; private set; }

    public DateTimeOffset UpdatedAt { get; private set; }

    /// <summary>The address as stored, or null when it is not an e-mail address at all.</summary>
    public static string? NormalizeEmail(string? email)
    {
        if (string.IsNullOrWhiteSpace(email))
        {
            return null;
        }

        string normalized = email.Trim().ToLowerInvariant();
        if (normalized.Length > EmailMaxLength || !MailAddress.TryCreate(normalized, out _))
        {
            return null;
        }

        return normalized;
    }

    public static User Register(string? email, DateTimeOffset createdAt)
    {
        string normalized = NormalizeEmail(email)
            ?? throw new DomainException("A valid e-mail address is required.");

        return new User
        {
            Id = Guid.CreateVersion7(),
            Email = normalized,
            CreatedAt = createdAt,
            UpdatedAt = createdAt,
        };
    }

    public void AssignPasswordHash(string passwordHash, DateTimeOffset changedAt)
    {
        if (string.IsNullOrEmpty(passwordHash))
        {
            throw new InvalidOperationException("The password hash is empty.");
        }

        PasswordHash = passwordHash;
        UpdatedAt = changedAt;
    }
}
```

`Domain/Identity/PasswordPolicy.cs`

```csharp
namespace Hackathon.Api.Domain;

public static class PasswordPolicy
{
    public const int MinLength = 8;

    public static void EnsureAcceptable(string? password)
    {
        if (string.IsNullOrEmpty(password) || password.Length < MinLength)
        {
            throw new DomainException($"A password has at least {MinLength} characters.");
        }
    }
}
```

## Persistence

`Persistence/Workspace/IWorkspaceContext.cs`

```csharp
namespace Hackathon.Api.Persistence;

/// <summary>The workspace the current request may see.</summary>
public interface IWorkspaceContext
{
    /// <summary>
    /// Workspace of the authenticated user, or <c>null</c> when no workspace is in scope. A request
    /// without a workspace matches no isolated rows at all.
    /// </summary>
    Guid? CurrentWorkspaceId { get; }
}
```

`Persistence/HackathonDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Storage;

namespace Hackathon.Api.Persistence;

/// <summary>The one context of the application, with every table of the model.</summary>
public sealed class HackathonDbContext : DbContext, IWorkspaceContext
{
    private readonly IWorkspaceContext workspaceContext;

    private Guid? lockedWorkspaceId;

    private Guid? lockingTransactionId;

    public HackathonDbContext(DbContextOptions<HackathonDbContext> options, IWorkspaceContext workspaceContext)
        : base(options)
    {
        this.workspaceContext = workspaceContext;
    }

    public DbSet<User> Users => Set<User>();

    public DbSet<Workspace> Workspaces => Set<Workspace>();

    public Guid? CurrentWorkspaceId => workspaceContext.CurrentWorkspaceId;

    /// <summary>
    /// Takes the write lock of a workspace for the rest of the current database transaction. A second writer
    /// of the same workspace waits until this transaction commits or rolls back; other workspaces do not wait.
    /// </summary>
    public async Task LockWorkspaceAsync(Guid workspaceId, CancellationToken cancellationToken)
    {
        IDbContextTransaction transaction = Database.CurrentTransaction
            ?? throw new InvalidOperationException("The workspace lock lasts as long as a database transaction; begin one first.");

        string workspaceKey = workspaceId.ToString();
        await Database.ExecuteSqlAsync(
            $"SELECT pg_advisory_xact_lock(hashtextextended({workspaceKey}, 0))",
            cancellationToken);

        lockedWorkspaceId = workspaceId;
        lockingTransactionId = transaction.TransactionId;
    }

    /// <summary>Whether the current database transaction holds the write lock of the workspace.</summary>
    public bool HoldsWorkspaceLock(Guid workspaceId)
    {
        return lockedWorkspaceId == workspaceId
            && lockingTransactionId is Guid transactionId
            && Database.CurrentTransaction?.TransactionId == transactionId;
    }

    protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
    {
        ModelConventions.ApplyTo(configurationBuilder);
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(HackathonDbContext).Assembly);
        ModelConventions.ApplyTo(modelBuilder);
        modelBuilder.ApplyWorkspaceIsolation(this);
    }
}
```

`Persistence/ModelConventions.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata;
using Microsoft.EntityFrameworkCore.Storage.ValueConversion;

namespace Hackathon.Api.Persistence;

/// <summary>
/// Numeric precision, temporal mapping, keys and enum storage, declared once for the whole model, so a new
/// entity cannot be added without them.
/// </summary>
public static class ModelConventions
{
    public const int Precision = 19;

    /// <summary>A bare decimal is a monetary amount.</summary>
    public const int AmountScale = 4;

    public static void ApplyTo(ModelConfigurationBuilder configurationBuilder)
    {
        ArgumentNullException.ThrowIfNull(configurationBuilder);

        configurationBuilder.Properties<decimal>()
            .HavePrecision(Precision, AmountScale);

        // A value with another scale gets its own type and its own line here, e.g.:
        // configurationBuilder.Properties<Rate>().HaveConversion<RateConverter>().HavePrecision(Precision, Rate.Scale);

        configurationBuilder.Properties<DateOnly>()
            .HaveColumnType("date");

        configurationBuilder.Properties<DateTimeOffset>()
            .HaveColumnType("timestamptz");
    }

    /// <summary>
    /// Keys come from <see cref="Guid.CreateVersion7()"/> in the domain, so neither EF Core nor the database
    /// may generate one. Enums are stored under the name they carry in the code, never as an ordinal.
    /// </summary>
    public static void ApplyTo(ModelBuilder modelBuilder)
    {
        ArgumentNullException.ThrowIfNull(modelBuilder);

        foreach (IMutableEntityType entityType in modelBuilder.Model.GetEntityTypes())
        {
            foreach (IMutableProperty keyProperty in entityType.GetDeclaredKeys().SelectMany(key => key.Properties))
            {
                if (keyProperty.ClrType == typeof(Guid))
                {
                    keyProperty.ValueGenerated = ValueGenerated.Never;
                }
            }

            foreach (IMutableProperty property in entityType.GetDeclaredProperties())
            {
                Type? enumType = EnumTypeOrNull(property.ClrType);
                if (enumType is not null)
                {
                    ValueConverter enumToText = EnumToText(enumType);
                    property.SetValueConverter(enumToText);
                }
            }
        }
    }

    private static Type? EnumTypeOrNull(Type clrType)
    {
        Type underlying = Nullable.GetUnderlyingType(clrType) ?? clrType;

        return underlying.IsEnum ? underlying : null;
    }

    private static ValueConverter EnumToText(Type enumType)
    {
        Type converterType = typeof(EnumToStringConverter<>).MakeGenericType(enumType);
        return (ValueConverter)Activator.CreateInstance(converterType)!;
    }
}
```

`Persistence/Workspace/WorkspaceIsolation.cs`

```csharp
using System.Linq.Expressions;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata;

namespace Hackathon.Api.Persistence;

/// <summary>
/// Declares data isolation once, for the whole model: every entity implementing <see cref="IWorkspaceOwned"/>
/// gets a global filter limiting it to the current workspace.
/// </summary>
public static class WorkspaceIsolation
{
    public static void ApplyWorkspaceIsolation<TContext>(this ModelBuilder modelBuilder, TContext context)
        where TContext : DbContext, IWorkspaceContext
    {
        ArgumentNullException.ThrowIfNull(modelBuilder);
        ArgumentNullException.ThrowIfNull(context);

        foreach (IMutableEntityType entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (!typeof(IWorkspaceOwned).IsAssignableFrom(entityType.ClrType))
            {
                continue;
            }

            LambdaExpression filter = BuildFilter(entityType.ClrType, context);
            modelBuilder.Entity(entityType.ClrType).HasQueryFilter(filter);
        }
    }

    private static LambdaExpression BuildFilter<TContext>(Type entityClrType, TContext context)
        where TContext : DbContext, IWorkspaceContext
    {
        ParameterExpression entity = Expression.Parameter(entityClrType, "entity");
        MemberExpression workspaceId = Expression.Property(entity, nameof(IWorkspaceOwned.WorkspaceId));

        // Rooted at the DbContext instance on purpose. EF Core caches the model once per context type and
        // rewrites a DbContext-typed node to the context executing the query, so the workspace is read per
        // query. Rooting it at the injected IWorkspaceContext would freeze the first request's workspace.
        ConstantExpression contextConstant = Expression.Constant(context, typeof(TContext));
        MemberExpression currentWorkspaceId = Expression.Property(contextConstant, nameof(IWorkspaceContext.CurrentWorkspaceId));

        UnaryExpression nullableWorkspaceId = Expression.Convert(workspaceId, typeof(Guid?));
        BinaryExpression body = Expression.Equal(nullableWorkspaceId, currentWorkspaceId);

        return Expression.Lambda(body, entity);
    }
}
```

`Persistence/Workspace/WorkspaceWriteGuard.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.ChangeTracking;
using Microsoft.EntityFrameworkCore.Diagnostics;

namespace Hackathon.Api.Persistence;

/// <summary>
/// Refuses to save a change of a workspace's data unless the current database transaction holds the write lock
/// of that same workspace. A handler that forgot the lock fails in its first test instead of racing in production.
/// </summary>
public sealed class WorkspaceWriteGuard : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(DbContextEventData eventData, InterceptionResult<int> result)
    {
        ArgumentNullException.ThrowIfNull(eventData);

        EnsureLocked(eventData.Context);

        return base.SavingChanges(eventData, result);
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(eventData);

        EnsureLocked(eventData.Context);

        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }

    private static void EnsureLocked(DbContext? context)
    {
        if (context is not HackathonDbContext db)
        {
            return;
        }

        foreach (EntityEntry<IWorkspaceOwned> entry in db.ChangeTracker.Entries<IWorkspaceOwned>())
        {
            bool written = entry.State is EntityState.Added or EntityState.Modified or EntityState.Deleted;
            if (written && !db.HoldsWorkspaceLock(entry.Entity.WorkspaceId))
            {
                throw new InvalidOperationException(
                    $"{entry.Entity.GetType().Name} of workspace {entry.Entity.WorkspaceId} is saved without the lock of "
                    + "that workspace. Begin a database transaction and call LockWorkspaceAsync before the first read.");
            }
        }
    }
}
```

`Persistence/PostgresErrors.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Npgsql;

namespace Hackathon.Api.Persistence;

public static class PostgresErrors
{
    public const string UniqueViolation = "23505";

    public static bool IsUniqueViolation(DbUpdateException exception)
    {
        return exception.InnerException is PostgresException postgres
            && postgres.SqlState == UniqueViolation;
    }
}
```

`Persistence/Identity/UserConfiguration.cs` i `Persistence/Workspace/WorkspaceConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace Hackathon.Api.Persistence;

public sealed class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.HasKey(user => user.Id);
        builder.Property(user => user.Email).HasMaxLength(User.EmailMaxLength);
        builder.HasIndex(user => user.Email).IsUnique();
    }
}
```

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace Hackathon.Api.Persistence;

public sealed class WorkspaceConfiguration : IEntityTypeConfiguration<Workspace>
{
    public void Configure(EntityTypeBuilder<Workspace> builder)
    {
        builder.HasKey(workspace => workspace.Id);
        builder.HasIndex(workspace => workspace.OwnerUserId).IsUnique();
        builder.HasOne<User>()
            .WithOne()
            .HasForeignKey<Workspace>(workspace => workspace.OwnerUserId)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

**Dane referencyjne** (słowniki wspólne dla wszystkich workspace'ów) idą ścieżką: plik JSON jako `EmbeddedResource` → `HasData` w `…Configuration` → migracja → tabela. Aplikacja czyta słownik z tabeli, nigdy z pliku. Encja słownika trafia na listę współdzielonych w teście izolacji.

## Infrastructure

`Infrastructure/Time/IClock.cs` i `SystemClock.cs`

```csharp
namespace Hackathon.Api.Infrastructure;

public interface IClock
{
    DateTimeOffset UtcNow { get; }
}
```

```csharp
namespace Hackathon.Api.Infrastructure;

public sealed class SystemClock : IClock
{
    public DateTimeOffset UtcNow => DateTimeOffset.UtcNow;
}
```

`Infrastructure/Identity/AuthClaims.cs`

```csharp
namespace Hackathon.Api.Infrastructure;

public static class AuthClaims
{
    public const string WorkspaceId = "workspace_id";
}
```

`Infrastructure/Identity/CurrentUser.cs`

```csharp
using System.Security.Claims;

namespace Hackathon.Api.Infrastructure;

/// <summary>
/// The signed-in user of the current request and the workspace they work in. A handler takes them from here
/// rather than as parameters: they say who asks, not what is asked.
/// </summary>
public sealed class CurrentUser(IHttpContextAccessor httpContextAccessor)
{
    public Guid UserId => ClaimedId(ClaimTypes.NameIdentifier);

    public Guid WorkspaceId => ClaimedId(AuthClaims.WorkspaceId);

    private Guid ClaimedId(string claimType)
    {
        ClaimsPrincipal principal = httpContextAccessor.HttpContext?.User
            ?? throw new DomainException("The request is not signed in.", StatusCodes.Status401Unauthorized);

        string? value = principal.FindFirstValue(claimType);
        if (!Guid.TryParse(value, out Guid id))
        {
            throw new DomainException("The request is not signed in.", StatusCodes.Status401Unauthorized);
        }

        return id;
    }
}
```

`Infrastructure/Identity/SignInCookie.cs`

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;

namespace Hackathon.Api.Infrastructure;

public static class SignInCookie
{
    public static async Task IssueAsync(HttpContext http, User user, Guid workspaceId)
    {
        ArgumentNullException.ThrowIfNull(http);
        ArgumentNullException.ThrowIfNull(user);

        string userIdText = user.Id.ToString();
        string workspaceIdText = workspaceId.ToString();
        var identity = new ClaimsIdentity(
            [
                new Claim(ClaimTypes.NameIdentifier, userIdText),
                new Claim(ClaimTypes.Email, user.Email),
                new Claim(AuthClaims.WorkspaceId, workspaceIdText),
            ],
            CookieAuthenticationDefaults.AuthenticationScheme);

        await http.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, new ClaimsPrincipal(identity));
    }
}
```

`Infrastructure/Workspace/WorkspaceContext.cs`

```csharp
namespace Hackathon.Api.Infrastructure;

/// <summary>Per-request workspace scope. Stays empty until sign-in sets it.</summary>
public sealed class WorkspaceContext : IWorkspaceContext
{
    public Guid? CurrentWorkspaceId { get; private set; }

    public void SetCurrentWorkspace(Guid workspaceId)
    {
        if (workspaceId == Guid.Empty)
        {
            throw new InvalidOperationException("The workspace id of the request scope is empty.");
        }

        CurrentWorkspaceId = workspaceId;
    }
}
```

`Infrastructure/Http/WorkspaceScopeMiddleware.cs`

```csharp
using System.Security.Claims;

namespace Hackathon.Api.Infrastructure;

/// <summary>
/// Copies the workspace id from the cookie onto the request-scoped <see cref="WorkspaceContext"/> so the
/// isolation filter can see it.
/// </summary>
public sealed class WorkspaceScopeMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext http, WorkspaceContext workspaceContext)
    {
        if (http.User.Identity?.IsAuthenticated == true)
        {
            string? value = http.User.FindFirstValue(AuthClaims.WorkspaceId);
            if (Guid.TryParse(value, out Guid workspaceId))
            {
                workspaceContext.SetCurrentWorkspace(workspaceId);
            }
        }

        await next(http);
    }
}
```

`Infrastructure/Http/DomainExceptionFilter.cs`

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

namespace Hackathon.Api.Infrastructure;

/// <summary>Turns a refusal of the domain into a response; anything else stays a 500.</summary>
public sealed class DomainExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        if (context.Exception is not DomainException refusal)
        {
            return;
        }

        context.Result = new ObjectResult(new { error = refusal.Message })
        {
            StatusCode = refusal.StatusCode,
        };
        context.ExceptionHandled = true;
    }
}
```

## Konto i sesja

`Features/Register/` — `RegisterRequest`, `RegisterResponse`, `RegisterResult`, `RegisterHandler`, `RegisterController`.

```csharp
namespace Hackathon.Api.Features.Register;

public sealed record RegisterRequest(string? Email, string? Password);
```

```csharp
namespace Hackathon.Api.Features.Register;

public sealed record RegisterResponse(Guid UserId, Guid WorkspaceId);
```

```csharp
namespace Hackathon.Api.Features.Register;

/// <summary>The response, and the user the controller issues the sign-in cookie for.</summary>
public sealed record RegisterResult(RegisterResponse Response, User User);
```

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Storage;

namespace Hackathon.Api.Features.Register;

public sealed class RegisterHandler(HackathonDbContext db, IPasswordHasher<User> passwords, IClock clock)
{
    public async Task<RegisterResult> HandleAsync(RegisterRequest request, CancellationToken cancellationToken)
    {
        ArgumentNullException.ThrowIfNull(request);

        PasswordPolicy.EnsureAcceptable(request.Password);

        DateTimeOffset now = clock.UtcNow;
        var user = User.Register(request.Email, now);
        string passwordHash = passwords.HashPassword(user, request.Password!);
        user.AssignPasswordHash(passwordHash, now);
        var workspace = Workspace.CreateFor(user.Id, now);

        await using IDbContextTransaction databaseTransaction = await db.Database.BeginTransactionAsync(cancellationToken);

        // Nobody else knows the new workspace yet, so nobody waits; the lock is taken because the write guard
        // accepts no workspace data — e.g. default records created at registration — without it.
        await db.LockWorkspaceAsync(workspace.Id, cancellationToken);

        db.Users.Add(user);
        db.Workspaces.Add(workspace);

        try
        {
            await db.SaveChangesAsync(cancellationToken);
            await databaseTransaction.CommitAsync(cancellationToken);
        }
        catch (DbUpdateException exception) when (PostgresErrors.IsUniqueViolation(exception))
        {
            throw new DomainException("An account with this e-mail already exists.", StatusCodes.Status409Conflict);
        }

        return new RegisterResult(new RegisterResponse(user.Id, workspace.Id), user);
    }
}
```

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace Hackathon.Api.Features.Register;

[ApiController]
[AllowAnonymous]
[Route("auth")]
public sealed class RegisterController(RegisterHandler handler) : ControllerBase
{
    [HttpPost("register")]
    public async Task<ActionResult<RegisterResponse>> Register(
        [FromBody] RegisterRequest request,
        CancellationToken cancellationToken)
    {
        RegisterResult registered = await handler.HandleAsync(request, cancellationToken);
        await SignInCookie.IssueAsync(HttpContext, registered.User, registered.Response.WorkspaceId);

        return Created(string.Empty, registered.Response);
    }
}
```

`Features/SignIn/` — ten sam układ: `SignInRequest(string? Email, string? Password)`, `SignInResponse(Guid UserId, Guid WorkspaceId)`, `SignInOutcome(SignInResponse Response, User User)`, kontroler `[AllowAnonymous] [Route("auth")]` z `[HttpPost("sign-in")]`.

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace Hackathon.Api.Features.SignIn;

public sealed class SignInHandler(HackathonDbContext db, IPasswordHasher<User> passwords)
{
    public async Task<SignInOutcome> HandleAsync(SignInRequest request, CancellationToken cancellationToken)
    {
        ArgumentNullException.ThrowIfNull(request);

        // One answer for every failure, so the response does not tell which addresses have an account.
        string? email = User.NormalizeEmail(request.Email);
        if (email is null || string.IsNullOrEmpty(request.Password))
        {
            throw new DomainException("Invalid e-mail or password.", StatusCodes.Status401Unauthorized);
        }

        User? user = await db.Users.SingleOrDefaultAsync(candidate => candidate.Email == email, cancellationToken);
        if (user is null)
        {
            throw new DomainException("Invalid e-mail or password.", StatusCodes.Status401Unauthorized);
        }

        PasswordVerificationResult verification = passwords.VerifyHashedPassword(user, user.PasswordHash, request.Password);
        if (verification == PasswordVerificationResult.Failed)
        {
            throw new DomainException("Invalid e-mail or password.", StatusCodes.Status401Unauthorized);
        }

        // Before sign-in no workspace is in scope, so the workspace is read by its owner, not through the filter.
        Workspace workspace = await db.Workspaces.SingleAsync(candidate => candidate.OwnerUserId == user.Id, cancellationToken);

        return new SignInOutcome(new SignInResponse(user.Id, workspace.Id), user);
    }
}
```

`Features/SignOut/SignOutController.cs` — bez handlera, bo nie dotyka danych:

```csharp
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Mvc;

namespace Hackathon.Api.Features.SignOut;

[ApiController]
[Route("auth")]
public sealed class SignOutController : ControllerBase
{
    [HttpPost("sign-out")]
    public async Task<IActionResult> Post()
    {
        await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);

        return NoContent();
    }
}
```

## Program.cs

```csharp
using System.Text.Json.Serialization;
using Hackathon.Api.Features.Register;
using Hackathon.Api.Features.SignIn;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc.Authorization;
using Microsoft.EntityFrameworkCore;
using Scalar.AspNetCore;

WebApplicationBuilder builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers(options =>
    {
        options.Filters.Add(new AuthorizeFilter());
        options.Filters.Add<DomainExceptionFilter>();
    })
    .AddJsonOptions(options =>
        options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter()));

builder.Services.AddOpenApi();

builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.Cookie.Name = "Hackathon.Auth";
        options.Cookie.HttpOnly = true;
        options.Cookie.SameSite = SameSiteMode.Lax;
        options.Cookie.SecurePolicy = CookieSecurePolicy.SameAsRequest;
        options.SlidingExpiration = true;
        options.ExpireTimeSpan = TimeSpan.FromDays(14);
        options.Events.OnRedirectToLogin = context =>
        {
            context.Response.StatusCode = StatusCodes.Status401Unauthorized;
            return Task.CompletedTask;
        };
        options.Events.OnRedirectToAccessDenied = context =>
        {
            context.Response.StatusCode = StatusCodes.Status403Forbidden;
            return Task.CompletedTask;
        };
    });

builder.Services.AddAuthorization();

// Infrastructure
builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddSingleton<IPasswordHasher<User>, PasswordHasher<User>>();
builder.Services.AddScoped<WorkspaceContext>();
builder.Services.AddScoped<IWorkspaceContext>(services => services.GetRequiredService<WorkspaceContext>());
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<CurrentUser>();

// Account and session
builder.Services.AddScoped<RegisterHandler>();
builder.Services.AddScoped<SignInHandler>();

// Each resource adds its queries and handlers here, one line each.

string? connectionString = builder.Configuration.GetConnectionString("Hackathon");
builder.Services.AddDbContext<HackathonDbContext>(options =>
    options
        .UseNpgsql(connectionString)
        .AddInterceptors(new WorkspaceWriteGuard()));

WebApplication app = builder.Build();

app.UseAuthentication();
app.UseMiddleware<WorkspaceScopeMiddleware>();
app.UseAuthorization();

// Development only: the document describes every endpoint, including the shape of the sign-in request.
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi().AllowAnonymous();
    app.MapScalarApiReference().AllowAnonymous();
}

app.MapControllers();
app.Run();

public partial class Program;
```

`public partial class Program;` udostępnia typ dla `WebApplicationFactory<Program>` w testach.

## Testy szkieletu

`tests/Hackathon.Tests/Persistence/PostgresFixture.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Npgsql;
using Testcontainers.PostgreSql;

namespace Hackathon.Tests;

/// <summary>One real PostgreSQL 17 instance shared by the integration tests; every test gets its own database.</summary>
public sealed class PostgresFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer container = new PostgreSqlBuilder("postgres:17")
        .WithCommand("-c", "max_connections=1000")
        .Build();

    public Task InitializeAsync()
    {
        return container.StartAsync();
    }

    public Task DisposeAsync()
    {
        return container.DisposeAsync().AsTask();
    }

    public async Task<string> CreateMigratedDatabaseAsync()
    {
        string databaseName = $"test_{Guid.CreateVersion7():n}";
        string serverConnectionString = container.GetConnectionString();

        await using (var connection = new NpgsqlConnection(serverConnectionString))
        {
            await connection.OpenAsync();
            await using var command = new NpgsqlCommand($"""CREATE DATABASE "{databaseName}" """, connection);
            await command.ExecuteNonQueryAsync();
        }

        string connectionString = new NpgsqlConnectionStringBuilder(serverConnectionString)
        {
            Database = databaseName,
        }.ConnectionString;

        await using HackathonDbContext context = TestDbContexts.Production(connectionString, new WorkspaceContext());
        await context.Database.MigrateAsync();

        return connectionString;
    }
}
```

`tests/Hackathon.Tests/Persistence/PostgresCollection.cs`

```csharp
namespace Hackathon.Tests;

[CollectionDefinition(Name)]
public sealed class PostgresCollection : ICollectionFixture<PostgresFixture>
{
    public const string Name = "postgres";
}
```

`tests/Hackathon.Tests/Persistence/TestDbContexts.cs`

```csharp
using Microsoft.EntityFrameworkCore;

namespace Hackathon.Tests;

internal static class TestDbContexts
{
    /// <summary>Building a model needs a provider, not a reachable server.</summary>
    private const string ModelOnlyConnectionString = "Host=localhost;Database=model_only";

    public static HackathonDbContext Production(string connectionString, IWorkspaceContext workspaceContext)
    {
        DbContextOptions<HackathonDbContext> options = new DbContextOptionsBuilder<HackathonDbContext>()
            .UseNpgsql(connectionString)
            .AddInterceptors(new WorkspaceWriteGuard())
            .Options;

        return new HackathonDbContext(options, workspaceContext);
    }

    public static HackathonDbContext ProductionModel()
    {
        return Production(ModelOnlyConnectionString, new WorkspaceContext());
    }
}
```

`tests/Hackathon.Tests/Features/ApiFactory.cs` i `TestApi.cs`

```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;

namespace Hackathon.Tests;

internal sealed class ApiFactory(string connectionString) : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseSetting("ConnectionStrings:Hackathon", connectionString);
        builder.UseSetting("Logging:LogLevel:Default", "Warning");
        builder.UseEnvironment("Development");
    }
}
```

```csharp
using System.Net.Http.Json;
using Hackathon.Api.Features.Register;
using Microsoft.AspNetCore.Mvc.Testing;

namespace Hackathon.Tests;

/// <summary>A migrated database of its own behind a signed-in client.</summary>
internal static class TestApi
{
    private static readonly WebApplicationFactoryClientOptions Cookies = new()
    {
        AllowAutoRedirect = false,
        HandleCookies = true,
    };

    public static async Task<ApiFactory> FactoryAsync(PostgresFixture postgres)
    {
        string connectionString = await postgres.CreateMigratedDatabaseAsync();

        return new ApiFactory(connectionString);
    }

    public static async Task<HttpClient> SignedInAsync(PostgresFixture postgres, string email)
    {
        ApiFactory factory = await FactoryAsync(postgres);

        return await SignInAsync(factory, email);
    }

    public static async Task<HttpClient> SignInAsync(ApiFactory factory, string email)
    {
        HttpClient client = factory.CreateClient(Cookies);
        HttpResponseMessage registered = await client.PostAsJsonAsync("/auth/register", new RegisterRequest(email, "password1"));
        registered.EnsureSuccessStatusCode();

        return client;
    }
}
```

`tests/Hackathon.Tests/Persistence/Workspace/WorkspaceIsolationTests.cs`

```csharp
using Microsoft.EntityFrameworkCore.Metadata;

namespace Hackathon.Tests;

public sealed class WorkspaceIsolationTests
{
    /// <summary>
    /// Entities deliberately shared across workspaces, each with the reason. Every other entity is workspace
    /// data. Adding a line here is a decision visible in review, not an accident.
    /// </summary>
    private static readonly IReadOnlyDictionary<Type, string> SharedAcrossWorkspaces = new Dictionary<Type, string>
    {
        [typeof(Workspace)] = "the workspace itself",
        [typeof(User)] = "read at sign-in, before any workspace is known",
    };

    [Fact]
    public void Every_entity_is_workspace_owned_unless_listed_as_shared()
    {
        using HackathonDbContext context = TestDbContexts.ProductionModel();

        string[] unmarked = context.Model.GetEntityTypes()
            .Where(entityType => !entityType.IsOwned())
            .Where(entityType => !SharedAcrossWorkspaces.ContainsKey(entityType.ClrType))
            .Where(entityType => !typeof(IWorkspaceOwned).IsAssignableFrom(entityType.ClrType))
            .Select(entityType => entityType.ShortName())
            .ToArray();

        Assert.Empty(unmarked);
    }

    [Fact]
    public void Every_workspace_owned_entity_has_the_isolation_filter()
    {
        using HackathonDbContext context = TestDbContexts.ProductionModel();

        string[] unprotected = context.Model.GetEntityTypes()
            .Where(entityType => typeof(IWorkspaceOwned).IsAssignableFrom(entityType.ClrType))
            .Where(entityType => entityType.GetDeclaredQueryFilters()?.Any() != true)
            .Select(entityType => entityType.ShortName())
            .ToArray();

        Assert.Empty(unprotected);
    }
}
```

`tests/Hackathon.Tests/Architecture/DependencyRulesTests.cs`

```csharp
using System.Reflection;
using NetArchTest.Rules;

namespace Hackathon.Tests;

/// <summary>
/// The boundaries from docs/architecture/01-structure-conventions.md, checked on the compiled assembly. The check
/// reads IL, so a dependency hidden in a method body counts the same as a using directive.
/// </summary>
public sealed class DependencyRulesTests
{
    private const string Root = "Hackathon.Api";
    private const string DomainNamespace = Root + ".Domain";
    private const string PersistenceNamespace = Root + ".Persistence";
    private const string QueriesNamespace = Root + ".Queries";
    private const string InfrastructureNamespace = Root + ".Infrastructure";
    private const string FeaturesNamespace = Root + ".Features";

    private static readonly Assembly Api = typeof(HackathonDbContext).Assembly;

    [Fact]
    public void Domain_knows_neither_EF_Core_nor_HTTP_nor_the_rest_of_the_application()
    {
        TestResult result = Types.InAssembly(Api)
            .That().ResideInNamespace(DomainNamespace)
            .ShouldNot().HaveDependencyOnAny(
                "Microsoft.EntityFrameworkCore",
                "Microsoft.AspNetCore",
                PersistenceNamespace,
                QueriesNamespace,
                InfrastructureNamespace,
                FeaturesNamespace)
            .GetResult();

        AssertPasses(result);
    }

    [Fact]
    public void Persistence_knows_neither_queries_nor_infrastructure_nor_features()
    {
        TestResult result = Types.InAssembly(Api)
            .That().ResideInNamespace(PersistenceNamespace)
            .ShouldNot().HaveDependencyOnAny(QueriesNamespace, InfrastructureNamespace, FeaturesNamespace)
            .GetResult();

        AssertPasses(result);
    }

    [Fact]
    public void Persistence_holds_no_query()
    {
        TestResult result = Types.InAssembly(Api)
            .That().ResideInNamespace(PersistenceNamespace)
            .ShouldNot().HaveNameEndingWith("Query")
            .GetResult();

        AssertPasses(result);
    }

    [Fact]
    public void Queries_know_neither_infrastructure_nor_features()
    {
        TestResult result = Types.InAssembly(Api)
            .That().ResideInNamespace(QueriesNamespace)
            .ShouldNot().HaveDependencyOnAny(InfrastructureNamespace, FeaturesNamespace)
            .GetResult();

        AssertPasses(result);
    }

    [Fact]
    public void A_class_reading_the_database_in_queries_is_an_injected_query()
    {
        TestResult result = Types.InAssembly(Api)
            .That().ResideInNamespace(QueriesNamespace)
            .And().AreNotNested()
            .And().HaveDependencyOn(PersistenceNamespace + ".HackathonDbContext")
            .Should().HaveNameEndingWith("Query")
            .And().NotBeStatic()
            .GetResult();

        AssertPasses(result);
    }

    [Fact]
    public void Infrastructure_knows_neither_queries_nor_features()
    {
        TestResult result = Types.InAssembly(Api)
            .That().ResideInNamespace(InfrastructureNamespace)
            .ShouldNot().HaveDependencyOnAny(QueriesNamespace, FeaturesNamespace)
            .GetResult();

        AssertPasses(result);
    }

    [Fact]
    public void Feature_folders_do_not_share_code()
    {
        string[] featureFolders = FeatureNamespaces()
            .Select(FeatureFolderOf)
            .Distinct()
            .ToArray();

        List<string> crossings = [];
        foreach (string folder in featureFolders)
        {
            foreach (string otherFolder in featureFolders.Where(other => other != folder))
            {
                TestResult result = Types.InAssembly(Api)
                    .That().ResideInNamespace(folder)
                    .ShouldNot().HaveDependencyOn(otherFolder)
                    .GetResult();

                if (!result.IsSuccessful)
                {
                    crossings.Add($"{folder} -> {otherFolder}");
                }
            }
        }

        Assert.Empty(crossings);
    }

    [Fact]
    public void A_subfolder_of_a_feature_does_not_reach_into_its_siblings()
    {
        string[] subfolders = FeatureNamespaces()
            .Select(SubfolderOf)
            .OfType<string>()
            .Where(subfolder => !subfolder.EndsWith(".Converters", StringComparison.Ordinal))
            .Distinct()
            .ToArray();

        List<string> crossings = [];
        foreach (string subfolder in subfolders)
        {
            string featureFolder = FeatureFolderOf(subfolder);
            foreach (string sibling in subfolders.Where(other => other != subfolder && FeatureFolderOf(other) == featureFolder))
            {
                TestResult result = Types.InAssembly(Api)
                    .That().ResideInNamespace(subfolder)
                    .ShouldNot().HaveDependencyOn(sibling)
                    .GetResult();

                if (!result.IsSuccessful)
                {
                    crossings.Add($"{subfolder} -> {sibling}");
                }
            }
        }

        Assert.Empty(crossings);
    }

    private static IEnumerable<string> FeatureNamespaces()
    {
        return Api.GetTypes()
            .Select(type => type.Namespace)
            .OfType<string>()
            .Where(name => name.StartsWith(FeaturesNamespace + ".", StringComparison.Ordinal))
            .Distinct();
    }

    /// <summary>The feature folder is the first level under <c>Features</c>; its subfolders belong to it.</summary>
    private static string FeatureFolderOf(string featureNamespace)
    {
        int nameStart = FeaturesNamespace.Length + 1;
        int nameEnd = featureNamespace.IndexOf('.', nameStart);

        return nameEnd < 0 ? featureNamespace : featureNamespace[..nameEnd];
    }

    /// <summary>The second level under <c>Features</c>; null for a namespace of the feature folder itself.</summary>
    private static string? SubfolderOf(string featureNamespace)
    {
        string featureFolder = FeatureFolderOf(featureNamespace);
        if (featureNamespace == featureFolder)
        {
            return null;
        }

        string subfolderName = featureNamespace[(featureFolder.Length + 1)..].Split('.')[0];

        return featureFolder + "." + subfolderName;
    }

    private static void AssertPasses(TestResult result)
    {
        string[] offenders = (result.FailingTypes ?? []).Select(type => type.FullName ?? type.Name).ToArray();

        Assert.Empty(offenders);
    }
}
```

Do szkieletu warto dopisać jeszcze dwa testy na prawdziwej bazie: **strażnik zapisu** (zapis encji workspace'u bez `LockWorkspaceAsync` rzuca `InvalidOperationException`) i **rejestracja/logowanie/wylogowanie** przez API (201, 409 dla zajętego e-maila, 401 dla złego hasła, 401 na chronionym endpoincie po wylogowaniu).

## Pierwsza migracja

```bash
docker compose up -d
dotnet dotnet-ef migrations add Initial --project src/Hackathon.Api
dotnet dotnet-ef database update --project src/Hackathon.Api
dotnet test
```

**Docker i Testcontainers:** jeśli `dotnet test` kończy się błędem „client version 1.44 is too new”, lokalny Docker Engine jest starszy niż wymaga Testcontainers — zaktualizuj Docker Desktop albo ustaw zmienną środowiskową `DOCKER_API_VERSION=1.43` dla testów (np. w pliku `.runsettings`).
