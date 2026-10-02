# Produkt

Co budujemy. Uzupełnia zespół; agent czyta, a dopisuje tylko brakujące nazwy do słownika (i mówi o tym). Założenia agenta trafiają do [`assumptions.md`](assumptions.md), nie tutaj.

## Cel

<!-- Jedno-dwa zdania: jaki problem rozwiązujemy i dla kogo. -->

## Użytkownicy

<!-- Kto loguje się do aplikacji. Czy dane są prywatne dla użytkownika (workspace), czy wspólne dla wszystkich? -->

## Zasoby

<!-- Jeden wiersz na encję. Nazwa jest nazwą w kodzie i w API. -->

| Encja | Czym jest | Należy do workspace'u? |
|---|---|---|
| | | tak / nie (słownik wspólny) |

## Przypadki użycia

<!-- Jeden blok na przypadek użycia. Kryteria są tym, co zamieniamy na testy. -->

### UC-01 — <nazwa>

- **Kto:** 
- **Co robi:** 
- **Kryteria:**
  - [ ] 
  - [ ] 
- **Odmowy:** <!-- kiedy system odmawia i z jakim kodem: 400 / 404 / 409 -->

## Reguły

<!-- Reguły, które nie mieszczą się w jednym przypadku użycia: dozwolone stany, limity, co z czym się nie łączy. -->

## Słownik

<!-- Nazwy w kodzie: encje, pola, wartości enum (UPPER_SNAKE_CASE). Agent nie wymyśla synonimów. -->

| Nazwa w kodzie | Znaczenie | Wartości (dla enum) |
|---|---|---|
| | | |

## Poza zakresem

<!-- Czego świadomie nie budujemy na hackathonie. Agent tego nie implementuje, nawet jeśli to łatwe. -->

- reset hasła, potwierdzenie e-mail, 2FA, role
- 
