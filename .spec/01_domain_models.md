# 01 – Modele Domenowe (Domain Models)

> **Status:** `v1.0 – baseline`  
> **Ostatnia aktualizacja:** 2026-10-15  
> **Właściciel:** Eryk Poczykowski

---

## Diagram encji i relacji

```
City (1) ──< (N) Cinema (1) ──< (N) Hall (1) ──< (N) Seat
                                     │
                              (N) Screening (N) >── (1) Movie
                                     │
                              (N) Reservation (1) ──< (N) ReservedSeat
                                     │
                                   User (1)
```

---

## 1. App: `users`

### 1.1. Model `User`

Rozszerza `AbstractUser` z Django.

| Pole         | Typ                                        | Ograniczenia / Uwagi                          |
|--------------|--------------------------------------------|-----------------------------------------------|
| `id`         | `AutoField` (PK)                           | Auto-generowane                               |
| `email`      | `EmailField`                               | `unique=True`; używany jako login             |
| `username`   | `CharField(150)`                           | Wymagane przez `AbstractUser`; zachowane      |
| `first_name` | `CharField(150)`                           | Opcjonalne                                    |
| `last_name`  | `CharField(150)`                           | Opcjonalne                                    |
| `role`       | `CharField(10)`, choices=`ROLE_CHOICES`    | Wartości: `'admin'`, `'client'`. Default: `'client'` |
| `is_active`  | `BooleanField`                             | Default: `True`                               |
| `created_at` | `DateTimeField(auto_now_add=True)`         | Automatyczne                                  |
| `updated_at` | `DateTimeField(auto_now=True)`             | Automatyczne                                  |

**Stałe:**
```python
ROLE_CHOICES = [
    ('admin', 'Administrator'),
    ('client', 'Klient'),
]
```

**Właściwości:**
- `is_admin` → `property` zwracająca `self.role == 'admin'`
- `is_client` → `property` zwracająca `self.role == 'client'`

**Uwagi:**
- `AUTH_USER_MODEL = 'users.User'` musi być ustawione w `settings.py`
- Logowanie odbywa się przez pole `email` (nie `username`). Wymaga ustawienia `AUTHENTICATION_BACKENDS` lub nadpisania `USERNAME_FIELD = 'email'`
- Pole `username` pozostaje wymagane przez `AbstractUser`, jednak nie jest używane do logowania

---

## 2. App: `cinema`

### 2.1. Model `City`

| Pole   | Typ                  | Ograniczenia / Uwagi       |
|--------|----------------------|----------------------------|
| `id`   | `AutoField` (PK)     | Auto-generowane            |
| `name` | `CharField(100)`     | `unique=True`              |
| `slug` | `SlugField(100)`     | `unique=True`; auto-generowany z `name` |

**Meta:** `ordering = ['name']`, `verbose_name = 'Miasto'`

---

### 2.2. Model `Cinema`

| Pole      | Typ                  | Ograniczenia / Uwagi                  |
|-----------|----------------------|---------------------------------------|
| `id`      | `AutoField` (PK)     | Auto-generowane                       |
| `city`    | `ForeignKey(City)`   | `on_delete=CASCADE`, `related_name='cinemas'` |
| `name`    | `CharField(200)`     | Nazwa kina, np. "Helios Galeria Mokotów" |
| `address` | `CharField(300)`     | Pełny adres                           |
| `created_at` | `DateTimeField(auto_now_add=True)` | Automatyczne          |
| `updated_at` | `DateTimeField(auto_now=True)`    | Automatyczne          |

**Meta:** `ordering = ['city', 'name']`, `verbose_name = 'Kino'`  
**Unique together:** `('city', 'name')` — dwa kina w tym samym mieście nie mogą mieć identycznej nazwy

---

### 2.3. Model `Hall`

Sala kinowa należąca do konkretnego kina.

| Pole             | Typ                     | Ograniczenia / Uwagi                           |
|------------------|-------------------------|------------------------------------------------|
| `id`             | `AutoField` (PK)        | Auto-generowane                                |
| `cinema`         | `ForeignKey(Cinema)`    | `on_delete=CASCADE`, `related_name='halls'`    |
| `name`           | `CharField(50)`         | Np. "Sala 1", "Sala VIP"                       |
| `rows`           | `PositiveSmallIntegerField` | Liczba rzędów, min=1, max=50              |
| `seats_per_row`  | `PositiveSmallIntegerField` | Liczba miejsc w rzędzie, min=1, max=50    |

**Meta:** `ordering = ['cinema', 'name']`, `verbose_name = 'Sala'`  
**Unique together:** `('cinema', 'name')` — sala o danej nazwie jest unikalna w ramach kina  
**Właściwość:** `total_seats` → `property` zwracająca `self.rows * self.seats_per_row`

---

### 2.4. Model `Seat`

Pojedyncze miejsce w sali kinowej. Generowane automatycznie przy tworzeniu sali.

| Pole     | Typ                  | Ograniczenia / Uwagi                       |
|----------|----------------------|--------------------------------------------|
| `id`     | `AutoField` (PK)     | Auto-generowane                            |
| `hall`   | `ForeignKey(Hall)`   | `on_delete=CASCADE`, `related_name='seats'`|
| `row`    | `PositiveSmallIntegerField` | Numer rzędu (1-indexed)             |
| `number` | `PositiveSmallIntegerField` | Numer miejsca w rzędzie (1-indexed) |

**Meta:** `ordering = ['hall', 'row', 'number']`, `verbose_name = 'Miejsce'`  
**Unique together:** `('hall', 'row', 'number')` — każde miejsce jest unikalne w sali  
**Właściwość:** `label` → `property` zwracająca np. `"R3/M7"` (rząd 3, miejsce 7)

> [!NOTE]
> Miejsca (`Seat`) są tworzone automatycznie w sygnale `post_save` modelu `Hall` lub przez dedykowaną metodę serwisową `Hall.generate_seats()`. Nie są tworzone ręcznie przez API.

---

## 3. App: `movies`

### 3.1. Model `Genre`

| Pole   | Typ              | Ograniczenia / Uwagi |
|--------|------------------|----------------------|
| `id`   | `AutoField` (PK) | Auto-generowane      |
| `name` | `CharField(50)`  | `unique=True`        |

**Meta:** `ordering = ['name']`, `verbose_name = 'Gatunek'`

---

### 3.2. Model `Movie`

| Pole           | Typ                       | Ograniczenia / Uwagi                          |
|----------------|---------------------------|-----------------------------------------------|
| `id`           | `AutoField` (PK)          | Auto-generowane                               |
| `title`        | `CharField(255)`          | Wymagane                                      |
| `genre`        | `ForeignKey(Genre)`       | `on_delete=PROTECT`, `related_name='movies'`  |
| `duration_min` | `PositiveSmallIntegerField` | Czas trwania w minutach, min=1              |
| `description`  | `TextField`               | `blank=True`                                  |
| `age_rating`   | `CharField(5)`, choices   | Wartości: `'G'`, `'PG'`, `'PG-13'`, `'R'`, `'NC-17'` |
| `release_date` | `DateField`               | Data premiery                                 |
| `created_at`   | `DateTimeField(auto_now_add=True)` | Automatyczne                       |
| `updated_at`   | `DateTimeField(auto_now=True)`    | Automatyczne                       |

**Meta:** `ordering = ['-release_date']`, `verbose_name = 'Film'`

---

## 4. App: `screenings`

### 4.1. Model `Screening`

Seans filmowy – połączenie filmu, sali i czasu.

| Pole         | Typ                    | Ograniczenia / Uwagi                                      |
|--------------|------------------------|-----------------------------------------------------------|
| `id`         | `AutoField` (PK)       | Auto-generowane                                           |
| `movie`      | `ForeignKey(Movie)`    | `on_delete=PROTECT`, `related_name='screenings'`          |
| `hall`       | `ForeignKey(Hall)`     | `on_delete=PROTECT`, `related_name='screenings'`          |
| `start_time` | `DateTimeField`        | Data i godzina rozpoczęcia seansu                         |
| `end_time`   | `DateTimeField`        | Obliczane: `start_time + duration`. Tylko do odczytu.     |
| `price`      | `DecimalField(6,2)`    | Cena biletu w PLN, min=0.01                               |
| `is_active`  | `BooleanField`         | Default: `True`. Nieaktywny = niewidoczny dla klientów    |
| `created_at` | `DateTimeField(auto_now_add=True)` | Automatyczne                                  |
| `updated_at` | `DateTimeField(auto_now=True)`    | Automatyczne                                  |

**Meta:** `ordering = ['start_time']`, `verbose_name = 'Seans'`

**Właściwości:**
- `end_time` → `property` obliczające `start_time + timedelta(minutes=movie.duration_min)`
- `available_seats_count` → liczba wolnych miejsc (wszystkie miejsca sali minus zarezerwowane)

**Walidacja modelu (`clean()`):**
- `start_time` musi być w przyszłości przy tworzeniu
- W danej sali nie może być dwóch aktywnych seansów nakładających się czasowo (kolizja czasowa)

> [!IMPORTANT]
> Pole `end_time` jest **wyliczane** i nie jest przechowywane w bazie danych. Jest właściwością (property) modelu.

---

## 5. App: `reservations`

### 5.1. Model `Reservation`

Głowica rezerwacji — łączy użytkownika z seansem.

| Pole          | Typ                       | Ograniczenia / Uwagi                                     |
|---------------|---------------------------|----------------------------------------------------------|
| `id`          | `AutoField` (PK)          | Auto-generowane                                          |
| `user`        | `ForeignKey(User)`        | `on_delete=PROTECT`, `related_name='reservations'`       |
| `screening`   | `ForeignKey(Screening)`   | `on_delete=PROTECT`, `related_name='reservations'`       |
| `status`      | `CharField(20)`, choices  | Wartości: `'confirmed'`, `'cancelled'`. Default: `'confirmed'` |
| `total_price` | `DecimalField(8,2)`       | Obliczane i zapisywane przy tworzeniu rezerwacji         |
| `created_at`  | `DateTimeField(auto_now_add=True)` | Automatyczne                                   |
| `updated_at`  | `DateTimeField(auto_now=True)`    | Automatyczne                                   |

**Stałe:**
```python
STATUS_CHOICES = [
    ('confirmed', 'Potwierdzona'),
    ('cancelled', 'Anulowana'),   # zarezerwowane na przyszłość
]
```

**Meta:** `ordering = ['-created_at']`, `verbose_name = 'Rezerwacja'`

> [!NOTE]
> Pole `status = 'cancelled'` jest **zarezerwowane do przyszłej implementacji** funkcji anulowania. W obecnej wersji API anulowanie nie jest dostępne. Rezerwacja po utworzeniu ma zawsze status `'confirmed'`.

---

### 5.2. Model `ReservedSeat`

Pojedyncze zarezerwowane miejsce w ramach rezerwacji.

| Pole          | Typ                         | Ograniczenia / Uwagi                                        |
|---------------|-----------------------------|-------------------------------------------------------------|
| `id`          | `AutoField` (PK)            | Auto-generowane                                             |
| `reservation` | `ForeignKey(Reservation)`   | `on_delete=CASCADE`, `related_name='reserved_seats'`        |
| `seat`        | `ForeignKey(Seat)`          | `on_delete=PROTECT`, `related_name='reserved_seats'`        |

**Meta:** `verbose_name = 'Zarezerwowane miejsce'`  
**Unique together:** `('reservation__screening', 'seat')` — realizowane przez constraint na poziomie bazy:

```python
class Meta:
    constraints = [
        models.UniqueConstraint(
            fields=['seat'],
            condition=models.Q(reservation__status='confirmed'),
            name='unique_active_seat_per_screening'
        )
    ]
```

> [!IMPORTANT]
> Unikalność zajętości miejsca per seans jest egzekwowana przez niestandardowy `UniqueConstraint` z `condition`. Szczegóły w `03_business_invariants.md`.
