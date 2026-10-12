# Plan testu odświeżania profilu (od 12.10.2026)

Pełny przebieg pętli odświeżania na **nowej firmie** założonej na tej stronie. Stara firma testowa ma zmieszane ślady wcześniejszych testów (chronione sekcje, niezastosowana runda A), więc się nie nadaje. Starsze, pojedyncze scenariusze są w [TEST-SCENARIOS.md](TEST-SCENARIOS.md).

## Jak publikować kroki

Każdy krok to osobna gałąź `test/step-N`. Gałęzie są ułożone jedna na drugiej (step-2 zawiera step-1 itd.). Strona na GitHub Pages pokazuje zawsze to, co jest na `main`, więc krok publikuje się przez przesunięcie `main` do jego gałęzi:

```bash
git push origin test/step-1:main     # potem test/step-2:main itd.
```

Po pushu odczekaj ok. 1 min na GitHub Pages, sprawdź zmianę w przeglądarce, potem **„Uruchom teraz”** w adminie (firma → „Odświeżanie profilu”). Lokalnie aktualizacja sekcji kończy się po czasie z `PROFILE_REFRESH_WEBHOOK_WAIT_MINUTES` (10 min).

Powrót do stanu sprzed testu: `git push --force origin <sha sprzed kroku 1>:main`.

## Krok 0 — przygotowanie

1. Nowa firma w aplikacji, www `https://nmarszalekwenet.github.io/testing-site/` (lokalnie NIP i tak jest pomijany, `PROFILE_GEN_IGNORE_NIP=on`). Onboarding do stanu `finished`.
2. Flaga `profile_refresh`: `allowed_company_ids` = tylko nowa firma, `rollout_percentage = 0`. Tryb `profile_refresh.mode = "live"`.
3. Zapisz 7 sekcji v1 i sprawdź, które sekcje wymieniają godziny pogotowia, parking, czas wyceny i ogrzewanie podłogowe — to doprecyzowuje oczekiwania w krokach 2, 5 i 6.
4. „Uruchom teraz” → **crawl bazowy**: 21 stron `ok`, `baseline: true`, brak paczki zmian, brak propozycji.

## Krok 1 — automatyczna aktualizacja, jedna sekcja na zmianę (`test/step-1`)

| Zmiana | Oczekiwany efekt |
|---|---|
| `/kontakt`: Niedziela `nieczynne` → `10:00–13:00 (tylko telefonicznie)` | **Dane kontaktowe** → opublikowana nowa wersja (`site_refresh`) |
| `/cennik` + `/oferta/kotly-gazowe`: Pierwsze uruchomienie kotła **450 → 490 zł** | **Usługi i produkty** → opublikowana nowa wersja; `internal_data` zostaje na 450 zł (sekcja poza pętlą) |

Pozostałe sekcje: `no_change`. Brak propozycji do akceptacji.

## Krok 2 — jeden fakt w wielu sekcjach (`test/step-2`)

| Zmiana | Oczekiwany efekt |
|---|---|
| `/oferta/pogotowie-grzewcze`: zgłoszenia awaryjne dla klientów bez abonamentu **pn–pt 7:00–20:00 → 7:00–21:00** | Opublikowane **wszystkie sekcje, które wymieniają te godziny** (lista z kroku 0.3). W żadnej sekcji nie zostaje „20:00” w tym kontekście. Sekcje bez tych godzin — bez zmian. Wiersz cennika „Wyjazd awaryjny pn–pt 7:00–20:00” (stawka dzienna) się nie zmienia |

## Krok 3 — nowa informacja trafia do jednej sekcji (`test/step-3`)

| Zmiana | Oczekiwany efekt |
|---|---|
| Nowa podstrona `/oferta/klimatyzacja` (montaż klimatyzatora split, **1 900 zł**) + kafelki na `/` i `/oferta/` + grupa w `/cennik` | **Usługi i produkty** → nowa usługa z ceną. Dopuszczalne: krótka wzmianka w **Profilu firmy**. Pozostałe sekcje bez zmian |

## Krok 4 — zmiany bez znaczenia dla profilu (`test/step-4`)

| Zmiana | Oczekiwany efekt |
|---|---|
| `/o-nas`: przeredagowane dwa zdania, te same fakty | opis zmian „Brak zmian merytorycznych”, **zero nowych wersji** |
| Nowy wpis `/blog/jak-odpowietrzyc-grzejniki` (link na `/` i `/blog/`) | jak wyżej |
| Stopka na wszystkich stronach: „… Wszystkie dane są przykładowe.” | stopka jest wycinana — strony różnią się tylko nią, więc **nie liczą się jako zmienione** |

## Krok 5 — usunięcie strony (`test/step-5a`, potem `test/step-5b`)

| Akcja | Oczekiwany efekt |
|---|---|
| step-5a: usunięta `/oferta/ogrzewanie-podlogowe` (linki zostają, 404) → crawl 1 | strona `missing_count = 1`, profil bez zmian, brak pauzy |
| bez zmian na stronie → drugi raz „Uruchom teraz” | strona usunięta; **ogrzewanie podłogowe znika** z Usług i produktów i z każdej sekcji, która je wymienia; brak pauzy (1 z 23 stron) |
| step-5b: strona przywrócona → crawl | strona wraca jako nowa; usługa wraca do Usług i produktów |

## Krok 6 — ręczne poprawki i propozycje (`test/step-6`)

**Przed publikacją kroku**, w UI:
- klient — Usługi i produkty: cena przeglądu pompy ciepła → **500 zł**;
- klient — Dane kontaktowe: dopisuje „**Bezpłatny parking dla klientów przy biurze.**”;
- admin z zaznaczonym „w imieniu klienta” — Jak współpracujemy: dopisuje zdanie (np. o wycenie w PDF);
- admin z **odznaczonym** „w imieniu klienta” — Pytania klientów: dopisuje dowolne zdanie.

| Zmiana na stronie | Oczekiwany efekt |
|---|---|
| przegląd pompy ciepła **510 → 530 zł**, abonament **720 → 750 zł/rok** (`/cennik`, `/oferta/przeglady-serwis`) | **Usługi i produkty → propozycja**: diff 500 → 530 zł z **konfliktem** (Twoja wersja 500 / strona 530) i 720 → 750 zł **bez konfliktu**, choć to ta sama linia |
| `/kontakt`: „Przy biurze nie ma parkingu — najbliższy parking publiczny jest 200 m dalej, przy rynku.” | **Dane kontaktowe → propozycja**: diff zastępuje zdanie klienta o parkingu wersją ze strony + **konflikt** |
| `/jak-pracujemy`: wycena **2 → 4 dni roboczych** | **Jak współpracujemy → propozycja bez konfliktu**, dopisane zdanie zostaje dosłownie |
| (ta sama zmiana) | **Pytania klientów → opublikowana automatycznie** (poprawka admina bez „w imieniu klienta” nie chroni sekcji) |

## Krok 7 — decyzje i kolejne crawle (`test/step-7a`, `7b`, `7c`)

| Akcja | Oczekiwany efekt |
|---|---|
| **Akceptuj** Usługi i produkty, **odrzuć** Dane kontaktowe; „Uruchom teraz” bez zmian na stronie | brak zmian i nowych propozycji; **odrzucona zmiana nie wraca** |
| step-7a: `/kontakt` nowy e-mail `serwis@example.com` → crawl | propozycja dla Danych kontaktowych zawiera **tylko nowy e-mail**, bez odrzuconego parkingu |
| step-7b: rozdzielacz **1 450 → 1 500 zł** → crawl, **bez decyzji** | propozycja dla Usług i produktów z 1 500 zł |
| step-7c: audyt pompy ciepła **600 → 650 zł** → crawl | poprzednia propozycja `superseded`; **nowa zawiera obie zmiany** (1 500 i 650 zł). Tylko 650 zł = błąd: zmiana z 7b przepadła |

## Krok 8 — zabezpieczenia

Scenariusze 7A, 7B, 8 i 10 z [TEST-SCENARIOS.md](TEST-SCENARIOS.md): masowe usunięcie stron (pauza na drugim crawlu), opróżnienie treści (pauza od razu), przywrócenie i „Sprawdźcie ponownie”, nieczytelna strona główna (pauza `read_failures` na trzecim crawlu).
