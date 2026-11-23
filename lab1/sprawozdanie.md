# Laboratorium 01 – Podstawy pracy z agentem

Repozytorium: https://github.com/Paffk0o/repozytorium

- **Aplikacja:** Claude Code 2.1.330
- **Model:** claude-opus-5-5
- **Poziom rozumowania:** medium
- **Ustawienia:** tryb uprawnień `auto` (polecenia wykonywane bez osobnego zatwierdzania każdego z nich)
- **Sesje:** [sesje/sesja-1.jsonl](sesje/sesja-1.jsonl) – całe zadanie w jednej sesji

## 1. Zlecenie

**Wynik:** raport dla `project/data/exercise.csv` zapisany w `project/results/exercise-summary.json`.

**Dozwolone zmiany:** w etapie 1 żadnych zmian w kodzie, danych ani testach. W etapie 2 zmiana etykiety jednej wiadomości i odświeżenie raportu. Pliki pomocnicze do próby z punktu 2 agent mógł tworzyć tylko w katalogu tymczasowym poza projektem.

**Odbiór:** agent miał wskazać miejsce raportu i odczytać jego treść z dysku, pokazać użyte dane, wykonane polecenie, wydruk i kod wyjścia każdego polecenia, podać wynik testu, napisać, czy polecenia wykonał mój komputer, czy inne środowisko, oraz zgłosić niewykonane kroki.

Zakres zaproponował agent po przeczytaniu `README.md`, `data/README.md` i `describe_data.py`. Przyjąłem go i dopisałem kody wyjścia oraz informację o środowisku.

## 2. Jak powstał raport

| | |
|---|---|
| Wejście | `data/exercise.csv` |
| Polecenie | `py -3 describe_data.py --data data/exercise.csv --out results/exercise-summary.json` |
| Katalog roboczy | `C:\Users\szcze\Desktop\WAT\7sem\MSI\LAB1\project` |
| Narzędzie | PowerShell w Claude Code (odczyty plików: narzędzie Read, zmiana etykiety: narzędzie Edit) |
| Środowisko | mój komputer (host PAWEŁ, Windows 11), Python 3.14.5 (`pythoncore-3.14-64\python.exe`) |
| Wynik procesu | `Report saved: ...\results\exercise-summary.json`, `Messages: 8; labels: {'ham': 4, 'spam': 4}`, kod wyjścia 0 |
| Raport końcowy | [exercise-summary.json](https://github.com/Paffk0o/repozytorium/blob/main/lab1/project/results/exercise-summary.json) |

**Podział ról:** model wybierał polecenie, Claude Code wywoływał narzędzie PowerShell, mój komputer uruchamiał Pythona ze skryptem, a wydruk i kod wyjścia wracały do modelu. W sesji widać rzeczywiste wywołania i ich wyniki. Sama komenda w odpowiedzi agenta nie byłaby dowodem. To, że skrypt działał u mnie, potwierdza też sekcja `runtime` raportu: lokalny interpreter i lokalny katalog roboczy.

**Liczebności:** raport końcowy pokazuje 8 wiadomości: 4 ham i 4 spam. To 8 rekordów CSV, a nie 8 linii, bo plik ma 203 linie: pole `text` zawiera całą treść maila. W etapie 1, przed zmianą danych, było 5 ham i 3 spam. Liczby zgadzają się z etykietami w CSV, które agent policzył osobno, wyszukiwaniem.

**Skąd wiem, że raport pochodzi z danego uruchomienia:**
- przed pierwszym uruchomieniem pliku raportu nie było,
- skrypt zakończył się kodem 0 i wypisał `Report saved`,
- `input_file` wskazuje plik z polecenia,
- `input_sha256` zgadza się z sumą pliku danych zmierzoną tuż przed uruchomieniem.

Szczegóły końcowego uruchomienia są w odpowiedzi 5.

Uwaga: Claude Code uruchomiłem w folderze `LAB1`, nadrzędnym względem `project`. Polecenia wykonywały się jednak w katalogu `project`, co potwierdza wynik `Get-Location` w sesji.

## 3. Wyjaśniony fragment

**Plik i miejsce:** `describe_data.py`, linie 55–62. To blok `try`/`except` w `main()`: wywołanie `describe()`, utworzenie katalogu, zapis raportu i obsługa błędów.

**Moje wyjaśnienie:** skrypt najpierw czyta i sprawdza cały plik CSV w pamięci, a raport zapisuje dopiero na końcu, gdy wszystko się zgadza. Przy błędzie w danych (`ValueError`), w pliku (`OSError`), w kodowaniu (`UnicodeError`) albo w formacie CSV (`csv.Error`) wypisuje `Error: ...`, kończy się kodem 1 i niczego nie zapisuje. Stary raport zostaje więc na dysku i po samym pliku nie wiadomo, czy ostatnie uruchomienie się udało. Trzeba patrzeć na wydruk i kod wyjścia. Agent dopowiedział, że błąd systemowy w trakcie samego zapisu, np. pełny dysk, mógłby zostawić plik obcięty.

**Mały przypadek:** w katalogu leży już raport z udanego uruchomienia (5 ham, 3 spam). Uruchamiamy skrypt z tym samym `--out` na kopii `exercise.csv`, w której 6. rekord (`exercise-006`) ma etykietę `Spam` z wielkiej litery.

**Przewidywanie, zapisane przed sprawdzeniem:**
1. `Spam` nie przejdzie sprawdzenia w linii 26, bo etykieta jest porównywana dokładnie z `ham` i `spam`, a wielkość liter ma znaczenie.
2. Skrypt wypisze `Error: Record 6: label must be ham or spam.` i zakończy się kodem 1, bez `Report saved`.
3. W pliku raportu zostanie stara treść (5 ham, 3 spam).

**Sprawdzenie:** bezpieczna próba na moim komputerze, w katalogu tymczasowym poza projektem. Najpierw agent zrobił stary raport z niezmienionej kopii danych (kod 0, 5 ham, 3 spam). Potem przygotował kopię z etykietą `Spam` i uruchomił skrypt z tym samym `--out`. Wynik: `Error: Record 6: label must be ham or spam.`, kod wyjścia 1. Suma SHA-256 i czas modyfikacji raportu były przed próbą i po niej identyczne, a w raporcie została stara treść.

Pierwsza próba się nie udała. Polecenie przygotowujące kopię było źle zapisane, więc kopia nie powstała, a skrypt dostał nieistniejący plik. Wynik: `Error: [Errno 2] No such file or directory`, kod 1, raport również nietknięty. To ten sam mechanizm, tylko przez gałąź `OSError`.

**Wniosek:** wszystkie trzy przewidywania się potwierdziły. Różnicy nie było, ale próba pokazała jeszcze jedno. W tym przypadku stary raport dało się rozpoznać po `input_file`, który wskazywał inny plik niż ostatnie polecenie. Gdy kolejne uruchomienia używają tego samego wejścia, ten trop nie działa i rozstrzyga tylko kod wyjścia oraz wydruk. W projekcie podczas tego ćwiczenia nic się nie zmieniło.

## 4. Test

**Polecenie:** `py -3 -m unittest discover -s tests -v`, uruchomione w katalogu `project`.

**Wynik:** `test_cli_writes_expected_counts ... ok`, `Ran 1 test in 0.084s`, `OK`, kod wyjścia 0.

**Co sprawdza:** test uruchamia `describe_data.py` na `tests/fixtures/messages.csv`, czyli 3 sztucznych wiadomościach (2 ham, 1 spam, każda w kilku liniach). Raport zapisuje do katalogu tymczasowego, który potem usuwa. Sprawdza, czy:
- program kończy się kodem 0,
- plik raportu powstał,
- `input_file` jest plikiem testowym,
- `total_messages` wynosi 3,
- `label_counts` to 2 ham i 1 spam.

Oczekiwane wartości są wpisane na stałe w teście, więc nie pochodzą z testowanego programu.

**Czego nie sprawdza:** nie czyta mojego `exercise-summary.json`. Suma tego pliku była przed testem i po nim taka sama. Nie dowodzi też poprawności dla każdych danych ani jakości przyszłego modelu. W etapie 2 testu nie powtarzałem, bo kod się nie zmienił, a zmienione dane nie są wejściem testu.

## 5. Zmiana danych

**Zmiana:** wiadomość `exercise-001`, etykieta `ham` → `spam`. To celowa symulacja nowej wersji danych, a nie ocena treści wiadomości. Agent pokazał dowód na podstawie porównania z kopią sprzed zmiany: nadal 8 rekordów, zmienione jedno pole w jednym rekordzie, jedyna różnica w linii 24, suma pliku `95e9049f…` → `7dc8080d…`. Dwie pierwsze próby edycji narzędzie odrzuciło: w jednej tekst przed i po był identyczny, w drugiej dopasowanie było niejednoznaczne. Nic się wtedy nie zmieniło. Przed odświeżeniem raportu nie uruchamiano ani skryptu, ani testu.

**Przewidywanie przed wykonaniem:** dotychczasowy raport nie opisuje aktualnych danych. Powstał przed zmianą, sam się nie aktualizuje, a jego `input_sha256` nie zgadza się z obecnym plikiem. Po odświeżeniu: `total_messages` zostaje 8, ham spada z 5 do 4, spam rośnie z 3 do 4.

**Operacje i ich cel:**
1. Ponowne uruchomienie `describe_data.py` tym samym poleceniem. Tylko tak powstaje raport z aktualnych danych, bez ręcznej edycji liczb. Plik zostanie nadpisany, ale treść raportu z etapu 1 jest zachowana w sesji.
2. Odczyt raportu z dysku.
3. Potwierdzenie, że raport jest nowy: kod 0 i `Report saved`, nowszy czas modyfikacji, `input_sha256` równy `7dc8080d…`. Agent zaproponował jeszcze pomiar stanu raportu i danych tuż przed uruchomieniem oraz porównanie pole po polu. Przyjąłem to.

Testu nie powtarzałem z powodu opisanego w odpowiedzi 4.

**Porównanie przed i po:**

| Pole | Etap 1 | Etap 2 | Przewidywanie |
|---|---|---|---|
| `input_file` | `...\project\data\exercise.csv` | bez zmian | bez zmian ✅ |
| `input_sha256` | `95e9049f…6ed07` | `7dc8080d…18b8152` | `7dc8080d…` ✅ |
| `total_messages` | 8 | 8 | 8 ✅ |
| `ham` | 5 | 4 | 4 ✅ |
| `spam` | 3 | 4 | 4 ✅ |
| `runtime` | Python 3.14.5, `...\project` | bez zmian | bez zmian ✅ |

**Z którego uruchomienia pochodzi raport:** z uruchomienia o 19:10:02 (kod wyjścia 0, `Report saved` z liczbami 4/4). Czas modyfikacji pliku zmienił się z 19:04:44.046 (etap 1) na 19:10:02.594. Suma pliku raportu zmieniła się z `b9bead71…` na `af302133…`. `input_sha256` jest równy sumie danych zmierzonej tuż przed uruchomieniem. Raportu nie poprawiałem ręcznie.

**Wniosek:** zmieniły się tylko `input_sha256` i `label_counts`, tak jak przewidywałem. Ta sama ścieżka `input_file` przy innej sumie kontrolnej pokazuje, że to ten sam plik w nowej wersji treści. Dlatego raport trzeba odświeżyć po każdej zmianie danych, bo sam się nie zaktualizuje.
