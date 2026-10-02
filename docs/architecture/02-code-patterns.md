# Wzorce kodu

Pliki, które powstają przy każdym wycinku. Przykład — sale (`Room`) i rezerwacje (`Booking`) — pokazuje wzorzec, nie temat projektu. Kopiując, zmieniasz nazwy i reguły; strukturę, styl i kolejność kroków zostawiasz.

Wszystkie przykłady spełniają reguły z [`01-structure-conventions.md`](01-structure-conventions.md): jawne typy, wynik wywołania w nazwanej zmiennej, metody w klamrach, jeden typ na plik.

## Kolejność pracy nad wycinkiem

1. **Słownik:** nazwy encji, pól i wartości enum z [`../product.md`](../product.md). Brakujące — dopisz.
2. **Test domeny** dla reguł encji (bez bazy) i **test API** dla przypadku użycia — choćby szkic z oczekiwanymi kodami HTTP.
3. **`Domain/<Temat>/`** — encja, enumy, ewentualny typ reguły.
4. **`Persistence/<Temat>/`** — `…Configuration`, `DbSet` w `HackathonDbContext`.
5. **Migracja** — `dotnet dotnet-ef migrations add <Nazwa> --project src/Hackathon.Api`, przejrzenie wygenerowanego pliku.
6. **`Queries/<Temat>/`** — tylko jeśli odczyt jest potrzebny w więcej niż jednym handlerze.
7. **`Features/<Zasoby>/`** — żądania, odpowiedź, konwerter, handlery, kontroler.
8. **`Program.cs`** — rejestracja handlerów i zapytań.
9. `dotnet build`, `dotnet test`, sprawdzenie w Scalar.

## Domain — enum

`Domain/Rooms/RoomStatus.cs`

```csharp
namespace Hackathon.Api.Domain;

public enum RoomStatus
{
    ACTIVE,
    ARCHIVED,
}
```

## Domain — encja

`Domain/Rooms/Room.cs`

```csharp
namespace Hackathon.Api.Domain;

/// <summary>
/// A room people book by the hour. An archived room keeps its past bookings and takes no new one.
/// </summary>
public sealed class Room : IWorkspaceOwned
{
    public const int NameMaxLength = 100;

    private Room()
    {
    }

    public Guid Id { get; private set; }

    public Guid WorkspaceId { get; private set; }

    public string Name { get; private set; } = null!;

    public int Capacity { get; private set; }

    public RoomStatus Status { get; private set; }

    public DateTimeOffset CreatedAt { get; private set; }

    public DateTimeOffset UpdatedAt { get; private set; }

    public bool IsActive => Status == RoomStatus.ACTIVE;

    public static Room Create(Guid workspaceId, string? name, int capacity, DateTimeOffset createdAt)
    {
        if (workspaceId == Guid.Empty)
        {
            throw new InvalidOperationException("The workspace id of a new room is empty.");
        }

        var room = new Room
        {
            Id = Guid.CreateVersion7(),
            WorkspaceId = workspaceId,
            Status = RoomStatus.ACTIVE,
            CreatedAt = createdAt,
        };

        room.Rename(name, createdAt);
        room.ChangeCapacity(capacity, createdAt);
        return room;
    }

    public void Rename(string? name, DateTimeOffset changedAt)
    {
        if (string.IsNullOrWhiteSpace(name))
        {
            throw new DomainException("A room needs a name.");
        }

        string trimmed = name.Trim();
        if (trimmed.Length > NameMaxLength)
        {
            throw new DomainException($"A room name has at most {NameMaxLength} characters.");
        }

        Name = trimmed;
        UpdatedAt = changedAt;
    }

    public void ChangeCapacity(int capacity, DateTimeOffset changedAt)
    {
        if (capacity < 1)
        {
            throw new DomainException("A room holds at least one person.");
        }

        Capacity = capacity;
        UpdatedAt = changedAt;
    }

    public void Archive(DateTimeOffset changedAt)
    {
        if (!IsActive)
        {
            throw new DomainException("The room is already archived.", StatusCodes.Status409Conflict);
        }

        Status = RoomStatus.ARCHIVED;
        UpdatedAt = changedAt;
    }

    /// <summary>A booking checks its own room before it takes a slot.</summary>
    public void EnsureBookable()
    {
        if (!IsActive)
        {
            throw new DomainException("An archived room cannot be booked.");
        }
    }
}
```

Na co patrzeć:

- prywatny konstruktor dla EF, `private set` wszędzie, stan zmieniają tylko metody;
- fabryka o nazwie czynności nadaje `Id` z `Guid.CreateVersion7()` i woła te same metody co późniejsza edycja — reguła nazwy jest w jednym miejscu;
- pusty `workspaceId` to błąd programisty (`InvalidOperationException`), zła nazwa to odmowa dla użytkownika (`DomainException`);
- czas przychodzi parametrem, encja nie woła zegara.

## Domain — encja z nawigacją i warunkiem zapytania

`Domain/Bookings/Booking.cs`

```csharp
using System.Linq.Expressions;

namespace Hackathon.Api.Domain;

/// <summary>One room reserved for one time slot on one day.</summary>
public sealed class Booking : IWorkspaceOwned
{
    /// <summary>
    /// The bookings that hold their slot. An expression, so the same rule filters rows in the database
    /// and is never rewritten by hand in a query.
    /// </summary>
    public static readonly Expression<Func<Booking, bool>> HoldsSlot = booking =>
        booking.Status == BookingStatus.CONFIRMED;

    private Booking()
    {
    }

    public Guid Id { get; private set; }

    public Guid WorkspaceId { get; private set; }

    public Guid RoomId { get; private set; }

    public Room Room { get; private set; } = null!;

    public Guid BookedByUserId { get; private set; }

    public DateOnly Day { get; private set; }

    public TimeOnly StartsAt { get; private set; }

    public TimeOnly EndsAt { get; private set; }

    public BookingStatus Status { get; private set; }

    public DateTimeOffset CreatedAt { get; private set; }

    public DateTimeOffset UpdatedAt { get; private set; }

    public static Booking Book(
        Room room,
        Guid bookedByUserId,
        DateOnly day,
        TimeOnly startsAt,
        TimeOnly endsAt,
        DateTimeOffset createdAt)
    {
        ArgumentNullException.ThrowIfNull(room);

        if (bookedByUserId == Guid.Empty)
        {
            throw new InvalidOperationException("The user id of a new booking is empty.");
        }

        room.EnsureBookable();
        EnsureSlot(startsAt, endsAt);

        return new Booking
        {
            Id = Guid.CreateVersion7(),
            WorkspaceId = room.WorkspaceId,
            RoomId = room.Id,
            Room = room,
            BookedByUserId = bookedByUserId,
            Day = day,
            StartsAt = startsAt,
            EndsAt = endsAt,
            Status = BookingStatus.CONFIRMED,
            CreatedAt = createdAt,
            UpdatedAt = createdAt,
        };
    }

    /// <summary>Needs <see cref="Room"/> loaded: a slot moves only within a room that still takes bookings.</summary>
    public void Reschedule(DateOnly day, TimeOnly startsAt, TimeOnly endsAt, DateTimeOffset changedAt)
    {
        EnsureConfirmed();
        Room.EnsureBookable();
        EnsureSlot(startsAt, endsAt);

        Day = day;
        StartsAt = startsAt;
        EndsAt = endsAt;
        UpdatedAt = changedAt;
    }

    public void Cancel(DateTimeOffset changedAt)
    {
        EnsureConfirmed();

        Status = BookingStatus.CANCELLED;
        UpdatedAt = changedAt;
    }

    /// <summary>Slots touching at an edge (9:00–10:00 and 10:00–11:00) do not overlap.</summary>
    public bool Overlaps(DateOnly day, TimeOnly startsAt, TimeOnly endsAt)
    {
        return Day == day && StartsAt < endsAt && startsAt < EndsAt;
    }

    private static void EnsureSlot(TimeOnly startsAt, TimeOnly endsAt)
    {
        if (endsAt <= startsAt)
        {
            throw new DomainException("A booking ends after it starts.");
        }
    }

    private void EnsureConfirmed()
    {
        if (Status != BookingStatus.CONFIRMED)
        {
            throw new DomainException("A cancelled booking cannot change.", StatusCodes.Status409Conflict);
        }
    }
}
```

Rezerwacja sprawdza swoją salę, bo ją ma (nawigacja). **Nie** sprawdza, czy termin jest wolny — to zależy od innych rezerwacji, czyli danych spoza niej.

## Domain — typ reguły używanej przez kilka handlerów

„Termin sali jest wolny” potrzebują `CreateBookingHandler` i `RescheduleBookingHandler`. Reguła dostaje dane, na których decyduje, i jest testowalna bez bazy.

`Domain/Bookings/RoomSchedule.cs`

```csharp
namespace Hackathon.Api.Domain;

/// <summary>
/// The bookings holding a slot in one room on one day, and whether another slot still fits between them.
/// </summary>
public sealed class RoomSchedule
{
    private readonly IReadOnlyList<Booking> bookingsHoldingSlot;

    public RoomSchedule(IReadOnlyList<Booking> bookingsHoldingSlot)
    {
        ArgumentNullException.ThrowIfNull(bookingsHoldingSlot);

        this.bookingsHoldingSlot = bookingsHoldingSlot;
    }

    /// <param name="movedBookingId">A booking being rescheduled does not collide with its own old slot.</param>
    public void EnsureFree(DateOnly day, TimeOnly startsAt, TimeOnly endsAt, Guid? movedBookingId)
    {
        bool taken = bookingsHoldingSlot.Any(booking =>
            booking.Id != movedBookingId && booking.Overlaps(day, startsAt, endsAt));

        if (taken)
        {
            throw new DomainException("The room is already booked at that time.", StatusCodes.Status409Conflict);
        }
    }
}
```

## Persistence — mapowanie

`Persistence/Rooms/RoomConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace Hackathon.Api.Persistence;

public sealed class RoomConfiguration : IEntityTypeConfiguration<Room>
{
    public void Configure(EntityTypeBuilder<Room> builder)
    {
        builder.HasKey(room => room.Id);

        builder.HasOne<Workspace>()
            .WithMany()
            .HasForeignKey(room => room.WorkspaceId)
            .OnDelete(DeleteBehavior.Restrict);

        builder.Property(room => room.Name).HasMaxLength(Room.NameMaxLength);

        builder.HasIndex(room => new { room.WorkspaceId, room.Name })
            .IsUnique()
            .HasDatabaseName("IX_Rooms_OneNamePerWorkspace");

        builder.ToTable(table =>
        {
            table.HasCheckConstraint("CK_Rooms_Capacity_Positive", "\"Capacity\" >= 1");
        });
    }
}
```

`Persistence/Bookings/BookingConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace Hackathon.Api.Persistence;

public sealed class BookingConfiguration : IEntityTypeConfiguration<Booking>
{
    public void Configure(EntityTypeBuilder<Booking> builder)
    {
        builder.HasKey(booking => booking.Id);

        builder.HasOne<Workspace>()
            .WithMany()
            .HasForeignKey(booking => booking.WorkspaceId)
            .OnDelete(DeleteBehavior.Restrict);

        // Workspace and room together: a booking cannot point at another workspace's room.
        builder.HasOne(booking => booking.Room)
            .WithMany()
            .HasForeignKey(booking => new { booking.WorkspaceId, booking.RoomId })
            .HasPrincipalKey(room => new { room.WorkspaceId, room.Id })
            .HasConstraintName("FK_Bookings_Room_SameWorkspace")
            .OnDelete(DeleteBehavior.Restrict);

        builder.HasOne<User>()
            .WithMany()
            .HasForeignKey(booking => booking.BookedByUserId)
            .OnDelete(DeleteBehavior.Restrict);

        builder.HasIndex(booking => new { booking.WorkspaceId, booking.RoomId, booking.Day });

        builder.ToTable(table =>
        {
            table.HasCheckConstraint("CK_Bookings_EndsAfterStart", "\"EndsAt\" > \"StartsAt\"");
        });
    }
}
```

Precyzji, enumów jako tekstu ani `ValueGenerated.Never` tu nie ustawiasz — robią to globalne konwencje ze szkieletu. W `HackathonDbContext` dopisujesz:

```csharp
public DbSet<Room> Rooms => Set<Room>();

public DbSet<Booking> Bookings => Set<Booking>();
```

Po migracji sprawdzasz w wygenerowanym pliku: tabelę, klucz złożony do sali, indeks unikalny, `CHECK`, kolumnę `Status` jako `text`, brak indeksu zdublowanego przez inny.

## Queries — odczyt wspólny

`Queries/Bookings/RoomScheduleQuery.cs`

```csharp
using Microsoft.EntityFrameworkCore;

namespace Hackathon.Api.Queries;

/// <summary>The bookings that hold a slot in one room on one day, for the rule that decides whether a slot is free.</summary>
public sealed class RoomScheduleQuery(HackathonDbContext db)
{
    public async Task<RoomSchedule> OfDayAsync(Guid roomId, DateOnly day, CancellationToken cancellationToken)
    {
        List<Booking> bookings = await db.Bookings
            .AsNoTracking()
            .Where(Booking.HoldsSlot)
            .Where(booking => booking.RoomId == roomId && booking.Day == day)
            .ToListAsync(cancellationToken);

        return new RoomSchedule(bookings);
    }
}
```

Bez `Where(booking => booking.WorkspaceId == ...)` — izolację daje filtr. Warunek „trzyma termin” pochodzi z `Booking.HoldsSlot`, nie jest przepisany.

## Features — żądania i odpowiedź

`Features/Bookings/CreateBookingRequest.cs`

```csharp
namespace Hackathon.Api.Features.Bookings;

public sealed record CreateBookingRequest(Guid RoomId, DateOnly Day, TimeOnly StartsAt, TimeOnly EndsAt);
```

`Features/Bookings/RescheduleBookingRequest.cs`

```csharp
namespace Hackathon.Api.Features.Bookings;

public sealed record RescheduleBookingRequest(DateOnly Day, TimeOnly StartsAt, TimeOnly EndsAt);
```

`Features/Bookings/ListBookingsRequest.cs`

```csharp
namespace Hackathon.Api.Features.Bookings;

/// <param name="Status">CONFIRMED or CANCELLED; all bookings when left out.</param>
public sealed record ListBookingsRequest(DateOnly? Day, string? Status);
```

`Features/Rooms/CreateRoomRequest.cs`

```csharp
namespace Hackathon.Api.Features.Rooms;

/// <param name="Capacity">Nullable on purpose: a field missing from the JSON must be refused, not read as 0.</param>
public sealed record CreateRoomRequest(string? Name, int? Capacity);
```

`Features/Bookings/BookingResponse.cs`

```csharp
namespace Hackathon.Api.Features.Bookings;

public sealed record BookingResponse(
    Guid Id,
    Guid RoomId,
    DateOnly Day,
    TimeOnly StartsAt,
    TimeOnly EndsAt,
    string Status);
```

`Features/Rooms/RoomResponse.cs` — analogicznie, z `RoomConverter.ToResponse()` obok:

```csharp
namespace Hackathon.Api.Features.Rooms;

public sealed record RoomResponse(Guid Id, string Name, int Capacity, string Status);
```

## Features — konwerter

`Features/Bookings/BookingConverter.cs`

```csharp
namespace Hackathon.Api.Features.Bookings;

internal static class BookingConverter
{
    public static BookingResponse ToResponse(this Booking booking)
    {
        string status = booking.Status.ToString();

        return new(booking.Id, booking.RoomId, booking.Day, booking.StartsAt, booking.EndsAt, status);
    }
}
```

Konwerter przepisuje pola. Nie czyta bazy, nie liczy reguł.

## Features — handler zapisu

`Features/Bookings/CreateBookingHandler.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Storage;

namespace Hackathon.Api.Features.Bookings;

public sealed class CreateBookingHandler(
    HackathonDbContext db,
    CurrentUser currentUser,
    RoomScheduleQuery roomSchedule,
    IClock clock)
{
    public async Task<BookingResponse> HandleAsync(CreateBookingRequest request, CancellationToken cancellationToken)
    {
        ArgumentNullException.ThrowIfNull(request);

        await using IDbContextTransaction databaseTransaction = await db.Database.BeginTransactionAsync(cancellationToken);
        await db.LockWorkspaceAsync(currentUser.WorkspaceId, cancellationToken);

        Room room = await db.Rooms.SingleOrDefaultAsync(candidate => candidate.Id == request.RoomId, cancellationToken)
            ?? throw new DomainException("The room does not exist.", StatusCodes.Status404NotFound);

        RoomSchedule schedule = await roomSchedule.OfDayAsync(room.Id, request.Day, cancellationToken);
        schedule.EnsureFree(request.Day, request.StartsAt, request.EndsAt, movedBookingId: null);

        var booking = Booking.Book(room, currentUser.UserId, request.Day, request.StartsAt, request.EndsAt, clock.UtcNow);
        db.Bookings.Add(booking);

        await db.SaveChangesAsync(cancellationToken);
        await databaseTransaction.CommitAsync(cancellationToken);

        return booking.ToResponse();
    }
}
```

Kroki zawsze w tej kolejności: transakcja → blokada → wczytanie (404, jeśli brak) → reguły → zmiana przez metodę encji → `SaveChanges` → commit → odpowiedź.

`Features/Bookings/RescheduleBookingHandler.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Storage;

namespace Hackathon.Api.Features.Bookings;

public sealed class RescheduleBookingHandler(
    HackathonDbContext db,
    CurrentUser currentUser,
    RoomScheduleQuery roomSchedule,
    IClock clock)
{
    public async Task<BookingResponse> HandleAsync(
        Guid bookingId,
        RescheduleBookingRequest request,
        CancellationToken cancellationToken)
    {
        ArgumentNullException.ThrowIfNull(request);

        await using IDbContextTransaction databaseTransaction = await db.Database.BeginTransactionAsync(cancellationToken);
        await db.LockWorkspaceAsync(currentUser.WorkspaceId, cancellationToken);

        Booking booking = await db.Bookings
                .Include(candidate => candidate.Room)
                .SingleOrDefaultAsync(candidate => candidate.Id == bookingId, cancellationToken)
            ?? throw new DomainException("The booking does not exist.", StatusCodes.Status404NotFound);

        RoomSchedule schedule = await roomSchedule.OfDayAsync(booking.RoomId, request.Day, cancellationToken);
        schedule.EnsureFree(request.Day, request.StartsAt, request.EndsAt, booking.Id);

        booking.Reschedule(request.Day, request.StartsAt, request.EndsAt, clock.UtcNow);

        await db.SaveChangesAsync(cancellationToken);
        await databaseTransaction.CommitAsync(cancellationToken);

        return booking.ToResponse();
    }
}
```

## Features — handler zapisu z unikalnością i polem nullowalnym

`Features/Rooms/CreateRoomHandler.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Storage;

namespace Hackathon.Api.Features.Rooms;

public sealed class CreateRoomHandler(HackathonDbContext db, CurrentUser currentUser, IClock clock)
{
    public async Task<RoomResponse> HandleAsync(CreateRoomRequest request, CancellationToken cancellationToken)
    {
        ArgumentNullException.ThrowIfNull(request);

        if (request.Capacity is not int capacity)
        {
            throw new DomainException("A room needs a capacity.");
        }

        await using IDbContextTransaction databaseTransaction = await db.Database.BeginTransactionAsync(cancellationToken);
        await db.LockWorkspaceAsync(currentUser.WorkspaceId, cancellationToken);

        var room = Room.Create(currentUser.WorkspaceId, request.Name, capacity, clock.UtcNow);
        db.Rooms.Add(room);

        try
        {
            await db.SaveChangesAsync(cancellationToken);
            await databaseTransaction.CommitAsync(cancellationToken);
        }
        catch (DbUpdateException exception) when (PostgresErrors.IsUniqueViolation(exception))
        {
            throw new DomainException("A room with this name already exists.", StatusCodes.Status409Conflict);
        }

        return room.ToResponse();
    }
}
```

`databaseTransaction` jest zadeklarowana przed `try` — gdyby `await using` leżało w `try`, transakcja byłaby zamknięta, zanim kod wejdzie do `catch`.

## Features — handler odczytu

`Features/Bookings/ListBookingsHandler.cs`

```csharp
using Microsoft.EntityFrameworkCore;

namespace Hackathon.Api.Features.Bookings;

public sealed class ListBookingsHandler(HackathonDbContext db)
{
    public async Task<IReadOnlyList<BookingResponse>> HandleAsync(
        ListBookingsRequest request,
        CancellationToken cancellationToken)
    {
        ArgumentNullException.ThrowIfNull(request);

        IQueryable<Booking> query = db.Bookings.AsNoTracking();

        if (request.Day is DateOnly day)
        {
            query = query.Where(booking => booking.Day == day);
        }

        if (request.Status is not null)
        {
            if (!NamedEnum.TryParse(request.Status, out BookingStatus status))
            {
                throw new DomainException("Status is CONFIRMED or CANCELLED.");
            }

            query = query.Where(booking => booking.Status == status);
        }

        List<Booking> bookings = await query
            .OrderBy(booking => booking.Day)
            .ThenBy(booking => booking.StartsAt)
            .ToListAsync(cancellationToken);

        return bookings.Select(booking => booking.ToResponse()).ToList();
    }
}
```

Handler tylko czytający nie otwiera transakcji i nie bierze blokady. Lista jest odczytem jednego handlera, więc zostaje w nim — nie trafia do `Queries/`.

## Features — kontroler

`Features/Bookings/BookingsController.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace Hackathon.Api.Features.Bookings;

[ApiController]
[Authorize]
[Route("bookings")]
public sealed class BookingsController(
    CreateBookingHandler create,
    RescheduleBookingHandler reschedule,
    CancelBookingHandler cancel,
    ListBookingsHandler list) : ControllerBase
{
    [HttpGet]
    public Task<IReadOnlyList<BookingResponse>> Get(
        [FromQuery] ListBookingsRequest request,
        CancellationToken cancellationToken)
    {
        return list.HandleAsync(request, cancellationToken);
    }

    [HttpPost]
    public async Task<ActionResult<BookingResponse>> Post(
        [FromBody] CreateBookingRequest request,
        CancellationToken cancellationToken)
    {
        BookingResponse booking = await create.HandleAsync(request, cancellationToken);

        return Created($"/bookings/{booking.Id}", booking);
    }

    [HttpPost("{bookingId:guid}/reschedule")]
    public Task<BookingResponse> Reschedule(
        Guid bookingId,
        [FromBody] RescheduleBookingRequest request,
        CancellationToken cancellationToken)
    {
        return reschedule.HandleAsync(bookingId, request, cancellationToken);
    }

    [HttpPost("{bookingId:guid}/cancel")]
    public Task<BookingResponse> Cancel(Guid bookingId, CancellationToken cancellationToken)
    {
        return cancel.HandleAsync(bookingId, cancellationToken);
    }
}
```

| Czynność | HTTP | Odpowiedź |
|---|---|---|
| Lista | `GET /<zasoby>` z filtrami w query | 200 + tablica |
| Szczegóły | `GET /<zasoby>/{id}` | 200 / 404 |
| Utworzenie | `POST /<zasoby>` | 201 + obiekt |
| Edycja pól | `PATCH /<zasoby>/{id}` | 200 + obiekt |
| Czynność domenowa | `POST /<zasoby>/{id}/<czynność>` | 200 + obiekt |
| Usunięcie | `DELETE /<zasoby>/{id}` | 204 |

Handler bez treści żądania (szczegóły, usunięcie, czynność bez danych) przyjmuje tylko parametry trasy. `CancelBookingHandler` wygląda jak `RescheduleBookingHandler` bez żądania i bez `RoomScheduleQuery`: transakcja, blokada, wczytanie (404), `booking.Cancel(clock.UtcNow)`, zapis, commit.

## Program.cs — rejestracja

Każdy nowy handler i zapytanie rejestrujesz jawnie, w bloku swojego zasobu:

```csharp
builder.Services.AddScoped<RoomScheduleQuery>();
builder.Services.AddScoped<CreateBookingHandler>();
builder.Services.AddScoped<RescheduleBookingHandler>();
builder.Services.AddScoped<CancelBookingHandler>();
builder.Services.AddScoped<ListBookingsHandler>();
```

## Testy — domena bez bazy

`tests/Hackathon.Tests/Domain/Bookings/RoomScheduleTests.cs`

```csharp
using Microsoft.AspNetCore.Http;

namespace Hackathon.Tests;

public sealed class RoomScheduleTests
{
    private static readonly Guid WorkspaceId = Guid.CreateVersion7();

    private static readonly Guid UserId = Guid.CreateVersion7();

    private static readonly DateTimeOffset Now = new(2026, 10, 2, 12, 0, 0, TimeSpan.Zero);

    private static readonly DateOnly Monday = new(2026, 10, 5);

    [Fact]
    public void A_slot_touching_another_at_its_edge_is_free()
    {
        var room = Room.Create(WorkspaceId, "Room A", 8, Now);
        var morning = Booking.Book(room, UserId, Monday, new TimeOnly(9, 0), new TimeOnly(10, 0), Now);
        var schedule = new RoomSchedule([morning]);

        schedule.EnsureFree(Monday, new TimeOnly(10, 0), new TimeOnly(11, 0), movedBookingId: null);
    }

    [Fact]
    public void An_overlapping_slot_is_refused_with_a_conflict()
    {
        var room = Room.Create(WorkspaceId, "Room A", 8, Now);
        var morning = Booking.Book(room, UserId, Monday, new TimeOnly(9, 0), new TimeOnly(10, 0), Now);
        var schedule = new RoomSchedule([morning]);

        DomainException refusal = Assert.Throws<DomainException>(() =>
        {
            schedule.EnsureFree(Monday, new TimeOnly(9, 30), new TimeOnly(10, 30), movedBookingId: null);
        });

        Assert.Equal(StatusCodes.Status409Conflict, refusal.StatusCode);
    }

    [Fact]
    public void A_booking_moved_within_its_own_slot_does_not_collide_with_itself()
    {
        var room = Room.Create(WorkspaceId, "Room A", 8, Now);
        var morning = Booking.Book(room, UserId, Monday, new TimeOnly(9, 0), new TimeOnly(10, 0), Now);
        var schedule = new RoomSchedule([morning]);

        schedule.EnsureFree(Monday, new TimeOnly(9, 30), new TimeOnly(10, 30), morning.Id);
    }
}
```

## Testy — przypadek użycia przez API

`tests/Hackathon.Tests/Features/Bookings/BookingApiTests.cs`

```csharp
using System.Net;
using System.Net.Http.Json;
using Hackathon.Api.Features.Bookings;
using Hackathon.Api.Features.Rooms;

namespace Hackathon.Tests;

[Collection(PostgresCollection.Name)]
public sealed class BookingApiTests(PostgresFixture postgres)
{
    private static readonly DateOnly Monday = new(2026, 10, 5);

    [Fact]
    public async Task A_slot_taken_by_another_booking_is_refused_with_a_conflict()
    {
        HttpClient client = await TestApi.SignedInAsync(postgres, "booking-overlap@example.com");
        Guid roomId = await CreateRoomAsync(client, "Room A");

        HttpResponseMessage first = await client.PostAsJsonAsync(
            "/bookings",
            new CreateBookingRequest(roomId, Monday, new TimeOnly(9, 0), new TimeOnly(10, 0)));
        HttpResponseMessage overlapping = await client.PostAsJsonAsync(
            "/bookings",
            new CreateBookingRequest(roomId, Monday, new TimeOnly(9, 30), new TimeOnly(11, 0)));
        HttpResponseMessage adjacent = await client.PostAsJsonAsync(
            "/bookings",
            new CreateBookingRequest(roomId, Monday, new TimeOnly(10, 0), new TimeOnly(11, 0)));

        Assert.Equal(HttpStatusCode.Created, first.StatusCode);
        Assert.Equal(HttpStatusCode.Conflict, overlapping.StatusCode);
        Assert.Equal(HttpStatusCode.Created, adjacent.StatusCode);
    }

    /// <summary>
    /// No booking handler filters by workspace by hand; the isolation filter alone keeps another
    /// workspace's room and booking out of reach.
    /// </summary>
    [Fact]
    public async Task A_room_of_another_workspace_is_not_found()
    {
        ApiFactory factory = await TestApi.FactoryAsync(postgres);
        HttpClient ada = await TestApi.SignInAsync(factory, "ada-booking@example.com");
        HttpClient jan = await TestApi.SignInAsync(factory, "jan-booking@example.com");
        Guid janRoomId = await CreateRoomAsync(jan, "Room A");

        HttpResponseMessage booked = await ada.PostAsJsonAsync(
            "/bookings",
            new CreateBookingRequest(janRoomId, Monday, new TimeOnly(9, 0), new TimeOnly(10, 0)));
        BookingResponse[]? adaBookings = await ada.GetFromJsonAsync<BookingResponse[]>("/bookings");

        Assert.Equal(HttpStatusCode.NotFound, booked.StatusCode);
        Assert.Empty(adaBookings!);
    }

    private static async Task<Guid> CreateRoomAsync(HttpClient client, string name)
    {
        HttpResponseMessage response = await client.PostAsJsonAsync("/rooms", new CreateRoomRequest(name, 8));
        response.EnsureSuccessStatusCode();
        RoomResponse? room = await response.Content.ReadFromJsonAsync<RoomResponse>();

        return room!.Id;
    }
}
```

Każdy test API dostaje własną bazę (`TestApi.SignedInAsync`) i własny adres e-mail — testy nie dzielą stanu i mogą biec równolegle w ramach kolekcji. Każdy zasób ma test „cudzy workspace dostaje 404”.
