# 00 – Wytyczne Architektoniczne (Architecture Guidelines)

> **Status:** `v1.0 – baseline`  
> **Ostatnia aktualizacja:** 2026-10-15  
> **Właściciel:** Eryk Poczykowski

---

## 1. Stack Technologiczny

| Komponent         | Wersja / Pakiet                          | Uwagi                                    |
|-------------------|------------------------------------------|------------------------------------------|
| Python            | 3.13.x                                   | Obowiązująca dla całego projektu         |
| Django            | 6.1.*                                    | Wyłączna wersja frameworka               |
| Django REST Framework | 3.16.*                               | Wyłączna biblioteka do budowania REST API|
| Autentykacja JWT  | `djangorestframework-simplejwt` 5.4.*    | Jedyna dopuszczona metoda autentykacji   |
| Dokumentacja API  | `drf-spectacular` 0.28.*                 | Generowanie Swagger UI i schematu OpenAPI|
| Baza danych       | PostgreSQL 16.x (produkcja) / SQLite (dev) | SQLite wyłącznie do lokalnego dev/testów |
| Testy             | `rest_framework.test.APITestCase`        | Wyłączna klasa bazowa dla testów API     |

### 1.1. Guardrails – zasady nieprzekraczalne

> [!IMPORTANT]
> 1. **Zakaz instalacji nowych bibliotek** bez uprzedniego dodania ich do tego pliku w sekcji 1.
> 2. **Django 6.1.*** jest obowiązującą wersją — agenci nie mogą używać API usuniętego lub nieistniejącego w tej wersji.
> 3. Plik `requirements.txt` jest jedynym źródłem zależności — bez `pip install` poza nim.
> 4. Cała logika biznesowa pozostaje w warstwie **modeli i serwisów** — widoki (views) są wyłącznie warstwą HTTP.

---

## 2. Struktura Projektu Django

```text
cinemahub/                          ← katalog główny
├── .spec/                          ← specyfikacja 
├── config/                         ← konfiguracja Django (settings, urls, wsgi)
│   ├── settings/
│   │   ├── base.py
│   │   ├── development.py
│   │   └── production.py
│   ├── urls.py
│   └── wsgi.py
├── apps/
│   ├── cinema/                     ← Kina, Sale, Miasta
│   ├── movies/                     ← Filmy, Gatunki
│   ├── screenings/                 ← Seanse (Repertuar)
│   ├── reservations/               ← Rezerwacje i zajęte miejsca
│   └── users/                      ← Użytkownicy, role, JWT
├── docs/
│   └── sdd_report.md
├── manage.py
└── requirements.txt
```

### 2.1. Wewnętrzna struktura każdej aplikacji Django

Każda aplikacja w `apps/` musi zawierać:

```text
apps/<nazwa>/
├── __init__.py
├── admin.py          ← Rejestracja modeli w panelu Django Admin
├── apps.py
├── models.py         ← Modele ORM
├── serializers.py    ← Serializery DRF
├── views.py          ← ViewSety DRF
├── urls.py           ← Routing aplikacji
├── permissions.py    ← Niestandardowe klasy uprawnień (jeśli wymagane)
└── tests/
    ├── __init__.py
    ├── test_models.py
    └── test_api.py
```

---

## 3. Konwencje Kodowania

### 3.1. Ogólne zasady (PEP 8)
- Maksymalna długość linii: **100 znaków**
- Wcięcia: **4 spacje** (bez tabulatorów)
- Nazwy klas: `PascalCase`
- Nazwy funkcji i zmiennych: `snake_case`
- Nazwy stałych: `UPPER_SNAKE_CASE`
- Każdy moduł musi zawierać docstring opisujący jego przeznaczenie
- Każda klasa modelu musi zawierać `class Meta` z `ordering` i `verbose_name`

### 3.2. Modele Django
- Każdy model dziedziczy z `django.db.models.Model`
- Pola `created_at` i `updated_at` (typ `DateTimeField(auto_now_add/auto_now)`) są **wymagane** we wszystkich modelach poza prostymi słownikami
- Metoda `__str__` jest **obowiązkowa** w każdym modelu
- Niestandardowa logika biznesowa umieszczana jest w metodach modelu lub dedykowanych klasach serwisowych (nie w widokach)

### 3.3. Serializery DRF
- Preferowane `ModelSerializer` nad ręcznym `Serializer`
- Walidacja danych wejściowych odbywa się wyłącznie przez metody `validate_<pole>` i `validate()`
- Serializery odczytu i zapisu mogą być rozdzielone (np. `ScreeningReadSerializer` vs `ScreeningWriteSerializer`) gdy pola się różnią

### 3.4. Widoki DRF
- Wyłącznie `ModelViewSet` lub `ReadOnlyModelViewSet` — **zakaz** używania `APIView` i `@api_view` bez uzasadnienia w specyfikacji
- Niestandardowe akcje realizowane przez `@action` decorator
- Każdy ViewSet musi jawnie definiować `permission_classes` i `authentication_classes`
- Paginacja jest **obowiązkowa** dla endpointów zwracających listy (domyślnie 20 elementów na stronę)

### 3.5. URL Routing
- Wyłącznie `DefaultRouter` z DRF
- Prefiks wersjonowania API: `/api/v1/`
- Nazwy URL: `<app_name>:<model>-<action>` (np. `cinema:cinema-list`)

---

## 4. Autentykacja i Autoryzacja

### 4.1. JWT (JSON Web Tokens)
- Tokeny generowane przez `djangorestframework-simplejwt`
- **Access token:** ważność 60 minut
- **Refresh token:** ważność 7 dni
- Nagłówek autoryzacji: `Authorization: Bearer <access_token>`

### 4.2. Role użytkowników

| Rola    | Opis                                                                 |
|---------|----------------------------------------------------------------------|
| `admin` | Pełen dostęp CRUD do wszystkich zasobów. Zarządza kinami, seansami. |
| `client`| Może przeglądać repertuar i zarządzać własnymi rezerwacjami.         |

- Implementacja ról przez pole `role` w modelu `User` (wybór z enum)
- Niestandardowe klasy uprawnień zdefiniowane w `apps/users/permissions.py`
- Endpointy publiczne (bez autentykacji): przeglądanie repertuaru, listy kin, listy miast

---

## 5. Obsługa Błędów

Wszystkie odpowiedzi błędów muszą mieć spójny format JSON:

```json
{
  "error": "Krótki opis błędu",
  "detail": "Szczegółowy opis dla dewelopera (opcjonalny)",
  "code": "machine_readable_error_code"
}
```

## 6. Dokumentacja API (Swagger)

- Swagger UI dostępny pod: `/api/schema/swagger-ui/`
- ReDoc dostępny pod: `/api/schema/redoc/`
- Schemat OpenAPI 3.0 generowany automatycznie przez `drf-spectacular`
- Każdy endpoint musi zawierać dekorator `@extend_schema` z opisem i przykładami odpowiedzi

---

## 7. Testy Automatyczne

- Klasa bazowa: `rest_framework.test.APITestCase`
- Minimalny pokrycie testami dla: modeli, serializerów, kluczowych endpointów
- Każdy test sprawdza dokładnie jeden przypadek (zasada single assertion)
- Nazewnictwo testów: `test_<co_testujemy>_<oczekiwany_rezultat>` (np. `test_reserve_seat_when_already_taken_returns_409`)
- Testy uruchamiane komendą: `python manage.py test apps`

---

## 8. Git i Wersjonowanie

- Branch główny: `main`
- Nowe funkcje: `feat/<nazwa-funkcji>` (np. `feat/reservations`)
- Poprawki: `fix/<opis-błędu>`
- Format commitów (Conventional Commits):
  ```
  feat(reservations): implement seat reservation with atomic transaction
  fix(screening): correct available seats count query
  docs(spec): update 02_api_contracts with reservation endpoint
  ```
- Każdy commit odpowiada jednej logicznej zmianie (zakaz "mega-commitów")
