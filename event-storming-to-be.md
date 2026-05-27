# Event Storming: Proces reklamacji TO-BE

**Legenda:**
- 🟠 Zdarzenie domenowe
- 🟡 Aktor
- 🤖 System / automatyzacja

---

```mermaid
flowchart TD
    A1["🟡 Klient"]
    A2["🟡 Specjalista serwisu"]
    A3["🟡 Dział jakości"]

    E1["🟠 Mail z reklamacją odebrany\n(webhook, sprawdzenie spam)"]
    E2{"Mail kompletny?\n(zdjęcie + nr zamówienia + opis)"}
    E3["🟠 Mail przekazany do specjalisty\n(brakujące dane)"]
    E4["🤖 LLM kategoryzuje wadę\n(tekst + zdjęcia)"]
    E5["🤖 Dane pobrane z SAP\n(zamówienie + batch)"]
    E6["🟠 Ticket Complaint utworzony w JIRA\n(kategoria + dane z SAP)"]
    E7{"SAP wystarcza\ndo rozstrzygnięcia?"}
    E8["🟠 Ticket Correction utworzony w JIRA\n(dla działu jakości)"]
    E9["🤖 Odpowiedź wysłana do klienta\nautomatycznie"]
    E10["🟠 Ticket przekazany do specjalisty"]
    E11["🟠 Odpowiedź wysłana do klienta\nprzez specjalistę"]

    A1 --> E1
    E1 --> E2
    E2 -->|Nie| E3
    E3 --> A2
    E2 -->|Tak| E4
    A2 -->|uzupełnia| E4
    E4 --> E5
    E5 --> E6
    E6 --> E7
    E7 -->|Nie| E10
    E10 --> A2
    E7 -->|Tak| E8
    E8 --> A3
    E8 --> E9
    E9 --> A1
    A2 --> E11
    E11 --> A1

```
