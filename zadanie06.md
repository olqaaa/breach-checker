# Zadanie 6

## 1. Własne testy

### Wynik `docker compose exec backend pytest -v`
```
docker compose exec backend pytest -v
=========================== test session starts ===========================
platform linux -- Python 3.12.16, pytest-8.4.2, pluggy-1.7.0 -- /usr/local/bin/python3.12
cachedir: .pytest_cache
rootdir: /app
collected 8 items

tests/test_check.py::test_leaked_hash_is_found PASSED               [ 12%]
tests/test_check.py::test_unknown_hash_is_not_found PASSED          [ 25%]
tests/test_check.py::test_missing_hash_field PASSED                 [ 37%]
tests/test_check.py::test_plaintext_password_is_rejected PASSED     [ 50%]
tests/test_check.py::test_uppercase_leaked_hash_is_found PASSED     [ 62%]
tests/test_check.py::test_too_short_hash_is_rejected PASSED         [ 75%]
tests/test_check.py::test_non_hex_characters_are_rejected PASSED    [ 87%]
tests/test_check.py::test_non_json_body_is_rejected PASSED          [100%]

============================ 8 passed in 0.21s ============================
```

### Wynik `docker compose exec backend pytest` po usunięciu `.lower()`
```
docker compose exec backend pytest
=========================== test session starts ===========================
platform linux -- Python 3.12.16, pytest-8.4.2, pluggy-1.7.0
rootdir: /app
collected 8 items

tests/test_check.py ....F...                                        [100%]

================================ FAILURES =================================
___________________ test_uppercase_leaked_hash_is_found ___________________

client = <FlaskClient <Flask 'app'>>

    def test_uppercase_leaked_hash_is_found(client):
        response = client.post("/check", json={"hash": LEAKED.upper()})
>       assert response.status_code == 200
E       assert 400 == 200
E        +  where 400 = <WrapperTestResponse streamed [400 BAD REQUEST]>.status_code

tests/test_check.py:28: AssertionError
========================= short test summary info =========================
FAILED tests/test_check.py::test_uppercase_leaked_hash_is_found - assert 400 == 200
======================= 1 failed, 7 passed in 0.26s =======================

What's next:
    Debug this Compose error with Gordon → docker ai "help me fix this compose error"
```

Testy przygotowane na zajęciach używały w stałych `LEAKED` i `NOT_LEAKED` wyłącznie małych liter, więc przechodziły walidację wyrażeniem regularnym `[0-9a-f]{40}` bez konwersji wielkości znaków. Dopiero nowy test wysyłający poprawny skrót SHA-1 zapisany wielkimi literami (`LEAKED.upper()`) wykazał, że bez `.lower()` aplikacja niesłusznie odrzuca takie żądanie kodem `400`.

---

## 2. Awaria a blokada

### Wynik `curl.exe` przy zatrzymanej bazie `db`
```
HTTP 500
```

### Wynik `docker compose ps -a` oraz `docker inspect` po restarcie backendu
```
docker compose ps -a
NAME                        IMAGE                     COMMAND                  SERVICE    CREATED         STATUS                     PORTS
breach-checker-backend-1    breach-checker-backend    "flask --app app run…"   backend    6 hours ago     Exited (1) 1 second ago
breach-checker-db-1         postgres:16-alpine        "docker-entrypoint.s…"   db         6 hours ago     Exited (0) 3 minutes ago
breach-checker-frontend-1   breach-checker-frontend   "/docker-entrypoint.…"   frontend   4 minutes ago   Up 4 minutes               0.0.0.0:8080->80/tcp, [::]:8080->80/tcp

docker inspect --format '{{.State.Status}} {{.State.ExitCode}}' breach-checker-backend-1
exited 1
```

### Porównanie awarii z blokadą z kroku 4

| Kryterium | Awaria (zatrzymana baza `db`) | Blokada (`LOCK TABLE`) |
| :--- | :--- | :--- |
| **Status w `docker compose ps`** | `Up (unhealthy)` / po restarcie `Exited (1)` | `Up (healthy)` |
| **CPU w `docker stats`** | `0.00%` (kontener zatrzymany) | Bliskie `0%` (proces czeka na odpowiedź bazy) |
| **Ostatnie linie logów** | Pełny ślad stosu (`psycopg2.OperationalError: could not translate host name "db"`) | Urwany wpis na `check: hash=7c4a8...` (brak statusu odpowiedzi HTTP) |
| **Co dostał klient** | `HTTP 500` ze stroną debuggera Flask/Werkzeug (`blad.html`) | Wiszące żądanie, a następnie błąd `504 Gateway Time-out` z `nginx` |
| **Po jakim czasie** | Natychmiast (od razu po próbie połączenia z bazą) | Po 60 sekundach (domyślny timeout proxy `nginx`) |

Plik `blad.html` zawierał interaktywną stronę błędu debuggera Werkzeug wraz z pełnym śladem stosu (*traceback*), ścieżkami plików na serwerze oraz fragmentami kodu źródłowego aplikacji. Na serwerze produkcyjnym ujawnia to atakującemu wewnętrzną architekturę i wersje bibliotek, a aktywny debugger pozwala na zdalne wykonanie kodu Pythona na serwerze po odgadnięciu lub przejęciu kodu PIN.

---

## 3. Frontend w trybie deweloperskim

### Wynik `curl.exe http://localhost:8080/nginx.conf` przed poprawką
```
server {
    listen 80;

    # Pliki strony (index.html, style.css, script.js)
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    # Wszystko pod /api/ nginx przekazuje do usługi backend w sieci Compose
    location /api/ {
        proxy_pass http://backend:5000/;
    }
}
```

### Wynik `curl.exe -I http://localhost:8080/nginx.conf` po poprawce
```
HTTP/1.1 404 Not Found
Server: nginx/1.31.8
Date: Wed, 26 Nov 2026 15:57:46 GMT
Content-Type: text/html
Content-Length: 153
Connection: keep-alive
```

W produkcyjnym `frontend/Dockerfile` pliki są kopiowane jawnie z nazwy (`COPY index.html style.css script.js /usr/share/nginx/html/`), dzięki czemu pliki konfiguracyjne nie trafiają do katalogu serwowanego przez `nginx`. Montowanie całego katalogu roboczego bezpośrednio do katalogu głównego serwera WWW jest ryzykowne, ponieważ publicznie udostępnia pliki infrastrukturalne (np. `nginx.conf`, `Dockerfile`), ujawniając konfigurację i strukturę aplikacji.