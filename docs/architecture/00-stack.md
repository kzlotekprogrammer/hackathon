# Stack techniczny

Decyzje obowiązujące. Nie są propozycją i nie wymagają potwierdzenia przed użyciem.

| Warstwa | Wybór |
|---|---|
| Baza danych | **PostgreSQL 17** (obraz `postgres:17`) |
| Backend | **.NET 10**, C#, `Nullable` i `ImplicitUsings` włączone; SDK przypięty do 10.0.x w `backend/global.json` |
| ORM i migracje | **EF Core** + `Npgsql.EntityFrameworkCore.PostgreSQL`; narzędzie `dotnet-ef` lokalnie w `backend/dotnet-tools.json` |
| API | **REST — kontrolery**, nie Minimal API |
| Testy | **xUnit**; testy integracyjne na prawdziwym PostgreSQL przez **Testcontainers**; granice architektury przez **NetArchTest** |
| Środowisko lokalne | **Docker Compose** z `postgres:17` |
| Dokumentacja API | **OpenAPI** (`Microsoft.AspNetCore.OpenApi`) + UI **Scalar**, wyłącznie w `Development` |
| Styl kodu | `.editorconfig` + `EnforceCodeStyleInBuild` + StyleCop (tylko SA1402, SA1649) — błędy stylu wychodzą w `dotnet build` |

Bez MediatR, AutoMappera, Swashbuckle, repozytoriów generycznych i ASP.NET Core Identity jako frameworka.

## Klucze główne

**`Guid` we wszystkich encjach**, generowany przez **`Guid.CreateVersion7()`**, nie `Guid.NewGuid()`.

- Klucz jest znany **przed** zapisem, więc encja i jej dzieci powstają razem w jednym `SaveChanges` bez zapisów pośrednich.
- UUID v7 ma znacznik czasu w części wiodącej — kolejne klucze są rosnące i nie rozrzucają wstawek po indeksie.
- Klucz ustawia domena w metodzie fabrycznej encji. W modelu EF wszystkie klucze `Guid` mają globalnie `ValueGenerated.Never` — ani EF, ani baza nie generują identyfikatora za aplikacją.

## Liczby

| Rodzaj wartości | Typ w C# | Typ w bazie |
|---|---|---|
| Liczba całkowita (sztuki, pojemność, kolejność) | `int` / `long` | `integer` / `bigint` |
| Kwota pieniężna | `decimal` | `NUMERIC(19,4)` — globalna konwencja |
| Wartość o innej skali (kurs, procent, ilość ułamkowa) | **własny typ wartości** zarejestrowany w konwencji | `NUMERIC(19,8)` |

`float`, `double` i `real` nie występują w wartościach biznesowych w żadnej postaci.

> **Precyzję ustawia globalna konwencja, nie atrybut na encji.** EF Core bez jawnej konfiguracji potrafi zmapować `decimal` na węższą kolumnę i **po cichu obciąć wartość**. Konwencja rozróżnia skalę **po typie CLR**, nie po nazwie pola: goły `decimal` to kwota, inna skala wymaga własnego typu. Konwencję pokrywa test zapisujący i odczytujący wartość o pełnej precyzji.

**Zaokrąglanie:** mnożenie w pełnej precyzji, zaokrąglenie **raz, na końcu**, trybem `MidpointRounding.AwayFromZero`. Wartość podana przez użytkownika z nadmiarem miejsc jest błędem wejścia — odrzucamy ją, nie zaokrąglamy po cichu. Sumy pokazywane użytkownikowi liczymy z tych samych zaokrąglonych wartości, które widzi w wierszach.

## Enumy w bazie

**Enum zapisujemy jako tekst nazwy wariantu** (`ACTIVE`, `ARCHIVED`), nigdy jako liczbę porządkową. Ustawia to globalna konwencja — nowy enum jest objęty automatycznie.

- bazę czyta człowiek — `'ARCHIVED'` jest samoopisujące, `2` nie jest;
- `CHECK`-i i indeksy częściowe pisze się przeciw tym wartościom;
- przestawienie kolejności wariantów przy ordinalach to cicha zmiana znaczenia danych.

**Konsekwencja:** nazwy wariantów są kontraktem bazy. Przemianowanie wariantu wymaga migracji.

Warianty enumów piszemy `UPPER_SNAKE_CASE` — tak samo w C#, w bazie i w JSON. Żądanie przenosi enum jako `string`, a handler parsuje go przez `NamedEnum.TryParse`, który odrzuca liczby (`"1"`).

## Daty i czas

| Rodzaj | Typ w C# | Typ w bazie |
|---|---|---|
| Data biznesowa (dzień rezerwacji, termin) | `DateOnly` | `date` — **bez konwersji stref** |
| Godzina biznesowa | `TimeOnly` | `time` |
| Znacznik techniczny (`CreatedAt`, `UpdatedAt`) | `DateTimeOffset` | `timestamptz`, w UTC |

Bieżący czas daje `IClock` (`Infrastructure/`), nie `DateTimeOffset.UtcNow` — test może go ustawić.

## Nazewnictwo w bazie

**Obowiązują domyślne konwencje EF Core.** Nazwy tabel i kolumn wynikają z nazw typów i właściwości w C#. Nie konfigurujemy `snake_case` ani własnego schematu nazw.

Nazwy constraintów i indeksów, które mają znaczenie biznesowe, nadajemy jawnie (`IX_Bookings_OnePerRoomAndSlot`, `CK_Rooms_Capacity_Positive`), żeby było widać, którą regułę łamie zapis.

## Izolacja danych między workspace'ami

Każdy użytkownik pracuje we własnym **workspace** — technicznym kontenerze swoich danych. Każde zapytanie musi być ograniczone do workspace'u zalogowanego użytkownika. Jedno pominięte ograniczenie oznacza pokazanie komuś cudzych danych.

**Rozwiązanie: globalny filtr EF Core.** Regułę deklarujemy raz, przy konfiguracji modelu, i EF dokłada warunek do każdego zapytania. Nikt go nie pisze ręcznie, więc nikt nie może go zapomnieć.

- Encje objęte izolacją implementują znacznik `IWorkspaceOwned` (`Guid WorkspaceId { get; }`).
- Filtr buduje się w wyrażeniu zaczepionym o **instancję `DbContext`** (kontekst implementuje `IWorkspaceContext`). Nie wolno zaczepiać go o wstrzyknięty `IWorkspaceContext` osobno: EF cache'uje model raz na typ kontekstu i zamroziłby workspace pierwszego żądania — wyciek danych.
- Test przechodzi po wszystkich encjach: każda jest albo `IWorkspaceOwned`, albo jawnie wpisana na listę współdzielonych z powodem. Nowa encja bez znacznika wywala test, a nie produkcję.
- Klucz obcy do encji tego samego workspace'u jest złożony (`WorkspaceId` + `Id`), żeby baza nie pozwoliła wskazać cudzego wiersza.

> Jeśli produkt nie ma danych prywatnych użytkownika ani zespołu — wszystko jest wspólne — ten mechanizm się pomija, razem z blokadą workspace'u z [`01-structure-conventions.md`](01-structure-conventions.md). To decyzja zespołu przed pierwszą encją, nie w trakcie.

## Uwierzytelnianie

**Minimalne.** Rejestracja, logowanie, wylogowanie, hasło wyłącznie jako bezpieczny hash, dostęp wyłącznie do własnego workspace'u.

| Element | Wybór |
|---|---|
| Hashowanie hasła | `PasswordHasher<T>` z `Microsoft.AspNetCore.Identity` — **sama klasa**, jako narzędzie |
| Sesja | cookie (`AddAuthentication().AddCookie()`), `HttpOnly`, `SameSite=Lax` |
| Odpowiedź bez sesji | `401` / `403`, nie przekierowanie na stronę logowania |
| Endpointy | `POST /auth/register`, `POST /auth/sign-in`, `POST /auth/sign-out` |
| Domyślnie | globalny `AuthorizeFilter` — każdy kontroler wymaga zalogowania, wyjątki mają `[AllowAnonymous]` |

Cookie niesie `NameIdentifier` (id użytkownika) i `workspace_id`. Middleware kopiuje `workspace_id` do zakresu żądania, z którego czyta filtr izolacji.

**Nie instalujemy ASP.NET Core Identity jako frameworka** ani JWT. Reset hasła, potwierdzenie e-mail, 2FA, role i usunięcie konta są poza zakresem, nawet jeśli framework daje je gotowe.

## Dokumentacja API

Dokument OpenAPI pod `/openapi/v1.json`, UI Scalar pod `/scalar/v1`. Oba mapowane **tylko gdy `app.Environment.IsDevelopment()`**, z jawnym `AllowAnonymous()`.

## Błędy HTTP

| Sytuacja | Wyjątek | Odpowiedź |
|---|---|---|
| Żądanie łamie regułę produktu | `DomainException` (domyślnie 400) | `{ "error": "..." }` z kodem z wyjątku |
| Zasób nie istnieje albo należy do innego workspace'u | `DomainException(..., 404)` | 404 — nie zdradzamy, że cudzy wiersz istnieje |
| Konflikt z istniejącym stanem / naruszenie unikalności | `DomainException(..., 409)` | 409 |
| Błąd programisty (pusty identyfikator, brak transakcji) | `InvalidOperationException` | 500 |

Tłumaczenie robi `DomainExceptionFilter` w `Infrastructure/`. Komunikaty błędów po angielsku, zdaniem, które powie użytkownikowi, co poprawić.
