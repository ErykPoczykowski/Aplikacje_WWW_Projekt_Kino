# 04 – Opcjonalne Funkcjonalności (Optional Features Backlog)

> **Status:** `v1.0 – living document`  
> **Ostatnia aktualizacja:** 2026-10-15  
> **Właściciel:** Eryk Poczykowski

---

> [!IMPORTANT]
> **Instrukcja dla agentów AI:**  
> Ten plik jest **wyłącznie rejestrem pomysłów** — nie implementuj żadnej z poniższych funkcji, jeśli nie zostałeś o to wprost poproszony.  
> Gdy sugerujesz rozwiązania dodatkowe lub rozszerzenia systemu, **sprawdź najpierw ten plik** — jeśli pomysł już tu figuruje, odwołaj się do odpowiedniego identyfikatora (np. `OF-03`), zamiast opisywać go od nowa.  
> Jeśli proponujesz zupełnie nową funkcję nieobecną w tym pliku, zasugeruj jej dodanie tutaj przed implementacją.

---

## Legenda statusów

| Status       | Znaczenie                                                      |
|--------------|----------------------------------------------------------------|
| `💡 Pomysł`  | Wstępny pomysł, niezweryfikowany biznesowo                     |
| `📋 Gotowy`  | Opisany wystarczająco, można pisać specyfikację w `features/`  |
| `🚧 W toku`  | Specyfikacja w `features/` jest już tworzona lub istnieje      |
| `✅ Wdrożony`| Zaimplementowany w kodzie i pokryty testami                    |
| `❌ Odrzucony`| Świadomie odrzucony z uzasadnieniem                           |

---

## Kategoria A: Rezerwacje i Płatności

### OF-01: Anulowanie rezerwacji przez klienta
**Status:** `📋 Gotowy`  
**Opis:** Użytkownik może anulować własną rezerwację, zmieniając jej status na `'cancelled'`. Miejsca wracają do puli wolnych.  
**Ograniczenia do przemyślenia:**
- Limit czasowy: np. zakaz anulowania mniej niż 30 minut przed seansem
- Polityka zwrotów (poza zakresem v1 — brak integracji płatności)

**Powiązane modele:** `Reservation.status` (pole `'cancelled'` już istnieje w modelu)  
**Potencjalny endpoint:** `DELETE /api/v1/reservations/{id}/` lub `POST /api/v1/reservations/{id}/cancel/`

---

### OF-02: Historia rezerwacji klienta
**Status:** `📋 Gotowy`  
**Opis:** Dedykowany endpoint zwracający paginowaną historię wszystkich rezerwacji zalogowanego klienta, w tym rezerwacji na minione seanse.  
**Uwaga:** Obecny `GET /api/v1/reservations/` już zwraca listę rezerwacji klienta — ta funkcja to rozszerzenie o lepszą paginację, sortowanie i filtry (np. po statusie, dacie).  
**Potencjalny endpoint:** `GET /api/v1/reservations/history/?status=confirmed&ordering=-created_at`

---

### OF-03: Ceny ulgowe (Ticket Types)
**Status:** `💡 Pomysł`  
**Opis:** Różne typy biletów z różnymi cenami: normalny, ulgowy (student/emeryt), dziecięcy.  
**Wpływ na modele:**
- Nowy model `TicketType` (name, discount_percent)
- `ReservedSeat` zyskuje FK do `TicketType`
- `Reservation.total_price` obliczane per miejsce z uwzględnieniem zniżki

---

### OF-04: Kody rabatowe (`PromoCode`)
**Status:** `💡 Pomysł`  
**Opis:** System jednorazowych lub wielokrotnych kodów rabatowych z datą ważności i limitem użyć.  
**Wzorzec implementacji:** Opisany szczegółowo w `readme.md` (warsztat SDD) — gotowy do przeniesienia do `features/promo_codes.md`.  
**Pola modelu:** `code`, `discount_percent`, `max_usages`, `usages_count`, `valid_until`, `screening` (opcjonalne powiązanie z konkretnym seansem)

---

### OF-05: Integracja płatności online
**Status:** `💡 Pomysł`  
**Opis:** Integracja z bramką płatności (np. Stripe lub PayU) — rezerwacja jest finalizowana dopiero po potwierdzeniu płatności.  
**Wpływ na modele:** Nowy status rezerwacji `'pending_payment'`, nowy model `Payment`.  
**Uwaga:** Wymaga dokładnej analizy wymagań PCI-DSS — wysokie ryzyko implementacyjne.

---

### OF-06: Maksymalna liczba miejsc na rezerwację
**Status:** `💡 Pomysł`  
**Opis:** Ograniczenie liczby miejsc możliwych do zarezerwowania w jednym żądaniu (np. max 10 biletów).  
**Implementacja:** Prosta walidacja w serializerze — niezmiennik do `03_business_invariants.md`.

---

## Kategoria B: Użytkownicy i Role

### OF-07: Rola `cinema_manager` (Menedżer Kina)
**Status:** `💡 Pomysł`  
**Opis:** Pośrednia rola między `client` a `admin`. Menedżer może zarządzać seansami i salami tylko we własnym kinie, ale nie może tworzyć nowych kin ani miast.  
**Wpływ na model:** Nowe pole `managed_cinema` (FK) w modelu `User` lub osobny model `CinemaManager`.

---

### OF-08: Zmiana hasła i reset przez email
**Status:** `📋 Gotowy`  
**Opis:** Standardowe endpointy do zmiany hasła przez zalogowanego użytkownika oraz reset hasła przez link emailowy.  
**Potencjalne endpointy:**
- `POST /api/v1/auth/change-password/`
- `POST /api/v1/auth/password-reset/`
- `POST /api/v1/auth/password-reset/confirm/`

---

### OF-09: Profil użytkownika (edycja danych)
**Status:** `📋 Gotowy`  
**Opis:** Endpoint umożliwiający klientowi edycję własnego `first_name`, `last_name` (nie emaila i roli).  
**Potencjalny endpoint:** `PATCH /api/v1/auth/me/`

---

## Kategoria C: Filmy i Repertuar

### OF-10: Plakat filmowy (`Movie.poster`)
**Status:** `💡 Pomysł`  
**Opis:** Możliwość wgrania plakatu filmowego (obraz). Wymaga konfiguracji `MEDIA_ROOT` i `django-storages` lub lokalnego serwera plików.  
**Pole modelu:** `Movie.poster = ImageField(upload_to='posters/', null=True, blank=True)`  
**Uwaga:** Wymaga instalacji `Pillow` — dodać do `00_architecture_guidelines.md` po zatwierdzeniu.

---

### OF-11: Oceny i recenzje filmów (`MovieReview`)
**Status:** `💡 Pomysł`  
**Opis:** Zalogowani klienci mogą wystawiać ocenę (1–5 gwiazdek) i opcjonalną recenzję tekstową dla filmów, na które mieli rezerwację.  
**Pola modelu:** `MovieReview` (user FK, movie FK, rating 1-5, comment TextField, created_at)  
**Niezmiennik:** Klient może wystawić max 1 recenzję na film. Recenzja możliwa tylko po seansie (start_time w przeszłości).  
**Wzorzec implementacji:** Opisany w `readme.md` (zadanie indywidualne) jako przykład SDD.

---

### OF-12: Lista oczekujących (`WaitlistEntry`)
**Status:** `💡 Pomysł`  
**Opis:** Gdy seans jest wyprzedany, klient może dołączyć do kolejki oczekujących. Gdy miejsce się zwolni (np. po anulowaniu — OF-01), pierwsza osoba z kolejki otrzymuje powiadomienie.  
**Pola modelu:** `WaitlistEntry` (user FK, screening FK, created_at, notified_at nullable)  
**Zależności:** Wymaga OF-01 (anulowanie) i OF-14 (powiadomienia email).

---

### OF-13: Format 3D / IMAX (`Screening.format`)
**Status:** `💡 Pomysł`  
**Opis:** Dodanie pola `format` do modelu `Screening` określającego typ projekcji (2D, 3D, IMAX, 4DX).  
**Pole modelu:** `format = CharField(choices=[('2D','2D'),('3D','3D'),('IMAX','IMAX'),('4DX','4DX')], default='2D')`  
**Wpływ:** Różne ceny dla różnych formatów — interakcja z OF-03.

---

## Kategoria D: Powiadomienia i Integracje

### OF-14: Powiadomienia email
**Status:** `💡 Pomysł`  
**Opis:** Automatyczne emaile po kluczowych zdarzeniach.  
**Zdarzenia:**
- Potwierdzenie rezerwacji (z listą miejsc i szczegółami seansu)
- Przypomnienie 24h przed seansem
- Potwierdzenie anulowania (gdy OF-01 wdrożony)

**Implementacja:** Django `send_mail` + Celery do wysyłki asynchronicznej. Wymaga dodania `celery` i brokera (Redis) do stacku — zatwierdź w `00_architecture_guidelines.md`.

---

### OF-15: Eksport biletów do PDF
**Status:** `💡 Pomysł`  
**Opis:** Endpoint generujący bilet w formacie PDF z kodem QR zawierającym ID rezerwacji.  
**Potencjalny endpoint:** `GET /api/v1/reservations/{id}/ticket.pdf`  
**Wymagana biblioteka:** `reportlab` lub `weasyprint` — wymaga zatwierdzenia w `00_architecture_guidelines.md`.

---

## Kategoria E: Administracja i Raportowanie

### OF-16: Statystyki dla administratora
**Status:** `💡 Pomysł`  
**Opis:** Dedykowany endpoint zwracający zagregowane dane: liczba sprzedanych biletów, przychód, obłożenie sali per seans.  
**Potencjalny endpoint:** `GET /api/v1/admin/stats/?cinema=1&date_from=2026-10-01&date_to=2026-10-31`

---

### OF-17: Masowe zarządzanie seansami (bulk import)
**Status:** `💡 Pomysł`  
**Opis:** Endpoint pozwalający administratorowi na import repertuaru z pliku CSV lub JSON na dany tydzień.  
**Potencjalny endpoint:** `POST /api/v1/screenings/bulk/`

---

## Rejestr odrzuconych pomysłów

| ID     | Pomysł                     | Powód odrzucenia                                      | Data       |
|--------|----------------------------|-------------------------------------------------------|------------|
| —      | —                          | —                                                     | —          |

> Dodawaj tu pomysły, które zostały świadomie odrzucone, wraz z uzasadnieniem, aby unikać powracania do tych samych dyskusji.
