# Scenariusze testowe — pętla odświeżania profilu

Strona przedstawia fikcyjną firmę **Termowir — instalacje grzewcze** (Kraków i okolice). Służy do testowania pętli odświeżania profilu firmy w Asystent Cloud: najpierw generujemy profil z tej strony, a potem zmieniamy stronę między crawlami i sprawdzamy, co zrobi pętla.

- Adres strony głównej do wpisania w aplikacji: **`https://nmarszalekwenet.github.io/testing-site/`** (z ukośnikiem na końcu).
- Każda strona to osobny plik `.html`. Nie ma żadnego builda. Po `git push` GitHub Pages publikuje zmiany w ciągu około minuty. Przed uruchomieniem crawla sprawdź zmianę w przeglądarce, odświeżając bez cache.
- Każdy scenariusz wprowadzaj **osobnym commitem**. Wtedy wycofanie to po prostu `git revert <sha> && git push`.

## Ważne: jak crawler widzi tę stronę

1. **Strona działa w podkatalogu** `/testing-site/`. Crawler szuka `robots.txt` i `sitemap.xml` tylko w katalogu głównym domeny (`https://nmarszalekwenet.github.io/robots.txt`), a tam ich nie ma. Dlatego na tym hostingu crawler znajduje strony **wyłącznie przez linki na stronie głównej**. Strona główna linkuje wszystkie 21 podstron: menu, kafelki usług, sekcję „Z bloga” i stopkę. Wniosek: **nowa strona musi dostać link na `index.html`**, inaczej crawler jej nie zobaczy. Pliki `robots.txt` i `sitemap.xml` i tak aktualizuj, żeby strona była gotowa na hosting w katalogu głównym domeny.
2. Crawler wycina `<header>`, `<nav>` i `<footer>`, a czyta tylko to, co jest w `<main>`. Zmiany w menu i stopce są dla niego niewidoczne.
3. Każdy fakt jest w jednym miejscu (wyjątek: ceny usług są w cenniku i na stronie danej usługi):

| Fakt | Plik(i) |
|---|---|
| Ceny usług | `cennik.html` + `oferta/<usługa>.html` |
| Wyjazd diagnostyczny, koszty dojazdu | `cennik.html` |
| Adres, telefony, e-mail, godziny otwarcia biura, obszar działania | `kontakt.html` |
| Godziny i czas dojazdu pogotowia | `oferta/pogotowie-grzewcze.html` |
| Gwarancja na robociznę, płatności, terminy | `jak-pracujemy.html` |
| Gwarancja producenta i warunek przeglądów | `oferta/przeglady-serwis.html` |
| Warunki dla firm (netto + 23% VAT, 14 dni) | `dla-firm.html` |
| Dofinansowania (Czyste Powietrze, Moje Ciepło) | `faq.html` |
| Historia, zespół, uprawnienia | `o-nas.html` |

4. Progi bezpieczeństwa (domyślna konfiguracja):
   - strona znika dopiero po **2 kolejnych** crawlach bez niej (404 albo brak linku);
   - pauza `mass_removal`, gdy w jednym crawlu usunięto **> 30% stron i co najmniej 3 strony** albo tekst serwisu spadł o **> 50%**;
   - pauza `read_failures`, gdy strona główna nie wczytała się w **3 kolejnych** crawlach.
5. Strona, która ma w `<main>` mniej niż 50 znaków tekstu, ma status `empty`. Taki odczyt liczy się jako nieudany: zachowuje poprzednią długość tekstu, więc **nie** obniża licznika „tekst spadł o 50%”. Żeby wywołać spadek tekstu, zostaw na stronie krótki tekst dłuższy niż 50 znaków (patrz scenariusz 7B).

---

## Scenariusz 1 — zmiana ceny → aktualizacja sekcji „Usługi i produkty”

**Zmiana:** przegląd roczny kotła gazowego zdrożał z 390 zł na 430 zł.

- `cennik.html`: w wierszu `<td>Przegląd roczny kotła gazowego</td>` zmień `<td class="price">390 zł</td>` na `<td class="price">430 zł</td>`.
- `oferta/przeglady-serwis.html`: w wierszu
  `<tr><td>Przegląd roczny kotła gazowego</td><td class="price">390 zł</td></tr>`
  zmień `390 zł` na `430 zł`.

```bash
sed -i '' 's#<td class="price">390 zł</td>#<td class="price">430 zł</td>#' cennik.html oferta/przeglady-serwis.html
```

**Oczekiwany wynik:** zmiana merytoryczna (cena) na 2 stronach. Sekcja „Usługi i produkty” dostaje automatyczną aktualizację na 430 zł, bo klient nie edytował jej ręcznie. Inne sekcje bez zmian.

**Wycofanie:** `git revert` albo ten sam `sed` w drugą stronę (`430 zł` → `390 zł`).

---

## Scenariusz 2 — nowa usługa → nowa pozycja w ofercie

**Zmiana:** dodajemy montaż klimatyzacji.

1. `cp oferta/kotly-gazowe.html oferta/klimatyzacja.html`
2. W `oferta/klimatyzacja.html` zmień `<title>` na `Klimatyzacja i pompy ciepła powietrze–powietrze | Termowir — instalacje grzewcze Kraków`. Całą zawartość `<div class="wrap">` w `<main>` zastąp tym blokiem:

```html
			<h1>Montaż klimatyzacji</h1>
			<p class="lead">Od tego sezonu montujemy też klimatyzatory typu split, które latem chłodzą, a w okresach przejściowych mogą dogrzewać dom.</p>
			<h2>Na czym polega usługa</h2>
			<p>Dobieramy moc urządzenia do wielkości i nasłonecznienia pomieszczenia, montujemy jednostkę wewnętrzną i zewnętrzną, prowadzimy instalację freonową i odprowadzenie skroplin, a na koniec robimy próbę szczelności i uruchomienie. Montaż jednego klimatyzatora trwa zwykle jeden dzień.</p>
			<h2>Cena</h2>
			<div class="table-scroll">
				<table>
					<thead>
						<tr><th>Usługa</th><th>Cena</th></tr>
					</thead>
					<tbody>
						<tr><td>Montaż klimatyzatora split do 3,5 kW (robocizna, do 3 m instalacji)</td><td class="price">1 900 zł</td></tr>
					</tbody>
				</table>
			</div>
			<p>Cena jest brutto z 8% VAT. Koszty dojazdu podajemy w <a href="../cennik">cenniku</a>. Termin oględzin ustalisz przez stronę <a href="../kontakt">Kontakt</a>.</p>
```

3. W `oferta/index.html` i `index.html` dodaj kafelek obok pozostałych (skopiuj istniejący `<div class="card">` i podmień link na `oferta/klimatyzacja` / `klimatyzacja`, a tytuł na „Montaż klimatyzacji”). **Link na `index.html` jest obowiązkowy** (patrz uwaga 1).
4. W `cennik.html` dodaj grupę i wiersz z ceną 1 900 zł.
5. W `sitemap.xml` dodaj `<url>` z `https://nmarszalekwenet.github.io/testing-site/oferta/klimatyzacja`.

**Oczekiwany wynik:** nowa strona (`added`) i zmiany w ofercie, cenniku i na stronie głównej. Sekcja „Usługi i produkty” dostaje nową usługę (montaż klimatyzacji, 1 900 zł).

**Wycofanie:** `git revert`. Pierwszy crawl po wycofaniu oznaczy stronę jako brakującą, drugi jako usuniętą, a usługa zniknie z profilu.

---

## Scenariusz 3 — zmiana godzin otwarcia → aktualizacja sekcji „Dane kontaktowe”

**Zmiana:** w `kontakt.html`, w tabeli godzin:

- `<tr><td>Piątek</td><td>7:30–16:30</td></tr>` → `<tr><td>Piątek</td><td>7:30–14:00</td></tr>`
- `<tr><td>Sobota</td><td>8:00–12:00</td></tr>` → `<tr><td>Sobota</td><td>nieczynne</td></tr>`

**Oczekiwany wynik:** sekcja „Dane kontaktowe” zaktualizowana: piątek do 14:00, sobota nieczynne. Godziny pogotowia (osobna strona) bez zmian.

**Wycofanie:** `git revert` albo przywróć obie linie.

---

## Scenariusz 4 — kosmetyczna zmiana tekstu na `/o-nas` → brak zmian w profilu

**Zmiana:** w `o-nas.html` przeredaguj dwa pierwsze akapity bez zmiany faktów. Na przykład zamień

```html
<p class="lead">Termowir to rodzinna firma instalacyjna spod Wieliczki. Zajmujemy się ogrzewaniem domów, budynków wielorodzinnych i małych firm w Krakowie i okolicach od 2009 roku.</p>
```

na

```html
<p class="lead">Jesteśmy rodzinną firmą instalacyjną z okolic Wieliczki. Od 2009 roku zajmujemy się ogrzewaniem domów jednorodzinnych, budynków wielorodzinnych i niewielkich firm w Krakowie oraz okolicy.</p>
```

a w drugim akapicie „Zaczynał sam, z jednym samochodem i skrzynką narzędzi” na „Na początku pracował sam — miał jeden samochód i skrzynkę narzędzi”.

**Oczekiwany wynik:** strona ma nowy hash i diff, ale LLM uznaje zmianę za kosmetyczną. Żadna sekcja się nie zmienia i nie powstaje propozycja.

**Wycofanie:** `git revert`.

---

## Scenariusz 5 — tylko nowy wpis na blogu → brak zmian w profilu

**Zmiana:**

1. `cp blog/jak-przygotowac-kociol-do-sezonu.html blog/odpowietrzanie-grzejnikow.html`. W nowym pliku zmień `<title>`, `<h1>`, datę w `<p class="meta">` (np. `Opublikowano: 9 października 2026 · autor: Marek`) i przepisz treść na poradnik o odpowietrzaniu grzejników (bez cen, godzin i danych kontaktowych).
2. Dodaj link na `blog/index.html` (nowy `<article class="card">` na początku listy) i **na `index.html`** w sekcji „Z bloga” (pierwsza pozycja `<li>`).
3. Dodaj wpis do `sitemap.xml`.

**Oczekiwany wynik:** nowa strona typu `blog` i drobne zmiany (nowy link) na stronie głównej i liście bloga. LLM nie widzi faktów istotnych dla profilu, więc wszystkie sekcje zostają bez zmian.

**Wycofanie:** `git revert`.

---

## Scenariusz 6 — usunięcie jednej usługi → usunięcie dopiero po drugim crawlu

**Zmiana:** `git rm oferta/ogrzewanie-podlogowe.html`. Linki zostaw (realistyczny „martwy link”, strona zwraca 404). Wariant: usuń też kafelek z `index.html` i `oferta/index.html`, wiersze z `cennik.html` i wpis z `sitemap.xml`. Wynik powinien być ten sam.

**Oczekiwany wynik:**

- **crawl 1:** strona oznaczona jako brakująca (`missingCount = 1`). Nie ma zmiany „removed” i profil zostaje bez zmian.
- **crawl 2:** strona usunięta (`removed`). Sekcja „Usługi i produkty” traci ogrzewanie podłogowe (o ile cennik też nie ma już tej usługi; jeśli wiersze w cenniku zostały, LLM może słusznie ją zachować).
- Usunięcie 1 z 21 stron jest poniżej progu `mass_removal`, więc **brak pauzy**.

**Wycofanie:** `git revert`. Przywrócona strona wraca jako `added`.

---

## Scenariusz 7 — masowe usunięcie lub opróżnienie stron → pauza firmy

### 7A — usunięcie stron (pauza na drugim crawlu)

**Zmiana:** usuń 8 z 21 stron (38%, czyli > 30% i ≥ 3):

```bash
git rm oferta/kotly-gazowe.html oferta/pompy-ciepla.html oferta/ogrzewanie-podlogowe.html \
	oferta/przeglady-serwis.html oferta/pogotowie-grzewcze.html realizacje.html opinie.html dla-firm.html
git commit -m "test: scenariusz 7A — masowe usunięcie stron" && git push
```

**Oczekiwany wynik:**

- **crawl 1:** 8 stron ma `missingCount = 1`. Nic jeszcze nie jest usunięte, więc nie ma pauzy.
- **crawl 2:** 8 stron przechodzi do `removed` w jednym crawlu, co daje 8/21 > 30%. Firma dostaje pauzę `mass_removal`, a w aplikacji pojawia się baner z przyciskiem **„Sprawdźcie ponownie”**. Sekcje nie są aktualizowane.

### 7B — opróżnienie treści (pauza od razu, przez spadek tekstu > 50%)

**Zmiana:** w 13 stronach zastąp całą zawartość `<main>` krótkim komunikatem dłuższym niż 50 znaków, żeby odczyt nadal miał status `ok`:

```bash
python3 - <<'EOF'
import re
files = ["oferta/index.html", "oferta/kotly-gazowe.html", "oferta/pompy-ciepla.html",
	"oferta/ogrzewanie-podlogowe.html", "oferta/przeglady-serwis.html", "oferta/pogotowie-grzewcze.html",
	"faq.html", "realizacje.html", "o-nas.html", "jak-pracujemy.html",
	"blog/jak-przygotowac-kociol-do-sezonu.html", "blog/pompa-ciepla-w-starym-domu.html",
	"blog/czyste-powietrze-krok-po-kroku.html"]
stub = '<main>\n\t\t<div class="wrap">\n\t\t\t<h1>Strona w przebudowie</h1>\n\t\t\t<p>Pracujemy nad nową wersją tej podstrony. Zajrzyj tu ponownie za kilka dni.</p>\n\t\t</div>\n\t</main>'
for f in files:
	s = open(f, encoding="utf-8").read()
	open(f, "w", encoding="utf-8").write(re.sub(r"<main>.*?</main>", stub, s, flags=re.S))
EOF
```

**Oczekiwany wynik:** **już pierwszy crawl** daje spadek tekstu serwisu o ponad 50% i pauzę `mass_removal`. Sekcje bez zmian (zwłaszcza nie mogą stracić usług).

**Wycofanie (7A i 7B):** patrz scenariusz 8.

---

## Scenariusz 8 — przywrócenie strony → wznowienie przez „Sprawdźcie ponownie”

**Zmiana:** `git revert <sha scenariusza 7>` i `git push`. Odczekaj, aż GitHub Pages opublikuje zmianę, i sprawdź kilka podstron w przeglądarce.

**Oczekiwany wynik:**

- Pauza `mass_removal` **nie znika sama**: nowe crawle są wstrzymane do decyzji człowieka.
- W aplikacji kliknij **„Sprawdźcie ponownie”** na banerze. Crawl przechodzi, strony wracają (w 7A jako `restored`/`added`), nie ma pauzy, a profil wraca do stanu sprzed scenariusza 7 albo zostaje bez zmian, jeśli niczego nie zaktualizowano.
- Jeśli klikniesz **przed** przywróceniem strony, oczekuj ponownej pauzy.

---

## Scenariusz 9 — sprzeczność: klient poprawił cenę ręcznie, a potem zmienia się strona → propozycja z konfliktem

1. W aplikacji, jako klient, edytuj ręcznie sekcję „Usługi i produkty”: zmień cenę montażu pompy ciepła do 10 kW z **9 500 zł** na **9 900 zł** i zapisz. Sekcja staje się chroniona, bo autorem jest klient.
2. Na stronie zmień tę samą cenę na **10 200 zł**:

```bash
sed -i '' 's#<td class="price">9 500 zł</td>#<td class="price">10 200 zł</td>#' cennik.html oferta/pompy-ciepla.html
```

3. Uruchom crawl.

**Oczekiwany wynik:** sekcja **nie** jest nadpisywana automatycznie. Powstaje **propozycja** do akceptacji przez klienta, która pokazuje konflikt: profil ma 9 900 zł (ręczna edycja), strona ma 10 200 zł. Po akceptacji sekcja ma 10 200 zł, a po odrzuceniu zostaje 9 900 zł.

**Wycofanie:** `sed` w drugą stronę (`10 200 zł` → `9 500 zł`) albo `git revert`. Ręczną edycję w profilu cofnij w historii wersji sekcji. Uwaga: sekcja pozostaje chroniona na stałe, więc do kolejnych testów automatycznych aktualizacji tej sekcji użyj świeżej firmy.

---

## Scenariusz 10 (dodatkowy) — nieczytelna strona główna → pauza `read_failures`

**Zmiana:** w `index.html` zastąp zawartość `<div class="wrap">` w `<main>` pustą treścią (zostaw samo `<main><div class="wrap"></div></main>`). Menu i stopka zostają, więc crawler nadal znajduje podstrony, ale strona główna ma status `empty` (mniej niż 50 znaków), czyli odczyt nieudany.

**Oczekiwany wynik:** crawle 1 i 2 to nieudany odczyt strony głównej bez pauzy, a pozostałe strony czytają się normalnie. **Crawl 3** daje pauzę `read_failures`.

**Wycofanie:** `git revert` i push. Pauza `read_failures` znika sama przy następnym udanym crawlu.

> Uwaga: **nie** testuj tego przez usunięcie `index.html` (404). Na tym hostingu (bez `sitemap.xml` w katalogu głównym domeny) crawler nie znajdzie wtedy żadnej podstrony. Wszystkie zostaną oznaczone jako brakujące, a w crawlu 2 zadziała `mass_removal`, zanim zadziała `read_failures`.
