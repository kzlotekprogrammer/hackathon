# Struktura kodu i konwencje

## Układ repozytorium

```
<repo>/
├── AGENTS.md, CLAUDE.md
├── docs/
│   ├── product.md             co budujemy — źródło prawdy
│   ├── assumptions.md         założenia agenta do weryfikacji
│   └── architecture/          jak budujemy — rozstrzygnięte
└── backend/
    ├── .editorconfig, Directory.Build.props, docker-compose.yml, dotnet-tools.json
    ├── src/Hackathon.Api/
    └── tests/Hackathon.Tests/
```

## Jeden projekt — monolit

Backend jest **jednym projektem** z jedną bazą i jednym wdrożeniem. **Nie dzielimy go na moduły ani osobne projekty** (`.Domain.csproj`, `.Application.csproj`) z portami między nimi. Granice pilnuje test architektury na namespace'ach, a reguł pilnują encje, nie foldery.

## Układ kodu

```
backend/src/Hackathon.Api/
├── Domain/              encje, typy wartości, enumy, reguły — bez EF Core i HTTP
│   ├── DomainException.cs, NamedEnum.cs
│   ├── Workspace/       Workspace, IWorkspaceOwned
│   ├── Identity/        User, EmailAddress, PasswordPolicy
│   └── <Temat>/         np. Rooms/, Bookings/ — podfoldery tematyczne
│
├── Persistence/         DbContext, konwencje, filtr izolacji, mapowanie (…Configuration),
│   └── <Temat>/         blokada workspace'u i strażnik zapisu, seedy danych referencyjnych
│
├── Queries/             odczyty używane przez wiele handlerów (…Query)
│   └── <Temat>/
│
├── Features/            przypadki użycia — folder na zasób
│   ├── Register/        ┐
│   ├── SignIn/          │ czynności konta i sesji — jedyne foldery nazwane czynnością
│   ├── SignOut/         ┘
│   ├── Rooms/           kontroler, żądania, odpowiedzi, handlery, konwerter zasobu
│   └── Bookings/
│
├── Infrastructure/      mechanika żądania: filtr wyjątków, middleware, cookie, CurrentUser,
│                        kontekst workspace'u, zegar
│
├── Migrations/          migracje EF Core — jedna historia
├── GlobalUsings.cs
└── Program.cs           jedyne miejsce, które składa całość (DI, middleware)
```

### `Domain/` — encje i reguły

**Reguła rozstrzygająca:** jeżeli coś opisuje, **jakie stany są dozwolone albo jak system liczy**, należy do `Domain/`. Jeżeli opisuje, **co się dzieje po kliknięciu**, należy do `Features/`.

- **Encja pilnuje własnego stanu i tego, z czego się składa.** Warunek w rodzaju „zarchiwizowanej sali nie da się zarezerwować” albo „koniec rezerwacji jest po jej początku” jest metodą encji, nie `if` w handlerze. Składową jest też encja, na którą wskazuje nawigacja: rezerwacja sprawdza swoją salę, bo ją ma (`Booking.Room`).
- **Encja nie przyjmuje faktów spoza siebie jako flag.** Parametr `bool hasBookings` nie jest walidacją: encja nie może go sprawdzić, a `if (flaga) throw` tylko udaje, że pilnuje reguły. Za dane odpowiada handler, który je wczytuje.
- **Regułę łączącą encję z danymi spoza niej** rozstrzyga:
  - **handler**, jeśli potrzebuje jej tylko on — warunek z `DomainException` tuż po wczytaniu danych;
  - **nazwany typ reguły w `Domain/`**, jeśli potrzebuje jej kilka handlerów. Typ dostaje dane, na których decyduje, i da się go przetestować bez bazy. Przykład: `RoomSchedule` dostaje rezerwacje sali z danego dnia i rozstrzyga, czy termin jest wolny — używają go `CreateBookingHandler` i `RescheduleBookingHandler`;
  - **serwis domenowy w `Domain/`**, jeśli reguła należy do czynności, którą wykonuje wiele handlerów — serwis wykonuje całą czynność: metodę encji i reguły ważące ją względem innych agregatów.
- **Za spójność na zewnątrz encji odpowiada ten, kto woła jej metodę.** Encja nie tworzy innego agregatu ani nie pośredniczy w jego walidacji.
- **Dane, z których encja liczy własną treść, mogą być parametrem jej metody** — to obliczenie tego, czym encja jest, a nie sprawdzenie cudzego faktu.
- **Stan encji zmienia się wyłącznie przez jej metody.** Właściwości mają `private set`, konstruktor jest prywatny (dla EF), encja powstaje przez statyczną metodę fabryczną o nazwie czynności (`Booking.Book`, `Room.Create`).
- **Domena nie czyta bazy.** Dane referencyjne (słowniki) leżą w tabelach wypełnianych seedem w migracji; encja sięga do nich przez nawigację, a nie przez plik.
- **Kryterium, które musi być warunkiem zapytania** (np. które rezerwacje są aktywne) jest zapisane **raz**, w `Domain/`, jako `Expression<Func<T, bool>>` przy encji, i używane przez wszystkie zapytania. Dwa zapytania z tym samym warunkiem przepisanym ręcznie to zduplikowana reguła.

### `Persistence/` — baza

`HackathonDbContext` z `DbSet` dla wszystkich tabel, globalne konwencje (precyzja, enumy jako tekst, klucze, daty), filtr izolacji, mapowanie encji (`…Configuration : IEntityTypeConfiguration<T>`), konwertery wartości, seedy danych referencyjnych oraz blokada workspace'u ze strażnikiem zapisu. `Domain/` nie zna EF Core, więc mapowanie nie leży obok encji.

**`Persistence/` opisuje, jak model leży w bazie — nie odpowiada na pytania.** Zapytań tu nie ma.

### `Queries/` — odczyty wspólne

- **Każdy odczyt ma postać `…Query`**: klasa wstrzykiwana przez DI, która dostaje `HackathonDbContext` w konstruktorze. Bez klas statycznych z kontekstem w parametrze i bez innych przyrostków (`…Lookup`, `…Reader`, `…Repository`).
- **Zapytanie czyta, nie rozstrzyga.** Zwraca dane, na których decyduje reguła (np. `RoomSchedule` z rezerwacjami), a nie gotową odpowiedź „czy wolno”. Wynik jest typem z `Domain/` albo nazwanym rekordem obok zapytania.
- **Odczyt jednego handlera zostaje w handlerze.** Lista, szczegóły, wczytanie encji do zmiany to część przypadku użycia, nie zapytanie wspólne.

### `Features/` — przypadki użycia

**Jeden folder funkcji zawiera wszystko, czego ta funkcja potrzebuje** — kontroler, żądania, odpowiedzi, handlery i konwerter. Zmiana jednej funkcji dotyka jednego folderu.

- **Folder obejmuje jeden zasób i jego operacje**, nie jedną czynność: utworzenie, lista, zmiana i usunięcie rezerwacji leżą w `Features/Bookings/`. Nazwa to rzeczownik zasobu w liczbie mnogiej, nazwa encji, nie słowo z UI. Wyjątek: `Register`, `SignIn`, `SignOut`.
- **Zasób o wielu typach dzieli się na podfoldery typów** (np. `Payments/Card/`, `Payments/Transfer/`), z własnym kontrolerem, żądaniami i handlerami. W folderze nadrzędnym leży to, co wspólne dla typów (lista, szczegóły, usunięcie, `Converters/` z odpowiedzią wspólną). **Podfolder może sięgać do folderu nadrzędnego, nie do rodzeństwa.**
- **Foldery funkcji nie współdzielą kodu między sobą** — wspólne idzie do `Domain/` albo `Queries/`.
- **Przypadek użycia, który zapisuje encje z kilku obszarów, to jeden handler** — tworzy obie encje przez ich metody i zapisuje je w jednej transakcji. Bez pośrednika i bez portów.

Pliki w folderze funkcji:

| Plik | Nazwa |
|---|---|
| Kontroler | `<Zasób w l. mn.>Controller` — `BookingsController` |
| Handler | `<Czynność><Zasób>Handler` — `CreateBookingHandler`, `ListBookingsHandler`, `GetBookingHandler`, `UpdateRoomHandler`, `CancelBookingHandler`, `DeleteRoomHandler` |
| Żądanie | nazwa handlera z `Request` zamiast `Handler` — `CreateBookingRequest` |
| Odpowiedź | `<Zasób>Response` — `BookingResponse`; wspólna dla handlerów zasobu |
| Konwerter | `<Zasób>Converter` — klasa statyczna z metodami rozszerzającymi `ToResponse()` |

Czasowniki: `Create`, `Get`, `List`, `Update`, `Delete` plus czasowniki domenowe, gdy czynność nie jest zwykłą edycją (`Archive`, `Cancel`, `Reschedule`, `Approve`).

### Handler, żądanie, konwersja

- **Jeden handler = jedna czynność na jednym typie.** Jedna publiczna metoda `HandleAsync`. Handler nie obsługuje kilku typów przez wspólny interfejs ani flagi.
- **Kontroler tylko tłumaczy HTTP i woła handler — także przy odczycie.** Kontroler nie dostaje `DbContext` i nie czyta danych sam.
- **Każdy handler przyjmuje własny typ żądania** o nazwie zgodnej z nazwą handlera. Żądanie nie ma pól, których handler nie czyta. Parametry trasy (`{bookingId}`) idą osobnym argumentem `HandleAsync`.
- **Żądanie opisuje intencję, nie gotowy zapis.** Formularz podaje to, co wie użytkownik; to, co wynika z reguł (status, wyliczone pola), nadaje domena.
- **Konwersja jednego typu na drugi to metoda rozszerzająca w klasie `…Converter`** wewnątrz folderu funkcji. Konwersja przepisuje i parsuje pola; reguły należą do `Domain/`. Globalnego katalogu konwersji nie ma.
- **Workspace i użytkownik nie są parametrami handlera.** Mówią, kto pyta, a nie o co — handler bierze je z `CurrentUser`.
- **Zapytania nie filtrują ręcznie po `WorkspaceId`.** Ręczny warunek dublowałby filtr i sugerował, że bez niego dane wyciekną. Po identyfikatorze czyta się tylko encje bez filtra (`Workspace`, `User`) i kod działający przed zalogowaniem.
- **Handler czyta się bez skakania po plikach.** Wczytanie i zapis piszemy w każdym handlerze, nawet kosztem powtórzeń; handler nie woła innego handlera. Zduplikowane reguły biznesowe — nie; zduplikowane „wczytaj/zapisz” — tak.
- **`HandleAsync` czyta się jak spis kroków.** Długi, zamknięty blok (sprawdzenie warunków przed zapisem, zmiana kilku pól naraz) trafia do prywatnej metody tego samego handlera nazwanej tym, co robi (`EnsureNoOverlapAsync`, `ApplyChanges`). Pojedynczego wywołania nie opakowujesz w metodę.

### Styl C#, którego pilnujemy

| Reguła | Dobrze | Źle |
|---|---|---|
| `var` tylko z typem widocznym w linii | `var booking = Booking.Book(...);` · `var room = new Room(...);` | `var now = clock.UtcNow;` · `foreach (var booking in bookings)` · `var count = 0;` |
| Wynik wywołania do nazwanej zmiennej | `string hash = passwords.HashPassword(user, password);`<br>`user.AssignPasswordHash(hash, now);` | `user.AssignPasswordHash(passwords.HashPassword(user, password), now);` |
| `await` też do zmiennej | `List<Room> rooms = await query.ToListAsync(ct);`<br>`return rooms.Select(...)` | `return (await query.ToListAsync(ct)).Select(...)` |
| Metody w klamrach | `public Task<X> Get(...) { return handler.HandleAsync(...); }` | `public Task<X> Get(...) => handler.HandleAsync(...);` |
| `=>` zostaje | liczona właściwość `public bool IsActive => Status == ...;`, lambdy | — |
| Bez krotek | `return new RoomAvailability(isFree, nextFreeAt);` | `return (isFree, nextFreeAt);` |
| Jeden typ na plik | `BookingStatus.cs`, `Booking.cs` | enum w pliku encji |
| Klamry zawsze | `if (x) { throw ...; }` | `if (x) throw ...;` |
| Pełne nazwy | `booking`, `cancellationToken`, `databaseTransaction` | `b`, `ct`, `tx` |

Wyjątki od reguły o zagnieżdżonych wywołaniach: lambda w zapytaniu EF (drzewo wyrażeń nie przyjmuje instrukcji), inicjalizator pola statycznego, `nameof`. `new` w argumencie zostaje, ale wywołania w jego argumentach też idą do zmiennych. Lambda wykonywana w pamięci, która potrzebuje wyniku wywołania, dostaje ciało w klamrach ze zmienną. Wywołanie w gałęzi `?:` przerabiasz na `if` z wczesnym `return`.

## Równoczesne zapisy

Walidacje zapisu działają według schematu „odczytaj stan, sprawdź, zapisz”: „termin sali jest wolny”, „limit miejsc nie przekroczony”. Dwa równoczesne zapisy mogłyby przejść sprawdzenie na tym samym stanie i razem złamać regułę. Chroni przed tym **blokada workspace'u**.

**Handler, który zapisuje encję workspace'u** (`IWorkspaceOwned`), zaczyna od otwarcia transakcji i wzięcia blokady — **przed pierwszym odczytem**, na którym opiera się walidacja — a kończy commitem:

```csharp
await using IDbContextTransaction databaseTransaction = await db.Database.BeginTransactionAsync(cancellationToken);
await db.LockWorkspaceAsync(currentUser.WorkspaceId, cancellationToken);

// read, let the domain decide, SaveChanges

await databaseTransaction.CommitAsync(cancellationToken);
```

- **Blokada to `pg_advisory_xact_lock`** z kluczem wyliczonym z identyfikatora workspace'u. Drugi zapis tego samego workspace'u czeka na commit albo rollback pierwszego; inne workspace'y nie czekają. Blokada zwalnia się sama razem z transakcją.
- **O blokadzie decyduje to, co handler zapisuje**, nie metoda HTTP. Handler tylko czytający nie bierze blokady. Handler zapisujący wyłącznie encje spoza workspace'u (użytkownik przy logowaniu) też nie.
- **Rejestracja blokuje workspace, który właśnie tworzy** — jego identyfikator jest znany przed zapisem.
- **Strażnik zapisu** (interceptor `SaveChanges` w `Persistence/`) odrzuca zapis encji workspace'u, jeżeli bieżąca transakcja nie trzyma blokady tego samego workspace'u. Brakująca blokada wychodzi w pierwszym teście handlera.
- **Unikalność łapana w bazie zamienia się na 409**: `catch (DbUpdateException exception) when (PostgresErrors.IsUniqueViolation(exception))`.

**Odrzucone:** izolacja `SERIALIZABLE` (wymaga ponawiania), blokada w pamięci procesu (nie działa przy kilku instancjach), wersja wiersza (równoczesny zapis kończy się błędem zamiast poczekać), dekorator wokół handlerów, który brałby blokadę niewidocznie dla czytającego.

## `Infrastructure/` — mechanika żądania

Filtr zamieniający `DomainException` na odpowiedź, middleware zakresu workspace'u, cookie logowania, `CurrentUser`, kontekst workspace'u dla filtra izolacji, zegar. Nie trafiają tu encje ani przypadki użycia.

## Namespace

- **Wspólny dla katalogu w korzeniu:** `Hackathon.Api.Domain`, `Hackathon.Api.Persistence`, `Hackathon.Api.Queries`, `Hackathon.Api.Infrastructure`. Podfoldery organizują pliki, nie tworzą namespace'ów (typ o nazwie folderu dawałby `CS0118`). Te cztery są w `GlobalUsings.cs`.
- **W `Features/` namespace odpowiada folderowi** (`Hackathon.Api.Features.Bookings`) — tu folder jest granicą, a test sprawdza ją po namespace'ach.

## Kierunek zależności

| Kto | Może używać | Nie może używać |
|---|---|---|
| `Domain/` | wyłącznie siebie i .NET | EF Core, ASP.NET Core, `Persistence/`, `Queries/`, `Infrastructure/`, `Features/` |
| `Persistence/` | `Domain/`, EF Core | `Queries/`, `Infrastructure/`, `Features/` |
| `Queries/` | `Domain/`, `Persistence/`, EF Core | `Infrastructure/`, `Features/` |
| `Infrastructure/` | `Domain/`, `Persistence/`, ASP.NET Core | `Queries/`, `Features/` |
| `Features/` | wszystkiego powyżej | innego folderu funkcji; podfolder — rodzeństwa |
| `Program.cs` | wszystkiego | — |

Wyjątek techniczny: `DomainException` przyjmuje kod HTTP jako `int` ze stałych `StatusCodes` — stała wkompilowuje się jako liczba, więc `Domain/` nie zależy od ASP.NET Core w IL.

## Konwencje — skrót

| Rzecz | Zasada |
|---|---|
| Klucze | `Guid` z `Guid.CreateVersion7()`, ustawiany w fabryce encji |
| Kwoty | `decimal`, precyzja z globalnej konwencji |
| Enumy | `UPPER_SNAKE_CASE`, w bazie tekst nazwy; w żądaniu `string` parsowany przez `NamedEnum` |
| Daty biznesowe | `DateOnly` → `date`, bez stref |
| Znaczniki techniczne | `DateTimeOffset` → `timestamptz`, UTC, z `IClock` |
| Nazwy w bazie | domyślne EF Core |
| Migracje | `dotnet dotnet-ef migrations add <Nazwa>` — znacznik czasu dodaje EF; zmergowanej się nie edytuje |
| Testy | `tests/Hackathon.Tests`, foldery lustrzane względem kodu |
| JSON | `camelCase` (domyślny ASP.NET Core), enumy jako tekst |
