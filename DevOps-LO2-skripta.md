# DevOps — LO2 skripta (Managing and creating container images)

> Ispitne bilješke, kolegij **Intro to DevOps** (Algebra/Bernays). Praktični ispit **25.6. u 18:30**.
> Na ispitu dozvoljeno: službene doke (Podman/Docker/k8s/minikube/Red Hat) + **vlastite GitHub bilješke**. **AI zabranjen.**
> Format: **zadatak po zadatak**, svaki flag i linija razloženi, sve začkoljice (gotchas) označene, na kraju 3-rečenični sažetak.
> Sve odrađeno **hands-on** na ispitnom VM-u (preko VPN-a), **bez sudo**, rootless Podman.

---

## 0. STVARI KOJE SE PONAVLJAJU (zapamti jednom, vrijedi za sve zadatke)

### Folder-struktura
Krov je `~/devops` (ili `~/lo2-XX` direktno — svejedno). Svaki zadatak u svom podfolderu da se imena i build kontekst ne miješaju:
```bash
mkdir -p ~/lo2-XX && cd ~/lo2-XX
```
- `mkdir -p` — napravi mapu; `-p` = "ne buni se ako već postoji" (+ stvori roditeljske mape po potrebi).
- `&&` — "napravi prvo, pa **ako uspije**, napravi drugo". (Jedan `&` = pokreni u pozadini i ne čekaj → `cd` bi pao jer mapa još ne postoji. Zamka koja se javila uživo.)
- `cd ~/lo2-XX` — uđi u mapu. `~` = tvoj home (`/home/student`).

### Layeri (slojevi) — temelj cijelog LO2
- Image je **složen od layera**. Svaka instrukcija u Containerfileu (`FROM`, `RUN`, `COPY`, `ENV`...) stvara **jedan novi layer** povrh prethodnog.
- Layeri su **append-only (samo-dodajući) i nepromjenjivi**. Jednom napravljen layer se ne mijenja.
- Finalni image = zbroj svih layera.
- **Posljedica:** brisanje datoteke u kasnijem layeru NE smanjuje image — datoteka fizički ostaje u ranijem layeru, kasniji layer samo doda "whiteout" (oznaku: skriveno). Bajtovi se i dalje nose. (Vidi zadatak #21.)

### Build kontekst — zadnji argument `podman build`
```bash
podman build -t ime:tag -f Containerfile .
```
- `-t ime:tag` — **tag** = ime i verzija image-a (`ime:tag`). Bez `-t` image ostaje bezimen (samo hash ID). Kraće od `--tag`.
- `-f Containerfile` — **file** = koja datoteka je recept. `Containerfile` je default ime koje Podman traži sam, pa je `-f` tu suvišan; nužan tek kad recept ima nestandardno ime (zadatak #13). Kraće od `--file`.
- `.` — **build kontekst** = mapa čiji se sadržaj šalje build procesu kao materijal za `COPY`/`ADD`. Točka = trenutna mapa. **Obavezan argument** (build uvijek mora znati odakle vuče datoteke, makar ih ne koristio). Sve u toj mapi (osim onog u `.containerignore`) se zapakira i pošalje.
- `-f` i `.` su **neovisni**: `-f` kaže *gdje je recept*, `.` kaže *odakle materijal*. Mogu biti različite mape.

### Heredoc `<< 'EOF'` (kako upisujemo Containerfile)
```bash
cat > Containerfile << 'EOF'
...sadržaj...
EOF
```
- `cat > Containerfile` — preusmjeri ono što slijedi **u** datoteku `Containerfile` (`>` = prepiši ispočetka; `>>` bi dodavao).
- `<< 'EOF'` — "čitaj sve do retka koji kaže `EOF`". Navodnici oko `'EOF'` = bash **NE dira** `$...` u tijelu (upiše doslovno, da varijable kasnije popuni podman/sh, a ne shell pri pisanju datoteke). Dobra navika i kad nema varijabli.

### CMD sintaksa
- **Exec forma (preporučeno):** `CMD ["program", "arg1", "arg2"]` — JSON polje, bez shella, signali (Ctrl+C, stop) rade čisto.
- **Shell forma:** `CMD program arg1` — vrti se kroz `/bin/sh -c`, signali ne dolaze čisto do programa (LO5 #11).

### `--rm` za "zaviri i baci"
`podman run --rm ...` = pokreni privremeni kontejner i obriši ga čim završi. Koristi se kad samo želimo nešto provjeriti (verzija, lista paketa), a ne ostavljati kontejner.

---

## ZADATAK 1 — ARG za odabir base-tag-a pri buildu (`--build-arg`)

> Write a Containerfile that uses an ARG to choose the base-image tag at build time (--build-arg), and build it twice with different values.

```bash
cat > Containerfile << 'EOF'
ARG ALPINE_TAG=3.20
FROM alpine:${ALPINE_TAG}
CMD ["cat", "/etc/alpine-release"]
EOF

podman build -t argtest:320 --build-arg ALPINE_TAG=3.20 -f Containerfile .
podman build -t argtest:319 --build-arg ALPINE_TAG=3.19 -f Containerfile .

podman run --rm argtest:320   # -> 3.20.x
podman run --rm argtest:319   # -> 3.19.x
```

**Razlomljeno:**
- `ARG ALPINE_TAG=3.20` — deklarira **build-time varijablu** s default vrijednosti `3.20`. `ARG` iznad `FROM` smije se koristiti u samom `FROM`.
- `FROM alpine:${ALPINE_TAG}` — baza čiji tag dolazi iz varijable. `${...}` = "uvrsti vrijednost varijable".
- `--build-arg ALPINE_TAG=3.19` — pri buildu pregazi default vrijednost ARG-a. Tako isti Containerfile gradi različite verzije.
- `CMD ["cat", "/etc/alpine-release"]` — ispiše točnu verziju Alpinea da vidimo koja je baza ušla.

**Začkoljica:**
- **`ARG` je samo build-time — NE preživi u kontejner.** Unutar pokrenutog kontejnera ta varijabla ne postoji (za run-time treba `ENV`, vidi #15).
- Ako drugi build ne pokaže novu vrijednost → keš. Dodaj `--no-cache` (vidi #12).

**Sažetak:** `ARG` je varijabla koja postoji **samo dok traje build**, s default vrijednosti koju pri buildu pregaziš s `--build-arg`. Time isti Containerfile parametriziraš (npr. biraš tag baze) bez mijenjanja datoteke. Vrijednost se ne vidi unutar pokrenutog kontejnera — za to služi `ENV`.

---

## ZADATAK 2 — Multi-stage build (artefakt u jednoj fazi → samo rezultat u finalni image)

> Write a multi-stage Containerfile that builds an artifact in one stage and copies only the result into a minimal final image; compare the final image size to a single-stage build.

```bash
cat > main.go << 'EOF'
package main
import "fmt"
func main() { fmt.Println("Hello multi-stage") }
EOF

cat > Containerfile << 'EOF'
# FAZA 1: builder (ima cijeli Go toolchain)
FROM golang:1.22 AS builder
WORKDIR /src
COPY main.go .
RUN go build -o app main.go

# FAZA 2: finalni (minimalan, samo binarij)
FROM alpine:3.20
COPY --from=builder /src/app /app
CMD ["/app"]
EOF

podman build -t multi:final -f Containerfile .
podman images multi    # finalni ~10 MB umjesto ~270+ MB
```

**Razlomljeno:**
- `FROM golang:1.22 AS builder` — prva faza, dobiva ime `builder` (`AS`). Ima cijeli Go kompajler (~270 MB).
- `WORKDIR /src` — radni direktorij unutar image-a (stvori ga ako ne postoji i uđi u njega). Sve relativne putanje idu odavde.
- `COPY main.go .` — kopiraj izvorni kod u `/src`. `.` = trenutni WORKDIR.
- `RUN go build -o app main.go` — kompajliraj u binarij `app`. `-o app` = ime izlaza.
- `FROM alpine:3.20` — **druga faza počinje ispočetka** na maloj bazi. Sve iz prve faze (kompajler) ostaje izvan.
- `COPY --from=builder /src/app /app` — uzmi **samo** gotov binarij iz faze `builder` i ubaci u finalni image. `--from=builder` = "kopiraj iz te faze, ne iz konteksta".
- `CMD ["/app"]` — pokreni binarij.

**Začkoljica:**
- Poanta je da **build-alat (kompajler) ostaje izvan finalnog image-a** — nosiš samo rezultat. Ogromna ušteda (270 MB → 10 MB).
- Faze se referenciraju imenom (`AS builder`) ili rednim brojem (`--from=0`).

**Sažetak:** Multi-stage build dijeli Containerfile na više `FROM` faza; u jednoj se kompajlira/gradi s teškim alatima, a u finalnu se `COPY --from=` prenosi **samo gotov artefakt**. Finalni image tako ne nosi kompajler ni izvorni kod — desetke puta je manji. Faze se imenuju s `AS`.

---

## ZADATAK 3 — `.containerignore` (isključi datoteke iz build konteksta)

> Add a .containerignore file and demonstrate that ignored files are excluded from the build context / not copied.

```bash
echo "tajna" > secret.txt
echo "podaci" > data.txt

cat > .containerignore << 'EOF'
secret.txt
EOF

cat > Containerfile << 'EOF'
FROM alpine:3.20
COPY . /app
CMD ["ls", "/app"]
EOF

podman build -t ignoretest:1.0 -f Containerfile .
podman run --rm ignoretest:1.0   # vidi data.txt, NE vidi secret.txt
```

**Razlomljeno:**
- `echo "tajna" > secret.txt` — `echo` ispiše tekst, `>` ga preusmjeri u datoteku (stvori/prepiši).
- `.containerignore` — popis (jedan uzorak po retku) datoteka koje se **NE šalju** u build kontekst → `COPY` ih ne može pokupiti. (Ekvivalent Dockerovog `.dockerignore`.)
- `COPY . /app` — kopiraj **sve iz konteksta** u `/app`. Kontekst je već "očišćen" jer je `secret.txt` izbačen.
- `CMD ["ls", "/app"]` — izlistaj što je stvarno ušlo.

**Začkoljica (javila se uživo):**
- **`COPY requires at least two arguments`** → fali **razmak**: mora biti `COPY . /app` (točka, RAZMAK, odredište), NE `COPY ./app`. Bez razmaka Podman vidi samo jedan argument.

**Sažetak:** `.containerignore` izbacuje navedene datoteke iz build konteksta, pa ih `COPY .` ne pokupi — koristi se za tajne, smeće i velike nepotrebne datoteke. Sintaksa je jedan uzorak po retku. Pazi na razmak u `COPY . /app` (točka i odredište su dva odvojena argumenta).

---

## ZADATAK 4 — Tri taga odjednom + konvencija `registry/namespace/name:tag`

> Build an image and apply three tags at once (name:1.0, name:1, name:latest); explain the registry/namespace/name:tag convention.

```bash
cat > Containerfile << 'EOF'
FROM alpine:3.20
CMD ["echo", "tagged"]
EOF

podman build -t myapp:1.0 -t myapp:1 -t myapp:latest -f Containerfile .
podman images myapp   # tri retka, ISTI IMAGE ID
```

**Razlomljeno:**
- Više `-t` u istom buildu = **jedan image, više imena (tagova)**. Svi pokazuju na isti hash → isti IMAGE ID u listi.
- **Konvencija punog imena:** `registry/namespace/name:tag`
  - `registry` — gdje image živi (`docker.io`, `quay.io`). Ako se izostavi → podrazumijeva se Docker Hub.
  - `namespace` — račun/organizacija (`library` za službene, `tomislavb16` za tvoj).
  - `name` — ime image-a.
  - `tag` — verzija/oznaka. Ako se izostavi → **`latest`**.

**Začkoljica:**
- **`latest` NIJE "najnovije"** — to je samo **default oznaka** koja se uzme kad ne navedeš tag. Image označen `latest` može biti star; ime ne garantira ništa.
- Tagovi su samo **pokazivači** na isti image ID (kao više naljepnica na istoj kutiji).

**Sažetak:** Više `-t` zastavica daje jednom image-u više tagova koji svi pokazuju na isti IMAGE ID. Puno ime ide po konvenciji `registry/namespace/name:tag`, gdje izostavljeni registry znači Docker Hub, a izostavljeni tag znači `latest`. `latest` je default oznaka, ne jamstvo da je image najnoviji.

---

## ZADATAK 5 — `podman history` i smanjenje broja slojeva

> Inspect an image's layer history with podman history and explain how to reduce the number of layers.

```bash
podman history myapp:1.0
podman history --no-trunc --format "{{.Size}}\t{{.CreatedBy}}" myapp:1.0
```

**Razlomljeno:**
- `podman history <img>` — prikaže **layer po layer**: svaka instrukcija (`RUN`, `COPY`, `ENV`...) = jedan red, s veličinom koju je dodao.
- `--no-trunc` — ne skraćuj dugačke naredbe (pokaži cijele).
- `--format "{{.Size}}\t{{.CreatedBy}}"` — ispiši samo veličinu i naredbu koja je stvorila layer. `\t` = tab (poravnanje). Go-template sintaksa.

**Kako smanjiti broj slojeva:**
```dockerfile
# Loše — 3 RUN-a = 3 sloja
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# Dobro — 1 RUN = 1 sloj
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
```
- Spoji `RUN`-ove s `&&` i `\` (nastavak retka).
- Koristi multi-stage (zadatak #2).
- Kombiniraj `COPY` gdje ima smisla.

**Sažetak:** `podman history` pokazuje slojeve image-a redom, sa zaslugom svakog za veličinu. Manje slojeva = spajanje više `RUN` naredbi u jednu s `&&` i `\`, plus multi-stage build. Ovo direktno smanjuje veličinu i lakše je za održavanje.

---

## ZADATAK 6 — Non-root `USER` + zapisiv `WORKDIR`

> Write a Containerfile that sets a non-root USER and a writable WORKDIR; run it and confirm whoami is not root.

```bash
cat > Containerfile << 'EOF'
FROM alpine:3.20
RUN adduser -D -u 1001 appuser
RUN mkdir -p /data && chown appuser:appuser /data
USER appuser
WORKDIR /data
CMD ["whoami"]
EOF

podman build -t nonroot:1.0 -f Containerfile .
podman run --rm nonroot:1.0                                # -> appuser (NIJE root)
podman run --rm nonroot:1.0 id                             # uid=1001(appuser)
podman run --rm nonroot:1.0 sh -c 'touch /data/x && echo OK'   # -> OK (WORKDIR zapisiv)
```

**Razlomljeno:**
- `adduser -D -u 1001 appuser` — stvori korisnika `appuser`. `-D` = bez lozinke (disabled password, dovoljno za kontejner). `-u 1001` = fiksni UID.
- `mkdir -p /data && chown appuser:appuser /data` — napravi direktorij i **predaj ga korisniku** (`chown vlasnik:grupa putanja`).
- `USER appuser` — od ove točke nadalje (i pri runu) kontejner radi kao `appuser`, ne root.
- `WORKDIR /data` — radni direktorij; mora pripadati `appuser`-u inače nema prava pisanja.
- `CMD ["whoami"]` — ispiše tko je efektivni korisnik.

**Začkoljica:**
- **`WORKDIR` mora biti `chown`-an na non-root korisnika**, inače pisanje (`touch`) padne s "permission denied" (LO5 #17 je baš to).
- Redoslijed: prvo stvori korisnika i direktorij + `chown`, **pa** `USER`. Ako prebaciš na `USER` prerano, kasnije naredbe nemaju root prava.

**Sažetak:** Non-root kontejner postaviš s `USER <ime>` nakon što stvoriš korisnika (`adduser -D`) i `chown`-aš mu radni direktorij. `WORKDIR` mora biti u vlasništvu tog korisnika da bi pisanje radilo. Pokretanje kao non-root je sigurnosna najbolja praksa (manja šteta ako netko provali).

---

## ZADATAK 7 — `HEALTHCHECK` (+ `--format docker`!)

> Add a HEALTHCHECK to a Containerfile, run the container, and show the health status in podman ps / via podman healthcheck run.

```bash
cat > Containerfile << 'EOF'
FROM alpine:3.20
RUN apk add --no-cache curl
CMD ["sh", "-c", "while true; do echo -e 'HTTP/1.1 200 OK\n\nok' | nc -l -p 8080; done"]
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/ || exit 1
EOF

podman build --format docker -t healthy:1.0 -f Containerfile .
podman run -d --name hc healthy:1.0
podman ps                                              # STATUS: (healthy) / (starting)
podman healthcheck run hc                              # -> healthy
podman inspect hc --format '{{.State.Health.Status}}'  # -> healthy
podman rm -f hc
```

**Razlomljeno:**
- `apk add --no-cache curl` — instaliraj curl (treba za provjeru). `--no-cache` = ne ostavljaj keš indeksa.
- `HEALTHCHECK --interval=10s --timeout=3s --retries=3 CMD ...` — definira naredbu kojom Podman periodički provjerava radi li app stvarno (ne samo da proces živi).
  - `--interval=10s` — provjeravaj svakih 10 s.
  - `--timeout=3s` — ako provjera traje dulje od 3 s → neuspjeh.
  - `--retries=3` — tek nakon 3 uzastopna neuspjeha označi `unhealthy`.
- `curl -f ... || exit 1` — `-f` = neka curl vrati grešku na HTTP 4xx/5xx; `|| exit 1` = ako curl padne, vrati izlazni kod 1 (= unhealthy).
- **`--format docker`** pri **buildu** — KLJUČNO (vidi začkoljicu).

**Začkoljica (zlato, javila se uživo):**
- **`HEALTHCHECK ... will be ignored. Must use 'docker' format`** → Podman po defaultu gradi u **OCI** formatu koji **tiho ignorira** HEALTHCHECK. Moraš graditi s **`--format docker`** da HEALTHCHECK uđe u image.

**Sažetak:** `HEALTHCHECK` definira periodičnu provjeru je li aplikacija uistinu funkcionalna, sa statusom vidljivim u `podman ps` (`healthy`/`unhealthy`) ili kroz `podman healthcheck run`. Parametri `--interval/--timeout/--retries` podešavaju ritam i toleranciju. **Obavezno graditi s `--format docker`** — OCI format ga tiho ignorira.

---

## ZADATAK 8 — `ENTRYPOINT` vs `CMD` (+ override CMD pri runu)

> Demonstrate the difference between ENTRYPOINT and CMD in one Containerfile that uses both, then override CMD at podman run time.

```bash
cat > Containerfile << 'EOF'
FROM alpine:3.20
ENTRYPOINT ["echo", "Pozdrav,"]
CMD ["svijete"]
EOF

podman build -t epcmd:1.0 -f Containerfile .
podman run --rm epcmd:1.0            # -> Pozdrav, svijete   (ENTRYPOINT + CMD)
podman run --rm epcmd:1.0 Podmanu    # -> Pozdrav, Podmanu   (CMD pregažen)
```

**Razlomljeno:**
- `ENTRYPOINT ["echo", "Pozdrav,"]` — **fiksni** dio naredbe. Uvijek se izvrši; teško ga je promijeniti (treba poseban flag `--entrypoint`).
- `CMD ["svijete"]` — **zadani argument** koji se predaje entrypointu. **Lako se pregazi** — sve što napišeš iza imena image-a u `podman run` zamijeni CMD.
- Bez argumenata: `echo Pozdrav, svijete`. S argumentom `Podmanu`: `echo Pozdrav, Podmanu`.

**Začkoljica:**
- **ENTRYPOINT = fiksno, CMD = pregazivo.** Argumenti iz `podman run` mijenjaju **CMD**, ne ENTRYPOINT.
- Za pregaziti i ENTRYPOINT: `podman run --entrypoint <novo> image`.
- Tipičan obrazac: `ENTRYPOINT` = program, `CMD` = defaultni argumenti koje korisnik može promijeniti.

**Sažetak:** `ENTRYPOINT` je fiksni dio naredbe koji se uvijek izvrši, a `CMD` su zadani argumenti koje korisnik lako pregazi navodeći nove iza imena image-a. Zajedno daju "program + defaultni argument" obrazac. ENTRYPOINT se mijenja samo s `--entrypoint`.

---

## ZADATAK 9 — `ADD` vs `COPY`

> Explain the difference between ADD and COPY and write a Containerfile that justifies using each.

```bash
echo "sadrzaj" > file.txt
tar -czf arhiva.tar.gz file.txt

cat > Containerfile << 'EOF'
FROM alpine:3.20
COPY file.txt /copied/file.txt
ADD arhiva.tar.gz /added/
CMD ["ls", "-R", "/copied", "/added"]
EOF

podman build -t addcopy:1.0 -f Containerfile .
podman run --rm addcopy:1.0   # /added sadrzi RASPAKIRAN file.txt, /copied ima file.txt
```

**Razlomljeno:**
- `tar -czf arhiva.tar.gz file.txt` — napravi tar.gz arhivu. `-c` = create, `-z` = gzip kompresija, `-f` = ime datoteke.
- `COPY file.txt /copied/file.txt` — **čisto kopiranje** datoteke u image. **Default izbor.**
- `ADD arhiva.tar.gz /added/` — kao COPY, ali ima dvije "magije":
  1. **Lokalni tar arhiv automatski RASPAKIRA** na odredište.
  2. Može **skinuti datoteku s URL-a** (`ADD https://... /x`).
- `CMD ["ls", "-R", ...]` — `-R` = rekurzivno izlistaj (vidiš da je u `/added` raspakiran sadržaj).

**Začkoljica:**
- **Pravilo: koristi `COPY` osim ako baš trebaš ADD-ovu magiju** (raspakiranje tar-a ili URL). ADD je "previše pametan" — može iznenaditi (npr. neočekivano raspakira nešto).
- ADD s URL-om se ne preporučuje (bolje `RUN curl/wget` jer imaš kontrolu i čišćenje).

**Sažetak:** `COPY` čisto kopira datoteke iz konteksta u image i to je default izbor. `ADD` dodatno automatski raspakira lokalne tar arhive i može skidati s URL-a, pa ga koristiš samo kad ti baš ta funkcionalnost treba. Zbog "skrivene magije" ADD se inače izbjegava.

---

## ZADATAK 10 — `save` / `load` (air-gapped prijenos)

> Save an image to a tarball with podman save, remove it, then restore it with podman load; explain when you'd do this (air-gapped transfer).

```bash
podman save -o myapp.tar myapp:1.0      # spremi image u .tar
podman rmi myapp:1.0                     # obriši image lokalno
podman load -i myapp.tar                 # vrati ga iz .tar
podman images myapp                      # opet je tu
```

**Razlomljeno:**
- `podman save -o myapp.tar myapp:1.0` — spremi image (sve layere + metapodatke) u jednu `.tar` datoteku. `-o` = output datoteka.
- `podman rmi myapp:1.0` — obriši image. **`rmi` = remove IMAGE** (ne kontejner!).
- `podman load -i myapp.tar` — učitaj image natrag iz tarballa. `-i` = input datoteka.

**Začkoljica:**
- **`rmi` briše image, `rm` briše kontejner** — lako pomiješati.
- **Kad ovo koristiš:** **air-gapped** okruženje (računalo bez interneta) — image preneseš USB-om/mrežom kao datoteku, bez registra. Ili arhiviranje točne verzije.

**Sažetak:** `podman save -o file.tar img` pakira cijeli image u tarball, a `podman load -i file.tar` ga vraća — bez registra. Koristi se za prijenos u izolirana (air-gapped) okruženja ili arhiviranje. Zapamti: `rmi` = image, `rm` = kontejner.

---

## ZADATAK 11 — Pull po digestu (`name@sha256:...`)

> Pull an image by digest (name@sha256:...) and explain why it's more reproducible than a tag.

```bash
# 1) povuci po tagu pa pročitaj REGISTRYJEV digest
podman pull alpine:3.20
REGDIGEST=$(podman image inspect alpine:3.20 --format '{{.Digest}}')
echo "$REGDIGEST"      # sha256:....

# 2) povuci po digestu
podman pull alpine@$REGDIGEST
```

**Razlomljeno:**
- `REGDIGEST=$(...)` — spremi izlaz naredbe u varijablu. `$(...)` = "pokreni ovo i uvrsti rezultat".
- `--format '{{.Digest}}'` — izvuci **registryjev** digest (sha256 sažetak manifesta). Go-template.
- `echo "$REGDIGEST"` — `$ime` = pročitaj **vrijednost** varijable (bez `$` bi ispisao golo ime).
- `alpine@$REGDIGEST` — povuci točno taj image po digestu. `@sha256:...` = "baš ovaj sadržaj, ne pomični tag".

**Začkoljica (dvije, obje uživo):**
- **`manifest unknown`** pri pull po digestu → koristio si **`.RepoDigests[0]`** (LOKALNI digest) koji nakon `save`/`load` ili na drugom računalu ne mora postojati na Hubu. **Koristi `'{{.Digest}}'`** (registryjev digest).
- **Ispisuje ime varijable umjesto vrijednosti** → **fali `$`**. Pravilo: `$ime` = vrijednost, `ime` = goli tekst.

**Zašto je digest reproducibilniji od taga:**
- **Tag je pomičan** — `alpine:3.20` danas i za mjesec dana mogu biti **različiti** image-i (maintaineri pregaze tag novom verzijom).
- **Digest je nepromjenjiv** — `@sha256:...` uvijek pokazuje **točno isti sadržaj**. Garantirana reproducibilnost.

**Sažetak:** Pull po digestu (`name@sha256:...`) dohvaća **točno određeni** sadržaj image-a, dok je tag pomičan pokazivač koji maintaineri mogu pregaziti. Zato je digest reproducibilniji. Koristi **registryjev** digest iz `'{{.Digest}}'`, ne lokalni `.RepoDigests`, i ne zaboravi `$` pri čitanju varijable.

---

## ZADATAK 12 — `--no-cache` + layer caching

> Build an image with --no-cache and explain layer caching and when disabling it helps or hurts.

```bash
podman build -t cachetest:1.0 -f Containerfile .                # prvi build (gradi sve)
podman build -t cachetest:1.0 -f Containerfile .                # drugi (sve iz keša, brzo)
podman build --no-cache -t cachetest:1.0 -f Containerfile .     # ignorira keš, gradi sve nanovo
```

**Razlomljeno:**
- `--no-cache` — **ignoriraj keširane layere**, sagradi svaki nanovo.

**Kako keš radi (layer caching):**
- Podman kešira svaki layer. Pri ponovnom buildu, ako se instrukcija I sve iznad nje nisu promijenile → koristi keširani layer (brzo).
- **Keš se lomi odozgo prema dolje na PRVOJ promjeni.** Čim se jedan layer promijeni, svi ispod njega se grade nanovo (jer ovise o njemu).
- Zato se **`COPY` izvornog koda stavlja što kasnije** — da promjena koda ne ruši keš instalacije ovisnosti (LO5 #12).

**Kad `--no-cache` POMAŽE:**
- Build argovi/vanjski resursi se keširaju a ti želiš svjež povlak (npr. `apt-get update` da pokupi nove pakete).
- Sumnjaš da je keš "zastario" i daje krivi rezultat.

**Kad ŠKODI:**
- Bespotrebno sporo — sve se gradi iznova iako nije trebalo.

**Začkoljica:**
- Tipičan slučaj (javio se u #15): drugi build s `--build-arg ...=2.0` pokupi **stari** keširani `1.0` layer. Rješenje: `--no-cache`.

**Sažetak:** Layer caching ubrzava ponovne buildove koristeći nepromijenjene slojeve, lomeći se odozgo prema dolje na prvoj promjeni. `--no-cache` ignorira keš i gradi sve nanovo — korisno kad keš daje zastarjeli rezultat (npr. stari `apt-get update` ili build arg), ali bespotrebno sporo inače. Zato `COPY` koda ide kasno u Containerfileu.

---

## ZADATAK 13 — Build iz ne-defaultnog imena Containerfilea (`-f`) + build kontekst

> Build from a non-default Containerfile name using -f, and explain the build-context argument.

```bash
cat > Containerfile.dev << 'EOF'
FROM alpine:3.20
CMD ["echo", "dev build"]
EOF

podman build -t devbuild:1.0 -f Containerfile.dev .
podman run --rm devbuild:1.0   # -> dev build
```

**Razlomljeno:**
- `Containerfile.dev` — recept s **nestandardnim imenom** (Podman po defaultu traži `Containerfile`).
- `-f Containerfile.dev` — **ovdje je `-f` NUŽAN**, ne kozmetika: govori Podmanu koja datoteka je recept jer ime nije default.
- `.` — build kontekst (trenutna mapa). I dalje obavezan i neovisan o `-f`.

**Začkoljica:**
- **`-f` bira RECEPT, `.` (zadnji argument) bira MATERIJAL** (kontekst iz kojeg `COPY` čita). Mogu biti različite mape.
- **`checking on sources under <kontekst>: ... no such file`** → ili krivi kontekst, ili krivo ime datoteke u `COPY`. Pročitaj koju putanju terminal traži pa usporedi sa stvarnim stanjem.

**Sažetak:** `-f <ime>` govori Podmanu da koristi recept s nestandardnim imenom (default je `Containerfile`). Zadnji argument (`.`) je build kontekst — mapa iz koje `COPY`/`ADD` čitaju, neovisna o lokaciji recepta. Razdvajanje `-f` (recept) i konteksta (materijal) omogućuje npr. različite Containerfileove nad istom mapom.

---

## ZADATAK 14 — `COPY --chown` (vlasništvo datoteka u image-u)

> Use COPY --chown to set file ownership in the image and verify with ls -l inside the container.

```bash
echo "data" > app.conf

cat > Containerfile << 'EOF'
FROM alpine:3.20
RUN adduser -D -u 1001 appuser
COPY --chown=appuser:appuser app.conf /home/appuser/app.conf
USER appuser
CMD ["ls", "-l", "/home/appuser/app.conf"]
EOF

podman build -t chowntest:1.0 -f Containerfile .
podman run --rm chowntest:1.0   # vlasnik je appuser appuser, ne root
```

**Razlomljeno:**
- `COPY --chown=appuser:appuser app.conf /home/appuser/app.conf` — kopiraj datoteku I odmah joj **postavi vlasnika** u istom koraku. Format `--chown=vlasnik:grupa`.
- Bez `--chown` kopirana datoteka bi pripadala **root**-u (default), pa je non-root korisnik ne bi mogao mijenjati.
- `ls -l` — "long" listing, pokaže vlasnika/grupu/prava (`-l` = long format).

**Začkoljica:**
- **Vlasništvo se postavlja u ISTOM layeru** kao kopiranje. Alternativa (`COPY` pa zaseban `RUN chown`) stvara dodatni layer i dvostruko zauzeće (datoteka u dva layera).
- **`no such file`** u `COPY` → krivo ime datoteke. Provjeri da datoteka stvarno postoji u kontekstu.

**Sažetak:** `COPY --chown=user:group` kopira datoteku i u istom layeru joj postavlja vlasništvo, što je nužno da non-root korisnik smije pisati po njoj. Bez toga kopirana datoteka pripada rootu. Rješavanje u jednom koraku štedi layer u odnosu na zaseban `RUN chown`.

---

## ZADATAK 15 — `ARG` + `ENV` (build-time vs run-time varijable)

> Write a Containerfile combining ARG and ENV (build-time vs run-time variables) and demonstrate the difference.

```bash
cat > Containerfile << 'EOF'
ARG APP_VERSION=1.0
ENV BUILT_VERSION=${APP_VERSION}
FROM alpine:3.20
ENV BUILT_VERSION=${BUILT_VERSION}
CMD ["sh", "-c", "echo Verzija: $BUILT_VERSION"]
EOF

podman build -t argenv:1.0 --build-arg APP_VERSION=1.0 -f Containerfile .
podman build --no-cache -t argenv:2.0 --build-arg APP_VERSION=2.0 -f Containerfile .

podman run --rm argenv:1.0   # -> Verzija: 1.0
podman run --rm argenv:2.0   # -> Verzija: 2.0
podman run --rm argenv:2.0 env | grep BUILT_VERSION   # BUILT_VERSION postoji u kontejneru
```

**Razlomljeno:**
- `ARG APP_VERSION=1.0` — **build-time** varijabla (postoji samo dok traje build, pregaziva s `--build-arg`).
- `ENV BUILT_VERSION=${APP_VERSION}` — **run-time** varijabla; preuzima vrijednost iz ARG-a i **ostaje zapisana u image-u**.
- `CMD ["sh", "-c", "echo Verzija: $BUILT_VERSION"]` — `sh -c` jer trebamo shell da razriješi `$BUILT_VERSION`.

**Razlika (suština zadatka):**
- **`ARG`** — vidljiv **samo pri buildu**, ne postoji u pokrenutom kontejneru.
- **`ENV`** — vidljiv **i pri buildu i u kontejneru** (`podman run ... env` ga pokaže), može se pregaziti pri runu s `-e`.

**Začkoljica (obje uživo):**
- **`BUILT_VERSION=BUILT_VERSION` (ispisuje ime umjesto vrijednosti)** → fali **`$`** ispred varijable u CMD-u. `$ime` = vrijednost.
- **Drugi build pokaže staru vrijednost** → keš pokupio stari `1.0` layer. **`--no-cache`** na drugom buildu.

**Sažetak:** `ARG` je build-time varijabla koja ne preživi u kontejner, dok `ENV` postaje trajna varijabla okruženja vidljiva i pri buildu i u pokrenutom kontejneru. Tipično `ENV` preuzme vrijednost iz `ARG`-a da bi je "zapamtio". Pazi na `$` pri čitanju i na keš pri mijenjanju build arga (`--no-cache`).

---

## ZADATAK 16 — `podman login` + push u privatni repo (gdje se čuvaju kredencijali)

> podman login to a registry and push an image to a private repository; explain where/how the credentials are stored.

```bash
# login (token ide kroz stdin, ne na ekran)
cat ~/docker/token | podman login docker.io -u tomislavb16 --password-stdin   # -> Login Succeeded!

# tag pod privatni repo i push
podman tag myapp:1.0 docker.io/tomislavb16/private-app:1.0
podman push docker.io/tomislavb16/private-app:1.0

# gdje su kredencijali
cat ${XDG_RUNTIME_DIR}/containers/auth.json
```

**Razlomljeno:**
- `cat ~/docker/token | podman login ... --password-stdin` — pročitaj token iz datoteke i predaj ga loginu kroz **stdin**.
  - `|` = pipe (izlaz lijevo → ulaz desno).
  - `--password-stdin` = "ne traži lozinku interaktivno, čitaj sa standardnog ulaza". Sigurnije (token ne ostaje u shell historiji ni na ekranu).
- `-u tomislavb16` — korisničko ime.
- `podman tag ... docker.io/tomislavb16/private-app:1.0` — daj image puno ime s tvojim namespace-om (treba za push).
- `podman push ...` — gurni na registar.
- `${XDG_RUNTIME_DIR}/containers/auth.json` — gdje Podman sprema login token.

**Začkoljica (sve uživo):**
- **Kredencijali se čuvaju u `auth.json`** (`${XDG_RUNTIME_DIR}/containers/auth.json`) kao **base64** — to je **enkodiranje, NE enkripcija** (vidljivo, trivijalno reverzibilno). Nije sigurno za dijeljenje.
- **Docker Hub odbija običnu lozinku** (pogotovo uz Google OAuth login bez native lozinke) → treba **Access Token** (Read & Write).
- **`--pasword-stdin` (jedno `s`)** → tipfeler, padne. Mora `--password-stdin`.
- **`docker-io` vs `docker.io`** → crtica umjesto točke, padne.
- **Clipboard ne radi VM↔host** → token unosiš ručno (npr. `nano ~/token`) ili Firefox **unutar** VM-a (tada copy/paste radi).
- ⚠️ **Ne ispisuj token na ekran** (`cat token` ga pokaže) ako dijeliš screenshot — opozovi ga ako se otkrije.

**Sažetak:** `podman login` autenticira na registar (token kroz `--password-stdin` radi sigurnosti), nakon čega `podman push` gura tagiran image u tvoj namespace. Kredencijali se spremaju u `auth.json` kao **base64 (enkodiranje, ne enkripcija)** — vidljivi i reverzibilni. Docker Hub traži Access Token, ne običnu lozinku.

---

## ZADATAK 17 — Dangling images (`podman image prune` + `rmi`)

> Remove dangling images with podman image prune and a specific image with podman rmi; explain what "dangling" means.

```bash
# napravi dangling: sagradi pod istim tagom dvaput (drugi build "otme" tag)
podman build -t danglingdemo:1.0 -f Containerfile .
# (izmijeni Containerfile pa ponovo)
podman build -t danglingdemo:1.0 -f Containerfile .
podman images                          # vidiš <none> <none> red = dangling

podman rmi <ID>                        # obriši konkretan (po ID-u)
podman image prune                     # obriši SVE dangling odjednom
```

**Razlomljeno:**
- `podman images` — u listi se pojavi red s `<none>` u REPOSITORY i TAG → to je **dangling image**.
- `podman rmi <ID>` — obriši konkretan image po hash ID-u. (Vidio si **dvije** `Deleted:` linije: image + njegov sad-osirotjeli layer.)
- `podman image prune` — obriši **sve** dangling image-e odjednom (počisti smeće).

**Što je "dangling":**
- **Dangling = image bez taga** (`<none>:<none>`). Nastaje kad **reiskoristiš tag** pri rebuildu: novi image preuzme tag `danglingdemo:1.0`, a stari ostane bez imena (ali zauzima prostor).
- To NIJE isto što i neiskorišten image — dangling je specifično "bezimeni ostatak".

**Sažetak:** Dangling image je image bez taga (`<none>:<none>`), tipično nastao kad rebuild "otme" tag prethodnom image-u. `podman rmi <ID>` briše konkretan (uz osirotjele layere), a `podman image prune` počisti sve dangling odjednom. Tako se oslobađa prostor od nakupljenih bezimenih ostataka.

---

## ZADATAK 18 — Statički file-server (`python -m http.server`)

> Write a Containerfile for a static file server using python -m http.server over a directory you COPY in, expose the port, and run it.

```bash
mkdir -p site
echo "<h1>Bok iz kontejnera</h1>" > site/index.html

cat > Containerfile << 'EOF'
FROM python:3.12-alpine
WORKDIR /web
COPY site/ /web/
EXPOSE 8000
CMD ["python", "-m", "http.server", "8000"]
EOF

podman build -t staticserver:1.0 -f Containerfile .
podman run -d --name web18 -p 8000:8000 staticserver:1.0
curl http://localhost:8000      # -> <h1>Bok iz kontejnera</h1>
podman rm -f web18
```

**Razlomljeno:**
- `python:3.12-alpine` — Python na maloj Alpine bazi (~51 MB umjesto ~1 GB za običan `python:3.12`).
- `WORKDIR /web` — radni direktorij; `http.server` poslužuje datoteke iz njega.
- `COPY site/ /web/` — kopiraj sadržaj mape `site/` u `/web/`.
- `EXPOSE 8000` — **najavi** da servis sluša na 8000. (Samo dokumentacija, ne otvara port! Vidi #20.)
- `CMD ["python", "-m", "http.server", "8000"]` — pokreni Pythonov ugrađeni HTTP server na portu 8000. `-m` = pokreni modul kao skriptu.
- `-p 8000:8000` — **stvarno objavi** port (host 8000 → kontejner 8000).

**Začkoljica:**
- **`EXPOSE` ne otvara port** — moraš ga objaviti s `-p` pri runu. EXPOSE je samo najava (LO5 #15).

**Sažetak:** Pythonov `http.server` modul poslužuje statičke datoteke iz `WORKDIR`-a bez ikakvog dodatnog koda. `EXPOSE` samo dokumentira port, a stvarni pristup s hosta tražiš s `-p host:container`. Mala `-alpine` baza drži image višestruko manjim.

---

## ZADATAK 19 — Pinanje verzije paketa (reproducibilnost)

> Build an image that installs a specific pinned version of a package (apt or apk) and explain why pinning matters for reproducibility.

```bash
# 1) otkrij DOSTUPNU verziju (jer repo drži samo trenutnu)
podman run --rm alpine:3.20 sh -c "apk update >/dev/null && apk list curl"
# -> curl-8.14.1-r2 ...

# 2) pinaj na taj broj
cat > Containerfile << 'EOF'
FROM alpine:3.20
RUN apk add --no-cache curl=8.14.1-r2
CMD ["curl", "--version"]
EOF

podman build -t pinned-curl:1.0 -f Containerfile .
podman run --rm pinned-curl:1.0   # -> curl 8.14.1 ...
```

**Razlomljeno:**
- `apk update >/dev/null && apk list curl` — povuci svjež indeks (`>/dev/null` skriva bučni ispis) pa ispiši dostupnu verziju curl-a.
- `apk add --no-cache curl=8.14.1-r2` — instaliraj **točno** tu verziju. Sintaksa pina = **`paket=verzija`**. `--no-cache` = bez keša indeksa (Alpine specifično, drži image manjim).
- (Debian ekvivalent: `apt-get install -y curl=<verzija>`; dostupnu verziju nađeš s `apt-cache policy curl`.)

**Začkoljica (obje uživo):**
- **Alpine/Debian glavni repo drže SAMO trenutnu verziju** — stare se brišu. Slijepo prepisivanje broja iz tutoriala → **`no such package`** / **`Unable to locate package`**. Zato **prvo otkrij** dostupnu verziju pa pinaj na nju.
- **`exit status 127`** = command not found (krivo ime KOMANDE, npr. `app-get` umjesto `apt-get`).

**Zašto pinanje = reproducibilnost:**
- Bez pina (`apk add curl`) dobiješ **koju god** verziju repo trenutno nudi → isti Containerfile danas i sutra može dati različit binarij.
- S pinom (`curl=8.14.1-r2`) **uvijek** isti binarij → build je reproducibilan.

**Sažetak:** Pinanje verzije (`paket=verzija`) osigurava da isti Containerfile uvijek instalira identičan binarij — to je reproducibilnost. Repozitoriji drže samo trenutnu verziju, pa prvo otkriješ dostupnu (`apk list` / `apt-cache policy`) pa pinaš na nju, inače build pada s "no such package". `127` znači krivo ime komande.

---

## ZADATAK 20 — Inspekcija nepoznatog image-a pa pokretanje prema njemu

> Inspect an unknown image with podman image inspect to discover its default Entrypoint/Cmd, exposed ports, and env, then run it accordingly.

```bash
podman pull docker.io/library/nginx:1.27

# cijeli Config blok
podman image inspect nginx:1.27

# ciljano polje po polje (Go-template)
podman image inspect nginx:1.27 --format '{{.Config.Cmd}}'           # [nginx -g daemon off;]
podman image inspect nginx:1.27 --format '{{.Config.Entrypoint}}'    # [/docker-entrypoint.sh]
podman image inspect nginx:1.27 --format '{{.Config.ExposedPorts}}'  # map[80/tcp:{}]
podman image inspect nginx:1.27 --format '{{.Config.Env}}'           # [PATH=... NGINX_VERSION=...]

# pokreni PREMA onome što kaže (port 80 jer inspect tako kaže)
podman run -d --name web20 -p 8080:80 nginx:1.27
podman ps                          # PORTS: 0.0.0.0:8080->80/tcp
curl http://localhost:8080         # nginx welcome stranica
podman rm -f web20
```

**Razlomljeno:**
- `podman image inspect <img>` — ispiše dugačak JSON; sekcija **`Config`** nosi recept kako image radi.
- `--format '{{.Config.Cmd}}'` — izvuci pojedino polje. `.` = korijenski objekt; `.Config.Cmd` = uđi u `Config`, izvuci `Cmd`. Go-template.
- Polja koja čitaš: `Cmd` + `Entrypoint` (što se vrti), `ExposedPorts` (najavljeni port), `Env` (ugrađene varijable, npr. verzija).
- `-p 8080:80` — objavi baš port 80 jer nam je inspect to rekao (ne nagađanjem).

**Začkoljice (pojmovi):**
- **`ExposedPorts` vraća `map[80/tcp:{}]`** — to je mapa (ključ→vrijednost); ključ `80/tcp`, vrijednost prazna `{}` jer EXPOSE ne nosi vrijednost, samo najavu.
- **`EXPOSE`/`ExposedPorts` je SAMO najava** — ne otvara port. Inspect ti kaže *koji* port, pa ga objaviš s `-p`.
- **Entrypoint + Cmd zajedno:** kod nginxa efektivno `/docker-entrypoint.sh nginx -g "daemon off;"`. `daemon off;` drži nginx u **foregroundu** (PID 1 živi → kontejner živi).

**Sažetak:** `podman image inspect` otkriva kako nepoznat image treba pokrenuti: `Config` sadrži `Entrypoint`/`Cmd` (što se vrti), `ExposedPorts` (najavljeni port) i `Env` (ugrađene varijable), a pojedino polje izvučeš s `--format '{{.Config.X}}'`. `EXPOSE` je samo najava, pa port stvarno objaviš s `-p`. Tako pokrećeš image svjesno, prema njegovim metapodacima, a ne nagađanjem.

---

## ZADATAK 21 — Multi-line `RUN ... && ...` (instalacija + čišćenje u ISTOM layeru)

> Write a Containerfile with a multi-line RUN ... && ... that installs packages and cleans the package cache in the same layer; explain why cleanup must share the layer.

```bash
cat > Containerfile << 'EOF'
FROM debian:12
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*
CMD ["curl", "--version"]
EOF

podman build -t clean-layer:1.0 -f Containerfile .

# DOKAZ uštede: "loša" verzija bez čišćenja
cat > Containerfile.bad << 'EOF'
FROM debian:12
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl ca-certificates
CMD ["curl", "--version"]
EOF

podman build -t clean-layer:bad -f Containerfile.bad .
podman images clean-layer    # 1.0 = 138 MB, bad = 158 MB (+20 MB cache)
```

**Razlomljeno:**
- `RUN ... && \` — `\` na kraju retka = naredba se nastavlja u sljedećem; sve je **jedan** `RUN` = **jedan** layer (samo vizualno prelomljen). To je "multi-line RUN".
- `apt-get update` — povuci svjež indeks paketa (bez ovoga `install` padne — LO5 #10).
- `&&` — "pokreni sljedeće samo ako prethodno uspije".
- `apt-get install -y` — instaliraj; `-y` = automatski "yes" (build nije interaktivan).
- `--no-install-recommends` — instaliraj samo tražene pakete, ne i "preporučene" dodatke. Dodatna ušteda.
- `ca-certificates` — curl bez ovoga ne može preko HTTPS-a.
- `rm -rf /var/lib/apt/lists/*` — obriši preuzeti indeks paketa. **Ovo je čišćenje — u ISTOM RUN-u.**

**Zašto čišćenje MORA dijeliti layer (suština):**
- Layer se snima kao snapshot **na kraju** svoje `RUN` instrukcije.
- Što obrišeš **prije** tog kraja (u istom RUN-u) → nestane, nikad ne uđe u snapshot.
- Što obrišeš u **sljedećem** RUN-u → prekasno; cache je već zapečaćen u prethodnom layeru, drugi RUN doda samo "whiteout" (sakrije) ali bajtovi i dalje putuju → image ostaje velik.
- **Dokaz uživo:** čisti 138 MB, "loša" verzija (cache ostao) 158 MB → **+20 MB** na istoj instalaciji.

**Začkoljica:**
- **`exit status 100`** = apt-ova greška (komanda postoji ali apt ne može odraditi — krivo ime PAKETA, npr. `ca-certificated`, ili fali `apt-get update`). Razlikuj od `127` (krivo ime komande).
- **`debconf: unable to initialize frontend: Dialog`** je **bezopasno** — apt samo javlja da nema interaktivni terminal. Build svejedno prođe.

**Sažetak:** Sve operacije u jednom `RUN ... && ...` čine **jedan layer**, pa se cache obrisan unutar njega nikad ne snimi u image. Ako bi čišćenje bilo u zasebnom RUN-u, cache bi već bio zapečaćen u prethodnom layeru i samo "skriven" (whiteout) — image bi ostao velik (dokazano: 138 vs 158 MB). Zato čišćenje mora dijeliti layer s instalacijom. `100` = apt greška (paket), `127` = kriva komanda.

---

## ZADATAK 22 — Tag + push na DVA registra (image mirroring)

> Tag and push the same image to two different registries (e.g. Docker Hub and Quay) and explain when image mirroring is useful.

```bash
# 0) image koji guramo
podman build -t lo2-push:1.0 -f Containerfile .

# 1) login na oba (oba traže vlastiti login)
cat ~/docker/token | podman login docker.io -u tomislavb16 --password-stdin   # Login Succeeded!
podman login quay.io -u tbrodar                                                # (CLI/encrypted password)

# 2) isti image, dva imena (naljepnice)
podman tag lo2-push:1.0 docker.io/tomislavb16/lo2-push:1.0
podman tag lo2-push:1.0 quay.io/tbrodar/lo2-push:1.0
podman images lo2-push    # tri reda, ISTI IMAGE ID

# 3) push na oba
podman push docker.io/tomislavb16/lo2-push:1.0
podman push quay.io/tbrodar/lo2-push:1.0
```

**Razlomljeno:**
- `podman tag <staro-ime> <novo-ime>` — dodaj postojećem image-u novo (puno) ime. Image se NE kopira — dobiva dodatnu naljepnicu (isti IMAGE ID).
- Razlika je samo u `registar/račun` dijelu: `docker.io/tomislavb16` vs `quay.io/tbrodar`.
- `podman push <puno-ime>` — gurni image na registar naveden u imenu (Podman zna kamo iz prefiksa `docker.io/` / `quay.io/`).

**Začkoljica (sve uživo):**
- **Quay često odbije običnu lozinku za CLI** → generiraj **CLI / Encrypted password** (Quay → Account Settings → Generate Encrypted Password), pa njime login.
- **`Copying blob ... skipped: already exists`** pri push-u → registar već ima taj layer (npr. Alpine bazu), pa ga ne šalje ponovo. Normalno i poželjno.
- Ako registar ne stvara repo automatski → napravi ga u browseru (Public, Empty repository).

**Kad je image mirroring koristan (TEORIJA — moraš znati objasniti):**
- **Redundancija/dostupnost** — ako jedan registar padne ili je nedostupan, image povučeš s drugog. Nema jedne točke kvara.
- **Geografija/brzina** — timovi blizu jednog registra vuku brže s njega.
- **Različiti ekosustavi** — npr. firma na Red Hat stacku vuče s Quaya, drugi s Docker Huba.
- **Izbjegavanje rate-limita** — Docker Hub ograničava broj pull-ova za anonimne/free korisnike; mirror na Quay zaobilazi to. (Direktna veza na **LO4 #2** — pull limit!)

**Sažetak:** Isti image dobije dva puna imena (`podman tag`, isti IMAGE ID) i gurne se na oba registra zasebnim `push`-evima — to je image mirroring. Koristan je za redundanciju (jedan registar padne → vučeš s drugog), brzinu po geografiji, različite ekosustave i zaobilaženje Docker Hub rate-limita. Svaki registar traži vlastiti login (Quay često encrypted/CLI password).

---

## ZADATAK 23 — Lista OS paketa + vrijednost minimalne baze

> Run an image and list the OS packages installed inside it; explain the security/size value of choosing a minimal base image.

```bash
# prebroji pakete (alat ovisi o OS-u)
podman run --rm debian:12 sh -c "dpkg -l | wc -l"        # ~93 (Debian, apt svijet)
podman run --rm alpine:3.20 sh -c "apk info | wc -l"     # ~14 (Alpine, apk svijet)

# vidi KOJE pakete
podman run --rm alpine:3.20 apk info                     # musl, busybox, apk-tools...

# poveži s veličinom
podman images | grep -E "debian|alpine"                  # debian ~121 MB, alpine ~8 MB
```

**Razlomljeno:**
- `dpkg -l` — Debianov alat: izlistaj instalirane pakete. `apk info` — Alpineov ekvivalent.
- `| wc -l` — `wc` = word count; `-l` = broji **retke**. Umjesto da čitamo listu, samo je prebrojimo.
- `apk info` (bez wc) — same nazive paketa.
- `grep -E "debian|alpine"` — filtriraj retke koji sadrže "debian" ili "alpine". `-E` = prošireni regex; `|` unutar navodnika = "ili".

**Brojevi (uživo):**
- **Debian 12: 93 paketa, ~121 MB.** **Alpine 3.20: 14 paketa, ~8.1 MB.** Alpine ~15× manji.
- Bonus uvid: `python:3.12-alpine` = 51 MB vs običan `python:3.12` ~1 GB. Zato službeni image-i nude `-alpine`/`-slim` varijante.

**Začkoljica:**
- **`apk info` warning** (`opening from cache ... No such file`) je **bezopasan** — samo nema svjež indeks (nismo radili `apk update`). Brojanje svejedno radi.
- `dpkg -l` ima par redaka zaglavlja, pa je stvarni broj paketa malo manji od ispisa `wc -l` (ne mijenja poantu).

**Sigurnosna/veličinska vrijednost minimalne baze (TEORIJA):**
- **Manja površina napada** — svaki paket je potencijalna ranjivost (CVE). 93 paketa = 93 mete; 14 = puno manje.
- **Lakše održavanje** — manje komponenti za patchanje.
- **Nema alata za napadača** — "debela" baza nudi `curl`, kompajlere, shell-ove za daljnji napad; minimalna (ili "distroless") mu ne da ni shell.
- **Veličina** — manje paketa → manji image → brži pull/push, manje storagea.

**Sažetak:** Broj instaliranih paketa (`dpkg -l`/`apk info` + `wc -l`) direktno se odražava na veličinu image-a (Debian 93 paketa/121 MB vs Alpine 14/8 MB). Minimalna baza je i manja i sigurnija: manja površina napada (manje CVE-ova), lakše održavanje i nema gotovih alata kojima bi se napadač služio. Zato se biraju `-alpine`/`-slim` baze ili distroless.

---

# 🏆 KOLEKCIJA DIJAGNOSTIČKIH GREŠAKA (zlato za ispit)

Sve su se stvarno pojavile uživo. Svaka ima jasan uzrok i fix.

| Poruka / simptom | Uzrok | Fix |
|---|---|---|
| `COPY requires at least two arguments` | Fali **razmak** u COPY (`./app` vs `. /app`) | `COPY . /app` (točka, RAZMAK, odredište) |
| `HEALTHCHECK ... will be ignored. Must use 'docker' format` | OCI default tiho ignorira HEALTHCHECK | Buildaj s **`--format docker`** |
| `manifest unknown` (pull po digestu) | Korišten **lokalni** `.RepoDigests` digest | Koristi **`'{{.Digest}}'`** (registryjev) |
| Ispisuje ime varijable umjesto vrijednosti (`X=X`) | Fali **`$`** | `$ime` = vrijednost; bez `$` = goli tekst |
| Drugi build pokaže staru vrijednost build arga | **Keš** pokupio stari layer | **`--no-cache`** |
| `checking on sources ... no such file` | Krivi **kontekst** ili krivo ime datoteke u COPY | Pročitaj putanju koju traži, usporedi sa stvarnim |
| `cd lo2-XX: No such file` | Bio si izvan krova; mapa nije tu | Počni svaki zadatak s `mkdir -p ~/lo2-XX && cd ~/lo2-XX` |
| Jedan `&` umjesto `&&` (npr. `mkdir & cd`) | `&` = pokreni u pozadini i NE čekaj | Koristi **`&&`** (čekaj uspjeh pa nastavi) |
| `exit status 127` | **Command not found** (krivo ime komande, npr. `app-get`) | Ispravi ime komande (`apt-get`) |
| `exit status 100` | **apt greška** (krivo ime PAKETA, npr. `ca-certificated`, ili fali `apt-get update`) | Ispravi ime paketa / dodaj update |
| `Unable to locate package` / `no such package` (s pinom) | Repo drži samo trenutnu verziju; pin zastario | Otkrij dostupnu (`apk list`/`apt-cache policy`) pa pinaj |
| `--pasword-stdin` | Tipfeler (jedno `s`) | `--password-stdin` |
| `docker-io` umjesto `docker.io` | Crtica umjesto točke | `docker.io` |
| `access denied` pri push-u | Login istekao / krivi namespace | Ponovi `podman login`; provjeri `registar/račun` u tagu |
| `Please select an image` (ponudi registryje) nakon pada builda | Image ne postoji lokalno, pa `run` pita s kojeg registra povući | **Ctrl+C, ne biraj** — prvo popravi build |
| `debconf: unable to initialize frontend: Dialog` | apt nema interaktivni terminal | **Bezopasno** — ignoriraj, build prolazi |

---

# 📌 BRZA REFERENCA — LO2 komande

```bash
# BUILD
podman build -t ime:tag -f Containerfile .          # osnovni build
podman build -t a:1 -t a:2 -t a:latest .            # više tagova odjednom
podman build --no-cache -t ime .                    # ignoriraj keš
podman build --format docker -t ime .               # za HEALTHCHECK
podman build --build-arg KEY=VAL -t ime .           # pregazi ARG

# INSPEKCIJA
podman images [ime]                                 # popis image-a + SIZE
podman history --no-trunc img                       # slojevi po veličini
podman image inspect img                            # cijeli JSON config
podman image inspect img --format '{{.Config.Cmd}}' # ciljano polje

# PRIJENOS / REGISTRI
podman save -o file.tar img                         # image -> tar
podman load -i file.tar                             # tar -> image
podman login docker.io -u USER --password-stdin     # login (token kroz stdin)
podman tag img registar/racun/ime:tag               # dodaj puno ime
podman push registar/racun/ime:tag                  # gurni na registar
podman pull ime@sha256:...                          # pull po digestu

# ČIŠĆENJE
podman rmi ime[:tag]                                # obriši IMAGE (rm = kontejner!)
podman image prune                                  # obriši sve dangling
```

**Containerfile instrukcije:** `FROM` (baza) · `ARG` (build-time var) · `ENV` (run-time var) · `WORKDIR` (radni dir) · `COPY`/`ADD` (datoteke; ADD raspakira tar/URL) · `RUN` (naredba pri buildu = layer) · `USER` (non-root) · `EXPOSE` (najava porta, NE otvara) · `HEALTHCHECK` (provjera zdravlja) · `ENTRYPOINT` (fiksno) · `CMD` (pregazivo).

---

> **LO2 gotov — svih 23 zadataka hands-on. ✅**
> Sljedeće: **LO3** (pods, compose, mreže, secrets, volumes, `generate kube`) — most prema Kubernetesu (LO4).
