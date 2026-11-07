# Zadanie 3

## 1. Etykiety i pochodzenie warstw
```
PS C:\Users\olami\mw-kurs\breach-checker> docker inspect --format '{{json .Config.Labels}}' ola-frontend:v1
{"autor":"Aleksandra Michałkiewicz","data":"2026-11-05","maintainer":"NGINX Docker Maintainers <docker-maint@nginx.com>"}
PS C:\Users\olami\mw-kurs\breach-checker> docker inspect --format '{{json .Config.Labels}}' ola-frontend:latest
{"autor":"Aleksandra Michałkiewicz","data":"2026-11-05","maintainer":"NGINX Docker Maintainers <docker-maint@nginx.com>"}
```
```
PS C:\Users\olami\mw-kurs\breach-checker> docker history ola-frontend:v1
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
2f1f64a7bc2b   30 minutes ago   EXPOSE [80/tcp]                                 0B        buildkit.dockerfile.v0
<missing>      30 minutes ago   RUN /bin/sh -c htmlhint /usr/share/nginx/htm…   4.1kB     buildkit.dockerfile.v0
<missing>      30 minutes ago   COPY . /usr/share/nginx/html # buildkit         32.8kB    buildkit.dockerfile.v0
<missing>      30 minutes ago   RUN /bin/sh -c apk add --no-cache nodejs npm…   99.3MB    buildkit.dockerfile.v0
<missing>      30 minutes ago   LABEL autor=Aleksandra Michałkiewicz data=20…   0B        buildkit.dockerfile.v0
<missing>      2 days ago       RUN /bin/sh -c set -x     && apkArch="$(cat …   52.4MB    buildkit.dockerfile.v0
<missing>      2 days ago       ENV ACME_VERSION=0.4.1                          0B        buildkit.dockerfile.v0
<missing>      2 days ago       ENV NJS_RELEASE=1                               0B        buildkit.dockerfile.v0
<missing>      2 days ago       ENV NJS_VERSION=1.0.2                           0B        buildkit.dockerfile.v0
<missing>      2 days ago       CMD ["nginx" "-g" "daemon off;"]                0B        buildkit.dockerfile.v0
<missing>      2 days ago       STOPSIGNAL SIGQUIT                              0B        buildkit.dockerfile.v0
<missing>      2 days ago       EXPOSE map[80/tcp:{}]                           0B        buildkit.dockerfile.v0
<missing>      2 days ago       ENTRYPOINT ["/docker-entrypoint.sh"]            0B        buildkit.dockerfile.v0
<missing>      2 days ago       COPY 30-tune-worker-processes.sh /docker-ent…   16.4kB    buildkit.dockerfile.v0
<missing>      2 days ago       COPY 20-envsubst-on-templates.sh /docker-ent…   12.3kB    buildkit.dockerfile.v0
<missing>      2 days ago       COPY 15-local-resolvers.envsh /docker-entryp…   12.3kB    buildkit.dockerfile.v0
<missing>      2 days ago       COPY 10-listen-on-ipv6-by-default.sh /docker…   12.3kB    buildkit.dockerfile.v0
<missing>      2 days ago       COPY docker-entrypoint.sh / # buildkit          8.19kB    buildkit.dockerfile.v0
<missing>      2 days ago       RUN /bin/sh -c set -x     && addgroup -g 101…   5.75MB    buildkit.dockerfile.v0
<missing>      2 days ago       ENV DYNPKG_RELEASE=1                            0B        buildkit.dockerfile.v0
<missing>      2 days ago       ENV PKG_RELEASE=1                               0B        buildkit.dockerfile.v0
<missing>      2 days ago       ENV NGINX_VERSION=1.31.8                        0B        buildkit.dockerfile.v0
<missing>      2 days ago       LABEL maintainer=NGINX Docker Maintainers <d…   0B        buildkit.dockerfile.v0
<missing>      2 days ago       CMD ["/bin/sh"]                                 0B        buildkit.dockerfile.v0
<missing>      2 days ago       ADD alpine-minirootfs-3.24.4-x86_64.tar.gz /…   9.09MB    buildkit.dockerfile.v0
```
Odpowiedź: Warstwy bazowe to te od nginx:alpine, a moje to te dodane przez LABEL, RUN, COPY. 
Niektóre mają rozmiar 0B, ponieważ instrukcje takie jak LABEL, EXPOSE czy CMD dopisują tylko metadane, nie zmieniając fizycznego systemu plików.
Widać tylko moją zawartość w wyniku inspect, ponieważ docker inspect pyta bezpośrednio o konfigurację metadanych mojego nowo zbudowanego obrazu, która nadpisuje lub izoluje te właściwości.

## 2. RUN działa raz
build.txt (kontener 1): Sat Nov  7 22:36:30 UTC 2026
build.txt (kontener 2): Sat Nov  7 22:36:30 UTC 2026

```
PS C:\Users\olami\mw-kurs\breach-checker> docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED              STATUS              PORTS                                     NAMES
721dc66628f9   ola-frontend:v1   "/docker-entrypoint.…"   15 seconds ago       Up 15 seconds       0.0.0.0:8081->80/tcp, [::]:8081->80/tcp   kontener2
0a57a0b674aa   ola-frontend:v1   "/docker-entrypoint.…"   About a minute ago   Up About a minute   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   kontener1
```
Odpowiedź: Czas zapisany w pliku build.txt jest w obu kontenerach taki sam, ponieważ instrukcja RUN wykonuje się tylko raz – w trakcie budowania obrazu (docker build). Plik ten został fizycznie „wypieczony” i zapisany jako stały element w warstwie obrazu. Z tego powodu każdy nowy kontener utworzony z tego samego obrazu dziedziczy ten sam, niezmienny plik.

## 3. Nadpisanie CMD
```
root   /usr/share/nginx/html;
```
Wyjaśnienie: Pliki aplikacji kopiujemy właśnie do katalogu /usr/share/nginx/html, ponieważ pod tą ścieżką domyślny serwer NGINX szuka plików statycznych (strony internetowej), aby zaserwować je użytkownikom.

## 4. Diagnoza błędnego COPY
```
PS C:\Users\olami\mw-kurs\breach-checker> docker exec -it test-kopia ls /usr/share/nginx
html        index.html  script.js   style.css

What's next:
    Try Docker Debug for seamless, persistent debugging tools in any container or image → docker debug test-kopia
    Learn more at https://docs.docker.com/go/debug-cli/
```
```
PS C:\Users\olami\mw-kurs\breach-checker> docker exec -it test-kopia ls /usr/share/nginx/html
50x.html    build.txt   index.html

What's next:
    Try Docker Debug for seamless, persistent debugging tools in any container or image → docker debug test-kopia
    Learn more at https://docs.docker.com/go/debug-cli/
```
Odpowiedź: Narzędzie htmlhint sprawdziło plik index.html, który znajdował się w kontekście budowania na hoście (lub w ścieżce wskazanej w tamtym momencie). Budowanie przeszedł bez błędu, ponieważ instrukcja COPY . /usr/share/nginx/ wykonana przez Dockera była poprawna pod względem składniowym – skopiowała pliki z kontekstu do wskazanego katalogu docelowego. Problem polegał na tym, że NGINX serwuje strony domyślnie z podkatalogu /usr/share/nginx/html, a pliki wylądowały poziom wyżej (/usr/share/nginx/), przez co serwer ich nie widział i wyświetlił swoją domyślną stronę.

## 5. Tag a obraz
```
PS C:\Users\olami\mw-kurs\breach-checker> docker rmi breach-checker-frontend:latest
Untagged: breach-checker-frontend:latest
```
```
PS C:\Users\olami\mw-kurs\breach-checker> docker rmi breach-checker-frontend:v2
Untagged: breach-checker-frontend:v2
Deleted: sha256:d16f1be9c5c23e29c89d7d11bb81d0251f63ca4004b00320c571fcde374c7b23
```
Odpowiedź: Polecenie docker rmi faktycznie usuwa obraz z dysku (czyli kasuje jego fizyczne warstwy i zwalnia miejsce) dopiero wtedy, gdy zostanie usunięty ostatni tag (etykieta) wskazujący na dany IMAGE ID. Dopóki chociaż jedna etykieta wskazuje na dany zestaw warstw, Docker zachowuje je na dysku, wykonując jedynie operację "odklejenia" (Untagged).