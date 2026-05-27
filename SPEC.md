# Specyfikacja rozwiązania: Automatyzacja procesu reklamacji — Metalpol Sp. z o.o.

**Autor:** Joanna Morawska  
**Data:** 2026-05-27  
**Wersja:** 1.1

---

## 1. Cel dokumentu

<!-- Co opisuje ten dokument i jaki problem rozwiązuje -->

---

## 2. Kontekst biznesowy

### 2.1 Firma i proces

Metalpol Sp. z o.o. — producent komponentów metalowych dla branży automotive. 180 pracowników, 3 hale produkcyjne, dział serwisu posprzedażowego.

**Obecny proces reklamacji (AS-IS):**

1. Klient wysyła e-mail na reklamacje@metalpol.pl — zdjęcie wady, numer zamówienia, opis (PL lub EN).
2. Specjalista serwisu czyta e-mail, ręcznie przepisuje dane do Excela (Rejestr Reklamacji 2026.xlsx).
3. Kategoryzuje wadę (wizualna / wymiary / materiał / logistyka) — ocena subiektywna.
4. Tworzy ticket w JIRA (projekt REK, issue type Complaint), sprawdza zamówienie i batch w SAP.
5. Pisze odpowiedź klientowi — średnio 2 dni od zgłoszenia.
6. Jeśli wada potwierdzona → tworzy ticket korygujący dla działu jakości (JIRA, issue type Correction).

**Wolumen:**
- ~600 reklamacji/miesiąc średnio, szczyt sezonowy (maj–lipiec) do 2000/miesiąc.
- Średnia długość e-maila: ~150 słów + 1–3 zdjęcia (typowo 2–5 MB każde).
- ~85% reklamacji można rozstrzygnąć na podstawie danych w SAP (batch → operator → parametry produkcji).

**Dostępne systemy:**
- Microsoft 365 / Exchange (reklamacje@metalpol.pl) — Microsoft Graph API, OAuth2, webhook na nowe maile dostępny.
- SAP ERP (moduł PP/QM) — REST API: GET /api/v1/orders/{id}, GET /api/v1/batches/{id}, rate limit 100 req/min.
- JIRA Cloud (projekt REK) — REST API, issue types: Complaint, Correction.
- Wewnętrzna baza klientów — PostgreSQL (read-only), ~15k rekordów.
- Azure Blob Storage — SAS tokens, archiwum zdjęć wad.

### 2.2 Zidentyfikowane problemy

- ~40% maili trafia do spam lub jest czytane z opóźnieniem.
- Jeden specjalista obsługuje 30 reklamacji/dzień (w szczycie 80); backlog 2–3 dni.
- Kategoryzacja niespójna między specjalistami — ten sam typ wady raz "wizualna", raz "materiał".
- Brak metryk: ile reklamacji, jakie typy, które linie produkcyjne generują najwięcej problemów.
- SAP i JIRA nie komunikują się ze sobą — wszystko przechodzi ręcznie przez Excela.
  
### 2.3 Zakres automatyzacji
 
**Zakres automatyzacji:**
- Odbiór maili z reklamacjami (webhook, filtrowanie spamu)
- Walidacja nadawcy (baza klientów) i zamówienia (SAP)
- Wysyłka potwierdzenia odbioru do klienta
- Kategoryzacja wady przez LLM (tekst + zdjęcia)
- Tworzenie ticketu Complaint w JIRA z danymi z SAP
- Ocena czy sprawa może zostać rozstrzygnięta automatycznie
- Tworzenie ticketu Correction w JIRA i wysyłka odpowiedzi do klienta — na ścieżce gdzie SAP dostarcza wystarczające dane
  
**Obsługa manualna (human-in-the-loop):**
- Przypadki z niekompletnymi danymi: nieznany nadawca, brak zamówienia w SAP
- Przypadki gdzie dane z SAP lub wynik kategoryzacji LLM nie są wystarczające do podjęcia decyzji
- Odpowiedź do klienta na ścieżkach wymagających oceny specjalisty

---

## 3. Proponowane rozwiązanie — overview

Proponowane rozwiązanie eliminuje manualną obsługę przepływu danych między systemami — od odebrania maila, przez weryfikację w bazie klientów i SAP, po utworzenie ticketu w JIRA. Głównym celem jest usunięcie Excela jako pośrednika między systemami i skrócenie czasu obsługi reklamacji.
 
LLM został zastosowany w jednym, konkretnym miejscu: kategoryzacji wady na podstawie treści maila i zdjęć. Rozwiązuje to problem niespójnej klasyfikacji między specjalistami i pozwala systemowi automatycznie zamknąć ~85% przypadków, które można rozstrzygnąć na podstawie danych z SAP. Pozostałe przypadki trafiają do specjalisty z kompletem danych już zebranych przez automat.

---

## 4. Flow automatyzacji (TO-BE)

Pełny diagram procesu: [event-storming-to-be.md](event-storming-to-be.md)
 
Poniżej opisane są elementy dodane względem procesu AS-IS:
 
**Webhook i filtrowanie spamu**
Mail na reklamacje@metalpol.pl wyzwala webhook Microsoft Graph API. Reklamacja jest odbierana natychmiast, bez udziału specjalisty.
 
**Walidacja twarda**
System sprawdza czy nadawca jest w bazie klientów (PostgreSQL) i czy numer zamówienia istnieje w SAP. Maile które nie przechodzą walidacji są odrzucane bez tworzenia ticketu.
 
**Potwierdzenie odbioru**
Po pozytywnej walidacji system wysyła klientowi automatyczne potwierdzenie odbioru.
 
**Kategoryzacja przez LLM**
LLM analizuje treść maila i zdjęcia i przypisuje kategorię wady. Zastępuje subiektywną ocenę specjalisty ujednoliconą klasyfikacją.
 
**Tworzenie ticketu Complaint w JIRA**
Ticket tworzony automatycznie z kategorią wady i danymi z SAP (zamówienie, batch, parametry produkcji). Specjalista przejmujący sprawę ma wszystkie dane w jednym miejscu.
 
**Automatyczne zamknięcie — ~85% przypadków**
Jeśli dane z SAP wystarczają do rozstrzygnięcia reklamacji, system tworzy ticket Correction w JIRA i wysyła odpowiedź do klienta bez angażowania specjalisty.
 
**Przekazanie do specjalisty**
Przypadki wymagające oceny ludzkiej trafiają do specjalisty jako ticket w JIRA — z kategorią, danymi z SAP i historią maila.

---

## 5. Architektura i integracje
 
### 5.1 Komponenty systemu
 
**Istniejące systemy:**
 
| Komponent | Rola | Technologia/API |
|---|---|---|
| Microsoft 365 / Exchange | Odbiór maili z reklamacjami | Microsoft Graph API, OAuth2, webhook |
| PostgreSQL (baza klientów) | Walidacja nadawcy | REST, read-only, ~15k rekordów |
| SAP ERP (PP/QM) | Walidacja zamówienia, dane o batchu i parametrach produkcji | REST API: GET /api/v1/orders/{id}, GET /api/v1/batches/{id} |
| JIRA Cloud | Tworzenie ticketów Complaint i Correction | REST API, issue types: Complaint, Correction |
| Azure Blob Storage | Archiwizacja zdjęć z maili | SAS tokens |
 
**Nowe komponenty:**
 
| Komponent | Rola | Technologia/API |
|---|---|---|
| Serwis integracyjny | Orkiestracja całego flow — odbiera webhook, wywołuje kolejne systemy, podejmuje decyzje routingu | Do ustalenia (np. Azure Functions, Python) |
| LLM | Kategoryzacja wady na podstawie tekstu i zdjęć | Do ustalenia (np. GPT-4o, Claude) |


### 5.2 Zastosowanie LLM
 
**Dane wejściowe:**
- Treść maila (tekst, PL lub EN)
- Zdjęcia wad (1–3 obrazy, typowo 2–5 MB każdy)
  
**Oczekiwany output:**
- Kategoria wady: wizualna / wymiary / materiał / logistyka
- Rekomendacja ścieżki: automatyczna (dane z SAP wystarczają) lub manualna (wymaga oceny specjalisty)
- Poziom pewności kategoryzacji (do wykorzystania przy routingu)
- Wykryty język maila (PL lub EN) — do wykorzystania przy wysyłce odpowiedzi do klienta
  
**Wymagania wobec modelu:**
- Obsługa multimodalna (tekst + obraz)
- Wsparcie dla języka polskiego i angielskiego
- Dostępność przez API
  
### 5.3 Logika deterministyczna
 
- **Walidacja nadawcy** — sprawdzenie w PostgreSQL czy adres email nadawcy istnieje w bazie klientów. Binarna decyzja, nie wymaga interpretacji.
- **Walidacja zamówienia** — sprawdzenie w SAP czy numer zamówienia z maila istnieje i jest aktywny. Binarna decyzja.
- **Tworzenie ticketów w JIRA** — operacja na API, bez elementu oceny.
- **Wysyłka potwierdzenia i odpowiedzi** — szablonowe wiadomości, bez generowania treści przez LLM.

---

## 6. Kluczowe decyzje projektowe
 
### Decyzja 1: Twarde reguły przed LLM
Walidacja nadawcy (PostgreSQL) i zamówienia (SAP) realizowana przez reguły deterministyczne, nie przez LLM. Binarne sprawdzenia nie wymagają interpretacji — reguły są tańsze, szybsze i przewidywalne.
 
### Decyzja 2: LLM tylko do kategoryzacji i oceny ścieżki
LLM nie generuje odpowiedzi do klienta ani nie tworzy ticketów. Ograniczenie zakresu LLM do jednego kroku zmniejsza ryzyko błędu i koszt, a zakres można rozszerzać iteracyjnie.
 
### Decyzja 3: Autonomiczna odpowiedź tylko na ścieżce ~85%
System wysyła odpowiedź do klienta autonomicznie tylko gdy dane z SAP wystarczają do rozstrzygnięcia. Pozostałe przypadki obsługuje specjalista. Pozwala to zweryfikować dokładność systemu w produkcji przed rozszerzeniem autonomii.
 
### Decyzja 4: Jeden ticket Complaint na reklamację
Ticket Complaint tworzony jest zawsze — przy automatycznej ścieżce zostaje zamknięty przez orkiestrator po utworzeniu Correction, przy manualnej jest początkiem pracy specjalisty. Correction linkowany do Complaint zachowuje pełną historię reklamacji w JIRA.

### Decyzja 5: Ticket tylko dla zwalidowanych maili
Tickety tworzone są wyłącznie dla maili które przeszły walidację nadawcy i zamówienia. Maile odrzucone na etapie walidacji nie generują wpisów w JIRA.
 
### Decyzja 6: Single-pass architecture
Każdy system jest odpytywany raz, w kolejności wynikającej z dostępności danych wejściowych. Eliminuje to wielokrotne wywołania tych samych systemów, upraszcza logikę orkiestratora i ułatwia wykrywanie błędów.
 
### Decyzja 7: Synchroniczny flow
Flow realizowany synchronicznie — bez kolejkowania i priorytetyzacji. Wystarczające przy braku twardego SLA; upraszcza architekturę i obniża koszt wdrożenia.
 
### Decyzja 8: Język odpowiedzi wykrywany przez LLM
LLM wykrywa język maila i odpowiedź wysyłana jest w tym samym języku. Przy mailach mieszanych (PL/EN) LLM wybiera język dominujący.

### Decyzja 9: Brak automatycznego feedback loop
Model jest trenowany na historycznych danych przed wdrożeniem. Po uruchomieniu systemu korekty specjalistów nie wracają automatycznie do modelu. Przy wolumenie ~600 reklamacji miesięcznie koszt budowy i utrzymania feedback loop przewyższa korzyści — ewentualna aktualizacja modelu może zostać przeprowadzona na życzenie klienta.
 
---
 
## 7. Edge case'y i obsługa wyjątków
 
W przypadkach gdy automatyczne przetwarzanie nie jest możliwe, tworzony jest ticket Complaint i przekazywany do specjalisty. Zapewnia to ciągłość obsługi niezależnie od rodzaju wyjątku.
 
**Timeout SAP lub bazy klientów**
Jeśli system nie otrzyma odpowiedzi w określonym czasie, ticket zakładany i kierowany do specjalisty z informacją o niedostępności systemu. Próg timeout do ustalenia przed wdrożeniem.
 
**Walidacja nie przeszła (nieznany nadawca lub nieistniejące zamówienie)**
Brak ticketu, brak odpowiedzi do nadawcy. Wyjątek — jeśli nadawca jest w bazie klientów ale numer zamówienia nie przeszedł walidacji, wysyłana jest zwrotka z prośbą o weryfikację numeru. Decyzja o szczegółach obsługi tego przypadku do uzgodnienia z klientem (patrz sekcja 10).
 
**Mail bez zdjęcia lub załącznik w nieobsługiwanym formacie**
Ticket zakładany i kierowany do specjalisty. Potwierdzenie odbioru wysyłane do klienta.
 
**LLM zwraca niską pewność kategoryzacji**
Ticket zakładany z flagą niskiej pewności i kierowany do specjalisty. Próg pewności poniżej którego następuje eskalacja do ustalenia przed wdrożeniem.
 
**Mail jest odpowiedzią na istniejący wątek**
System sprawdza czy temat maila zawiera numer ticketu JIRA (np. RE: [REK-123]). Jeśli tak — mail dołączany do istniejącego ticketu, nowy ticket nie jest tworzony.
 
**Język mieszany PL/EN**
LLM wykrywa język dominujący i wysyła odpowiedź w tym języku.
 
---

## 8. Trade-offy
 
| Wybór | Zysk | Koszt |
|---|---|---|
| Synchroniczny flow | Prosta architektura, niższy koszt wdrożenia | Brak priorytetyzacji — przy twardym SLA może być niewystarczające |
| Brak feedback loop | Niższy koszt utrzymania, brak złożonej infrastruktury | Model nie poprawia się automatycznie — aktualizacja wymaga ręcznej interwencji |
| Twarde reguły do walidacji nadawcy i zamówienia | Przewidywalne, szybkie, tanie | Brak tolerancji na błędy — literówka w adresie lub numerze zamówienia skutkuje odrzuceniem |
| Edge case'y zawsze kierowane do specjalisty | Bezpieczeństwo — żadna reklamacja nie jest błędnie obsłużona automatycznie | Mniejsza automatyzacja — specjalista nadal obsługuje wyjątki |
| Walidacja nadawcy jako filtr | Eliminacja spamu i nieuprawnionych zgłoszeń | Wymaga od klientów wysyłania reklamacji z zarejestrowanego adresu email — konieczne poinformowanie klientów |
 
---

## 9. Poza zakresem projektu
 
**Warstwa raportowa**
Nowy proces generuje ustrukturyzowane dane w JIRA stanowiące podstawę do raportowania. Budowa dashboardu analitycznego pozostaje decyzją po stronie klienta i jest traktowana jako odrębny projekt.
 
**Aktualizacja modelu LLM**
Projekt nie obejmuje budowy mechanizmu feedback loop. Ewentualna aktualizacja modelu realizowana jest na zlecenie klienta.
 
**Zarządzanie bazą klientów**
System zakłada aktualność istniejącej bazy klientów (PostgreSQL, read-only). Proces jej utrzymania i aktualizacji pozostaje poza zakresem.
 
**Interfejs użytkownika dla specjalisty**
Specjalista obsługuje reklamacje przez JIRA. Budowa dedykowanego panelu nie jest częścią tego projektu.

---

## 10. Pytania otwarte / ryzyka

### 10.1 Niespójność danych w opisie procesu

Dane wolumenowe i wydajnościowe w briefie nie są w pełni spójne:

- 600 reklamacji/miesiąc ÷ 20 dni roboczych = 30 reklamacji/dzień → jeden specjalista jest na granicy wydajności przy normalnym wolumenie.
- W szczycie: 2000/miesiąc ÷ 20 dni = 100/dzień. Brief podaje, że jeden specjalista obsługuje w szczycie do 80/dzień — co oznacza, że backlog (2–3 dni) pojawia się realnie, ale tylko sezonowo, nie przez cały rok.
- Brief nie precyzuje ile osób liczy zespół serwisu.

**Założenie przyjęte na potrzeby analizy:** opisane problemy (backlog, niespójna kategoryzacja) są prawdziwe i wymagają rozwiązania. Dane traktuję jako orientacyjne.

**Pytanie do klienta (przed projektem):** Ile osób obsługuje reklamacje? Od odpowiedzi zależy podejście do standaryzacji kategoryzacji przez LLM:
- Jeśli jedna osoba → problem z kategoryzacją leży w niespójności w czasie (zmęczenie, kontekst) — LLM standaryzuje względem jednego wzorca.
- Jeśli kilka osób → każda może mieć inny punkt widzenia — LLM musi rozstrzygnąć sprzeczne wzorce, co wymaga wcześniejszej decyzji biznesowej: czyja kategoryzacja jest "złota".

### 10.2 Brak zdefiniowanych celów i KPI

Brief opisuje problemy, ale nie definiuje co oznacza sukces projektu.

**Pytania do klienta:**
- Jakie KPI ma optymalizować system? Czas odpowiedzi do klienta? Dokładność kategoryzacji? Redukcja backlogu?
- Kto będzie tworzył i monitorował metryki po wdrożeniu?

Odpowiedzi wpłyną na priorytety architektoniczne — inaczej projektuje się system pod "szybkość odpowiedzi", inaczej pod "jakość kategoryzacji".

### 10.3 Akceptowalny poziom błędu LLM

Brief nie określa wymaganej dokładności kategoryzacji ani konsekwencji błędu.

**Pytania do klienta:**
- Jaka dokładność kategoryzacji jest wystarczająca — 80%? 95%?
- Czy błędna kategoryzacja ma konsekwencje finansowe lub jakościowe (np. błędny ticket Correction w JIRA)?

Odpowiedź decyduje o tym czy LLM kategoryzuje autonomicznie, czy tylko sugeruje kategorię do zatwierdzenia przez człowieka.

### 10.4 SLA automatyzacji

Brief nie definiuje oczekiwanego czasu przetwarzania przez system.

**Pytanie do klienta:** Czy wystarczy "szybciej niż obecne 2 dni", czy klient oczekuje konkretnego SLA (np. odpowiedź w ciągu godziny)?

Odpowiedź wpływa na architekturę: synchroniczny flow (prosto, taniej) vs kolejkowanie z priorytetami (bardziej złożone, potrzebne przy twardym SLA).

### 10.5 Liczba i definicja kategorii wad

Obecny proces używa 4 kategorii: wizualna / wymiary / materiał / logistyka. Brief nie wyjaśnia czy kategoria wpływa na routing reklamacji ani na priorytety obsługi — jedynie na metryki.

**Obserwacja:** Granica między "wizualna" a "materiałowa" jest rozmyta — zdjęcie wady często nie pozwala rozstrzygnąć czy problem dotyczy powierzchni czy składu materiału. Wymiary i logistyka są obiektywnie rozróżnialne.

**Pytanie do klienta:** Czy można połączyć kategorie "wizualna" i "materiałowa" w jedną? Zmniejszy to koszt i złożoność szkolenia LLM oraz ryzyko niespójnej klasyfikacji na granicy między tymi kategoriami.

