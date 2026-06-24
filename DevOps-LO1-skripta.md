# DevOps — LO1: Uporaba kontejnera i kontejnerskih usluga (Podman)

> Bilješke za ispit (na ispitu dozvoljene). Svaki zadatak ima: cilj → naredbe (razložene) → poanta → zamke.
> Slika koja se svuda koristi: `docker.io/library/httpd` (Apache). Radni direktorij je `~/devops`.
> Okruženje: rootless Podman, minikube, kubectl, bez `sudo`.

## Rječnik kratica

- **Podman** — Pod Manager; alat za pokretanje kontejnera bez pozadinske usluge (daemonless).
- **conmon** — container monitor; mali nadzorni proces koji prati svaki kontejner umjesto centralne pozadinske usluge.
- **OCI** — Open Container Initiative; standard za format slika i izvršno okruženje (runtime) kontejnera.
- **cgroups** — control groups; kernelov mehanizam za ograničavanje resursa (CPU, memorija).
- **OOM** — Out Of Memory; kernel ubije proces kad kontejner prijeđe granicu memorije.
- **UID / GID** — User ID / Group ID; brojčani identitet korisnika i grupe.
- **PID** — Process ID; brojčani identitet procesa.
- **DNS** — Domain Name System; razrješavanje imena u adrese.
- **IP** — Internet Protocol (adresa); mrežna adresa čvora.
- **RAM** — Random Access Memory; radna memorija.
- **CPU** — Central Processing Unit; procesor.
- **JSON** — JavaScript Object Notation; tekstualni format zapisa podataka koji ispisuje `inspect`.
- **YAML** — YAML Ain't Markup Language; format za manifeste (s razmacima, ne tabovima).
- **systemd** — sustav za upravljanje uslugama na Linuxu (pokretanje na boot).
- **Quadlet** — suvremeni način pokretanja kontejnera preko systemd-a (datoteka `.container`).
- **SIGKILL / SIGTERM** — signali za prekid procesa (9 / 15).
- **regex** — regular expression; obrazac za pretragu teksta.

---

## 0. Temelji (osnova koja drži sve ostalo)

### Slika vs kontejner vs zapisivi sloj — najvažnije

```
┌─ HOST (VM) ───────────────────────────────────┐
│  tu se tipka `podman ...`; tu žive host dat.   │
│  ┌─ KONTEJNER (kad se pokrene slika) ────────┐ │
│  │  ZAPISIVI SLOJ  → čita i PIŠE (prolazno)  │ │
│  │     promjene: curl, /opt/marker.txt       │ │
│  │  SLIKA (httpd)  → SAMO čita (zamrznuto)   │ │
│  └────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

- **Slika** = recept, zamrznut na disku, **samo za čitanje**, nikad se sama ne mijenja. (Analogija: klasa.)
- **Kontejner** = pokrenuta slika = slika + tanki **zapisivi sloj** navrh. (Analogija: objekt.)
- Sve što se mijenja unutra (instalira paket, napravi datoteka) ide **samo u zapisivi sloj**, NE u sliku.
- **`podman rm`** briše kontejner + zapisivi sloj → promjene nestanu. Slika ostaje.
- **`podman commit`** zapeče zapisivi sloj u **novu sliku** → promjene postanu trajne.
- **`podman cp`** prenosi datoteke preko granice host ↔ kontejner.

### Struktura naredbe
```
podman run [opcije bilo kojim redom] IMAGE [komanda]
```
Što je **iza slike** = naredba koja se vrti unutar kontejnera (zamjenjuje zadanu).

### Zastavice (crtice)
- jedna crtica = kratko ime: `-d`, `-p`, `-v`
- dvije crtice = dugo ime: `--name`, `--restart`, `--memory`
- mogu se spajati: `-it` = `-i -t`

### `-p HOST:CONTAINER` (mapiranje porta)
- **desna** strana = port na kojem aplikacija sluša **unutra** (httpd = 80, fiksno)
- **lijeva** strana = slobodan izbor na hostu
- **Zamka:** obrnut redoslijed. `-p 80:8081` ≠ `-p 8081:80`.

### Rootless Podman — nema centralnu pozadinsku uslugu
- Docker ima pozadinsku uslugu (daemon) koja stalno motri kontejnere; **Podman nema**.
- Svaki kontejner ima mali nadzorni proces **`conmon`** koji ga prati.
- Posljedica: `podman stop/kill` = **namjerna** naredba → restart politika se NE okida.

---

## Zadatak 1 — httpd detached, prilagođeni port/ime/hostname

### Koncept
Cilj: pokrenuti httpd u pozadini, mapirati port 80→8081, dodijeliti prilagođeni `--name` i `--hostname`, potvrditi s `inspect`.

### Naredba
```bash
podman run -d -p 8081:80 --name web1 --hostname web-host docker.io/library/httpd
```

### Razlaganje
- `-d` — detached (u pozadini), terminal ostaje slobodan
- `-p 8081:80` — host 8081 → kontejner 80
- `--name web1` — ime kontejnera (za lako dohvaćanje)
- `--hostname web-host` — hostname *unutar* kontejnera

```bash
podman inspect --format '{{.Name}} {{.Config.Hostname}}' web1
```
- `--format '{{...}}'` — ispisuje samo zadana polja iz punog zapisa (Go-predložak; detaljno razloženo u zad. 9)

### Zašto
Detached + objavljen port = poslužitelj radi u pozadini i dostupan je na hostu. `--name` služi za dohvaćanje, `--hostname` je identitet unutar kontejnera.

---

## Zadatak 2 — `--restart=always` + dokaz restarta

### Koncept
Cilj: pokrenuti s restart politikom, "ubiti" glavni proces, dokazati da se Podman vrati.

### Naredba
```bash
podman run -d --restart=always --name restart-demo docker.io/library/httpd
```
- `--restart=always` — politika: uvijek vrati kontejner kad stane
  - opcije: `no` (zadano), `on-failure` (samo na grešku), `always` (uvijek)

### Razlaganje — kako ispravno dokazati
```bash
# OVO NE okida restart (namjeran stop):
podman kill restart-demo        # kontejner ostane Exited (137)

# OVO okida restart (vanjska, neočekivana smrt):
podman start restart-demo
podman inspect --format '{{.State.Pid}}' restart-demo   # nađi host-PID, npr. 14665
kill -9 14665                   # ubij ga S HOSTA, iza Podmanovih leđa
podman ps                       # opet "Up X seconds" = vratio se
podman inspect --format '{{.RestartCount}}' restart-demo # → 1
```

### Zašto / Gotče
- `podman kill` = namjerno gašenje → conmon zna da je namjerno → **bez restarta**.
- `kill -9 <host-pid>` = proces umre izvana, neočekivano → conmon to vidi → **restart se okine**.
- Isti signal (9), različit ishod — bitno je TKO ubija i je li namjerno.
- **`Exited (137)`** = `128 + 9`. U Linuxu exit kod ubijenog procesa = `128 + broj_signala`. Signal 9 = SIGKILL.
- `--restart=always` djeluje na: (1) stvarni pad procesa, (2) reboot (preko systemd-a, vidi zad. 22).

---

## Zadatak 3 — memorija + CPU limit

### Koncept
Cilj: pokrenuti s `--memory=256m --cpus=0.5`, provjeriti da su limiti primijenjeni.

### Naredba
```bash
podman run -d --memory=256m --name limit-demo docker.io/library/httpd
podman inspect --format '{{.HostConfig.Memory}}' limit-demo   # → 268435456 (256 MB u bajtovima)
podman stats --no-stream limit-demo                            # MEM USAGE / LIMIT → .../256MB
```

### Razlaganje
- `--memory=256m` — gornja granica RAM-a (kernel OOM-ubije kontejner ako prijeđe)
- `--cpus=0.5` — pola procesorske jezgre (vidi gotču dolje)
- `--no-stream` — jedan snimak pa izlaz; bez njega se `stats` vrti uživo i zaključa terminal

### Gotča (rootless cgroups v2) — VAŽAN LO5 PRIMJER
```
Error: OCI runtime error: crun: controller `cpu` is not available ...
```
- Limiti se nameću preko **cgroups v2** (kernelov mehanizam).
- U **rootless** načinu kernel korisniku delegira samo neke controllere: `memory` JEST, `cpu` NIJE.
- Provjera: `cat /sys/fs/cgroup/cgroup.controllers` (sustav ima `cpu`) ali korisniku nije delegiran.
- **Popravak (treba root):** dodati `Delegate=cpu cpuset io memory pids` u `user@.service` override + relogin. Bez root-a / sudo-a → CPU limit ne radi; koristi se rootful Podman ili admin.
- **Za ispit:** važno je razumjeti mehanizam (cgroups), grešku i tri popravka — ne prisiljavati tuđi VM.

---

## Zadatak 4 — varijable okruženja (`--env` vs `--env-file`)

### Koncept
Cilj: jedan kontejner s `--env`, drugi s `--env-file`, prikazati razliku s `exec ... env`.

### Naredba
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

### Razlaganje
- `--env KEY=VALUE` — ubacuje jednu varijablu ručno
- `printf 'APP_MODE=prod\nMAX_USERS=50\n' > vars.env` — radi datoteku `vars.env`: `printf` ispiše tekst, `\n` = novi red (svaka varijabla u svoj red), `>` preusmjeri taj tekst u datoteku (prepiše je)
- `--env-file datoteka` — učitava hrpu varijabli iz datoteke (urednije, lozinka se ne vidi u naredbi)
- `exec <c> env` — pokreće `env` *unutar* kontejnera; ispisuje SVE varijable
- `| grep APP` — filtrira samo retke s `APP` (`|` = cijev, prosljeđuje izlaz dalje)
- `grep -E 'APP_MODE|MAX_USERS'` — `-E` uključuje prošireni regex; tu `|` znači **ILI**, pa hvata retke koji sadrže `APP_MODE` *ili* `MAX_USERS` odjednom (bez `-E` bi `|` bio običan znak, ne "ili")

### Zašto / Gotče
- `env` ispisuje i hrpu "šuma": slika dodaje svoje (`HTTPD_VERSION`...), sustav svoje (`PATH`, `HOME`), a VM dodaje **proxy** varijable (`http_proxy`...). Sve to je normalno, nije greška.
- Imena varijabli pisati s **podvlakom** velikim slovima (`APP_COLOR`). Crtica (`APP-COLOR`) prolazi ali nije konvencija.

---

## Zadatak 5 — vlastiti mrežni most (bridge) + IP kontejnera

### Koncept
Cilj: napraviti korisnički most (user-defined bridge), spojiti kontejner, naći mu IP.

### Naredba
```bash
podman network create webnet
podman run -d --network webnet --name net1 docker.io/library/httpd
podman network inspect webnet     # subnet, gateway, spojeni kontejneri
podman inspect --format '{{.NetworkSettings.Networks.webnet.IPAddress}}' net1   # → 10.89.0.2
```

### Razlaganje
- `network create webnet` — novi virtualni preklopnik (switch)
- `--network webnet` — spaja kontejner na NJU umjesto na zadanu
- `network inspect` — subnet (`10.89.0.0/24`), gateway (`10.89.0.1`), `dns_enabled: true`

### Zašto
Most = privatni virtualni preklopnik; svaki kontejner dobije IP u istom subnetu. Na vlastitim mrežama Podman vrti **interni DNS** (`dns_enabled: true`) pa se kontejneri mogu zvati **po imenu**. Na zadanoj mreži tog imenovanja nema.

---

## Zadatak 6 — pause / unpause

### Koncept
Cilj: zamrznuti i odmrznuti kontejner, objasniti u kakvom su stanju procesi.

### Naredba
```bash
podman run -d --name pause-demo docker.io/library/httpd
podman pause pause-demo
podman inspect --format '{{.State.Status}}' pause-demo     # → paused  (pouzdan dokaz!)
podman unpause pause-demo
podman inspect --format '{{.State.Status}}' pause-demo     # → running
```

### Razlaganje
- `pause` — zamrzava SVE procese unutra (kernelov **freezer** cgroup)
- `unpause` — odmrzava, nastavlja točno gdje je stao

### Zašto
Pauzirani procesi su **zamrznuti u RAM-u** — drže memoriju i točno stanje, ali dobivaju **nula CPU**. Kao pauzirana sličica filma. (Različito od `stop`, koji procesu pošalje signal i on **izađe**.)

### Gotča (LO5)
`podman ps --filter name=pause-demo` zna vratiti **prazno** za pauzirani kontejner — izgleda kao da ne postoji. Goli `podman ps -a` ga uredno pokaže (`Paused`). Pouka: kad `--filter` da prazno a očekuje se kontejner, makne se filter i provjerava se golim `ps -a`, ili se koristi `inspect --format '{{.State.Status}}'`. Valja sumnjati i u alat kojim se gleda, ne samo u stvar.

---

## Zadatak 7 — rename

### Naredba
```bash
podman run -d --name old-name docker.io/library/httpd
podman rename old-name new-name
podman ps     # NAMES → new-name, ali CONTAINER ID je ISTI
```

### Zašto
Mijenja se samo naljepnica (ime). Container ID ostaje isti = to je isti kontejner.

---

## Zadatak 8 — `podman logs` (`--tail`, `--since`, `-f`)

### Naredba
```bash
podman run -d --name log-demo docker.io/library/httpd
podman logs --tail 5 log-demo      # zadnjih 5 redaka
podman logs --since 1m log-demo    # samo retci iz zadnje minute
podman logs -f log-demo            # živo praćenje; izlaz s Ctrl+C
```

### Razlaganje
- `--tail N` — samo zadnjih N redaka
- `--since 1m` — samo noviji od zadanog vremena (može i timestamp)
- `-f` (follow) — drži otvoreno, ispisuje nove retke uživo. **Ctrl+C** prekida praćenje (NE gasi kontejner).

### Gotča
`AH00558 ... ServerName` je samo Apacheovo gunđanje da nema postavljeno ime poslužitelja — bezopasno.

---

## Zadatak 9 — izvuci jedno polje (Go-predložak)

### Naredba
```bash
podman inspect inspect-demo | head -40                       # prvo zaviri u strukturu (JSON)
podman inspect --format '{{.Id}}' inspect-demo
podman inspect --format '{{.State.Status}}' inspect-demo
podman inspect --format '{{.State.Pid}}' inspect-demo
podman inspect --format '{{.Config.Image}}' inspect-demo
```

### Razlaganje
- `| head -40` — prosljeđuje ispis u `head` (`|` = cijev), koji pokaže samo prvih 40 redaka (da ne zatrpa)
- `inspect` bez formata = cijeli JSON; `--format '{{...}}'` = samo jedna vrijednost
- `{{ }}` = Go-predložak; `.` = cijeli objekt; `.State.Status` = spuštanje kroz razine točkom (kao u JSON-u)

### Zašto
Koristi se kad u skripti treba točno jedan podatak (npr. IP da se proslijedi dalje).

---

## Zadatak 10 — `podman cp` (host ↔ kontejner)

### Naredba
```bash
# Host → kontejner:
echo 'pozdrav s hosta' > fromhost.txt
podman cp fromhost.txt cp-demo:/tmp/fromhost.txt
podman exec cp-demo cat /tmp/fromhost.txt

# Kontejner → host:
podman exec cp-demo sh -c 'echo "pozdrav iz containera" > /tmp/fromcontainer.txt'
podman cp cp-demo:/tmp/fromcontainer.txt ./fromcontainer.txt
cat fromcontainer.txt
```

### Razlaganje
- `echo 'tekst' > datoteka` — `echo` ispiše tekst, `>` ga preusmjeri u datoteku (napravi/prepiše je)
- `podman cp SRC DEST` — kopira datoteku
- Strana s prefiksom `container:` (npr. `cp-demo:`) određuje **smjer**. Bez prefiksa = host.
- `sh -c '...'` — izvršava navedeni string kao shell-naredbu **unutar** kontejnera (treba kad se u `exec` liniji koristi `>`, `|` i slično)

---

## Zadatak 11 — exec shell + instalacija + (ne)preživljavanje ⭐

### Koncept
Cilj: ući u kontejner, instalirati paket, objasniti preživljava li to ponovno stvaranje.

### Naredba
```bash
podman exec -it exec-demo bash      # interaktivni shell UNUTAR containera
  # (sad unutra; prompt root@...#)
  apt-get update
  apt-get install -y curl
  which curl                        # → /usr/bin/curl
  exit                              # nazad na host

# DOKAZ da promjena NE preživi ponovno stvaranje:
podman rm -f exec-demo
podman run -d --name exec-demo docker.io/library/httpd
podman exec exec-demo sh -c 'which curl || echo "curl NIJE instaliran"'   # → curl NIJE instaliran
```

### Razlaganje
- `-it` = `-i` (interaktivno) + `-t` (terminal); `bash` = pokreće shell unutra
- `apt-get update` — osvježuje popis paketa; `apt-get install -y curl` — instalira (`-y` = automatski potvrdi); `which curl` — pokazuje putanju do programa (dokaz da je instaliran)
- `podman rm -f` — `-f` (force) briše kontejner i ako se vrti
- `||` = "ako prethodno ne uspije, izvrši ovo"

### Zašto (srž kontejnera)
Promjene u pokrenutom kontejneru žive u **zapisivom sloju** i preživljavaju `stop`/`start` (isti kontejner). Ali kad se kontejner **obriše i napravi novi iz slike**, sve je izgubljeno — slika se nije promijenila. Za trajnost: `podman commit` (zad. 16) ili Containerfile. *(Šum `debconf: unable to initialize frontend: Dialog` je samo jer unutra nema interaktivnog terminala za apt — bezopasno.)*

---

## Zadatak 12 — `podman top` i `podman stats`

### Naredba
```bash
podman top top-demo                    # PROCESI unutra (kao ps): USER, PID, COMMAND
podman stats --no-stream top-demo      # RESURSI: CPU%, MEM, NET IO
```

### Zašto
`top` = **što** se vrti (procesi), `stats` = **koliko** troši (resursi). Detalj: glavni proces drži `root` (PID 1), a radne (worker) procese httpd spušta na `www-data` — sigurnosna dobra praksa.

---

## Zadatak 13 — `--rm` (auto-uklanjanje)

### Naredba
```bash
podman run --rm --name rm-demo docker.io/library/httpd httpd -v   # ispiše verziju, izađe, sam se obriše
podman ps -a --filter name=rm-demo                                # prazno = očišćen
```

### Razlaganje
- `--rm` — briše kontejner automatski čim izađe
- `httpd -v` — kratka naredba (ispiši verziju) umjesto poslužitelja, da kontejner završi sam
- `ps -a` — `-a` (all) prikazuje i zaustavljene kontejnere, ne samo pokrenute; `--filter name=` traži po imenu (vidi zad. 15)

### Zašto / Gotče
Odlično za **kratkotrajne/jednokratne** kontejnere (test, build-korak) — ne gomilaju se mrtvi. **Što se gubi:** nakon izlaska nema logova, `inspect`, ni `diff` — sve nestane. Ne koristi se kad treba naknadno pregledati što se dogodilo.

---

## Zadatak 14 — bind-mount samo za čitanje (`:ro`)

### Naredba
```bash
mkdir -p ~/devops/romount && echo 'sadrzaj s hosta' > ~/devops/romount/host.txt
podman run -d --name ro-demo -v ~/devops/romount:/data:ro docker.io/library/httpd

podman exec ro-demo cat /data/host.txt                       # čitanje radi → sadrzaj s hosta
podman exec ro-demo sh -c 'echo test > /data/proba.txt'      # → Read-only file system (DOKAZ)
```

### Razlaganje
- `mkdir -p mapa` — radi mapu; `-p` napravi i roditelje ako fale i ne buni se ako već postoji
- `&&` — "ako je prethodno uspjelo, nastavi"; `echo ... > host.txt` upisuje tekst u datoteku (`>` = preusmjeri u datoteku, prepiše je)
- `-v IZVOR:ODREDIŠTE:ro` — povezuje host-mapu u kontejner, **samo za čitanje**
- greška `Read-only file system` je **očekivana i poželjna** = dokaz da `:ro` radi

### Zašto
Kontejner vidi host-mapu ali ne smije pisati. Korisno kad treba samo *čitati* konfiguraciju/podatke.

---

## Zadatak 15 — labele + `--filter`

### Naredba
```bash
podman run -d --name label1 --label app=web --label tier=frontend docker.io/library/httpd
podman run -d --name label2 --label app=db  --label tier=backend  docker.io/library/httpd

podman ps --filter label=tier=frontend     # samo label1
podman ps --filter label=app=db            # samo label2
```

### Razlaganje
- `--label kljuc=vrijednost` — proizvoljna oznaka (metapodatak); može ih više
- `--filter label=...` — prikazuje samo kontejnere s tom labelom

### Zašto
Labele organiziraju i filtriraju kontejnere (po aplikaciji, okolini, timu) — neprocjenjivo kad ih ima desetke.

---

## Zadatak 16 — `podman commit` (kontejner → nova slika) ⭐

### Naredba
```bash
podman exec commit-src sh -c 'echo "ja sam u imageu" > /opt/marker.txt'
podman commit commit-src my-httpd:v1
podman images --filter reference=my-httpd          # localhost/my-httpd  v1

podman run -d --name commit-test my-httpd:v1
podman exec commit-test cat /opt/marker.txt        # → ja sam u imageu
```

### Razlaganje
- `commit <kontejner> <ime:tag>` — pečenje trenutnog stanja kontejnera u novu sliku

### Zašto (poveži sa zad. 11)
U zad. 11 promjene nestanu kad se kontejner obriše. `commit` ih **trajno spremi u novu sliku** — pa svaki novi kontejner iz nje već ima tu promjenu. (U praksi se radi Containerfileom; commit pokazuje mehanizam.)

---

## Zadatak 17 — `podman diff` (A/C/D markeri)

### Naredba
```bash
podman diff commit-src
# A /opt/marker.txt      ← Added (dodano)
# C /opt                 ← Changed (izmijenjeno)
# A /usr/local/apache2/logs/httpd.pid
```

### Razlaganje
- `diff` — što se promijenilo u datotečnom sustavu kontejnera u odnosu na sliku
- **A** = Added, **C** = Changed, **D** = Deleted

### Zašto
`diff` pokazuje točno koje tragove je kontejner ostavio iznad slike (= sadržaj zapisivog sloja). Korisno prije `commit`-a (da se zna što se peče) i za troubleshooting.

---

## Zadatak 18 — `--network none` (bez mreže)

### Naredba
```bash
podman run -d --name none-demo --network none docker.io/library/httpd
podman exec none-demo sh -c 'ip addr 2>/dev/null || ls /sys/class/net'   # → samo lo (nema eth0)
podman exec none-demo sh -c 'getent hosts deb.debian.org || echo "NEMA MREZE - ocekivano"'
```

### Razlaganje
- `--network none` — nikakva mreža, samo loopback `lo`
- `2>/dev/null` — sakriva greške (kanal `2` = greške → "crna rupa"); `||` pokreće zamjenu ako prvo padne; `ip addr`/`getent` su Linux mrežni alati
- vidljiv je samo `lo`, vanjsko razrješavanje imena padne → dokaz izolacije

### Zašto (slučaj uporabe)
Potpuna mrežna izolacija. Korisno za obradu **nepouzdanih podataka** (parsiranje sumnjive datoteke bez rizika da "nazove kući"), čisto računanje, ili sigurnosno testiranje malwarea.

---

## Zadatak 19 — port samo na jednom IP-u (`127.0.0.1:...`)

### Naredba
```bash
podman run -d -p 127.0.0.1:8082:80 --name ip-demo docker.io/library/httpd
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8082            # → 200 (radi lokalno)
hostname -I                                                                # nađi vanjski IP, npr. 10.10.68.126
curl -s -o /dev/null -w '%{http_code}\n' --noproxy '*' --max-time 3 http://10.10.68.126:8082   # → 000/refused
```

### Razlaganje
- `-p 127.0.0.1:8082:80` — objavljuje port **samo** na localhostu (usporedi `-p 8081:80` = na svim IP-ovima)
- `hostname -I` — ispisuje IP adrese stroja (uzima se ona koja nije 127.x)
- `curl` zastavice: `-s` tiho, `-o /dev/null` baci sadržaj, `-w '%{http_code}'` ispiši samo status
- `--max-time 3` — odustani nakon 3 s; `--noproxy '*'` — zaobiđi proxy (vidi gotču)

### Gotča (LO5)
Bez `--noproxy`, `curl` na vanjski IP ide **kroz proxy** (jer `no_proxy` pokriva samo localhost) i proxy vrati `503` umjesto izravnog "refused". Definitivan dokaz binding-a je svejedno `podman port` / `inspect` (vidi zad. 20). Tipfeler `127.0.0.0.1` (jedan `0` viška) → "cannot parse as IP".

### Zašto
Vezanjem na `127.0.0.1` servis je vidljiv **samo lokalno** — za stvari koje smiju pričati samo s procesima na istom stroju (npr. baza iza aplikacije).

---

## Zadatak 20 — `podman port`

### Naredba
```bash
podman port ip-demo                                          # → 80/tcp -> 127.0.0.1:8082
podman inspect --format '{{.NetworkSettings.Ports}}' ip-demo # → map[80/tcp:[{127.0.0.1 8082}]]
```

### Zašto
`podman port` = brzi pregled "koji port vodi kamo"; `inspect` to potvrđuje iz punog zapisa. Dva pogleda na isto.

---

## Zadatak 21 — ne-root korisnik (`--user`)

### Naredba
```bash
podman run -d --user 1000 --name user-demo docker.io/library/httpd   # httpd ispadne (Exited 1) — OČEKIVANO
podman run --rm --user 1000 docker.io/library/httpd id               # → uid=1000(student) gid=0(root)
podman run --rm --user 1000 docker.io/library/httpd whoami           # → student
```

### Razlaganje
- `--user 1000` — pokreće glavni proces kao UID 1000 (umjesto root/UID 0)
- httpd kao ne-root zna **ispasti** (treba prava na neke datoteke) — to je dio pouke
- `id` / `whoami` dokazuju da nije root

### Zašto
`--user` spušta kontejner s root-a na običnog korisnika. **Princip najmanje privilegije** — ako netko provali u kontejner, ima manje ovlasti. (Zato httpd worker procese spušta na `www-data`, zad. 12.)

---

## Zadatak 22 — systemd unit / Quadlet (start na boot)

### Koncept
Cilj: generirati systemd unit (ili Quadlet `.container`) i objasniti kako pokreće kontejner na boot.

### Naredba / Manifest
```bash
# Stari način (ZASTARJELO u novim verzijama):
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

### Razlaganje
- `generate systemd` — ispisuje systemd unit za kontejner (u novim verzijama zastarjelo → koristi se Quadlet)
- `cat > datoteka << 'EOF' ... EOF` — upisuje sve retke između `EOF` oznaka u datoteku (zgodno za višeredni sadržaj); `mkdir -p` napravi mapu ako fali (vidi zad. 14)
- Quadlet `.container` datoteka: `[Container] Image=...` što pokrenuti, `PublishPort=` kao `-p`
- `[Install] WantedBy=default.target` — "pokreće se kad sustav dođe u normalno stanje" = **na boot**

### Zašto (poveži sa zad. 2)
systemd/Quadlet pretvara kontejner u **uslugu kojom upravlja sustav** — pokreće ga automatski **na boot** i restarta ako padne. `--restart=always` (zad. 2) djeluje na pad procesa; systemd pokriva pokretanje **nakon reboota**. To je "pravi" način da kontejner trajno radi na poslužitelju.

### Gotča
`podman generate systemd` je ZASTARJELO; suvremeni način je Quadlet `.container` datoteka.

---

## Zadatak 23 — `podman events` (životni ciklus)

### Koncept
Cilj: pratiti događaje u jednom terminalu dok se u drugom pokreće/gasi kontejner.

### Naredba
```bash
# Terminal 1:
podman events        # stoji i ispisuje uživo; izlaz s Ctrl+C

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

### Zašto
`events` pokazuje **životni ciklus** u stvarnom vremenu. Koristi se za **debugiranje** ("zašto se kontejner ugasio?") i nadzor. Ako nema drugog terminala: `podman events --since 1m --stream=false` pokaže nedavne događaje *nakon* akcije.

---

## Zbirka zamki (LO5 troubleshooting — zlato za ispit)

| # | Zamka | Rješenje / objašnjenje |
|---|-------|------------------------|
| 2 | `podman kill` ne okine restart | Namjeran stop. Za restart ubij host-PID s `kill -9` ili reboot. |
| 2 | `Exited (137)` | `128 + 9` = ubijen SIGKILL-om. `143` = `128+15` = SIGTERM. |
| 3 | `controller cpu is not available` | Rootless cgroups v2: `cpu` nije delegiran. Treba root (`Delegate=cpu`) ili rootful. `memory` radi. |
| 4 | `env` pun "šuma" | Slika + sustav + **proxy** varijable su normalne. Filtrira se s `grep`. |
| 6 | `ps --filter` skriva pauzirani kontejner | Makne se filter (`ps -a`) ili koristi `inspect --format '{{.State.Status}}'`. |
| 11 | promjena nestala nakon `rm` | Živjela u zapisivom sloju. Za trajnost: `commit` ili Containerfile. |
| 19 | vanjski curl daje `503` umjesto refused | Ide kroz proxy. Dodaj `--noproxy '*'`. Binding dokaži s `podman port`. |
| 21 | httpd ispadne uz `--user 1000` | Očekivano (treba root prava). UID dokaži s `id`/`whoami`. |
| 22 | `generate systemd` ZASTARJELO | Koristi se Quadlet `.container` datoteka. |

## Brza referenca

**Exit kodovi:** `0` čist izlaz · `137` SIGKILL (128+9) · `143` SIGTERM (128+15)
**diff markeri:** `A` Added · `C` Changed · `D` Deleted
**restart politike:** `no` · `on-failure` · `always`
**`-p` smjer:** `HOST:CONTAINER` (desno = fiksni port aplikacije unutra)
**`cp` smjer:** strana s `container:` prefiksom = unutar kontejnera
**čišćenje:** `podman rm -f <ime>` (i ako se vrti) · `podman rmi <slika>` · `podman network rm <mreza>`

---

## Ključne rečenice za pamćenje
1. **Slika je zamrznuta i dijeljena; kontejner je slika + privatni zapisivi sloj koji se briše zajedno s njim — osim ako se `commit`-om ne pretvori u novu sliku.**
2. **Podman nema centralnu pozadinsku uslugu; svaki kontejner motri `conmon`. Zato je ručni `stop`/`kill` namjeran (bez restarta), a vanjska/neočekivana smrt okida restart.**
3. **Kad alat (`ps --filter`) da neočekivano prazno — valja sumnjati u alat, ne samo u stvar. Provjeri se golim `ps -a` ili `inspect`.**
