# Feature: Repertuar Kinowy (`Screening`)

> **Status:** `v1.0 – baseline`  
> **Ostatnia aktualizacja:** 2026-10-15  
> **Powiązane modele:** `Screening`, `Movie`, `Hall`, `Cinema`, `City`  
> **Powiązane niezmienniki:** BI-07

---

## 1. Opis funkcjonalności

Repertuar to centralny punkt aplikacji. Użytkownik (zalogowany lub nie) może:
- Przeglądać dostępne seanse filtrowane po mieście, kinie, filmie lub dacie
- Sprawdzić mapę dostępności miejsc dla konkretnego seansu

Administrator może zarządzać seansami (CRUD).

---

## 2. Model danych (podsumowanie)

```
Screening
  ├── movie (FK → Movie)
  ├── hall (FK → Hall → Cinema → City)
  ├── start_time (DateTimeField)
  ├── end_time (property: start_time + movie.duration_min)
  ├── price (Decimal)
  └── is_active (bool)
```

---

## 3. Kontrakt API (podsumowanie)

| Metoda | Endpoint                          | Rola | Opis                               |
|--------|-----------------------------------|------|------------------------------------|
| GET    | `/api/v1/screenings/`             | 🌐   | Lista aktywnych seansów (repertuar)|
| GET    | `/api/v1/screenings/{id}/`        | 🌐   | Szczegóły seansu                   |
| GET    | `/api/v1/screenings/{id}/seats/`  | 🌐   | Mapa dostępności miejsc            |
| POST   | `/api/v1/screenings/`             | 👑   | Utwórz seans                       |
| PATCH  | `/api/v1/screenings/{id}/`        | 👑   | Edytuj seans                       |
| DELETE | `/api/v1/screenings/{id}/`        | 👑   | Usuń seans                         |

---

## 4. Filtry repertuaru

Endpoint `GET /api/v1/screenings/` obsługuje następujące parametry zapytania:

| Parametr | Typ    | Opis                                                    |
|----------|--------|---------------------------------------------------------|
| `cinema` | int    | ID kina                                                 |
| `city`   | int    | ID miasta (filtruje przez hall__cinema__city)           |
| `movie`  | int    | ID filmu                                                |
| `date`   | date   | Format YYYY-MM-DD — filtruje seanse na dany dzień       |

Domyślnie endpoint zwraca **wyłącznie aktywne seanse** (`is_active=True`) **w przyszłości** (`start_time > now()`).

---

## 5. Mapa dostępności miejsc

Endpoint `GET /api/v1/screenings/{id}/seats/` zwraca informacje o zajętości każdego miejsca.

**Implementacja zapytania:**
```python
# Efektywne zapytanie — jeden JOIN zamiast N+1
taken_seat_ids = ReservedSeat.objects.filter(
    reservation__screening=screening,
    reservation__status='confirmed'
).values_list('seat_id', flat=True)

seats = Seat.objects.filter(hall=screening.hall).annotate(
    is_available=Case(
        When(id__in=taken_seat_ids, then=Value(False)),
        default=Value(True),
        output_field=BooleanField()
    )
)
```

---

## 6. Kryteria Akceptacji (Given-When-Then)

### AC-01: Przeglądanie repertuaru bez logowania

```gherkin
Given istnieje seans aktywny w przyszłości
When anonimowy użytkownik wysyła GET /api/v1/screenings/
Then otrzymuje 200 OK
  And lista zawiera ten seans
  And odpowiedź zawiera pola: id, movie, hall, start_time, end_time, price, available_seats, total_seats
```

### AC-02: Filtrowanie repertuaru po mieście

```gherkin
Given istnieją seanse w Warszawie (city_id=1) i Krakowie (city_id=2)
When GET /api/v1/screenings/?city=1
Then lista zawiera tylko seanse z Warszawy
```

### AC-03: Filtrowanie repertuaru po dacie

```gherkin
Given seanse odbywają się 2026-10-20 i 2026-10-21
When GET /api/v1/screenings/?date=2026-10-20
Then lista zawiera tylko seanse z dnia 2026-10-20
```

### AC-04: Nieaktywne seanse ukryte dla publicznych użytkowników

```gherkin
Given seans o ID=5 ma is_active=False
When anonimowy użytkownik wysyła GET /api/v1/screenings/
Then seans 5 NIE jest na liście
When anonimowy użytkownik wysyła GET /api/v1/screenings/5/
Then otrzymuje 404 Not Found
```

### AC-05: Mapa dostępności miejsc

```gherkin
Given seans o ID=1 ma salę z 3 miejscami (ID: 1, 2, 3)
  And miejsce 2 jest zajęte (aktywna rezerwacja)
When GET /api/v1/screenings/1/seats/
Then otrzymuje 200 OK
  And seats[0] = {id: 1, is_available: true}
  And seats[1] = {id: 2, is_available: false}
  And seats[2] = {id: 3, is_available: true}
  And available_seats == 2
  And total_seats == 3
```

### AC-06: Kolizja czasowa przy tworzeniu seansu

```gherkin
Given w sali 1 istnieje seans od 18:00 do 20:46
When admin wysyła POST /api/v1/screenings/ z hall=1, start_time=20:00
Then otrzymuje 400 Bad Request
  And w body komunikat o kolizji czasowej
```

### AC-07: Admin widzi nieaktywne seanse

```gherkin
Given seans o ID=5 ma is_active=False
When admin wysyła GET /api/v1/screenings/ (bez filtrów)
Then seans 5 JEST na liście
```

> [!NOTE]
> Rozróżnienie: endpoint publiczny `/api/v1/screenings/` dla klientów filtruje `is_active=True`. Admin widzi wszystko — zrealizowane przez nadpisanie `get_queryset()` w ViewSet z sprawdzeniem roli.

---

## 7. Szczegóły implementacji

### 7.1. ViewSet z akcją `seats`

```python
# apps/screenings/views.py

class ScreeningViewSet(viewsets.ModelViewSet):
    serializer_class = ScreeningListSerializer
    
    def get_queryset(self):
        qs = Screening.objects.select_related('movie', 'hall__cinema__city')
        
        # Klient widzi tylko aktywne, przyszłe seanse
        if not (self.request.user.is_authenticated and self.request.user.is_admin):
            qs = qs.filter(is_active=True, start_time__gt=timezone.now())
        
        # Filtry
        cinema_id = self.request.query_params.get('cinema')
        city_id = self.request.query_params.get('city')
        movie_id = self.request.query_params.get('movie')
        date = self.request.query_params.get('date')
        
        if cinema_id:
            qs = qs.filter(hall__cinema_id=cinema_id)
        if city_id:
            qs = qs.filter(hall__cinema__city_id=city_id)
        if movie_id:
            qs = qs.filter(movie_id=movie_id)
        if date:
            qs = qs.filter(start_time__date=date)
        
        return qs
    
    @action(detail=True, methods=['get'], url_path='seats')
    def seats(self, request, pk=None):
        screening = self.get_object()
        # ... logika mapy miejsc
```

---

## 8. Rozszerzalność (Future Scope)

Następujące funkcje są **zaplanowane do przyszłej implementacji**:

- [ ] System ocen i recenzji filmów (`MovieReview`)
- [ ] Lista oczekujących (`WaitlistEntry`) gdy seans wyprzedany
- [ ] Filtry po gatunku filmowym w repertuarze
- [ ] Eksport repertuaru do iCal/PDF
