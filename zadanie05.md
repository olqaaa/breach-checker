# Zadanie 5

## 1. Healthcheck backendu

```
[+] up 6/6
 ✔ Image breach-checker-frontend       Built                     4.5s
 ✔ Image breach-checker-backend        Built                     4.4s
 ✔ Network breach-checker_default      Created                   0.1s
 ✔ Container breach-checker-db-1       Healthy                   6.4s
 ✔ Container breach-checker-backend-1  Healthy                  12.0s
 ✔ Container breach-checker-frontend-1 Started                  12.1s
```
```
NAME                        IMAGE                     COMMAND                  SERVICE    CREATED          STATUS                    PORTS
breach-checker-backend-1    breach-checker-backend    "python app.py"          backend    16 seconds ago   Up 9 seconds (healthy)    5000/tcp
breach-checker-db-1         postgres:16-alpine        "docker-entrypoint.s…"   db         16 seconds ago   Up 15 seconds (healthy)   5432/tcp
```

## 2. Trwałość bazy

### 2a. Odpowiedź API po dodaniu skrótu (`INSERT`)
```
docker compose exec db psql -U breach -d breachdb -c "INSERT INTO leaked_hashes VALUES ('bc45408273d7e1914b741c3407b0b559764f7876');"
INSERT 0 1

'{"hash":"bc45408273d7e1914b741c3407b0b559764f7876"}' | curl.exe -X POST http://localhost:8080/api/check -H "Content-Type: application/json" -d '@-'
{"found":true}
```

### 2b. `docker compose down` + `docker compose up -d`
```
'{"hash":"bc45408273d7e1914b741c3407b0b559764f7876"}' | curl.exe -X POST http://localhost:8080/api/check -H "Content-Type: application/json" -d '@-'
{"found":true}
```

### 2c. `docker compose down -v` + `docker compose up -d`
```
'{"hash":"bc45408273d7e1914b741c3407b0b559764f7876"}' | curl.exe -X POST http://localhost:8080/api/check -H "Content-Type: application/json" -d '@-'
{"found":false}
```

### Odpowiedź

Polecenie `docker compose down` usuwa jedynie kontenery i sieć projektu, pozostawiając nazwany wolumen z zapisanymi danymi bazy, dlatego dodany skrót przetrwał restart. Flaga `-v` usuwa także wolumen, trwale kasując całą bazę danych. 
Przy kolejnym uruchomieniu kontener `db` wykrył pusty katalog danych (`/var/lib/postgresql/data`) i automatycznie wykonał skrypt `/docker-entrypoint-initdb.d/init.sql`, który odtworzył tabelę oraz zaimportował pierwotne 89 skrótów z pliku `leaked-sha1.txt`.

## 3. Skalowanie

```
docker compose ps
NAME                        IMAGE                     COMMAND                  SERVICE    CREATED          STATUS                    PORTS
breach-checker-backend-1    breach-checker-backend    "python app.py"          backend    7 minutes ago    Up 7 minutes (healthy)    5000/tcp
breach-checker-backend-2    breach-checker-backend    "python app.py"          backend    12 seconds ago   Up 11 seconds (healthy)   5000/tcp
breach-checker-backend-3    breach-checker-backend    "python app.py"          backend    12 seconds ago   Up 11 seconds (healthy)   5000/tcp
breach-checker-db-1         postgres:16-alpine        "docker-entrypoint.s…"   db         7 minutes ago    Up 7 minutes (healthy)    5432/tcp
breach-checker-frontend-1   breach-checker-frontend   "/docker-entrypoint.…"   frontend   7 minutes ago    Up 7 minutes              0.0.0.0:8080->80/tcp, [::]:8080->80/tcp
```


```
docker compose exec frontend nslookup backend
Server:         127.0.0.11
Address:        127.0.0.11:53

Non-authoritative answer:
Name:   backend
Address: 172.19.0.3
Name:   backend
Address: 172.19.0.5
Name:   backend
Address: 172.19.0.6

Non-authoritative answer:
```

Logi `POST /check` z backendu przed restartem `frontend`:
```
docker compose logs backend | Select-String "POST /check"

backend-1  | 172.19.0.4 - - [19/Nov/2026 14:58:10] "POST /check HTTP
/1.1" 200 -
backend-1  | 172.19.0.4 - - [19/Nov/2026 15:07:10] "POST /check HTTP
/1.1" 200 -
backend-1  | 172.19.0.4 - - [19/Nov/2026 15:07:10] "POST /check HTTP
/1.1" 200 -
backend-1  | 172.19.0.4 - - [19/Nov/2026 15:07:10] "POST /check HTTP
/1.1" 200 -
backend-1  | 172.19.0.4 - - [19/Nov/2026 15:07:10] "POST /check HTTP
/1.1" 200 -
backend-1  | 172.19.0.4 - - [19/Nov/2026 15:07:10] "POST /check HTTP
/1.1" 200 -
backend-1  | 172.19.0.4 - - [19/Nov/2026 15:07:10] "POST /check HTTP
/1.1" 200 -
backend-1  | 172.19.0.4 - - [19/Nov/2026 15:07:10] "POST /check HTTP
/1.1" 200 -
backend-1  | 172.19.0.4 - - [19/Nov/2026 15:07:10] "POST /check HTTP
/1.1" 200 -
```

Logi `POST /check` z backendu po `docker compose restart frontend`:
```
docker compose logs backend | Select-String "POST /check"

backend-1  | 172.19.0.4 - - [19/Nov/2026 14:58:10] "POST /check HTTP
/1.1" 200 -
backend-2  | 172.19.0.4 - - [19/Nov/2026 15:10:08] "POST /check HTTP
/1.1" 200 -
backend-2  | 172.19.0.4 - - [19/Nov/2026 15:10:08] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:07] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:07] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:07] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:07] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:07] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:07] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:07] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:07] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:07] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:07] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:07] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:08] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:08] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:08] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:08] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:08] "POST /check HTTP
/1.1" 200 -
backend-3  | 172.19.0.4 - - [19/Nov/2026 15:10:08] "POST /check HTTP
/1.1" 200 -
backend-1  | 172.19.0.4 - - [19/Nov/2026 15:07:10] "POST /check HTTP
/1.1" 200 -
backend-1  | 172.19.0.4 - - [19/Nov/2026 15:07:10] "POST /check HTTP
/1.1" 200 -
backend-1  | 172.19.0.4 - - [19/Nov/2026 15:07:10] "POST /check HTTP
/1.1" 200 -
backend-1  | 172.19.0.4 - - [19/Nov/2026 15:07:10] "POST /check HTTP
/1.1" 200 -
```

### Odpowiedź

Serwer `nginx` rozwiązuje nazwę domenową `backend` przez wewnętrzny DNS Dockera tylko raz podczas swojego uruchamiania i zapisuje pierwszy uzyskany adres IP w pamięci podręcznej — dlatego przed restartem cały ruch trafiał do jednej repliki. Dopiero restart zmusił `nginx` do ponownego odpytania DNS-a, pobrania wszystkich trzech adresów IP i rozdzielania ruchu. 
Skalowanie nie zadziałałoby z sekcją `ports: "5000:5000"`, ponieważ jeden port (`5000`) na komputerze-hoście może być zajęty jednocześnie tylko przez jeden kontener, więc pozostałe repliki zgłosiłyby błąd konfliktu portów.