# DevOps LO1 — Podman: kompletna ispitna skripta

> Tvoje bilješke za ispit (na ispitu dozvoljene). Svaki zadatak ima: **cilj → komande (razložene) → poanta → zamke**.
> Image koji svuda koristimo: `docker.io/library/httpd` (Apache). Radimo u `~/devops`.

---

## 0. TEMELJI (ovo drži sve ostalo)

### Image vs Container vs Zapisivi sloj — NAJVAŽNIJE

```
┌─ HOST (tvoj VM) ──────────────────────────────┐
│  tu tipkaš `podman ...`; tu žive host datoteke │
│  ┌─ CONTAINER (kad pokreneš image) ──────────┐ │
│  │  ZAPISIVI SLOJ  → čita i PIŠE (prolazno)  │ │
│  │     tvoje promjene: curl, /opt/marker.txt │ │
│  │  IMAGE (httpd)  → SAMO čita (zamrznuto)   │ │
│  └────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

- **Image** = recept, zamrznut na disku, **samo za čitanje**, nikad se sam ne mijenja. (Analogija: klasa.)
- **Container** = pokrenut image = image + tanki **zapisivi sloj** navrh. (Analogija: objekt.)
- Sve što mijenjaš unutra (instaliraš paket, napraviš datoteku) ide **samo u zapisivi sloj**, NE u image.
- **`podman rm`** briše container + zapisivi sloj → promjene nestanu. Image ostaje.
- **`podman commit`** zapeče zapisivi sloj u **novi image** → promjene postanu trajne.
- **`podman cp`** prenosi datoteke preko granice host ↔ container.

### Struktura komande
```
podman run [opcije bilo kojim redom] IMAGE [komanda]
```
Što je **iza imagea** = komanda koja se vrti unutar containera (zamijeni default).

### Flagovi (crtice)
- jedna crtica = kratko ime: `-d`, `-p`, `-v`
- dvije crtice = dugo ime: `--name`, `--restart`, `--memory`
- mogu se spajati: `-it` = `-i -t`

### `-p HOST:CONTAINER` (mapiranje porta)
- **desna** strana = port na kojem app sluša **unutra** (httpd = 80, fiksno)
- **lijeva** strana = slobodan izbor na hostu
- **Zamka:** obrnut redoslijed. `-p 80:8081` ≠ `-p 8081:80`.

### Rootless Podman — nema centralni daemon
- Docker ima daemon koji stalno motri containere; **Podman nema**.
- Svaki container ima mali nadzorni procesić **`conmon`** koji ga prati.
- Posljedica: `podman stop/kill` = tvoja **namjerna** naredba → restart policy se NE okida.

---

## #1 — httpd detached, custom port/name/hostname

**Cilj:** pokreni httpd u pozadini, mapiraj port 80→8081, daj custom `--name` i `--hostname`, potvrdi s `inspect`.

```bash
podman run -d -p 8081:80 --name web1 --hostname web-host docker.io/library/httpd
```
- `-d` — detached (u pozadini), terminal ostaje slobodan
- `-p 8081:80` — host 8081 → container 80
- `--name web1` — ime containera (za lako dohvaćanje)
- `--hostname web-host` — hostname *unutar* containera

```bash
podman inspect --format '{{.Name}} {{.Config.Hostname}}' web1
```
- `--format '{{...}}'` — ispiši samo zadana polja iz punog zapisa (Go-template; detaljno razloženo u #9)

**Poanta:** detached + objavljen port = server radi u pozadini i dostupan je na hostu. `--name` je za tebe, `--hostname` je identitet unutar containera.

---

## #2 — `--restart=always` + dokaz restarta

**Cilj:** pokreni s restart politikom, "ubij" glavni proces, dokaži da se Podman vrati.

```bash
podman run -d --restart=always --name restart-demo docker.io/library/httpd
```
- `--restart=always` — politika: uvijek vrati container kad stane
  - opcije: `no` (default), `on-failure` (samo na grešku), `always` (uvijek)

**KLJUČNO — kako ispravno dokazati:**
```bash
# OVO NE okida restart (namjeran stop):
podman kill restart-demo        # container ostane Exited (137)

# OVO okida restart (vanjska, neočekivana smrt):
podman start restart-demo
podman inspect --format '{{.State.Pid}}' restart-demo   # nađi host-PID, npr. 14665
kill -9 14665                   # ubij ga S HOSTA, iza Podmanovih leđa
podman ps                       # opet "Up X seconds" = vratio se
podman inspect --format '{{.RestartCount}}' restart-demo # → 1
```

**Poanta / zamka:**
- `podman kill` = ti namjerno gasiš → conmon zna da je namjerno → **bez restarta**.
- `kill -9 <host-pid>` = proces umre izvana, neočekivano → conmon to vidi → **restart se okine**.
- Isti signal (9), različit ishod — bitno je TKO ubija i je li namjerno.
- **`Exited (137)`** = `128 + 9`. U Linuxu exit kod ubijenog procesa = `128 + broj_signala`. Signal 9 = SIGKILL.
- `--restart=always` djeluje na: (1) stvarni pad procesa, (2) reboot (preko systemd-a, vidi #22).

---

## #3 — memorija + CPU limit

**Cilj:** pokreni s `--memory=256m --cpus=0.5`, provjeri da su limiti primijenjeni.

```bash
podman run -d --memory=256m --name limit-demo docker.io/library/httpd
podman inspect --format '{{.HostConfig.Memory}}' limit-demo   # → 268435456 (256 MB u bajtovima)
podman stats --no-stream limit-demo                            # MEM USAGE / LIMIT → .../256MB
```
- `--memory=256m` — gornja granica RAM-a (kernel OOM-ubije container ako prijeđe)
- `--cpus=0.5` — pola procesorske jezgre (vidi zamku dolje)
- `--no-stream` — jedan snimak pa izađi; bez njega se `stats` vrti uživo i zaključa terminal

**ZAMKA (rootless cgroups v2) — VAŽAN LO5 PRIMJER:**
```
Error: OCI runtime error: crun: controller `cpu` is not available ...
```
- Limiti se nameću preko **cgroups v2** (kernelov mehanizam).
- U **rootless** načinu kernel korisniku delegira samo neke controllere: `memory` JEST, `cpu` NIJE.
- Provjera: `cat /sys/fs/cgroup/cgroup.controllers` (sistem ima `cpu`) ali korisniku nije delegiran.
- **Popravak (treba root):** dodaj `Delegate=cpu cpuset io memory pids` u `user@.service` override + relogin. Bez root-a / sudo-a → CPU limit ne ide; koristi rootful Podman ili admina.
- **Za ispit:** važno je razumjeti mehanizam (cgroups), grešku i tri popravka — ne prisiliti tuđi VM.

---

## #4 — environment varijable (`--env` vs `--env-file`)

**Cilj:** jedan container s `--env`, drugi s `--env-file`, pokaži razliku s `exec ... env`.

```bash
# Način 1: ručno
podman run -d --env APP_COLOR=blue --name env1 docker.io/library/httpd

# Način 2: iz datoteke
printf 'APP_MODE=prod\nMAX_USERS=50\n' > vars.env
podman run -d --env-file vars.env --name env2 docker.io/library/httpd

# Dokaz da su unutra:
podman exec env1 env | grep APP    # → APP_COLOR=blue
podman exec env2 env | grep -E 'APP_MODE|MAX_USERS'
```
- `--env KEY=VALUE` — ubaci jednu varijablu ručno
- `printf 'APP_MODE=prod\nMAX_USERS=50\n' > vars.env` — napravi datoteku `vars.env`: `printf` ispiše tekst, `\n` = novi red (svaka varijabla u svoj red), `>` preusmjeri taj tekst u datoteku (prepiše je)
- `--env-file datoteka` — učitaj hrpu varijabli iz datoteke (urednije, lozinka se ne vidi u komandi)
- `exec <c> env` — pokreni `env` *unutar* containera; ispiše SVE varijable
- `| grep APP` — filtriraj samo retke s `APP` (`|` = pipe, proslijedi izlaz dalje)
- `grep -E 'APP_MODE|MAX_USERS'` — `-E` uključi prošireni regex; tu `|` znači **ILI**, pa hvata retke koji sadrže `APP_MODE` *ili* `MAX_USERS` odjednom (bez `-E` bi `|` bio običan znak, ne "ili")

**Poanta / zamke:**
- `env` ispisuje i hrpu "šuma": image dodaje svoje (`HTTPD_VERSION`...), sistem svoje (`PATH`, `HOME`), a VM dodaje **proxy** varijable (`http_proxy`...). Sve to je normalno, nije greška.
- Imena varijabli pisati s **podvlakom** velikim slovima (`APP_COLOR`). Crtica (`APP-COLOR`) prolazi ali nije konvencija.

---

## #5 — vlastiti bridge network + IP containera

**Cilj:** napravi user-defined bridge, spoji container, nađi mu IP.

```bash
podman network create webnet
podman run -d --network webnet --name net1 docker.io/library/httpd
podman network inspect webnet     # vidiš subnet, gateway, spojene containere
podman inspect --format '{{.NetworkSettings.Networks.webnet.IPAddress}}' net1   # → 10.89.0.2
```
- `network create webnet` — novi virtualni switch
- `--network webnet` — spoji container na NJU umjesto na default
- `network inspect` — subnet (`10.89.0.0/24`), gateway (`10.89.0.1`), `dns_enabled: true`

**Poanta:** bridge = privatni virtualni switch; svaki container dobije IP u istom subnetu. Na vlastitim mrežama Podman vrti **interni DNS** (`dns_enabled: true`) pa se containeri mogu zvati **po imenu**. Na default mreži tog imenovanja nema.

---

## #6 — pause / unpause

**Cilj:** zamrzni i odmrzni container, objasni u kakvom su stanju procesi.

```bash
podman run -d --name pause-demo docker.io/library/httpd
podman pause pause-demo
podman inspect --format '{{.State.Status}}' pause-demo     # → paused  (pouzdan dokaz!)
podman unpause pause-demo
podman inspect --format '{{.State.Status}}' pause-demo     # → running
```
- `pause` — zamrzni SVE procese unutra (kernelov **freezer** cgroup)
- `unpause` — odmrzni, nastavlja točno gdje je stao

**Poanta:** paused procesi su **zamrznuti u RAM-u** — drže memoriju i točno stanje, ali dobivaju **nula CPU**. Kao pauzirana sličica filma. (Različito od `stop`, koji procesu pošalje signal i on **izađe**.)

**ZAMKA (LO5):** `podman ps --filter name=pause-demo` zna vratiti **prazno** za paused container — izgleda kao da ne postoji. Goli `podman ps -a` ga uredno pokaže (`Paused`). **Pouka: kad `--filter` da prazno a očekuješ container, makni filter i provjeri golim `ps -a`, ili koristi `inspect --format '{{.State.Status}}'`. Sumnjaj i u alat kojim gledaš, ne samo u stvar.**

---

## #7 — rename

```bash
podman run -d --name old-name docker.io/library/httpd
podman rename old-name new-name
podman ps     # NAMES → new-name, ali CONTAINER ID je ISTI
```
**Poanta:** mijenja se samo naljepnica (ime). Container ID ostaje isti = to je isti container.

---

## #8 — `podman logs` (`--tail`, `--since`, `-f`)

```bash
podman run -d --name log-demo docker.io/library/httpd
podman logs --tail 5 log-demo      # zadnjih 5 redaka
podman logs --since 1m log-demo    # samo retci iz zadnje minute
podman logs -f log-demo            # živo praćenje; izađi s Ctrl+C
```
- `--tail N` — samo zadnjih N redaka
- `--since 1m` — samo noviji od zadanog vremena (može i timestamp)
- `-f` (follow) — drži otvoreno, ispisuje nove retke uživo. **Ctrl+C** prekida praćenje (NE gasi container).

**Napomena:** `AH00558 ... ServerName` je samo Apacheovo gunđanje da nema postavljeno ime servera — bezopasno.

---

## #9 — izvuci jedno polje (Go-template)

```bash
podman inspect inspect-demo | head -40                       # prvo zaviri u strukturu (JSON)
podman inspect --format '{{.Id}}' inspect-demo
podman inspect --format '{{.State.Status}}' inspect-demo
podman inspect --format '{{.State.Pid}}' inspect-demo
podman inspect --format '{{.Config.Image}}' inspect-demo
```
- `| head -40` — proslijedi ispis u `head` (`|` = pipe), koji pokaže samo prvih 40 redaka (da te ne zatrpa)
- `inspect` bez formata = cijeli JSON; `--format '{{...}}'` = samo jedna vrijednost
- `{{ }}` = Go-template; `.` = cijeli objekt; `.State.Status` = spuštaš se kroz razine točkom (kao u JSON-u)

**Poanta:** koristiš kad u skripti trebaš točno jedan podatak (npr. IP da ga proslijediš dalje).

---

## #10 — `podman cp` (host ↔ container)

```bash
# Host → container:
echo 'pozdrav s hosta' > fromhost.txt
podman cp fromhost.txt cp-demo:/tmp/fromhost.txt
podman exec cp-demo cat /tmp/fromhost.txt

# Container → host:
podman exec cp-demo sh -c 'echo "pozdrav iz containera" > /tmp/fromcontainer.txt'
podman cp cp-demo:/tmp/fromcontainer.txt ./fromcontainer.txt
cat fromcontainer.txt
```
- `echo 'tekst' > datoteka` — `echo` ispiše tekst, `>` ga preusmjeri u datoteku (napravi/prepiše je)
- `podman cp SRC DEST` — kopiraj datoteku
- Strana s prefiksom `container:` (npr. `cp-demo:`) određuje **smjer**. Bez prefiksa = host.
- `sh -c '...'` — izvrši navedeni string kao shell-naredbu **unutar** containera (treba kad u `exec` liniji koristiš `>`, `|` i slično)

---

## #11 — exec shell + instalacija + (ne)preživljavanje ⭐

**Cilj:** uđi u container, instaliraj paket, objasni preživljava li to ponovno stvaranje.

```bash
podman exec -it exec-demo bash      # interaktivni shell UNUTAR containera
  # (sad si unutra; prompt root@...#)
  apt-get update
  apt-get install -y curl
  which curl                        # → /usr/bin/curl
  exit                              # nazad na host

# DOKAZ da promjena NE preživi ponovno stvaranje:
podman rm -f exec-demo
podman run -d --name exec-demo docker.io/library/httpd
podman exec exec-demo sh -c 'which curl || echo "curl NIJE instaliran"'   # → curl NIJE instaliran
```
- `-it` = `-i` (interaktivno) + `-t` (terminal); `bash` = pokreni shell unutra
- `apt-get update` — osvježi popis paketa; `apt-get install -y curl` — instaliraj (`-y` = automatski potvrdi); `which curl` — pokaži putanju do programa (dokaz da je instaliran)
- `podman rm -f` — `-f` (force) obriši container i ako se vrti
- `||` = "ako prethodno ne uspije, izvrši ovo"

**POANTA (srž containera):** promjene u pokrenutom containeru žive u **zapisivom sloju** i preživljavaju `stop`/`start` (isti container). Ali kad container **obrišeš i napraviš novi iz imagea**, sve je izgubljeno — image se nije promijenio. Za trajnost: `podman commit` (#16) ili Dockerfile. *(Šum `debconf: unable to initialize frontend: Dialog` je samo jer unutra nema interaktivnog terminala za apt — bezopasno.)*

---

## #12 — `podman top` i `podman stats`

```bash
podman top top-demo                    # PROCESI unutra (kao ps): USER, PID, COMMAND
podman stats --no-stream top-demo      # RESURSI: CPU%, MEM, NET IO
```
**Poanta:** `top` = **što** se vrti (procesi), `stats` = **koliko** troši (resursi). Detalj: glavni proces drži `root` (PID 1), a radne (worker) procese httpd spušta na `www-data` — sigurnosna dobra praksa.

---

## #13 — `--rm` (auto-uklanjanje)

```bash
podman run --rm --name rm-demo docker.io/library/httpd httpd -v   # ispiše verziju, izađe, sam se obriše
podman ps -a --filter name=rm-demo                                # prazno = očišćen
```
- `--rm` — obriši container automatski čim izađe
- `httpd -v` — kratka komanda (ispiši verziju) umjesto servera, da container završi sam
- `ps -a` — `-a` (all) prikaže i zaustavljene containere, ne samo pokrenute; `--filter name=` traži po imenu (vidi #15)

**Poanta:** odlično za **kratkotrajne/jednokratne** containere (test, build-korak) — ne gomilaju se mrtvi. **Što gubiš:** nakon izlaska nema logova, `inspect`, ni `diff` — sve nestane. Ne koristi kad trebaš naknadno pregledati što se dogodilo.

---

## #14 — bind-mount read-only (`:ro`)

```bash
mkdir -p ~/devops/romount && echo 'sadrzaj s hosta' > ~/devops/romount/host.txt
podman run -d --name ro-demo -v ~/devops/romount:/data:ro docker.io/library/httpd

podman exec ro-demo cat /data/host.txt                       # čitanje radi → sadrzaj s hosta
podman exec ro-demo sh -c 'echo test > /data/proba.txt'      # → Read-only file system (DOKAZ)
```
- `mkdir -p mapa` — napravi mapu; `-p` napravi i roditelje ako fale i ne buni se ako već postoji
- `&&` — "ako je prethodno uspjelo, nastavi"; `echo ... > host.txt` upiše tekst u datoteku (`>` = preusmjeri u datoteku, prepiše je)
- `-v IZVOR:ODREDIŠTE:ro` — poveži host-mapu u container, **read-only**
- greška `Read-only file system` je **očekivana i poželjna** = dokaz da `:ro` radi

**Poanta:** container vidi host-mapu ali ne smije pisati. Korisno kad treba samo *čitati* konfiguraciju/podatke.

---

## #15 — labele + `--filter`

```bash
podman run -d --name label1 --label app=web --label tier=frontend docker.io/library/httpd
podman run -d --name label2 --label app=db  --label tier=backend  docker.io/library/httpd

podman ps --filter label=tier=frontend     # samo label1
podman ps --filter label=app=db            # samo label2
```
- `--label kljuc=vrijednost` — proizvoljna oznaka (metapodatak); može ih više
- `--filter label=...` — prikaži samo containere s tom labelom

**Poanta:** labele organiziraju i filtriraju containere (po aplikaciji, okolini, timu) — neprocjenjivo kad ih imaš desetke.

---

## #16 — `podman commit` (container → novi image) ⭐

```bash
podman exec commit-src sh -c 'echo "ja sam u imageu" > /opt/marker.txt'
podman commit commit-src my-httpd:v1
podman images --filter reference=my-httpd          # vidiš localhost/my-httpd  v1

podman run -d --name commit-test my-httpd:v1
podman exec commit-test cat /opt/marker.txt        # → ja sam u imageu
```
- `commit <container> <ime:tag>` — zapeci trenutno stanje containera u novi image

**POANTA (poveži s #11):** u #11 promjene nestanu kad obrišeš container. `commit` ih **trajno spremi u novi image** — pa svaki novi container iz njega već ima tu promjenu. (U praksi se radi Dockerfileom; commit pokazuje mehanizam.)

---

## #17 — `podman diff` (A/C/D markeri)

```bash
podman diff commit-src
# A /opt/marker.txt      ← Added (dodano)
# C /opt                 ← Changed (izmijenjeno)
# A /usr/local/apache2/logs/httpd.pid
```
- `diff` — što se promijenilo u filesystemu containera u odnosu na image
- **A** = Added, **C** = Changed, **D** = Deleted

**Poanta:** `diff` pokaže točno koje tragove je container ostavio iznad imagea (= sadržaj zapisivog sloja). Korisno prije `commit`-a (da znaš što pečeš) i za troubleshooting.

---

## #18 — `--network none` (bez mreže)

```bash
podman run -d --name none-demo --network none docker.io/library/httpd
podman exec none-demo sh -c 'ip addr 2>/dev/null || ls /sys/class/net'   # → samo lo (nema eth0)
podman exec none-demo sh -c 'getent hosts deb.debian.org || echo "NEMA MREZE - ocekivano"'
```
- `--network none` — nikakva mreža, samo loopback `lo`
- `2>/dev/null` — sakrij greške (kanal `2` = greške → "crna rupa"); `||` pokrene zamjenu ako prvo padne; `ip addr`/`getent` su Linux mrežni alati
- vidiš samo `lo`, vanjsko razrješavanje imena padne → dokaz izolacije

**Poanta / use case:** potpuna mrežna izolacija. Korisno za obradu **nepouzdanih podataka** (parsiranje sumnjive datoteke bez rizika da "nazove kući"), čisto računanje, ili sigurnosno testiranje malwarea.

---

## #19 — port samo na jednom IP-u (`127.0.0.1:...`)

```bash
podman run -d -p 127.0.0.1:8082:80 --name ip-demo docker.io/library/httpd
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8082            # → 200 (radi lokalno)
hostname -I                                                                # nađi vanjski IP, npr. 10.10.68.126
curl -s -o /dev/null -w '%{http_code}\n' --noproxy '*' --max-time 3 http://10.10.68.126:8082   # → 000/refused
```
- `-p 127.0.0.1:8082:80` — objavi port **samo** na localhostu (usporedi `-p 8081:80` = na svim IP-ovima)
- `hostname -I` — ispiše IP adrese stroja (uzmeš onu koja nije 127.x)
- `curl` flagovi: `-s` tiho, `-o /dev/null` baci sadržaj, `-w '%{http_code}'` ispiši samo status
- `--max-time 3` — odustani nakon 3 s; `--noproxy '*'` — zaobiđi proxy (vidi zamku)

**ZAMKA (LO5):** bez `--noproxy`, `curl` na vanjski IP ide **kroz proxy** (jer `no_proxy` pokriva samo localhost) i proxy vrati `503` umjesto direktnog "refused". Definitivan dokaz binding-a je svejedno `podman port` / `inspect` (vidi #20). Tipfeler `127.0.0.0.1` (jedan `0` viška) → "cannot parse as IP".

**Poanta:** vezanjem na `127.0.0.1` servis je vidljiv **samo lokalno** — za stvari koje smiju pričati samo s procesima na istom stroju (npr. baza iza aplikacije).

---

## #20 — `podman port`

```bash
podman port ip-demo                                          # → 80/tcp -> 127.0.0.1:8082
podman inspect --format '{{.NetworkSettings.Ports}}' ip-demo # → map[80/tcp:[{127.0.0.1 8082}]]
```
**Poanta:** `podman port` = brzi pregled "koji port vodi kamo"; `inspect` to potvrđuje iz punog zapisa. Dva pogleda na isto.

---

## #21 — ne-root korisnik (`--user`)

```bash
podman run -d --user 1000 --name user-demo docker.io/library/httpd   # httpd ispadne (Exited 1) — OČEKIVANO
podman run --rm --user 1000 docker.io/library/httpd id               # → uid=1000(student) gid=0(root)
podman run --rm --user 1000 docker.io/library/httpd whoami           # → student
```
- `--user 1000` — pokreni glavni proces kao UID 1000 (umjesto root/UID 0)
- httpd kao ne-root zna **ispasti** (treba prava na neke datoteke) — to je dio pouke
- `id` / `whoami` dokazuju da nisi root

**Poanta:** `--user` spušta container s root-a na običnog korisnika. **Princip najmanje privilegije** — ako netko provali u container, ima manje ovlasti. (Zato httpd worker procese spušta na `www-data`, #12.)

---

## #22 — systemd unit / Quadlet (start na boot)

**Cilj:** generiraj systemd unit (ili Quadlet `.container`) i objasni kako pokreće container na boot.

```bash
# Stari način (DEPRECATED u novim verzijama):
podman generate systemd --name boot-demo     # ispiše cijeli [Unit]/[Service]/[Install] blok

# Novi način (Quadlet):
mkdir -p ~/.config/containers/systemd
cat > ~/.config/containers/systemd/boot-demo.container << 'EOF'
[Container]
Image=docker.io/library/httpd
PublishPort=8083:80

[Install]
WantedBy=default.target
EOF
```
- `generate systemd` — ispiši systemd unit za container (u novim verzijama zastario → koristi Quadlet)
- `cat > datoteka << 'EOF' ... EOF` — upiši sve retke između `EOF` oznaka u datoteku (zgodno za višeredni sadržaj); `mkdir -p` napravi mapu ako fali (vidi #14)
- Quadlet `.container` datoteka: `[Container] Image=...` što pokrenuti, `PublishPort=` kao `-p`
- `[Install] WantedBy=default.target` — "pokreni kad sistem dođe u normalno stanje" = **na boot**

**Poanta (poveži s #2):** systemd/Quadlet pretvara container u **uslugu kojom upravlja sistem** — pokreće ga automatski **na boot** i restarta ako padne. `--restart=always` (#2) djeluje na pad procesa; systemd pokriva pokretanje **nakon reboota**. To je "pravi" način da container trajno radi na serveru.

---

## #23 — `podman events` (životni ciklus)

**Cilj:** prati događaje u jednom terminalu dok u drugom pokrećeš/gasiš container.

```bash
# Terminal 1:
podman events        # stoji i ispisuje uživo; izađi s Ctrl+C

# Terminal 2:
podman run -d --name events-demo docker.io/library/httpd
podman stop events-demo
podman start events-demo
podman rm -f events-demo
```
Uočeni slijed u Terminalu 1:
```
container create → image pull → container init → container start
→ container died → container cleanup        (stop)
→ container init → container start           (start)
→ container died → container remove          (rm)
```

**Poanta:** `events` pokaže **životni ciklus** u stvarnom vremenu. Koristi za **debugiranje** ("zašto mi se container ugasio?") i nadzor. Ako nemaš drugi terminal: `podman events --since 1m --stream=false` pokaže nedavne događaje *nakon* akcije.

---

## 📌 ZBIRKA ZAMKI (LO5 troubleshooting — zlato za ispit)

| # | Zamka | Rješenje / objašnjenje |
|---|-------|------------------------|
| 2 | `podman kill` ne okine restart | Namjeran stop. Za restart ubij host-PID s `kill -9` ili reboot. |
| 2 | `Exited (137)` | `128 + 9` = ubijen SIGKILL-om. `143` = `128+15` = SIGTERM. |
| 3 | `controller cpu is not available` | Rootless cgroups v2: `cpu` nije delegiran. Treba root (`Delegate=cpu`) ili rootful. `memory` radi. |
| 4 | `env` pun "šuma" | Image + sistem + **proxy** varijable su normalne. Filtriraj s `grep`. |
| 6 | `ps --filter` skriva paused container | Makni filter (`ps -a`) ili koristi `inspect --format '{{.State.Status}}'`. |
| 11 | promjena nestala nakon `rm` | Živjela u zapisivom sloju. Za trajnost: `commit` ili Dockerfile. |
| 19 | vanjski curl daje `503` umjesto refused | Ide kroz proxy. Dodaj `--noproxy '*'`. Binding dokaži s `podman port`. |
| 21 | httpd ispadne uz `--user 1000` | Očekivano (treba root prava). UID dokaži s `id`/`whoami`. |
| 22 | `generate systemd` DEPRECATED | Koristi Quadlet `.container` datoteku. |

## 📌 BRZE REFERENCE

**Exit kodovi:** `0` čist izlaz · `137` SIGKILL (128+9) · `143` SIGTERM (128+15)
**diff markeri:** `A` Added · `C` Changed · `D` Deleted
**restart politike:** `no` · `on-failure` · `always`
**`-p` smjer:** `HOST:CONTAINER` (desno = fiksni port appa unutra)
**`cp` smjer:** strana s `container:` prefiksom = unutar containera
**čišćenje:** `podman rm -f <ime>` (i ako se vrti) · `podman rmi <image>` · `podman network rm <mreza>`

---

## Ključne rečenice za pamćenje
1. **Image je zamrznut i dijeljen; container je image + tvoj privatni zapisivi sloj koji se briše zajedno s njim — osim ako ga `commit`-om ne pretvoriš u novi image.**
2. **Podman nema centralni daemon; svaki container motri `conmon`. Zato je ručni `stop`/`kill` namjeran (bez restarta), a vanjska/neočekivana smrt okida restart.**
3. **Kad alat (`ps --filter`) da neočekivano prazno — sumnjaj u alat, ne samo u stvar. Provjeri golim `ps -a` ili `inspect`.**
