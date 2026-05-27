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

<!-- Co automatyzujemy, czego NIE automatyzujemy i dlaczego -->

---

## 3. Proponowane rozwiązanie — overview

<!-- 2-3 zdania: co proponujesz i jaka jest główna idea -->

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

<!-- Co wymaga decyzji biznesowej lub dalszego discovery -->
