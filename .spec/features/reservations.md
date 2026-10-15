# Feature: System Rezerwacji Miejsc (`Reservation`)

> **Status:** `v1.0 – baseline`  
> **Ostatnia aktualizacja:** 2026-10-15  
> **Powiązane modele:** `Reservation`, `ReservedSeat`, `Seat`, `Screening`  
> **Powiązane niezmienniki:** BI-01, BI-02, BI-03, BI-04, BI-05, BI-06, BI-08, BI-10

---

## 1. Opis funkcjonalności

System rezerwacji umożliwia zalogowanym użytkownikom (rola `client`) zarezerwowanie konkretnych miejsc na wybranym seansie. Rezerwacja jest atomowa — albo wszystkie miejsca są zarezerwowane, albo żadne.

---

## 2. Model danych (podsumowanie)

```
Reservation
  ├── user (FK → User)
  ├── screening (FK → Screening)
  ├── status: 'confirmed' | 'cancelled'
  ├── total_price (Decimal, obliczane przez backend)
  └── ReservedSeat[]
        ├── reservation (FK)
        └── seat (FK → Seat)
```

---

## 3. Kontrakt API (podsumowanie)

| Metoda | Endpoint                     | Rola    | Opis                                      |
|--------|------------------------------|---------|-------------------------------------------|
| GET    | `/api/v1/reservations/`      | 🔐      | Lista moich rezerwacji (admin: wszystkie)  |
| POST   | `/api/v1/reservations/`      | 🔐      | Utwórz rezerwację                          |
| GET    | `/api/v1/reservations/{id}/` | 🔐      | Szczegóły rezerwacji (tylko właściciel/admin) |

---

## 4. Niezmienniki (Invariants)

Pełne definicje w `03_business_invariants.md`. Skrót:

1. **BI-01** — zakaz overbookingu (SELECT FOR UPDATE + atomic)
2. **BI-02** — miejsca muszą należeć do sali seansu
3. **BI-03** — seans musi być aktywny
4. **BI-04** — seans nie może być w przeszłości
5. **BI-05** — min. 1 miejsce w rezerwacji
6. **BI-06** — `total_price` obliczane przez backend
7. **BI-08** — izolacja danych (client widzi tylko swoje)
8. **BI-10** — brak duplikatów w `seat_ids`

---

## 5. Kryteria Akceptacji (Given-When-Then)

### AC-01: Pomyślna rezerwacja

```gherkin
Given zalogowany użytkownik z rolą 'client'
  And seans o ID=1 jest aktywny i w przyszłości
  And miejsca [5, 6] należą do sali seansu
  And miejsca [5, 6] są wolne
When wysyła POST /api/v1/reservations/ z body {"screening": 1, "seat_ids": [5, 6]}
Then otrzymuje 201 Created
  And w body jest "status": "confirmed"
  And w body jest "total_price": "50.00" (2 × 25.00 PLN)
  And w body jest lista "seats" zawierająca miejsca 5 i 6
  And w bazie istnieje Reservation z 2 obiektami ReservedSeat
```

### AC-02: Próba zajętego miejsca

```gherkin
Given zalogowany użytkownik A zarezerwował miejsce 5 na seansie 1
When zalogowany użytkownik B wysyła POST /api/v1/reservations/ z body {"screening": 1, "seat_ids": [5]}
Then otrzymuje 409 Conflict
  And w body jest "code": "seat_already_taken"
  And w bazie NIE istnieje nowa Reservation
```

### AC-03: Miejsce spoza sali seansu

```gherkin
Given seans 1 odbywa się w sali A
  And miejsce 99 należy do sali B
When zalogowany użytkownik wysyła POST /api/v1/reservations/ z body {"screening": 1, "seat_ids": [99]}
Then otrzymuje 400 Bad Request
  And w body jest komunikat o nieprzynależności miejsca do sali
```

### AC-04: Seans nieaktywny

```gherkin
Given seans o ID=2 ma is_active=False
When zalogowany użytkownik wysyła POST /api/v1/reservations/ z body {"screening": 2, "seat_ids": [1]}
Then otrzymuje 400 Bad Request
  And w body jest komunikat o nieaktywnym seansie
```

### AC-05: Seans w przeszłości

```gherkin
Given seans o ID=3 ma start_time w przeszłości
When zalogowany użytkownik wysyła POST /api/v1/reservations/ z body {"screening": 3, "seat_ids": [1]}
Then otrzymuje 400 Bad Request
  And w body jest komunikat o zakończonym seansie
```

### AC-06: Izolacja danych

```gherkin
Given użytkownik A ma rezerwację o ID=10
When użytkownik B wysyła GET /api/v1/reservations/10/
Then otrzymuje 404 Not Found lub 403 Forbidden
  And dane rezerwacji użytkownika A nie są ujawnione
```

### AC-07: Bez autentykacji

```gherkin
Given żądanie bez nagłówka Authorization
When wysyła POST /api/v1/reservations/ z body {"screening": 1, "seat_ids": [1]}
Then otrzymuje 401 Unauthorized
```

### AC-08: Pusta lista miejsc

```gherkin
When zalogowany użytkownik wysyła POST /api/v1/reservations/ z body {"screening": 1, "seat_ids": []}
Then otrzymuje 400 Bad Request
  And w body jest komunikat o wymaganiu min. 1 miejsca
```

---

## 6. Szczegóły implementacji

### 6.1. Metoda `create` w serializerze

```python
# apps/reservations/serializers.py

class ReservationCreateSerializer(serializers.Serializer):
    screening = serializers.PrimaryKeyRelatedField(queryset=Screening.objects.all())
    seat_ids = serializers.ListField(
        child=serializers.IntegerField(),
        min_length=1,
        error_messages={'min_length': "Rezerwacja musi zawierać co najmniej jedno miejsce."}
    )

    def validate(self, data):
        screening = data['screening']
        seat_ids = data['seat_ids']

        # BI-10: brak duplikatów
        if len(seat_ids) != len(set(seat_ids)):
            raise serializers.ValidationError("Lista miejsc zawiera duplikaty.")

        # BI-03: seans aktywny
        if not screening.is_active:
            raise serializers.ValidationError("Seans jest nieaktywny.")

        # BI-04: seans w przyszłości
        if screening.start_time <= timezone.now():
            raise serializers.ValidationError("Seans już się rozpoczął lub zakończył.")

        # BI-02: miejsca należą do sali seansu
        seats = Seat.objects.filter(id__in=seat_ids)
        if seats.count() != len(seat_ids):
            raise serializers.ValidationError("Jedno lub więcej miejsc nie istnieje.")
        for seat in seats:
            if seat.hall != screening.hall:
                raise serializers.ValidationError(
                    f"Miejsce o ID {seat.id} nie należy do sali tego seansu."
                )
        data['seats'] = seats
        return data

    def create(self, validated_data):
        # Wywołanie serwisu z logiką atomową (BI-01)
        from apps.reservations.services import ReservationService
        return ReservationService.create_reservation(
            user=self.context['request'].user,
            screening=validated_data['screening'],
            seats=validated_data['seats'],
        )
```

### 6.2. Serwis atomowy

```python
# apps/reservations/services.py

class ReservationService:
    @staticmethod
    @transaction.atomic
    def create_reservation(user, screening, seats):
        seat_ids = [s.id for s in seats]

        # BI-01: blokada pessimistyczna
        Screening.objects.select_for_update().get(id=screening.id)

        already_taken = ReservedSeat.objects.filter(
            reservation__screening=screening,
            reservation__status='confirmed',
            seat_id__in=seat_ids
        ).exists()

        if already_taken:
            raise SeatAlreadyTakenError()

        reservation = Reservation.objects.create(
            user=user,
            screening=screening,
            total_price=screening.price * len(seat_ids),
        )
        ReservedSeat.objects.bulk_create([
            ReservedSeat(reservation=reservation, seat=seat)
            for seat in seats
        ])
        return reservation
```

---

## 7. Rozszerzalność (Future Scope)

Następujące funkcje są **zaplanowane do przyszłej implementacji** i nie wolno ich implementować teraz:

- [ ] `DELETE /api/v1/reservations/{id}/` — anulowanie rezerwacji ze zmianą statusu na `'cancelled'`
- [ ] Limit czasu na anulowanie (np. 30 min przed seansem)
- [ ] Powiadomienie email po rezerwacji
- [ ] Kody rabatowe (`PromoCode`)
