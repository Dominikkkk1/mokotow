# Integracja sceny UE ze stroną - specyfikacja

Dokument dla zespołu Unreal Engine. Opisuje jedyną wiadomość, którą scena musi
wysłać, żeby odwiedzający mógł zapytać o konkretne mieszkanie, oraz zdarzenia
analityczne, z których strona już korzysta.

## Zasada podziału

**Scena decyduje KIEDY zapytać. Strona decyduje JAK wygląda pytanie.**

Formularz jest i pozostaje elementem HTML na stronie. Nie buduj pól tekstowych
wewnątrz sceny - w pixel streamingu pole tekstowe jest fragmentem obrazu wideo,
przez co:

- przeglądarka nie podpowie zapisanego adresu (brak autouzupełniania),
- na telefonie klawiatura ekranowa zasłania widok, a znaki gubią się przy lagu,
- tekst jest rozmyty, bo strumień jest kompresowany pod ruch, nie pod czcionkę,
- zerwanie strumienia kasuje wypełniony formularz, a StreamPixel zrywa je
  samoczynnie po 2,5 minuty bez ruchu,
- nie da się kliknąć w link do polityki prywatności ani rzetelnie odebrać zgody,
- dane i tak muszą wrócić do przeglądarki, żeby trafić do CRM - droga przez
  scenę jest dłuższa, nie krótsza.

## Wiadomość: otwórz formularz kontaktowy

Scena wysyła do rodzica (strony) przez `postMessage`:

```json
{ "type": "contact-form", "apartmentNumber": "13" }
```

| pole | typ | wymagane | opis |
| --- | --- | --- | --- |
| `type` | string | tak | zawsze `contact-form` |
| `apartmentNumber` | string | nie | numer mieszkania widoczny dla użytkownika, np. `"13"`. Pominięty lub pusty = zapytanie ogólne o inwestycję |

Strona po odebraniu tej wiadomości podnosi formularz z wybranym mieszkaniem,
a po wysłaniu tworzy w CRM leada przypiętego do tego lokalu.

Dopuszczalny jest zarówno obiekt, jak i string z JSON-em - strona normalizuje
oba warianty.

### Gdzie umieścić wyzwalacz

Naturalne miejsca po stronie sceny:

- przycisk **„Zapytaj o ten apartament"** w interfejsie mieszkania,
- zakończenie sekwencji prezentacyjnej mieszkania,
- kliknięcie w element wymagający kontaktu z handlowcem (np. „sprawdź dostępność").

Nie wysyłaj tej wiadomości automatycznie po upływie czasu - stroną steruje
własna logika zaproszeń opisana niżej i podwójna zaczepka byłaby natrętna.

## Zdarzenia analityczne, które strona już czyta

Scena wysyła je jako `{ "type": "analytics", ... }`. Strona rozkłada je na
zdarzenia CRM. Obsługiwane `actionType`:

| `actionType` | znaczenie | wykorzystywane pola |
| --- | --- | --- |
| `LocationEnter` | wejście do pomieszczenia | `locationName`, `apartmentNumber`, `elapsedSeconds` |
| `LocationLeave` | wyjście z pomieszczenia | `elapsedSeconds` |
| `PawnChange` | zmiana kamery / trybu | `pawnType`, `apartmentNumber` |
| `ApartmentBrowserFocus` | powrót do listy mieszkań | `elapsedSeconds` |

`elapsedSeconds` to liczba sekund od startu sesji, przekazywana jako string.
Z różnicy między wejściem a wyjściem strona liczy czas spędzony w pomieszczeniu
i zapisuje go przy mieszkaniu w CRM.

Nieznane wartości `actionType` są zapisywane w całości i nie są interpretowane -
można dodawać nowe bez ryzyka, że coś przestanie działać. Wymagają jednak
uzgodnienia, zanim zaczną cokolwiek znaczyć.

### Ważne przy `ApartmentBrowserFocus`

Pole `apartmentNumber` bywa tu wypełnione numerem podświetlonym na liście.
Strona celowo go ignoruje - podświetlenie na liście to nie jest obejrzenie
mieszkania i liczenie go zawyżałoby zainteresowanie w CRM.

### Bezczynność

Komunikaty własne sceny (w tym informacja o zerwaniu strumienia po braku ruchu)
nie są traktowane jako aktywność użytkownika. Jeśli scena zacznie wysyłać
osobne zdarzenie o bezczynności, prosimy o podanie jego `actionType` - wtedy
zastąpi ono stoper po stronie strony.

## Zaproszenia do kontaktu po stronie strony

Strona sama pokazuje nienachalny pasek z pytaniem, opierając się na zachowaniu,
nie na stoperze. Warunki (jednorazowo na wizytę):

- wyjście z mieszkania na listę po obejrzeniu co najmniej **2 pomieszczeń**,
- łącznie **90 sekund** w jednym mieszkaniu,
- zerwanie strumienia.

Pasek nie pojawi się, gdy formularz jest już otwarty albo został wysłany.

## Kontakt

Pytania do integracji: Codelivery.
