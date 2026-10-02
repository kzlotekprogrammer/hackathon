# AGENTS.md — Hackathon

Instrukcje dla agenta AI pracującego z tym repozytorium. Claude Code wczytuje je przez [`CLAUDE.md`](CLAUDE.md), inne narzędzia czytają ten plik bezpośrednio.

> **Nazwa robocza `Hackathon`** (`Hackathon.Api`, `Hackathon.Tests`, `HackathonDbContext`) jest do zamiany jednym „znajdź i zamień” na nazwę projektu. Przykłady w dokumentacji (sale i rezerwacje: `Room`, `Booking`) pokazują wzorzec i nie są tematem projektu.

## Repozytorium

| Ścieżka | Czym jest |
|---|---|
| [`docs/product.md`](docs/product.md) | Co budujemy: cel, zasoby, przypadki użycia, słownik nazw. **Źródło prawdy dla „co”.** |
| [`docs/architecture/`](docs/architecture/) | Jak budujemy: stack, struktura, konwencje, wzorce kodu. **Źródło prawdy dla „jak”. Rozstrzygnięte.** |
| [`docs/assumptions.md`](docs/assumptions.md) | Dziennik założeń przyjętych przez agenta tam, gdzie `product.md` milczy. |
| `backend/src/Hackathon.Api/` | Kod API — jeden projekt. |
| `backend/tests/Hackathon.Tests/` | Testy, foldery lustrzane względem kodu. |

## Zanim napiszesz pierwszą linijkę

1. [`docs/architecture/00-stack.md`](docs/architecture/00-stack.md) — stack, klucze, liczby, enumy, daty, izolacja danych, uwierzytelnianie.
2. [`docs/architecture/01-structure-conventions.md`](docs/architecture/01-structure-conventions.md) — układ katalogów, kierunek zależności, handler, zapytania, blokada zapisu.
3. [`docs/architecture/02-code-patterns.md`](docs/architecture/02-code-patterns.md) — wzorce plików, które piszesz codziennie: encja, mapowanie, handler, kontroler, zapytanie, testy.
4. [`docs/architecture/03-skeleton.md`](docs/architecture/03-skeleton.md) — szkielet budowany raz na starcie: `DbContext`, konwencje, izolacja, logowanie, fixture testów.
5. [`docs/product.md`](docs/product.md) — zasób albo przypadek użycia, nad którym pracujesz.

To, co opisuje `docs/architecture/`, jest rozstrzygnięte. Nie wybierasz tego ponownie i nie proponujesz alternatyw bez prośby — także wtedy, gdy alternatywa byłaby „szybsza na hackathon”.

## Tempo i zakres

- **Pracujesz pionowymi wycinkami.** Jeden zasób albo jeden przypadek użycia od końca do końca: encja → mapowanie → migracja → handler → kontroler → test. Następny wycinek zaczynasz, gdy poprzedni się buduje i przechodzi testy. Nie rozgrzebujesz kilku funkcji naraz.
- **Działający mniejszy zakres jest lepszy od rozgrzebanego większego.** Po każdym wycinku `dotnet build` i `dotnet test` są zielone.
- **Nie dokładasz rzeczy, o które nikt nie prosił:** reset hasła, potwierdzenie e-mail, role, 2FA, paginacja, cache, kolejki, soft delete, wersjonowanie API — nawet gdy framework daje je gotowe. Jeśli uważasz, że czegoś brakuje, mówisz o tym, zamiast to budować.
- **Tempo nie zwalnia z reguł kodu.** Skrót łamiący architekturę (kontroler z `DbContext`, encja z publicznymi setterami, `?? 0` na wartości biznesowej) kosztuje więcej przy debugowaniu przed demem, niż oszczędza.

## Luki w wymaganiach

- **Pytanie o rdzeń modelu** — co zapisujemy, jakie stany są dozwolone, kto widzi czyje dane — zatrzymuje pracę nad tym wycinkiem. Wypisz pytanie i zapytaj użytkownika. Kod napisany na domyśle o modelu zostałby wyrzucony.
- **Drobną lukę** zamykasz rozsądnym założeniem i dopisujesz je do [`docs/assumptions.md`](docs/assumptions.md): co założono, dlaczego, gdzie w kodzie. Domysłu nie wpisujesz do `product.md` jako reguły.
- **Nazwy encji, pól i wartości enum bierzesz ze słownika w `product.md`.** Brakującą nazwę dopisujesz do słownika i mówisz o tym w odpowiedzi — nie wymyślasz synonimów w kodzie.

## Reguły generowania kodu

Szczegóły i uzasadnienia są w `docs/architecture/`. Tu skrót do sprawdzenia przed oddaniem kodu.

**Struktura**

- **Pięć katalogów w korzeniu projektu:** `Domain/`, `Persistence/`, `Queries/`, `Features/`, `Infrastructure/` (plus `Migrations/`). Nie ma globalnych folderów `Controllers`, `Services`, `Repositories`, `Helpers`, `Utils`.
- **Kierunek zależności jest jeden:** `Features` → `Queries` → `Persistence` → `Domain`. `Domain/` nie zna EF Core ani ASP.NET Core. Foldery funkcji nie współdzielą kodu między sobą; wspólne idzie do `Domain/` albo `Queries/`. Pilnuje tego test architektury.
- **Nazwy w kodzie to nazwy encji, nie słowa z UI.** Folder funkcji to rzeczownik zasobu w liczbie mnogiej (`Features/Bookings/`), bez `Manage`/`Add`/`Record`.

**Encje i reguły**

- **Encja pilnuje własnego stanu i tego, z czego się składa** (także encji, do której ma nawigację). Stan zmienia się wyłącznie przez jej metody — bez publicznych setterów.
- **Encja nie przyjmuje faktów spoza siebie jako flag** (`bool hasBookings`). Regułę łączącą encję z danymi spoza niej rozstrzyga handler, jeśli potrzebuje jej tylko on, albo nazwany typ reguły w `Domain/`, jeśli potrzebuje jej kilka handlerów.
- **Domena nie czyta bazy.** Czego potrzebuje, wczytuje handler.
- **Odmowa biznesowa to `DomainException`** (z kodem HTTP, domyślnie 400). **Błąd programisty** — np. pusty identyfikator, którego nie miało prawa być — to `InvalidOperationException` (500).

**Handler, żądanie, kontroler**

- **Jeden handler = jedna czynność na jednym typie**, z własnym typem żądania o zgodnej nazwie: `CreateBookingHandler` ← `CreateBookingRequest`. Bez wspólnych interfejsów handlerów, bez flag prowadzących kilka typów jedną ścieżką, bez MediatR.
- **Kontroler tylko tłumaczy HTTP i woła handler** — także przy odczycie. Kontroler nie dostaje `DbContext`.
- **Handler sam mapuje żądanie.** Konwersja typu na typ to metoda rozszerzająca w klasie `…Converter` wewnątrz folderu funkcji. Metoda wyglądająca na konwersję nie sięga do bazy.
- **Handler nie przyjmuje `workspaceId` ani `userId`** — bierze je z `CurrentUser`. Zapytania nie filtrują ręcznie po `WorkspaceId`; izolację daje globalny filtr EF.
- **Handler zapisujący encję workspace'u otwiera transakcję i woła `db.LockWorkspaceAsync(currentUser.WorkspaceId, …)` przed pierwszym odczytem**, a po `SaveChanges` robi commit. Strażnik przy `SaveChanges` odrzuca zapis bez blokady.
- **Zmiana:** `PATCH` + `Update…Handler` dla niezależnych atrybutów — pola żądania nullowalne, **`null` znaczy „nie zmieniaj”**. `PUT` + `Revise…Handler` (albo `POST /{id}/<czynność>` z domenowym czasownikiem) dla treści sprawdzanej w całości — pola wymagane, brak to 400.
- **Usuwanie:** encja, na którą coś wskazuje, jest archiwizowana (`Archive…Handler`, status `ARCHIVED`), nie usuwana. Twarde `Delete` ma tylko liść. Gdy zależności blokują czynność, handler sprawdza je sam i odmawia z 409; naruszenia klucza obcego (23503) nie łapiesz — to znak brakującego sprawdzenia.
- **Czas bierzesz z `IClock`**, nie z `DateTimeOffset.UtcNow`.
- **Odczyt używany przez kilka handlerów to `…Query` w `Queries/`** — wstrzykiwana klasa, która czyta, a nie rozstrzyga. Odczyt jednego handlera zostaje w tym handlerze.
- **Każdy handler, zapytanie i serwis rejestrujesz jawnie w `Program.cs`** (`AddScoped<CreateBookingHandler>()`). Bez skanowania assembly.

**Kod czyta się sam**

- **Nie twórz klas pomocniczych ani statycznych wrapperów**, które tylko oszczędzają linijki i chowają prawdziwe wywołanie. Współdzielone są tylko reguły. Zanim dodasz nowy typ, sprawdź, czy domena już go nie ma.
- **Handler woła zapytania wprost, z nazwanymi zmiennymi**, nawet jeśli kilka linijek się powtórzy. Długi, zamknięty blok idzie do prywatnej metody tego samego handlera, żeby `HandleAsync` czytało się jak spis kroków. Pojedynczego wywołania nie opakowujesz w metodę.
- **`var` tylko tam, gdzie typ widać w tej samej linii** (`new`, rzutowanie, `as`, fabryka tego samego typu). Wynik metody, `await`, `foreach`, `out`, literał — typ jawnie. Pilnuje `.editorconfig` (IDE0008).
- **Bez zagnieżdżonych wywołań.** Wynik wywołania, także `await`, najpierw trafia do nazwanej zmiennej z jawnym typem, a dopiero potem do argumentu. `new` w argumencie zostaje. Wyjątki: lambda w zapytaniu EF, inicjalizator pola statycznego, `nameof`.
- **Metody w klamrach, także jednolinijkowe.** `=>` tylko przy liczonych właściwościach tylko do odczytu i w lambdach. Pilnuje `.editorconfig` (IDE0021–IDE0024, IDE0061).
- **Bez krotek (`ValueTuple`).** Dwie wartości zwracane albo przekazywane razem dostają nazwany typ.
- **Jeden typ na plik, nazwa pliku = nazwa typu.** Pomocnik jednej klasy zostaje w niej jako prywatny typ zagnieżdżony. Pilnuje StyleCop (SA1402, SA1649).
- **Zasób potrzebny w `catch` deklaruj przed `try`.** `await using` wewnątrz `try` zamyka zasób przed wejściem do `catch`.
- Komentarze i komunikaty błędów po angielsku, zawsze klamry, pełne nazwy zmiennych (`booking`, nie `b`; `cancellationToken`, nie `ct`).

**Dane**

- **Nieznane nigdy nie staje się `0` ani pustym stringiem.** Brak wartości to `null` albo odmowa. `?? 0` i `GetValueOrDefault()` na wartościach biznesowych są błędem. Pole liczbowe w żądaniu jest nullowalne (`int? Capacity`), a handler odrzuca brak — inaczej brakujące pole w JSON przyjdzie jako `0`.
- **Reguły przekrojowe egzekwuje także baza:** klucze obce w obrębie tego samego workspace'u, unikalność, `CHECK`. Naruszenie unikalności z wyścigu zamieniasz na 409.
- **Migracje:** generuje je `dotnet dotnet-ef`, nazwa ze znacznikiem czasu. Zmergowanej migracji się nie edytuje. Wygenerowaną migrację sprawdzasz sam (oczekiwane tabele i constrainty, brak zbędnych indeksów) i krótko opisujesz, co zawiera.

**Testy**

- Kryteria z `product.md` przenosisz na testy przed implementacją albo razem z nią.
- **Reguła encji → test domeny bez bazy. Przypadek użycia → test API** na prawdziwym PostgreSQL przez Testcontainers. Każdy zasób dostaje test „cudzy workspace dostaje 404”.
- Nazwa testu jest zdaniem o zachowaniu: `A_slot_taken_by_another_booking_is_refused_with_a_conflict`.

## Definicja ukończenia wycinka

`dotnet build` bez ostrzeżeń stylu i `dotnet test` przechodzą czysto, migracja wykonuje się na pustej bazie, endpoint jest widoczny w Scalar (`/scalar/v1`), założenia są w `docs/assumptions.md`.

## Komendy

Z katalogu `backend/`:

```bash
docker compose up -d                       # PostgreSQL 17 lokalnie
dotnet build
dotnet test
dotnet dotnet-ef migrations add <Nazwa> --project src/Hackathon.Api
dotnet dotnet-ef database update --project src/Hackathon.Api
dotnet run --project src/Hackathon.Api     # Scalar: /scalar/v1
```

`dotnet dotnet-ef` (dwa razy „dotnet”) woła lokalne narzędzie z `backend/dotnet-tools.json`. Samo `dotnet ef` może sięgnąć po globalne narzędzie w innej wersji.

## Git

- Commitujesz i pushujesz tylko na prośbę użytkownika.
- Przed pierwszym commitem w sesji: `git fetch` i sprawdzenie, czy bieżąca gałąź nie jest już scalona do `origin/main`. Jeśli jest, zakładasz nową gałąź od `origin/main`.
- Jeden wycinek = jeden commit albo kilka małych; komunikat mówi, co działa, a nie co ruszono.

## Współpraca

- Odpowiadasz po polsku; artefakty techniczne i komentarze w kodzie po angielsku.
- Nie zadajesz pytań, na które odpowiedź jest w `docs/`.
- **Pytanie to nie decyzja.** Gdy użytkownik pyta „dlaczego” albo kwestionuje rozwiązanie, odpowiadasz i wypisujesz, co jest do rozstrzygnięcia. Zmiany robisz dopiero, gdy wybierze kierunek.
- Gdy reguła jest w dwóch miejscach i mówi dwie różne rzeczy, zgłaszasz to, zamiast wybrać wygodniejszą wersję.
