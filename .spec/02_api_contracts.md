# 02 – Kontrakty API (API Contracts)

> **Status:** `v1.0 – baseline`  
> **Ostatnia aktualizacja:** 2026-10-15  
> **Właściciel:** Eryk Poczykowski  
> **Base URL:** `/api/v1/`

---

## Legenda uprawnień

| Symbol | Znaczenie                                    |
|--------|----------------------------------------------|
| 🌐     | Publiczny — bez autentykacji                 |
| 🔐     | Wymaga JWT (dowolna rola)                    |
| 👑     | Wymaga JWT z rolą `admin`                    |

---

## 1. Autentykacja (`/api/v1/auth/`)

### `POST /api/v1/auth/register/` 🌐

Rejestracja nowego użytkownika. Nowi użytkownicy zawsze otrzymują rolę `client`.

**Request body:**
```json
{
  "email": "jan.kowalski@example.com",
  "username": "jan_kowalski",
  "password": "SecurePass123!",
  "password_confirm": "SecurePass123!",
  "first_name": "Jan",
  "last_name": "Kowalski"
}
```

**Response `201 Created`:**
```json
{
  "id": 1,
  "email": "jan.kowalski@example.com",
  "username": "jan_kowalski",
  "first_name": "Jan",
  "last_name": "Kowalski",
  "role": "client"
}
```

**Błędy:**
- `400` — email już zajęty, hasła się nie zgadzają, brak wymaganych pól

---

### `POST /api/v1/auth/login/` 🌐

Logowanie i uzyskanie tokenów JWT.

**Request body:**
```json
{
  "email": "jan.kowalski@example.com",
  "password": "SecurePass123!"
}
```

**Response `200 OK`:**
```json
{
  "access": "<access_token>",
  "refresh": "<refresh_token>"
}
```

**Błędy:**
- `401` — nieprawidłowe dane logowania

---

### `POST /api/v1/auth/token/refresh/` 🌐

Odświeżenie access tokena.

**Request body:**
```json
{
  "refresh": "<refresh_token>"
}
```

**Response `200 OK`:**
```json
{
  "access": "<nowy_access_token>"
}
```

---

### `GET /api/v1/auth/me/` 🔐

Pobranie danych aktualnie zalogowanego użytkownika.

**Response `200 OK`:**
```json
{
  "id": 1,
  "email": "jan.kowalski@example.com",
  "username": "jan_kowalski",
  "first_name": "Jan",
  "last_name": "Kowalski",
  "role": "client"
}
```

---

## 2. Miasta (`/api/v1/cities/`)

### `GET /api/v1/cities/` 🌐

Lista wszystkich miast z kinami.

**Query params:** brak  
**Response `200 OK`:**
```json
{
  "count": 3,
  "next": null,
  "previous": null,
  "results": [
    { "id": 1, "name": "Warszawa", "slug": "warszawa" },
    { "id": 2, "name": "Kraków", "slug": "krakow" }
  ]
}
```

---

### `GET /api/v1/cities/{id}/` 🌐

Szczegóły miasta.

**Response `200 OK`:**
```json
{ "id": 1, "name": "Warszawa", "slug": "warszawa" }
```

---

### `POST /api/v1/cities/` 👑
### `PUT/PATCH /api/v1/cities/{id}/` 👑
### `DELETE /api/v1/cities/{id}/` 👑

CRUD dla administratorów. Struktura request body odpowiada polom modelu (bez `id`).

---

## 3. Kina (`/api/v1/cinemas/`)

### `GET /api/v1/cinemas/` 🌐

Lista kin. Obsługuje filtrowanie.

**Query params:**
| Parametr  | Typ  | Opis                                |
|-----------|------|-------------------------------------|
| `city`    | int  | ID miasta — filtruje kina w mieście |

**Response `200 OK`:**
```json
{
  "count": 2,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 1,
      "name": "Helios Galeria Mokotów",
      "city": { "id": 1, "name": "Warszawa", "slug": "warszawa" },
      "address": "ul. Wołoska 12, 02-675 Warszawa"
    }
  ]
}
```

---

### `GET /api/v1/cinemas/{id}/` 🌐

Szczegóły kina z listą sal.

**Response `200 OK`:**
```json
{
  "id": 1,
  "name": "Helios Galeria Mokotów",
  "city": { "id": 1, "name": "Warszawa", "slug": "warszawa" },
  "address": "ul. Wołoska 12, 02-675 Warszawa",
  "halls": [
    { "id": 1, "name": "Sala 1", "total_seats": 120 },
    { "id": 2, "name": "Sala VIP", "total_seats": 40 }
  ]
}
```

---

### `POST /api/v1/cinemas/` 👑
### `PUT/PATCH /api/v1/cinemas/{id}/` 👑
### `DELETE /api/v1/cinemas/{id}/` 👑

CRUD dla administratorów.

---

## 4. Sale (`/api/v1/halls/`)

### `GET /api/v1/halls/` 👑

Lista sal (tylko admin).

**Query params:**
| Parametr  | Typ  | Opis                     |
|-----------|------|--------------------------|
| `cinema`  | int  | ID kina — filtruje sale  |

---

### `GET /api/v1/halls/{id}/` 👑

Szczegóły sali z listą miejsc.

**Response `200 OK`:**
```json
{
  "id": 1,
  "cinema": 1,
  "name": "Sala 1",
  "rows": 10,
  "seats_per_row": 12,
  "total_seats": 120,
  "seats": [
    { "id": 1, "row": 1, "number": 1, "label": "R1/M1" },
    { "id": 2, "row": 1, "number": 2, "label": "R1/M2" }
  ]
}
```

---

### `POST /api/v1/halls/` 👑

Tworzenie sali. Miejsca są generowane automatycznie.

**Request body:**
```json
{
  "cinema": 1,
  "name": "Sala 2",
  "rows": 8,
  "seats_per_row": 15
}
```

**Response `201 Created`:** jak szczegóły sali powyżej.

---

### `PUT/PATCH /api/v1/halls/{id}/` 👑
### `DELETE /api/v1/halls/{id}/` 👑

CRUD dla administratorów.

> [!WARNING]
> Usunięcie sali kaskadowo usuwa wszystkie miejsca (`Seat`). Dozwolone tylko jeśli sala nie ma nadchodzących seansów.

---

## 5. Filmy (`/api/v1/movies/`)

### `GET /api/v1/movies/` 🌐

Lista filmów.

**Query params:**
| Parametr  | Typ    | Opis                              |
|-----------|--------|-----------------------------------|
| `genre`   | int    | ID gatunku — filtruje filmy       |
| `search`  | string | Szukaj po tytule (case-insensitive)|

**Response `200 OK`:**
```json
{
  "count": 10,
  "next": "...",
  "previous": null,
  "results": [
    {
      "id": 1,
      "title": "Diuna: Część Druga",
      "genre": { "id": 3, "name": "Sci-Fi" },
      "duration_min": 166,
      "age_rating": "PG-13",
      "release_date": "2024-03-01"
    }
  ]
}
```

---

### `GET /api/v1/movies/{id}/` 🌐

Szczegóły filmu.

**Response `200 OK`:**
```json
{
  "id": 1,
  "title": "Diuna: Część Druga",
  "genre": { "id": 3, "name": "Sci-Fi" },
  "duration_min": 166,
  "description": "Kontynuacja epickiej sagi...",
  "age_rating": "PG-13",
  "release_date": "2024-03-01"
}
```

---

### `POST /api/v1/movies/` 👑
### `PUT/PATCH /api/v1/movies/{id}/` 👑
### `DELETE /api/v1/movies/{id}/` 👑

CRUD dla administratorów.

---

## 6. Gatunki (`/api/v1/genres/`)

### `GET /api/v1/genres/` 🌐

Lista gatunków filmowych.

### `POST /api/v1/genres/` 👑
### `PUT/PATCH /api/v1/genres/{id}/` 👑
### `DELETE /api/v1/genres/{id}/` 👑

CRUD dla administratorów.

---

## 7. Seanse (`/api/v1/screenings/`)

### `GET /api/v1/screenings/` 🌐

Lista aktywnych seansów. Kluczowy endpoint dla repertuaru.

**Query params:**
| Parametr    | Typ    | Opis                                        |
|-------------|--------|---------------------------------------------|
| `cinema`    | int    | ID kina — filtruje seanse w kinie           |
| `city`      | int    | ID miasta — filtruje seanse w mieście       |
| `movie`     | int    | ID filmu — filtruje seanse danego filmu     |
| `date`      | date   | Format `YYYY-MM-DD` — filtruje po dacie     |

**Response `200 OK`:**
```json
{
  "count": 5,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 1,
      "movie": { "id": 1, "title": "Diuna: Część Druga", "duration_min": 166 },
      "hall": {
        "id": 1,
        "name": "Sala 1",
        "cinema": { "id": 1, "name": "Helios Galeria Mokotów" }
      },
      "start_time": "2026-10-20T18:00:00+02:00",
      "end_time": "2026-10-20T20:46:00+02:00",
      "price": "25.00",
      "available_seats": 85,
      "total_seats": 120
    }
  ]
}
```

---

### `GET /api/v1/screenings/{id}/` 🌐

Szczegóły seansu.

---

### `GET /api/v1/screenings/{id}/seats/` 🌐

Mapa dostępności miejsc dla danego seansu.

**Response `200 OK`:**
```json
{
  "screening_id": 1,
  "total_seats": 120,
  "available_seats": 85,
  "seats": [
    { "id": 1, "row": 1, "number": 1, "label": "R1/M1", "is_available": true },
    { "id": 2, "row": 1, "number": 2, "label": "R1/M2", "is_available": false }
  ]
}
```

---

### `POST /api/v1/screenings/` 👑
### `PUT/PATCH /api/v1/screenings/{id}/` 👑
### `DELETE /api/v1/screenings/{id}/` 👑

CRUD dla administratorów.

---

## 8. Rezerwacje (`/api/v1/reservations/`)

### `GET /api/v1/reservations/` 🔐

Lista rezerwacji aktualnie zalogowanego użytkownika.  
Admin widzi wszystkie rezerwacje. Client widzi tylko swoje.

**Response `200 OK`:**
```json
{
  "count": 2,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 1,
      "screening": {
        "id": 1,
        "movie_title": "Diuna: Część Druga",
        "start_time": "2026-10-20T18:00:00+02:00",
        "cinema_name": "Helios Galeria Mokotów"
      },
      "status": "confirmed",
      "total_price": "50.00",
      "seats": [
        { "id": 5, "label": "R3/M5" },
        { "id": 6, "label": "R3/M6" }
      ],
      "created_at": "2026-10-15T14:30:00+02:00"
    }
  ]
}
```

---

### `POST /api/v1/reservations/` 🔐

Tworzenie nowej rezerwacji. **Kluczowy endpoint — wymaga transakcji atomowej.**

**Request body:**
```json
{
  "screening": 1,
  "seat_ids": [5, 6]
}
```

**Response `201 Created`:**
```json
{
  "id": 1,
  "screening": {
    "id": 1,
    "movie_title": "Diuna: Część Druga",
    "start_time": "2026-10-20T18:00:00+02:00",
    "cinema_name": "Helios Galeria Mokotów"
  },
  "status": "confirmed",
  "total_price": "50.00",
  "seats": [
    { "id": 5, "label": "R3/M5" },
    { "id": 6, "label": "R3/M6" }
  ],
  "created_at": "2026-10-15T14:30:00+02:00"
}
```

**Błędy:**
- `400` — brak `seat_ids`, pusta lista miejsc, miejsce nie należy do sali seansu
- `404` — seans nie istnieje
- `409 Conflict` — przynajmniej jedno miejsce jest już zajęte

---

### `GET /api/v1/reservations/{id}/` 🔐

Szczegóły rezerwacji. Użytkownik może zobaczyć tylko swoje rezerwacje (admin — wszystkie).

**Błędy:**
- `403` — próba dostępu do cudzej rezerwacji przez klienta
- `404` — rezerwacja nie istnieje

---

> [!NOTE]
> **Anulowanie rezerwacji:** Endpoint `DELETE /api/v1/reservations/{id}/` oraz zmiana statusu na `'cancelled'` są **zarezerwowane do przyszłej implementacji**. Aktualnie metoda DELETE zwraca `405 Method Not Allowed`.
