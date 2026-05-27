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

<!-- Opis krok po kroku jak działa nowy proces — od maila do zamknięcia ticketu -->

---

## 5. Architektura i integracje

### 5.1 Diagram komponentów
<!-- Odniesienie do diagramu Event Storming TO-BE -->

### 5.2 Komponenty systemu

| Komponent | Rola | Technologia/API |
|---|---|---|
| | | |

### 5.3 Gdzie jest LLM i do czego służy
<!-- Co robi model AI, na jakich danych, z jakim promptem — ogólnie -->

### 5.4 Gdzie jest logika reguł (bez LLM)
<!-- Co NIE powinno iść przez LLM i dlaczego -->

---

## 6. Kluczowe decyzje projektowe

<!-- Format: Decyzja → Uzasadnienie → Alternatywa którą odrzuciłam i dlaczego -->

### Decyzja 1: 
### Decyzja 2: 
### Decyzja 3: 

---

## 7. Edge case'y i obsługa wyjątków

<!-- Co się dzieje gdy: mail jest niekompletny / SAP nie odpowiada / LLM zwraca niską pewność kategoryzacji / itd. -->

---

## 8. Trade-offy

| Wybór | Zysk | Koszt |
|---|---|---|
| | | |

---

## 9. Co zostało poza zakresem (i dlaczego)

<!-- Czego świadomie nie robię w tej wersji -->

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

