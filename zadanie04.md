# Zadanie 4

## 1. Endpoint GET /health

```
PS C:\Users\olami\mw-kurs\breach-checker> curl.exe http://localhost:5000/health
{"status":"ok"}
```

```
172.18.0.1 - - [12/Nov/2026 14:34:04] "GET /health HTTP/1.1" 200 -
```

## 2. Kontener diagnostyczny healthcheck-probe

```
PS C:\Users\olami\mw-kurs\breach-checker> docker run --rm --network breach-net healthcheck-probe
backend: OK
```

Sonda bez `--network`:
```
PS C:\Users\olami\mw-kurs\breach-checker> docker run --rm healthcheck-probe

backend: UNREACHABLE
```

Sonda w sieci `breach-net`, po `docker stop backend`:
```
PS C:\Users\olami\mw-kurs\breach-checker> docker stop backend
backend
PS C:\Users\olami\mw-kurs\breach-checker> docker run --rm --network breach-net healthcheck-probe
backend: UNREACHABLE
```

Dlaczego bez `--network` nazwa `backend` nie działa, choć backend jest uruchomiony?
Bez opcji --network kontener uruchamia się w domyślnej sieci bridge, która nie posiada wbudowanego serwera DNS (nie tłumaczy nazw kontenerów na adresy IP) oraz jest odizolowana od sieci breach-net, w której działa backend. Z tego powodu sonda nie jest w stanie ani przetłumaczyć nazwy backend na adres IP, ani się z nim połączyć.


## 3. Trwałość danych

### 3a. Warstwa zapisywalna

Odczyt pliku po `docker restart backend`:
```
PS C:\Users\olami\mw-kurs\breach-checker> docker exec backend cat /tmp/notatka.txt
notatka
```

Odczyt pliku po `docker rm -f backend` i ponownym `docker run`:
```
PS C:\Users\olami\mw-kurs\breach-checker> docker exec backend cat /tmp/notatka.txt
cat: /tmp/notatka.txt: No such file or directory

```

### 3b. Wolumen

Zawartość `historia.txt` (dwie linie z różnymi godzinami):
```
PS C:\Users\olami\mw-kurs\breach-checker> docker run --rm -v breach-dane:/dane alpine sh -c "date >> /dane/historia.txt; cat /dane/historia.txt"
Thu Nov 12 15:03:16 UTC 2026
Thu Nov 12 15:03:32 UTC 2026
```

### Odpowiedzi (2-3 zdania)

- Dlaczego plik w `/tmp` przetrwał restart, ale nie `rm`:

Każdy kontener posiada własną warstwę zapisywalną, która jest zachowywana podczas zatrzymywania i restartu (docker restart), ale zostaje bezpowrotnie usunięta wraz z kontenerem przy wykonaniu docker rm. Nowo uruchomiony kontener startuje od zera z czystej warstwy obrazu.

- Dlaczego `historia.txt` przetrwała, choć oba kontenery zostały usunięte:

Dane zostały zapisane w nazwanym wolumenie Dockera (breach-dane), który ma własny cykl życia niezależny od kontenerów. Usunięcie kontenerów (flaga --rm) nie kasuje wolumenu, dzięki czemu kolejny kontener mógł go zamontować i odczytać wcześniejsze wpisy.

- Czy lista wycieków przetrwałaby `docker rm` i dlaczego:

Tak, ponieważ plik leaked-sha1.txt znajduje się fizycznie na dysku komputera-hosta i jest podłączony do kontenera jako bind mount (-v ./data/...). Usunięcie kontenera (docker rm) odpina jedynie ten punkt montowania, nie naruszając plików na dysku hosta.