# 03 – Niezmienniki Biznesowe (Business Invariants)

> **Status:** `v1.0 – baseline`  
> **Ostatnia aktualizacja:** 2026-10-15  
> **Właściciel:** Eryk Poczykowski

---

> [!IMPORTANT]
> Poniższe reguły są **Niezmienialne** — naruszenie któregokolwiek z nich stanowi błąd krytyczny. Każdy agent AI otrzymujący zadanie implementacyjne musi bezwzględnie przestrzegać tych zasad. Niezmienniki są weryfikowane przez testy automatyczne `APITestCase`.

---

## BI-01: Zakaz overbookingu (Atomic Seat Reservation)

**Opis:** Żadne miejsce w danym seansie nie może być przypisane do więcej niż jednej aktywnej rezerwacji (`status='confirmed'`).

**Reguła:**
```
∀ seat ∈ Seat, ∀ screening ∈ Screening:
  COUNT(ReservedSeat WHERE seat=seat AND reservation__screening=screening AND reservation__status='confirmed') ≤ 1
```

**Implementacja:**
- Rezerwacja miejsc musi być wykonana wewnątrz bloku `transaction.atomic()` z `select_for_update()` na zajętych miejscach, aby zapobiec wyścigowi danych (race condition) przy współbieżnych żądaniach
- Naruszenie przez `IntegrityError` (unikalny constraint bazy danych) musi być przechwytywane i zwracane jako `409 Conflict`

**Pseudokod implementacji:**
```python
from django.db import transaction, IntegrityError

def reserve_seats(user, screening_id, seat_ids):
    with transaction.atomic():
        # Zablokuj wiersze, aby zapobiec race condition
        screening = Screening.objects.select_for_update().get(id=screening_id)
        
        # Sprawdź czy miejsca są wolne
        already_taken = ReservedSeat.objects.filter(
            reservation__screening=screening,
            reservation__status='confirmed',
            seat_id__in=seat_ids
        ).exists()
        
        if already_taken:
            raise SeatAlreadyTakenError("Jedno lub więcej miejsc jest już zajętych")
        
        # Utwórz rezerwację
        reservation = Reservation.objects.create(
            user=user,
            screening=screening,
            total_price=screening.price * len(seat_ids)
        )
        ReservedSeat.objects.bulk_create([
            ReservedSeat(reservation=reservation, seat_id=sid)
            for sid in seat_ids
        ])
        return reservation
```

**Test weryfikujący:**
- `test_concurrent_reservation_returns_409_for_second_request`
- `test_reserve_already_taken_seat_returns_409`

---

## BI-02: Walidacja przynależności miejsc do seansu

**Opis:** Miejsca podane w żądaniu rezerwacji muszą należeć do sali, w której odbywa się seans.

**Reguła:**
```
∀ seat_id ∈ request.seat_ids:
  Seat.objects.get(id=seat_id).hall == Screening.objects.get(id=screening_id).hall
```

**Implementacja:** Walidacja w serializerze rezerwacji w metodzie `validate()`.

**Błąd:** `400 Bad Request` z komunikatem `"Miejsce o ID {id} nie należy do sali tego seansu."`

**Test weryfikujący:**
- `test_reserve_seat_from_wrong_hall_returns_400`

---

## BI-03: Zakaz rezerwacji w nieaktywnym seansie

**Opis:** Rezerwacja może być dokonana wyłącznie dla seansu z flagą `is_active=True`.

**Reguła:**
```
Screening.is_active == True → rezerwacja dozwolona
Screening.is_active == False → rezerwacja niemożliwa (400)
```

**Implementacja:** Walidacja w serializerze lub widoku rezerwacji.

**Błąd:** `400 Bad Request` z komunikatem `"Seans jest nieaktywny."`

**Test weryfikujący:**
- `test_reserve_inactive_screening_returns_400`

---

## BI-04: Zakaz rezerwacji przeszłego seansu

**Opis:** Nie można dokonać rezerwacji na seans, który już się rozpoczął lub zakończył.

**Reguła:**
```
Screening.start_time > timezone.now() → rezerwacja dozwolona
Screening.start_time ≤ timezone.now() → rezerwacja niemożliwa (400)
```

**Implementacja:** Walidacja w serializerze rezerwacji.

**Błąd:** `400 Bad Request` z komunikatem `"Seans już się rozpoczął lub zakończył."`

**Test weryfikujący:**
- `test_reserve_past_screening_returns_400`

---

## BI-05: Minimalna liczba miejsc w rezerwacji

**Opis:** Rezerwacja musi obejmować przynajmniej jedno miejsce.

**Reguła:**
```
len(request.seat_ids) ≥ 1
```

**Implementacja:** Walidacja w serializerze rezerwacji.

**Błąd:** `400 Bad Request` z komunikatem `"Rezerwacja musi zawierać co najmniej jedno miejsce."`

**Test weryfikujący:**
- `test_reserve_with_empty_seat_list_returns_400`

---

## BI-06: Obliczanie ceny całkowitej

**Opis:** Cena całkowita rezerwacji jest zawsze wyliczana przez backend — klient nie może jej podać.

**Reguła:**
```
Reservation.total_price = Screening.price × len(seat_ids)
```

**Implementacja:** Pole `total_price` jest `read_only` w serializerze. Wartość obliczana jest w metodzie `create()` serializera lub w metodzie serwisowej.

**Test weryfikujący:**
- `test_total_price_is_calculated_correctly`

---

## BI-07: Kolizja czasowa seansów w tej samej sali

**Opis:** W danej sali nie mogą odbywać się dwa aktywne seanse nakładające się czasowo.

**Reguła:**
```
∀ s1, s2 ∈ Screening WHERE s1.hall == s2.hall AND s1 ≠ s2 AND s1.is_active AND s2.is_active:
  s1.end_time ≤ s2.start_time OR s2.end_time ≤ s1.start_time
```

**Implementacja:** Walidacja w metodzie `clean()` modelu `Screening` oraz w serializerze (dla API).

**Błąd:** `400 Bad Request` z komunikatem `"W tej sali istnieje już seans w podanym przedziale czasowym."`

**Test weryfikujący:**
- `test_create_overlapping_screening_returns_400`

---

## BI-08: Izolacja danych użytkowników (Data Isolation)

**Opis:** Użytkownik z rolą `client` może zobaczyć i zarządzać wyłącznie własnymi rezerwacjami.

**Reguła:**
```
GET /api/v1/reservations/ → zwraca tylko Reservation WHERE user == request.user (dla roli 'client')
GET /api/v1/reservations/{id}/ → zwraca 403 jeśli Reservation.user != request.user (dla roli 'client')
```

**Implementacja:** Nadpisanie `get_queryset()` w `ReservationViewSet`:
```python
def get_queryset(self):
    if self.request.user.is_admin:
        return Reservation.objects.all()
    return Reservation.objects.filter(user=self.request.user)
```

**Test weryfikujący:**
- `test_client_cannot_access_other_users_reservation_returns_403`

---

## BI-09: Rola `client` przy rejestracji

**Opis:** Każdy nowo zarejestrowany użytkownik przez publiczny endpoint rejestracji otrzymuje rolę `client`. Rola `admin` może być nadana wyłącznie przez innego administratora przez panel Django Admin lub dedykowany endpoint (poza zakresem v1).

**Reguła:**
```
POST /api/v1/auth/register/ → User.role = 'client' (zawsze, ignorując pole 'role' w body)
```

**Implementacja:** Pole `role` jest ignorowane lub ustawiane na `'client'` w serialzerze rejestracji.

**Test weryfikujący:**
- `test_registration_always_creates_client_role`
- `test_registration_with_admin_role_in_body_still_creates_client`

---

## BI-10: Unikalność miejsc w liście rezerwacji

**Opis:** Lista `seat_ids` w żądaniu rezerwacji nie może zawierać duplikatów.

**Reguła:**
```
len(request.seat_ids) == len(set(request.seat_ids))
```

**Implementacja:** Walidacja w serializerze rezerwacji.

**Błąd:** `400 Bad Request` z komunikatem `"Lista miejsc zawiera duplikaty."`

**Test weryfikujący:**
- `test_reserve_with_duplicate_seat_ids_returns_400`
