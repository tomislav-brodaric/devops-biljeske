# DevOps LO5 — Troubleshooting (rješavanje problema s isporukom)

Okruženje: isti rootless VM (Podman + minikube + kubectl, **bez sudo**). Nastavak na LO1–LO4.

**Format svakog zadatka (po PDF-u):** *uočavanje greške → ispravak → objašnjenje zašto je bila kriva.*
Većina se rješava rezoniranjem iz LO1–4; hands-on samo gdje se želi vidjeti uživo.

---

## Rječnik kratica

- **OOM / OOMKilled** — Out Of Memory; jezgrin OOM-killer ubije proces koji probije memorijski limit (izlazni kod 137).
- **SIGTERM / SIGKILL** — signal za uredno gašenje (može se uhvatiti) / signal za prisilno gašenje (kod 9, ne može se ignorirati).
- **PID 1** — prvi proces u kontejneru; kontejner živi dok taj proces traje.
- **TTY** — terminalski uređaj; `-t` dodjeljuje pseudo-TTY za interaktivni rad.
- **DNS** — Domain Name System; razlučivanje imena u IP adrese.
- **FQDN** — Fully Qualified Domain Name; puno ime, npr. `<svc>.<ns>.svc.cluster.local`.
- **NAT** — Network Address Translation; prosljeđivanje s host porta na port kontejnera (`-p`).
- **NXDOMAIN** — odgovor DNS-a da traženo ime ne postoji.
- **PV / PVC** — PersistentVolume (trajni svezak) / PersistentVolumeClaim (zahtjev za trajnim sveskom).
- **RWO** — ReadWriteOnce; pristupni način svezka (jedan čvor čita i piše).
- **CNI** — Container Network Interface; mrežni dodatak klastera.
- **CRD** — Custom Resource Definition; korisnički definiran tip resursa.
- **RS** — ReplicaSet; objekt koji Deployment koristi za održavanje replika.
- **ClusterIP / NodePort / LoadBalancer** — tipovi Servisa (interni IP / port na čvoru / vanjski balanser).
- **CoreDNS** — DNS poslužitelj klastera (ClusterIP servisa `kube-dns`, obično 10.96.0.10).
- **TLS** — Transport Layer Security; šifriranje prometa (edge / passthrough / re-encrypt kod Route/Ingress).
- **L4 / L7** — sloj 4 (transportni, TCP/UDP) / sloj 7 (aplikacijski, HTTP).
- **TTL** — Time To Live; rok trajanja (npr. `--event-ttl` za k8s događaje).
- **GA** — General Availability; značajka izašla iz bete (stabilna).
- **CI/CD** — Continuous Integration / Continuous Delivery; automatizirana izgradnja i isporuka.
- **YAML / JSON** — formati zapisa manifesta; kubectl parsira YAML u JSON prije slanja serveru.
- **cgroup** — kontrolna grupa jezgre; provodi tvrde limite resursa (npr. `--memory`, `limits.memory`).
- **SELinux** — sigurnosni model oznaka; `:Z`/`:z` relabela bind-mount za pristup iz kontejnera.

## Pregled — 43 stavke u 3 skupine

| Skupina | Tema | Zadaci |
|---------|------|--------|
| **A** | Pogrešne `podman run` naredbe | 1–8 |
| **B** | Pogrešni Containerfile/Dockerfile | 9–18 (zad. 9 = naslov skupine, nije primjer) |
| **C** | k8s dijagnostika | 19–43 |

Napredak: **Skupina A KOMPLETNA** ✓ (1–8) · **Skupina B KOMPLETNA** ✓ (9–18) · **Skupina C KOMPLETNA** ✓ (19–43) → **LO5 GOTOV**

---

## VM-specifične zamke (rootless, bez sudo) — vrijede kroz cijeli LO5

Popis se dopunjava kako problemi naiđu.

- **Host port < 1024 = privilegiran** (80, 443…) → rootless Podman ga **ne smije** vezati (`Permission denied`). Hostovski port uvijek treba biti **≥ 1024**.
- **Neuspjeli `run` ostavi kreirani kontejner** koji i dalje drži `--name` → idući `run` s istim imenom pukne na *"name already in use"*. Rješava se s `podman rm <ime>` ili zastavicom `--replace`.
- `minikube tunnel` NE radi (nema sudo) → LoadBalancer ostaje `<pending>`; pristupa se preko NodePorta / `port-forward`.

---

# Skupina A — pogrešne `podman run` naredbe

## Anatomija `podman run` + zastavice (vrijedi za cijelu Skupinu A)

Opći oblik:
```
podman run [ZASTAVICE] SLIKA [NAREDBA] [ARGUMENTI]
```
- `podman run` — stvara NOVI kontejner iz slike i pokreće ga
- `SLIKA` — **jedini obavezni** pozicijski argument (npr. `docker.io/library/nginx`)
- `[NAREDBA]` — neobavezno; gazi default naredbu slike (npr. `... busybox sleep infinity`)

**Zastavice koje se ponavljaju kroz skupinu A:**
- `-d` (`--detach`) — pokreće u **pozadini**; terminal odmah slobodan, ispiše se samo ID kontejnera
- `--name <ime>` — ime kontejnera (inače Podman dodijeli nasumično, npr. `keen_pasteur`); mora biti **slobodno**
- `-p <host>:<kontejner>` (`--publish`) — objavi/mapiraj port: **host : kontejner** (lijevo host, desno port na kojem proces unutra sluša). Radi NAT preko granice mrežnog namespace-a.
- `-e <KLJUČ>=<vrij>` (`--env`) — postavi env-varijablu. **`-e KLJUČ` bez `=` preuzima vrijednost s HOSTA** (vidi zad. 2).
- `--rm` — **automatski obriši** kontejner čim STANE (vidi zad. 5)
- `--network <mod>` — mrežni način; default je `bridge` (vlastiti namespace + DNS po imenu na *custom* mrežama). `--network host` = dijeli hostov namespace (vidi zad. 3).
- `--replace` — ako kontejner s tim `--name` već postoji, ugasi ga i zamijeni novim (umjesto greške „name already in use")
- `-it` — `-i` (interaktivno, drži stdin otvoren) + `-t` (dodijeli pseudo-TTY); za rad u shellu (NIJE „detached koji radi")
- `-f` (na `rm`) — `--force`: kontejner se prvo ugasi ako radi, pa ukloni

**Dijagnostičke naredbe (također po dijelovima):**
- `podman ps` — popis kontejnera koji **rade**
- `podman ps -a` — `-a` (`--all`) = i **zaustavljeni/izašli** (tu se vide `Exited (...)`)
- `podman logs <ime>` — stdout/stderr kontejnera (često doslovno imenuje uzrok pada)
- `podman port <ime>` — popis mapiranja porta za kontejner
- `podman inspect <ime>` — puni JSON detalji kontejnera

**Naziv slike — `registry/namespace/ime:tag`:**
- `docker.io/library/nginx` → registry `docker.io`, namespace `library` (službene slike), ime `nginx`
- bez taga → povlači `:latest`

---

## Zadatak 1 — obrnut `-p host:kontejner` (+ dvije VM-zamke uživo)

**Pogrešna naredba:**
```bash
podman run -d --name web -p 80:8080 docker.io/library/nginx
```

**Razlaganje naredbe:**
- `podman run` — stvara i pokreće novi kontejner
- `-d` — u pozadini (ispiše samo ID)
- `--name web` — ime kontejnera = `web`
- `-p 80:8080` — mapiranje **host 80 → kontejner 8080** ← **OVDJE greška** (obrnuto; nginx sluša na 80, ne 8080)
- `docker.io/library/nginx` — slika (tag `:latest`)

**Što se stvarno dogodilo na VM-u (provjereno):**
```
Error: pasta failed with exit code 1:
Failed to bind port 80 (Permission denied) for option '-t 80-80:8080-8080'
```

### Greška 1 — portovi su obrnuti

`-p` se uvijek čita kao **`host:kontejner`**. Ovdje `80:8080` znači: host **80** → kontejner **8080**. Ali nginx *unutar* kontejnera sluša na **80**, ne na 8080. Na 8080 unutra nitko ne sluša → ništa ne odgovara.

- desni broj (kontejner) MORA biti port na kojem proces unutra stvarno sluša → nginx = **80**
- lijevi broj (host) je proizvoljan port kojim pristupaš izvana

### Greška 2 (bonus, rootless VM) — host port 80 je privilegiran

Naredba nije ni stigla do nginxa — pukla je ranije, na **hostovskoj** strani:

- portovi **< 1024** su privilegirani (80, 443…)
- rootless Podman (bez sudo na ovom VM-u) **ne smije** vezati privilegirani host port → `Permission denied`
- `pasta` = rootless mrežni backend koji radi prosljeđivanje porta; on je javio grešku
- u ispisu se vidi i obrnutost: `-t 80-80:8080-8080` (host 80 → kontejner 8080)
- **Pravilo za VM: hostovski port uvijek ≥ 1024.**

### Greška 3 (bonus) — neuspjeli run zauzme ime

Nakon ispravka na `-p 8080:80`, idući pokušaj pukne ovako:
```
Error: ... the container name "web" is already in use by <id> ...
You have to remove that container ... or use --replace ...
```

- prvi (neuspjeli) `run` je **već kreirao** kontejner i zauzeo ime "web" PRIJE nego što je mreža pukla
- ime sad visi zauzeto → sudar pri ponovnom `run`

### Ispravak

```bash
podman rm web                                               # makni mrtvi kontejner koji drži ime
podman run -d --name web -p 8080:80 docker.io/library/nginx
```

- `podman rm web` = obriši zaustavljeni kontejner (oslobodi ime)
  - ako `rm` kaže da kontejner radi → `podman rm -f web` (`-f` = force: gasi pa uklanja)
- `-p 8080:80` = host **8080** (≥ 1024 ✓) → kontejner **80** (gdje nginx sluša)

Jednolinijski (sama poruka to predlaže):
```bash
podman run -d --replace --name web -p 8080:80 docker.io/library/nginx
```
- `--replace` = ako kontejner s tim imenom postoji, tiho se uklanja i pravi novi

**Provjera:**
```bash
podman ps                  # web: Up, PORTS 0.0.0.0:8080->80/tcp
curl localhost:8080        # vraća nginx welcome HTML
```

### Zašto (srž za ispit)

Redoslijed u `-p` je **uvijek host:kontejner**; desni broj mora pogoditi port na kojem proces *unutra* stvarno sluša. U rootless modu host port mora biti **≥ 1024**. I: svaki `run` (čak i neuspjeli) rezervira `--name`, pa ime treba osloboditi (`rm`) ili pregaziti (`--replace`).

### Gotče

- Na **non-rootless** okruženju `-p 80:8080` ne bi pucalo na dozvoli — samo se ništa ne bi vidjelo (tihi kvar, teže za uočiti). Na ovom VM-u glasno pukne zbog privilegiranog porta → ispada lakše.
- `--rm` ovdje NIJE korišten; da jest, mrtvi kontejner bi se sam maknuo i ne bi bilo sudara imena (ali bi onda nestao i za inspekciju). Veza: zad. 5.
- `docker.io/library/nginx` = puni naziv slike: registry `docker.io`, namespace `library` (službene slike), ime `nginx`. Bez taga → `:latest`.
- **`Error: requires at least 1 arg(s), only received 0`** → zaboravljena **slika** na kraju (npr. `podman run -d --replace --name web -p 8080:80` bez naziva slike). Slika je jedini OBAVEZNI pozicijski argument; sve ostalo (`-d`, `--name`, `-p`, `--replace`) su zastavice. Dodaj naziv slike na kraj.

---

## Zadatak 2 — `-e MYSQL_ROOT_PASSWORD` bez vrijednosti (kontejner izađe pri initu)

**Pogrešna naredba:**
```bash
podman run -d --name db -e MYSQL_ROOT_PASSWORD docker.io/library/mysql
```

**Razlaganje naredbe:**
- `podman run` — stvara i pokreće novi kontejner
- `-d` — u pozadini
- `--name db` — ime kontejnera = `db`
- `-e MYSQL_ROOT_PASSWORD` — env-varijabla **bez `=vrijednost`** ← **OVDJE greška** (preuzima vrijednost s hosta; nema je → prazno)
- `docker.io/library/mysql` — slika (tag `:latest`)

**Što se stvarno dogodilo na VM-u (provjereno):**

`podman ps -a` (`-a` = prikaži i izašle kontejnere):
```
CONTAINER ID  IMAGE         ...  STATUS                     ...  NAMES
42cba4381b89  mysql:latest  ...  Exited (1) About a minute  ...  db
```
`Exited (1)` = izašao s greškom (exit code 1) — NE radi.

`podman logs db` (zašto je stao — log sam imenuje uzrok):
```
[ERROR] [Entrypoint]: Database is uninitialized and password option is not specified
    You need to specify one of the following as an environment variable:
    - MYSQL_ROOT_PASSWORD
    - MYSQL_ALLOW_EMPTY_PASSWORD
    - MYSQL_RANDOM_ROOT_PASSWORD
```

### Greška i uzrok — `-e KEY` (bez `=value`) ≠ `-e KEY=value`

- `-e MYSQL_ROOT_PASSWORD=secret` → postavi varijablu na vrijednost `secret`
- `-e MYSQL_ROOT_PASSWORD` (bez `=`) → znači **„proslijedi vrijednost te varijable s HOSTA"**. Pošto na hostu ta varijabla NIJE postavljena, u kontejner ne dođe nikakva vrijednost → mysql vidi prazno → odbija inicijalizaciju bez root lozinke → izlazi s kodom 1.

### Ispravak

```bash
podman rm db
podman run -d --name db -e MYSQL_ROOT_PASSWORD=secret docker.io/library/mysql
```
- ključ je `=secret` (bilo koja ne-prazna vrijednost)
- alternative koje sam log nudi:
  - `-e MYSQL_ALLOW_EMPTY_PASSWORD=yes` — dozvoli praznu lozinku (SAMO za test, nesigurno)
  - `-e MYSQL_RANDOM_ROOT_PASSWORD=yes` — mysql generira nasumičnu i ispiše je u log

**Provjera:**
```bash
podman ps          # db sad: Up (a ne Exited)
podman logs db     # pred kraj: "ready for connections"
```

### Zašto (srž za ispit)

Forma `-e VAR` bez znaka jednakosti NE postavlja vrijednost — ona *preuzima* `VAR` iz okruženja hosta. Nema li je host, varijabla efektivno ostaje prazna. mysqlov entrypoint zahtijeva da root lozinka bude zadana (ili eksplicitno dozvoljena prazna), inače prekida init i izlazi. Lekcija: za zadavanje vrijednosti uvijek `KEY=VALUE`.

### Gotče

- `Exited (0)` vs `Exited (1)`: `0` = uredan izlazak (npr. posao gotov), bilo što ≠ 0 = greška. mysql ovdje daje `1`.
- **log uvijek prvo:** kod baza/aplikacija koje „samo izađu", `podman logs <ime>` gotovo uvijek doslovno imenuje uzrok — kao ovdje.
- ista `requires at least 1 arg(s)` greška se ponovila jer je opet falila slika na kraju (vidi Zadatak 1).
- **`&` umjesto `/` u imenu slike** (`docker.io/library&mysql`): `&` je bashu znak za *pokreni u pozadini*, pa se naredba prelomi na `podman run ... docker.io/library` (u pozadini) + zasebnu naredbu `mysql`. Simptomi: `[1] <pid>`, `bash: mysql: command not found`, i pull `docker.io/library/library:latest` → `access ... denied`. Ispravak: kosa crta `/`, ne `&`.

---

## Zadatak 3 — `--network host` + `-p` (zašto je `-p` ignoriran)

**Pogrešna naredba:**
```bash
podman run -d --name app --network host -p 8080:80 docker.io/library/nginx
```

**Razlaganje naredbe:**
- `podman run` — stvara i pokreće novi kontejner
- `-d` — u pozadini
- `--name app` — ime kontejnera = `app`
- `--network host` — dijeli **hostov** mrežni namespace ← **OVDJE greška** (uz ovo `-p` nema smisla)
- `-p 8080:80` — mapiranje host 8080 → kontejner 80 (ispravno po sebi, ali ga host net poništava)
- `docker.io/library/nginx` — slika (tag `:latest`)

**Što se stvarno dogodilo na VM-u (provjereno):**

Odmah nakon `run`:
```
Port mappings have been discarded because "host" network namespace mode does not support them
e75e8b1a5ca5...
```
`podman port app` → ništa (nema mapiranja); `podman ps` → stupac `PORTS` prazan.

`podman logs app`:
```
... Configuration complete; ready for start up
[emerg] bind() to 0.0.0.0:80 failed (13: Permission denied)
nginx: [emerg] bind() to 0.0.0.0:80 failed (13: Permission denied)
```

### Greška i uzrok — `-p` nema smisla uz host networking

Dvije neovisne stvari u jednom ispisu:

1. **`-p 8080:80` je odbačen** (Podman to doslovno kaže). Kod normalne (bridge) mreže kontejner dobije VLASTITI mrežni namespace s vlastitim IP-jem, a `-p host:kontejner` postavi NAT/prosljeđivanje preko te granice. Uz `--network host` kontejner NEMA zaseban namespace — dijeli hostov. Nema granice preko koje bi se NAT-alo → `-p` nema što raditi → Podman ga baci uz upozorenje.
2. **nginx zatim padne na bind 0.0.0.0:80.** Pošto dijeli hostovu mrežu, nginx pokušava vezati **host** port 80 IZRAVNO. Rootless korisnik ne smije port < 1024 → `Permission denied` (isti uzrok kao Zadatak 1, samo sad iznutra).

### Ispravak

**Namjera se vidi iz same naredbe:** `-p 8080:80` je naveden → namjera je mapiranje porta. Dakle `--network host` je greška/višak, a NE `-p`.

**Glavni fix — ukloniti `--network host`** (povratak na bridge + `-p`, radna forma iz Zadatka 1):
```bash
podman run -d --replace --name app -p 8080:80 docker.io/library/nginx
```
Provjera (provjereno uživo ✓): `podman ps` → `PORTS 0.0.0.0:8080->80/tcp`; `podman port app` → `80/tcp -> 0.0.0.0:8080`; `curl localhost:8080` → nginx welcome.

(Alternativa samo ako je host net stvarno potreban — što ovdje nije slučaj: ukloniti `-p` i pustiti nginx na port ≥ 1024.)

### Zašto (srž za ispit)

`-p` (publish) ima smisla SAMO kad postoji granica mrežnog namespace-a između kontejnera i hosta — tada se NAT-a s host porta na kontejnerski. `--network host` ukida tu granicu: kontejner koristi hostov mrežni stog izravno, pa procesi unutra vežu hostove portove direktno. Zato Podman `-p` jednostavno odbaci. Posljedica na rootless VM-u: izravan bind na 80 puca na dozvoli.

### Gotče

- `podman port <ime>` je brz dokaz: uz host net vrati prazno (nema mapiranja).
- `--network host` zna biti zamka jer „radi" za portove ≥ 1024 i bez `-p`, ali `-p` tiho ignorira → lako se previdi da publish ništa ne čini.
- host networking žrtvuje izolaciju (kontejner vidi sve hostove interfejse/portove) → koristi se rijetko (izvedba, mrežni alati), nikad kao default.

---

## Zadatak 4 — busybox odmah izađe `Exited (0)` (nema dugotrajne naredbe)

**Pogrešna naredba:**
```bash
podman run -d --name c1 docker.io/library/busybox
```

**Razlaganje naredbe:**
- `podman run` — stvara i pokreće novi kontejner
- `-d` — u pozadini
- `--name c1` — ime kontejnera = `c1`
- `docker.io/library/busybox` — slika (tag `:latest`)
- *(nema NAREDBE na kraju)* ← **OVDJE greška**: pokreće se default `sh`, koji s `-d` (bez stdina) odmah izađe

**Što se stvarno dogodilo na VM-u (provjereno):**

Naredba „uspije" (ispiše ID), ali `podman ps -a`:
```
CONTAINER ID  IMAGE           COMMAND  ...  STATUS                     ...  NAMES
8d2223112326  busybox:latest  sh       ...  Exited (0) About a minute  ...  c1
```
`COMMAND = sh`, `STATUS = Exited (0)`. `podman ps` (bez `-a`) ga NE prikazuje. `podman logs c1` je prazan (tihi kvar).

### Greška i uzrok — kontejner nema dugotrajan glavni proces

- busybox default naredba je `sh`. S `-d` (detached) nema priključenog terminala/stdina → `sh` odmah dođe do EOF i uredno izađe (kod **0**).
- **Kontejner živi točno koliko i njegov PID 1.** Izađe li glavni proces → kontejner staje. Nije pad (kod 0 = uredno), nego naprosto nema što raditi.
- Tihi kvar: nema poruke o grešci ni u logu. Dijagnoza ide preko `STATUS` u `podman ps -a`, ne preko `logs`.

### Ispravak — daj mu dugotrajnu naredbu

```bash
podman run -d --replace --name c1 docker.io/library/busybox sleep infinity
```
`podman ps` → `c1 ... sleep infinity ... Up`. **(provjereno uživo ✓)**

Alternative: `sleep 1d`, `tail -f /dev/null`. (Za interaktivni rad bio bi `-it`, ali to nije „detached koji radi".)

### Zašto (srž za ispit)

Kontejner nije VM — nema „pozadinski OS koji ostaje upaljen". On je omotač oko JEDNOG procesa (PID 1). Kad taj proces završi, kontejner završi, bez obzira na izlazni kod. Da bi „radio u pozadini", mora imati proces koji ne završava (server, ili umjetno `sleep`/`tail -f`).

### Gotče

- `Exited (0)` ≠ kvar — to je uredan izlazak. Problem je samo što se htjelo da radi.
- **`--replace` na `sleep infinity` kontejneru → `WARN: StopSignal SIGTERM failed ... resorting to SIGKILL`** (viđeno uživo). Razlog: **PID 1 u kontejneru ne dobiva default obradu signala** — jezgra ne primjenjuje zadane akcije signala na PID 1. `sleep` ne instalira vlastiti SIGTERM-handler → ignorira SIGTERM → Podman nakon 10 s grace perioda šalje SIGKILL (kod 9, ne može se ignorirati). Bezopasno ovdje, ali isti mehanizam je razlog zašto je važna pravilna obrada signala / exec forma → **veza: Zadatak 11**.
- `sleep` BEZ argumenta (`busybox sleep`) → greška „missing operand"; uvijek `sleep <trajanje>` ili `sleep infinity`.
- typo `sčeep` umjesto `sleep` (hrvatska tipkovnica, `č`↔`l`) → još jedna retype-zamka; uhvatiti prije Entera ili `^C`.

---

## Zadatak 5 — `--rm` + `-d` (kontejner nestane prije čitanja loga)

**Pogrešna naredba:**
```bash
podman run --rm -d --name job docker.io/library/alpine echo hello
```

**Razlaganje naredbe:**
- `podman run` — stvara i pokreće novi kontejner
- `--rm` — **obriši kontejner čim STANE** ← ključno za ovu zamku
- `-d` — u pozadini
- `--name job` — ime kontejnera = `job`
- `docker.io/library/alpine` — slika (tag `:latest`)
- `echo hello` — NAREDBA: ispiši „hello" i odmah završi

**Što se stvarno dogodilo na VM-u (provjereno):**

Naredba „uspije" (ispiše ID), a onda `podman logs job`:
```
Error: no container with name or ID "job" found: no such container
```

### Greška i uzrok — auto-brisanje stigne prerano

- `echo hello` se izvrši u trenu, ispiše, i kontejner odmah izađe (kod 0).
- `--rm` znači *„obriši kontejner čim stane"* → kontejner **nestane istog trena** kad `echo` završi.
- Dok se utipka `podman logs job`, kontejnera **više nema** → „no such container".
- To je `--rm` + detached zamka: spaja se auto-brisanje s procesom koji završi prebrzo da bi ga se stiglo pregledati. (Kod dugotrajnog procesa `--rm` je OK — briše tek pri ručnom zaustavljanju.)

### Ispravak — ovisno o namjeri

**A) Vidjeti izlaz odmah → ukloniti `-d`** (pokretanje u prvom planu):
```bash
podman run --rm --name job docker.io/library/alpine echo hello
```
Ispiše `hello` ravno u terminal, pa se počisti. **(provjereno uživo ✓)**
- bez `-d` proces drži terminal dok ne završi → stdout je vidljiv direktno

**B) Log mora PREŽIVJETI za kasnije → ukloniti `--rm`**:
```bash
podman run -d --replace --name job docker.io/library/alpine echo hello
podman logs job        # ispiše: hello
```
**(provjereno uživo ✓)**
- kontejner ostane kao `Exited (0)` i dalje postoji → `logs` ga može pročitati
- `--replace` jer ime `job` može biti zauzeto od ranijeg pokušaja

### Zašto (srž za ispit)

`--rm` je vezan uz **prestanak rada** kontejnera, ne uz `run`. Kod procesa koji odmah završi (`echo`, batch posao), kontejner se obriše prije nego se dođe do `logs`. Ako je potreban ispis kratkotrajnog posla: ili se gleda u prvom planu (bez `-d`), ili se ne briše (izostavi se `--rm`), pa se očisti ručno.

### Gotče

- `--rm` je super za dugotrajne ad-hoc kontejnere (sami se počiste), ali za „pokreni-pa-pročitaj" kratke poslove uništi dokaz.
- `podman logs` radi i na **zaustavljenom** kontejneru — sve dok kontejner POSTOJI (nije obrisan).
- redoslijed `--rm -d` vs `-d --rm` nije bitan; bitno je da su zajedno na kratkotrajnom procesu.

---

## Zadatak 6 — `--memory 8m` premali za mysql (OOMKilled)

**Pogrešna naredba:**
```bash
podman run -d -p 8080:80 --memory 8m docker.io/library/mysql
```

**Razlaganje naredbe:**
- `podman run` — stvara i pokreće kontejner
- `-d` — u pozadini
- `-p 8080:80` — host 8080 → kontejner 80 *(usput pogrešno: mysql sluša na 3306, ne 80)*
- `--memory 8m` — tvrdi limit memorije = **8 MB** ← **OVDJE greška**
- `docker.io/library/mysql` — slika
- *(nema `--name`, nema `MYSQL_ROOT_PASSWORD`)*

**Što se stvarno dogodilo na VM-u (provjereno):**

Doslovna naredba pukne ranije (port zauzet):
```
Failed to bind port 8080 (Address already in use) for option '-t 8080-8080:80-80'
```
→ ostavi kontejner u stanju `Created` (nikad ne krene).

Kad izoliramo memoriju (dodana lozinka, maknut sukob porta, `--memory 8m`), `podman ps -a`:
```
24a0d480a5a4  mysql:latest  mysqld  ...  Exited (137)  ...  db2
```
`podman logs db2` stane na prvom retku:
```
[Note] [Entrypoint]: Entrypoint script for MySQL Server 9.7.1-1.el9 started.
```

### Greška i uzrok — 8 MB je premalo; jezgra ubije proces (OOM)

- mysql treba **stotine MB** samo za podizanje (InnoDB buffer pool, konekcijski baferi…). 8 MB ni izbliza nije dovoljno.
- Kad proces probije cgroup memorijski limit, **jezgrin OOM-killer** ga ubije sa **SIGKILL**.
- **`Exited (137)` = OOM kill.** Računica: izlazni kod ubijenog procesa = `128 + broj_signala`; SIGKILL = 9 → `128 + 9 = 137`. (Valja zapamtiti: **137 = OOM**.)
- Log stane na prvom retku jer je proces ubijen usred dizanja, prije ičega korisnog.

### Ispravak — podizanje memorije (ili uklanjanje limita)

```bash
podman run -d --replace --name db2 -e MYSQL_ROOT_PASSWORD=secret --memory 512m docker.io/library/mysql
```
`podman ps` → `db2 ... Up`; `podman logs db2` → pred kraj `ready for connections`. **(provjereno uživo ✓)**
- 512 MB je dovoljno za default mysql; alternativno potpuno izostavi `--memory`.

### Zašto (srž za ispit)

`--memory` postavlja **tvrdu** gornju granicu kroz cgroups. To NIJE preporuka — to je zid: probije li ga proces, jezgra ga ubije (SIGKILL, kod 137), bez milosti i bez aplikacijske poruke. Memorijski limit mora pokriti stvarne potrebe procesa pri vršnom opterećenju (kod baza i tijekom dizanja).

### Bonus — `--container_aware is OFF` (suptilno, pojavi se u logu)

U logu uspješnog db2 (512m):
```
[Warning] Server ignores the discovered container restrictions as --container_aware is OFF
... MySQL Server has access to 16497238016 bytes of physical memory.
```
- mysql tu javlja **16 GB** (memorija HOSTA), ne 512 MB iz limita — jer mu je `--container_aware` isključen pa NE čita cgroup limit.
- **ALI limit svejedno vrijedi:** cgroup je kernelski zid neovisan o tome „vidi" li ga aplikacija. Zato je 8m svejedno ubilo proces — mysql je mislio da ima 16 GB, alocirao po tome, probio 8 MB cgroup limit → OOM. Aplikacijska (ne)svijest o limitu ne mijenja da ga jezgra provodi.

### Gotče

- **`Exited (137)` → OOM** (premalo memorije). Ekvivalent u Kubernetesu je status **OOMKilled** → dio C, zad. 23.
- doslovna naredba je pala na **`Address already in use`** za 8080 jer ga već drži `app` (nginx) → prije ponovnog `-p` provjeriti zauzete portove (`podman ps`).
- `-p 8080:80` za mysql je dvostruko krivo: i port zauzet, i krivi kontejnerski port (mysql = 3306).

---

## Zadatak 7 — DNS po imenu ne radi na default mreži

**Pogrešne naredbe:**
```bash
podman run -d --name a docker.io/library/alpine sleep 1d
podman run -d --name b docker.io/library/alpine sleep 1d
podman exec a ping b
```

**Razlaganje naredbi:**
- prva dva `run` — dva kontejnera (`a`, `b`), slika alpine, naredba `sleep 1d` (žive ~1 dan da ostanu Up)
- `podman exec a ping b` — `exec` = pokreće naredbu u VEĆ pokrenutom kontejneru; ovdje: iz `a` šalje ping na `b` **po imenu** ← tu pukne

**Što se stvarno dogodilo na VM-u (provjereno):**

Na default mreži:
```
ping: bad address 'b'
```
`bad address 'b'` = busybox ping nije razlučio ime `b` u IP (DNS ne radi).

### Greška i uzrok — default mreža nema DNS za imena kontejnera

- Kontejneri bez `--network` idu na **default** mrežu. Ona NE rješava imena kontejnera u IP.
- Razlučivanje imena (`b` → IP) radi **samo na korisnički definiranoj (custom) mreži**, gdje Podmanov `aardvark-dns` servisira imena.
- Zato `ping b` na defaultu kaže „bad address" — ne zna tko je `b`.

### Ispravak — napraviti custom mrežu i spojiti oba kontejnera na nju

```bash
podman network create mojamreza
podman run -d --replace --name a --network mojamreza docker.io/library/alpine sleep 1d
podman run -d --replace --name b --network mojamreza docker.io/library/alpine sleep 1d
podman exec a ping -c 3 b
```
Rezultat **(provjereno uživo ✓)**:
```
PING b (10.89.0.3): 56 data bytes
64 bytes from 10.89.0.3: seq=0 ttl=42 time=0.079 ms
...
3 packets transmitted, 3 packets received, 0% packet loss
```
`b` se razlučio u `10.89.0.3` (raspon custom mreže) i ping prolazi.

- `podman network create <ime>` — stvara korisničku bridge mrežu (s DNS-om)
- `--network mojamreza` — zakači kontejner na tu mrežu
- `-c 3` (na ping) — pošalji 3 paketa pa stani (inače ping ide u beskonačno)

### Zašto (srž za ispit)

Na default Podman mreži nema servisa za razlučivanje imena — komunikacija ide samo po IP-u. Čim se kontejneri stave na **istu korisnički definiranu** mrežu, `aardvark-dns` automatski razlučuje **imena kontejnera** (i `--network-alias`) u njihove IP-jeve. Zato „aplikacija nađe bazu po imenu" radi tek na custom mreži (isto vrijedi za compose/pod — oni implicitno stvaraju takvu mrežu).

### Gotče

- `ping: bad address 'b'` = problem **razlučivanja imena**, NE mrežne povezanosti. Po IP-u bi i na defaultu prošlo.
- opet iskočio `SIGTERM failed ... SIGKILL` kad je `--replace` gasio stare `sleep 1d` kontejnere (isti PID-1 razlog kao zad. 4) — bezopasno.
- DNS radi za imena na ISTOJ custom mreži; kontejneri na različitim mrežama se opet ne vide po imenu (veza: LO3 segmentacija).

---

## Zadatak 8 — bind-mount: relativna putanja + SELinux `:Z`

**Pogrešna naredba:**
```bash
podman run -d --name web -v ./html:/usr/share/nginx/html docker.io/library/nginx
```

**Razlaganje naredbe:**
- `podman run` `-d` `--name web` — poznato
- `-v ./html:/usr/share/nginx/html` — bind-mount: hostov folder → u kontejner na nginx web-root ← **ovdje dvije zamke**
- `docker.io/library/nginx` — slika

**Što se stvarno dogodilo na VM-u (provjereno):**

Prvo je puklo na **putanji**, ne na SELinuxu:
```
Error: lstat html: no such file or directory
```

### Greška 1 — relativna putanja `./html` ovisi o trenutnom radnom folderu

- `-v ./html:...` je **relativna** putanja → računa se od **trenutnog** radnog foldera.
- Prompt je bio `[student@vm69-61 ~]$` (dakle u `~`), a mapa je bila u `~/devops/lo5/html` → `./html` je pokazivao na `~/html` kojeg nema → `lstat html: no such file`.
- **Fix: apsolutna putanja** — neovisna o trenutnom radnom folderu:
```bash
-v ~/devops/lo5/html:/usr/share/nginx/html
```
(`~/` shell sam proširi u `/home/student`.)

### Greška 2 (po PDF-u) — nedostaje SELinux relabel `:Z`

Teorija (vrijedi za **enforcing** SELinux, npr. RHEL/Fedora host):
- bind-mountani hostov folder zadržava svoju SELinux oznaku (npr. `home_root_t`), koju kontejnerski proces (`container_t`) NE smije čitati → SELinux odbije pristup (datoteke „nevidljive" / 403).
- **`:Z`** = relabela volumen **privatnom** oznakom `container_file_t` (samo ovaj kontejner)
- **`:z`** (malo) = **dijeljena** oznaka (više kontejnera smije isti volumen)
```bash
-v ~/devops/lo5/html:/usr/share/nginx/html:Z
```

### Nalaz na OVOM VM-u (provjereno uživo — bitno!)

Testirano na ČISTOJ mapi (`html2`), mount **bez** `:Z`, prvi pokušaj:
```
curl localhost:8091  →  <h1>Test bez Z</h1>
```
**Radi i bez `:Z`.** Zaključak: rootless Podman na ovom VM-u SELinux **ne provodi striktno** na bind-mountu → datoteke su vidljive bez relabela.

⚠️ **Zamka u testiranju:** ako se prvo pokrene mount **s** `:Z`, on TRAJNO relabela folder na hostu → kasniji mount bez `:Z` zatekne već ispravnu oznaku i „radi", pa se lažno zaključi da `:Z` ne treba. Zato čisti test treba **nova mapa** + mount **bez** `:Z` PRVI.

### Zašto (srž za ispit)

`-v host:kontejner` montira hostov folder u kontejner. Dvije neovisne stvari mogu zapeti: (1) **putanja** — relativni `./` ovisi o `pwd`, pa je apsolutna sigurnija; (2) **SELinux** — na enforcing sustavima kontejner ne smije čitati tuđe oznake, pa treba `:Z`/`:z` da Podman relabela. Na ovom konkretnom VM-u (2) nije prepreka, ali `:Z` je svejedno ispravan, prenosiv odgovor.

### Gotče

- `lstat <x>: no such file or directory` na `-v` → putanja ne postoji (često relativna `./` iz krivog foldera). Rješava se apsolutnom putanjom.
- `:Z` je **destruktivan na razini hosta** — relabela stvarni folder; oprez ako taj folder dijeli i host-proces.
- `echo "..."` s nezatvorenim navodnikom → shell čeka nastavak (prompt `>`); zatvori navodnik ili `^C` pa ponovi.

---

# Skupina B — pogrešni Containerfile / Dockerfile

## Zadatak 9 — (naslov skupine, NIJE primjer)

PDF, str. 7: *„For each Containerfile/Dockerfile: identify the mistake, fix it, and explain what was wrong."*
To je uvodna uputa za skupinu B, ne zadatak — nema priloženog Containerfilea ni greške za popraviti. Primjeri kreću od zad. 10.

**Podsjetnik na alat — `podman build`:**
```
podman build -t <ime> .
```
- `build` — sagradi sliku iz `Containerfile`a u build contextu
- `-t <ime>` — tag (ime) rezultirajuće slike
- `.` — **build context** = folder koji se šalje builderu (tu se traži `Containerfile`)
- `-f <putanja>` — koristi Containerfile s drugog imena/lokacije (inače default `./Containerfile`)


## Zadatak 10 — `apt-get install` bez `apt-get update` (build padne)

**Pokvaren Containerfile:**
```dockerfile
FROM debian:12
RUN apt-get install -y nginx
```

**Razlaganje Containerfilea:**
- `FROM debian:12` — bazna slika: Debian 12, tag `12`
- `RUN apt-get install -y nginx` — u build koraku instaliraj nginx; `-y` = automatski „da" na sve upite ← **OVDJE greška** (nema `update` prije)

**Build naredba (razloženo):**
```bash
podman build -t z10test .
```
- `build` — sagradi sliku iz `Containerfile`a
- `-t z10test` — **tag/ime** slike = `z10test` (bez `:tag` → `:latest`); bez `-t` slika ostane bezimena `<none>`
- `.` — **build context** = trenutni folder (tu se traži `Containerfile`)
- *(napomena: isti `-t` na `podman run` znači pseudo-TTY, NE tag — ovdje je build → tag)*

**Što se stvarno dogodilo na VM-u (provjereno):**

`STEP 1/2` (FROM) prođe, pa `STEP 2/2` (RUN) padne:
```
STEP 2/2: RUN apt-get install -y nginx
Reading package lists...
E: Unable to locate package nginx
Error: building at STEP "RUN apt-get install -y nginx": ... exit status 100
```

### Greška i uzrok — prazan indeks paketa; `install` ne zna gdje je nginx

- Debian bazna slika dolazi s **praznim** `/var/lib/apt/lists` (indeks paketa) da slika bude manja.
- `apt-get install` čita taj indeks da nađe odakle povući paket. Prazan indeks → **„Unable to locate package nginx"**.
- `exit status 100` = apt-ov standardni kod za grešku → cijeli build pukne na tom `RUN` koraku.

### Ispravak — `apt-get update` PRIJE installa, u ISTOM `RUN`

```dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y nginx
```
Build sad prođe `STEP 2/2` i sagradi sliku. **(provjereno uživo ✓)**
```
REPOSITORY         TAG     IMAGE ID      SIZE
localhost/z10test  latest  4c986f9a4654  159 MB
```
(`localhost/` prefiks = lokalno sagrađena slika.)

### Zašto (srž za ispit) — zašto `&&` i zašto isti sloj

- `apt-get update` napuni indeks → tek onda `install` zna odakle povući nginx.
- **`&&`** = pokreće `install` SAMO ako `update` uspije (lančanje uspjeha).
- **Isti `RUN` (jedan sloj):** da su odvojeni (`RUN update` pa `RUN install`), Docker bi cacheirao sloj s `update`. Kasniji build bi preskočio `update` (cache hit) i koristio **zastarjeli** indeks → opet „Unable to locate" ili stara verzija. Zato par `update && install` UVIJEK ide u istom `RUN`.

### Gotče

- klasična greška: zaboraviti `update` → „Unable to locate package".
- ne razdvajaj `update` i `install` u dva `RUN`-a (cache zamka gore).
- u istom `RUN`-u se obično doda i čišćenje cachea radi manje slike (vidi zad. 16): `&& rm -rf /var/lib/apt/lists/*`.
- typo u putanji: `cd ~/devops/lo5/10` umjesto `z10` → `No such file`; valja paziti na prefiks foldera.

---

## Zadatak 11 — `CMD` shell forma vs exec forma (signali / čisto gašenje)

> Tip „pročitaj Dockerfile i uoči grešku" — nema komandi za izvođenje, rješava se iz teksta.

**Pokvaren Containerfile:**
```dockerfile
FROM python:3.11
COPY app.py /app/app.py
WORKDIR /app
CMD python app.py
```

**Razlaganje Containerfilea:**
- `FROM python:3.11` — baza s Pythonom 3.11
- `COPY app.py /app/app.py` — kopira `app.py` s build contexta u sliku na `/app/app.py`
- `WORKDIR /app` — postavi radni direktorij na `/app` (vrijedi i kao `cd` za daljnje naredbe)
- `CMD python app.py` — default naredba pri pokretanju ← **OVDJE greška: shell forma**

### Greška i uzrok — shell forma → PID 1 je `sh`, signali se gube

- `CMD python app.py` je **shell forma** → Docker je interno pretvori u `/bin/sh -c "python app.py"`.
- Posljedica: **PID 1 je `sh`**, a `python` mu je dijete.
- Kad Podman šalje `SIGTERM` (`stop`, `--replace`), signal ode `sh`-u, a **`sh` ga NE prosljeđuje** djetetu → `python` ga nikad ne dobije → ne može se uredno ugasiti → istekne grace period pa `SIGKILL`.
- To je isti `SIGTERM failed → SIGKILL` mehanizam viđen u zad. 4 i 7 (PID 1 ne dobiva default obradu signala), ovdje uzrokovan baš shell formom.

### Ispravak — exec forma (JSON niz)

```dockerfile
CMD ["python", "app.py"]
```
- `["python", "app.py"]` = **exec forma** → Docker pokreće `python` IZRAVNO, bez `sh` omotača
- sad je **`python` PID 1** → prima signale izravno → uredno gašenje (`SIGTERM` stigne do njega)

### Zašto (srž za ispit)

Shell forma (`CMD naredba`) ubacuje `/bin/sh -c` između init sustava i samog procesa. Taj `sh` postaje PID 1 i ne prosljeđuje signale, pa se aplikacija ne gasi čisto (gubi se i propagacija exit koda). Exec forma (`CMD ["…","…"]`) pokreće binar izravno kao PID 1 → ispravno rukovanje signalima. Pravilo: **`CMD`/`ENTRYPOINT` u exec formi** kad god je potrebno čisto gašenje.

**Sažeto:**
- shell forma: `CMD python app.py` → PID 1 = `sh`, python dijete (signali se gube)
- exec forma: `CMD ["python","app.py"]` → PID 1 = `python` (signali rade)

### Gotče

- shell forma IMA jednu prednost: radi shell-proširenja (`$VAR`, `&&`, `|`). Ako je to potrebno, koristi se svjesno: `CMD ["sh","-c","python app.py"]` (eksplicitan sh) ili dodaj `--init` da PID 1 bude pravi init koji prosljeđuje signale.
- isto vrijedi i za `ENTRYPOINT` — i njega drži u exec formi.

---

## Zadatak 12 — redoslijed slojeva za cache (`COPY` manifest prije koda)

> Tip „pročitaj Dockerfile" — rješava se iz teksta, bez gradnje.

**Pokvaren Containerfile:**
```dockerfile
FROM node:20
COPY . /app
WORKDIR /app
RUN npm install
```

**Razlaganje Containerfilea:**
- `FROM node:20` — baza s Node.js 20
- `COPY . /app` — kopira **cijeli** build context (sav kod) u sliku na `/app` ← **OVDJE greška: prerano kopira sve**
- `WORKDIR /app` — radni direktorij `/app`
- `RUN npm install` — instaliraj ovisnosti (čita `package.json` u `/app`)

### Podsjetnik — zašto uopće `COPY` i što je build context

- Slika se gradi **izolirano** i NE vidi host-disk. Datoteke s hosta (`app.py`, `package.json`) postoje u slici tek kad ih se unese.
- `podman build -t ime .` → točka `.` = **build context** (folder spakiran i poslan builderu; „vreća datoteka" koju build smije koristiti).
- `COPY izvor odredište` = izvadi datoteku **iz contexta** i stavi je **u sliku**. Lanac: izvorni folder → (`.`) context → (`COPY`) u sliku.
- Bez `COPY`-a `RUN npm install` nema `package.json` → nema što instalirati. Zato uopće postoji pitanje „što i kojim redom kopirati".

### Greška i uzrok — `COPY . /app` ruši cache na svaku izmjenu koda

- Docker cacheira svaki sloj; sloj se poništi čim mu se promijeni ulaz.
- **Pravilo: kad se sloj poništi, poništavaju se i SVI slojevi ispod njega.**
- `COPY . /app` u jedan sloj ubacuje **sav** kod. Promijeni li se jedan redak bilo koje `.js` datoteke → taj `COPY` sloj se poništi → **i `RUN npm install` ispod njega** → ovisnosti se reinstaliraju, iako se `package.json` nije ni dotaknuo. Spor build.

### Ispravak — manifest u svoj sloj, instalacija, pa ostatak koda

```dockerfile
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
```

**Zašto DVA `COPY`-a (srž pitanja):** odvajaju dvije stvari koje se mijenjaju različitom brzinom u zasebne slojeve.
- `COPY package*.json ./` — **SLOJ 1**: samo `package.json` + `package-lock.json` (mijenja se rijetko)
- `RUN npm install` — **SLOJ 2**: ovisi SAMO o sloju 1
- `COPY . .` — **SLOJ 3**: sav kod (mijenja se stalno), tek POSLIJE instalacije

Što Docker sad vidi:
- mijenja li se **kod** → mijenja se samo sloj 3; sloj 1 netaknut → `npm install` (sloj 2) ostaje **cacheiran** ✓ (brz build)
- mijenja li se **`package.json`** → sloj 1 se mijenja → poništava se i sloj 2 (`npm install` se vrti) i sloj 3. I treba — ovisnosti su se stvarno promijenile.

Da je samo JEDAN `COPY . .` (manifest + kod u istom sloju), Docker ne bi razlikovao „promijenio se kod" od „promijenile se ovisnosti" → svaka izmjena ruši `npm install`. Vraćamo se na pokvareno.

### Zašto (srž za ispit)

Slaži slojeve **od najmanje promjenjivog prema najpromjenjivijem**. Iznad skupog `RUN`-a (instalacija ovisnosti) smije biti samo ono što se rijetko mijenja (manifest); promjenjivi dio (kod) ide ISPOD njega. Tako `RUN` ostaje cacheiran dok god se manifest ne mijenja. Univerzalno vrijedi (Node `package.json`, Python `requirements.txt`, Maven `pom.xml`…).

### Gotče

- prvi `COPY` postoji JEDINO da manifest dobije vlastiti sloj iznad `RUN install`.
- `.containerignore` (ekvivalent `.dockerignore`) dodatno smanji context i izbjegne nepotrebna poništavanja (npr. izostave se `node_modules`, `.git`).
- isti princip = razlog zašto par `update && install` ide zajedno (zad. 10) — sve se vrti oko toga kada se sloj poništi.

---

## Zadatak 13 — `ENV PATH=/app/bin` prebriše sistemski PATH

> Tip „pročitaj Dockerfile" — rješava se iz teksta, bez gradnje.

**Pokvaren Containerfile:**
```dockerfile
FROM alpine:3.20
ENV PATH=/app/bin
RUN apk add --no-cache curl
```

**Razlaganje Containerfilea:**
- `FROM alpine:3.20` — baza Alpine 3.20
- `ENV PATH=/app/bin` — postavi varijablu okruženja `PATH` na `/app/bin` ← **OVDJE greška: PREBRIŠE cijeli PATH**
- `RUN apk add --no-cache curl` — instaliraj `curl`; `--no-cache` = ne čuvaj apk indeks (manja slika)

### Što je `PATH`

Popis foldera (odvojenih `:`) u kojima shell traži izvršne programe. Normalno otprilike: `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`. Upišeš `curl` → shell prolazi te foldere dok ga ne nađe (`curl` je u `/usr/bin`, `apk` u `/sbin`).

### Greška i uzrok — `ENV PATH=...` ZAMIJENI, ne dodaje

- `ENV PATH=/app/bin` ne **dodaje** `/app/bin` — nego **zamijeni cijeli PATH** tom jednom stavkom. PATH sad sadrži SAMO `/app/bin`.
- Posljedica: već `RUN apk add ... curl` padne jer se **`apk` ne nađe** (`apk` je u `/sbin`, kojeg više nema na PATH-u) → „apk: not found".
- I da prođe, kasnije se ni `curl`/`sh` ne bi našli iz istog razloga.

### Ispravak — PROŠIRI PATH (referenciraj stari `$PATH`)

```dockerfile
FROM alpine:3.20
ENV PATH=/app/bin:$PATH
RUN apk add --no-cache curl
```
- `/app/bin:$PATH` = nova stavka `/app/bin`, pa `:`, pa **stari sadržaj** (`$PATH` se proširi u zatečenu vrijednost)
- `/app/bin` dobije prioritet (prvi je), ali svi sistemski folderi ostaju → `apk`, `curl`, `sh` se nalaze

### Zašto (srž za ispit) — kad uključiti `$STARO`, a kad ne

Razlika ovisi o tome **nadograđuješ li popis ili postavljaš fiksnu vrijednost**:
- **Nadograđuješ popis** (PATH, `LD_LIBRARY_PATH`, `PYTHONPATH`…) → UVIJEK uključi staro: `ENV PATH=/app/bin:$PATH`. Inače prepišeš i pobrišeš zatečeno.
- **Postavljaš novu/fiksnu vrijednost** koja nema korisno staro stanje → samo upiši: `ENV NODE_ENV=production`, `ENV APP_PORT=8080`. Nema što čuvati.

Kontrolni test: *„ima li ova varijabla već neku korisnu vrijednost koju bih izgubio?"* PATH ima (sistemski folderi) → čuvaj (`:$PATH`); `NODE_ENV` nema → slobodno prepiši.

(Napomena: ovo NIJE pravilo iz LO1 — u terminalu je PATH već postavljen od shella pa ga se ne dira. Tek kad se u Containerfileu sam postavi `ENV PATH`, na korisniku je da ne pregazi ono što je baza složila.)

**Sažeto:**
- `ENV PATH=/app/bin` → PATH = samo `/app/bin` (sistemski alati nestanu)
- `ENV PATH=/app/bin:$PATH` → PATH = `/app/bin` + sve od prije (ispravno)

### Gotče

- isti zid vrijedi za bilo koju popisnu varijablu — izostavi li se `$STARO` → varijabla je zbrisana.
- redoslijed u PATH-u određuje prioritet: što je lijevo, traži se prvo (`/app/bin:$PATH` daje prednost vlastitim binarima).

---

## Zadatak 14 + 15 — `EXPOSE` ne objavljuje port (samo dokumentira) + nesklad portova

> Tip „pročitaj Dockerfile" — rješava se iz teksta. U PDF-u su 14 (Containerfile) i 15 (pitanje) jedan primjer.

**Containerfile (zad. 14):**
```dockerfile
FROM ubuntu:24.04
EXPOSE 8080
CMD ["python3", "-m", "http.server", "3000"]
```
**Pitanje (zad. 15):** mapira se `-p 8080:8080` ali ništa ne odgovara — koji je nesklad, i što `EXPOSE` zapravo radi?

**Razlaganje Containerfilea:**
- `FROM ubuntu:24.04` — baza Ubuntu 24.04
- `EXPOSE 8080` — **dokumentira** da kontejner „koristi" port 8080 ← **tu je nesklad**
- `CMD ["python3", "-m", "http.server", "3000"]` — exec forma; pokreće Python HTTP server
  - `-m http.server` = pokreće ugrađeni modul `http.server`
  - `3000` = port na kojem server STVARNO sluša

### Greška i uzrok — dvije stvari

**1) Nesklad portova:** `EXPOSE 8080`, ali server sluša na **3000**. Mapira se `-p 8080:8080` → host gađa kontejnerski port 8080, a tamo nitko ne sluša (server je na 3000) → prazno. (Isti tip greške kao zad. 1, samo unutar slike.)

**2) Što `EXPOSE` zapravo radi:** **ništa funkcionalno — samo dokumentira.** To je metapodatak u slici koji *označava* namjeravani port. NE otvara port, NE mapira ništa na host. Stvarno objavljivanje radi jedino **`-p` na `podman run`**. Može se mapirati i port koji nije u `EXPOSE`, i obrnuto — `EXPOSE` bez `-p` ne čini ništa.

### Ispravak — uskladi portove + stvarno mapiraj

```dockerfile
FROM ubuntu:24.04
EXPOSE 3000
CMD ["python3", "-m", "http.server", "3000"]
```
```bash
podman run -d -p 8080:3000 ime    # host 8080 → kontejner 3000 (gdje server sluša)
```

### Zašto (srž za ispit)

- desni broj u `-p` mora biti port na kojem proces STVARNO sluša → ovdje **3000**.
- `EXPOSE` uskladi na 3000 radi **dokumentacije** (da slika pošteno objavi svoj port), ali to je kozmetika — funkcionalno radi tek `-p 8080:3000`.
- ključ: **`EXPOSE` = dokumentacija/namjera; `-p` = stvarno objavljivanje porta.**

### Gotče

- `EXPOSE` ipak ima koristi: `podman run -P` (veliko P) automatski mapira SVE `EXPOSE` portove na nasumične host portove; i alati (compose, orkestratori) ga čitaju kao nagovještaj. Ali bez `-p`/`-P` sam po sebi ne otvara ništa.
- klasična zamka: uskladi se `EXPOSE` ali se zaboravi da to ništa ne mijenja dok se ne doda `-p`.

---

## Zadatak 16 — smanji veličinu slike: čišćenje cachea u ISTOM sloju

**Pokvaren Containerfile:**
```dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y build-essential
```
Pitanje: slika je ogromna — kako smanjiti veličinu u istom sloju?

**Razlaganje Containerfilea:**
- `FROM debian:12` — baza Debian 12
- `RUN apt-get update && apt-get install -y build-essential` — osvježi indeks pa instaliraj `build-essential` (gcc, make, headeri… velik paket)

### Greška i uzrok — smeće ostaje u sloju

Dvije stvari nepotrebno bubre sliku:
1. **apt indeks** koji `update` skine u `/var/lib/apt/lists/` (ne treba nakon instalacije)
2. preporučeni (nenužni) paketi koje `install` povuče

Ključno je **gdje** brišeš: svaki `RUN` = JEDAN sloj, i sloj pamti SVE što je u njemu nastalo. Obrišeš li indeks u DRUGOM `RUN`-u, raniji sloj ga i dalje nosi — brisanje u kasnijem sloju samo ga „sakrije", ne smanji raniji sloj.

### Ispravak — počisti u ISTOM `RUN`-u

```dockerfile
FROM debian:12
RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*
```
- `&& rm -rf /var/lib/apt/lists/*` — obriši apt indeks **u istom sloju** → datoteke nikad trajno ne uđu u sloj
- `--no-install-recommends` — ne povlači preporučene (nenužne) pakete → dodatno manja slika
- `\` na kraju retka = nastavi naredbu u sljedeći redak (čitljivije, ali i dalje JEDAN `RUN` = jedan sloj)

### Mjereno na VM-u (provjereno uživo)

```
localhost/z16mrsava   latest   446 MB
localhost/z16debela   latest   546 MB
```
**100 MB uštede** (546 → 446, ~18%) samo čišćenjem u istom sloju + `--no-install-recommends`. Kod lakših paketa je postotak veći.

### Zašto (srž za ispit) — zašto baš isti sloj

- slojevi su **kumulativni i nepromjenjivi** — kad nešto uđe u sloj, ostaje u toj slici zauvijek.
- `RUN install` (sloj A) pa `RUN rm` (sloj B): sloj A i dalje sadrži indeks (veći je), sloj B samo dodaje „oznaku brisanja" → ukupna slika NIJE manja.
- `RUN install && rm` (jedan sloj): indeks se stvori i obriše UNUTAR istog sloja → u finalni sloj uđe samo ono što ostane → slika je stvarno manja.

**Sažeto:**
- odvojeni `RUN`-ovi za install i cleanup → cleanup beskoristan (raniji sloj nosi smeće)
- jedan `RUN` s `&& rm -rf …` → smeće nikad ne ostane u sloju

### Gotče

- `-f Containerfile.ime` na buildu = gradi iz tog imena datoteke (kad imaš više Containerfilea u istoj mapi).
- isti princip = razlog zašto `update && install` ide zajedno (zad. 10): sve ovisi o tome što ostaje u sloju.
- za još manju sliku → multi-stage (zad. 18): tamo finalna slika uopće ne nosi build alate.

---

## Zadatak 17 — `USER` prije `COPY`/`RUN` (permission denied)

> Tip „pročitaj Dockerfile" — rješava se iz teksta.

**Pokvaren Containerfile:**
```dockerfile
FROM python:3.11
USER appuser
COPY requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
```
Pitanje: permission denied — što je krivo s redoslijedom `USER`/`COPY` i vlasništvom?

**Razlaganje Containerfilea:**
- `FROM python:3.11` — baza s Pythonom
- `USER appuser` — od sad nadalje svi koraci rade kao `appuser` (ne root) ← **OVDJE greška: prerano**
- `COPY requirements.txt /app/requirements.txt` — kopira datoteku u sliku
- `RUN pip install -r ...` — instaliraj pakete iz requirements

### Greška i uzrok — prerani prelazak na neprivilegiranog korisnika

Prebaciš se na `appuser` prerano, pa koraci poslije nemaju root prava:
- `COPY` stvara `/app/...` — `/app` možda ne postoji, a `appuser` nema pravo stvarati/pisati u njega → **permission denied**
- `RUN pip install` — piše u sistemske `site-packages` (vlasništvo roota) → **permission denied**
- i da `COPY` prođe, datoteka bi mogla dobiti krivo vlasništvo za `appuser`.

### Ispravak — privilegirani koraci kao root; `USER` na kraju, tik prije pokretanja

```dockerfile
FROM python:3.11
WORKDIR /app
COPY requirements.txt ./
RUN pip install -r requirements.txt
RUN useradd -m appuser && chown -R appuser /app
USER appuser
CMD ["python", "app.py"]
```
- `WORKDIR /app` — stvara i ulazi u `/app` (još kao root → smije)
- `COPY` + `RUN pip install` — instalacija dok si **root** (ima prava na `/app` i sistemske pakete)
- `RUN useradd -m appuser && chown -R appuser /app` — napravi korisnika, daj mu vlasništvo nad `/app`
- `USER appuser` — prebaci se na neprivilegiranog TEK SADA
- `CMD [...]` — aplikacija se IZVODI kao `appuser` (sigurnost u runtime-u)

### Zašto (srž za ispit)

Build koraci koji traže prava (pisanje u sistemske foldere, instalacija) rade kao **root**. `USER` na neprivilegiranog stavi se **što kasnije** — idealno tik prije `CMD`. Dobiješ oboje: build prolazi (root prava tijekom gradnje), a kontejner se *vrti* kao ne-root (sigurnije u radu). Sigurnost se tiče RUNTIME-a, ne build-a.

Ako baš `COPY` mora postaviti vlasništvo odmah: `COPY --chown=appuser:appuser src dst` — ali korisnik mora postojati prije toga.

**Sažeto:**
- `USER` prerano → `COPY`/`pip install` nemaju prava → permission denied
- `USER` tik prije `CMD` → privilegirani build kao root, ne-root tek u runtime-u

### Gotče

- `USER` vrijedi za sve korake ISPOD sebe (i za runtime), dok ga sljedeći `USER` ne promijeni.
- za root korak usred „ne-root" dijela može se privremeno vratiti (`USER root` pa natrag), ali bolje je sve privilegirano odraditi gore.

---

## Zadatak 18 — multi-stage build (finalna slika nosi samo binar)

**Pokvaren Containerfile:**
```dockerfile
FROM golang:1.22
COPY . /src
WORKDIR /src
RUN go build -o app .
CMD ["/src/app"]
```
Pitanje: finalna slika je ~1 GB — kako bi multi-stage to popravio?

**Razlaganje Containerfilea:**
- `FROM golang:1.22` — baza s cijelim Go toolchainom (kompajler, biblioteke) — velika
- `COPY . /src` + `WORKDIR /src` + `RUN go build -o app .` — sagradi binar `app`
- `CMD ["/src/app"]` — pokreće binar

### Greška i uzrok — finalna slika nosi cijeli toolchain

Za **pokretanje** Go programa treba samo gotov (statički) binar. Ali ova slika u sebi nosi **cijeli golang toolchain + izvorni kod** (~876 MB izmjereno), iako ništa od toga ne treba u runtime-u.

### Ispravak — multi-stage: gradnja u jednoj fazi, kopiranje samo binara u minimalnu

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o app .

FROM gcr.io/distroless/static-debian12
COPY --from=build /src/app /app
CMD ["/app"]
```

**Razlaganje (ključ):**
- `FROM golang:1.22 AS build` — **prva faza**, imenovana `build` (ima Go toolchain)
- `COPY . .` + `RUN go build -o app .` — sagradi binar `app` unutar te faze
- `FROM gcr.io/distroless/static-debian12` — **druga faza** kreće od NULE: minimalna slika (samo runtime, bez shella, bez apt-a)
- `COPY --from=build /src/app /app` — prekopira **samo binar** iz faze `build` u finalnu sliku
- `CMD ["/app"]` — pokreće taj binar
- **Sve iz prve faze (kompajler, izvor) NE ide u finalnu sliku** — samo ono što izričito `COPY --from`-aš

### Mjereno na VM-u (provjereno uživo)

```
localhost/z18multi    5.05 MB
localhost/z18debela   876 MB
```
**876 MB → 5 MB — slika ~174× manja.** U build ispisu se vide dvije faze: `[1/2]` STEP-ovi = faza `build` (golang), `[2/2]` STEP-ovi = finalna faza (distroless + `COPY --from`).

### Zašto (srž za ispit)

Build-time alati (kompajler, dev biblioteke, izvorni kod) i run-time potrebe su različite stvari. Multi-stage ih razdvaja: gradiš u „debeloj" fazi s alatima, a u finalnu „mršavu" sliku preneseš samo **artefakt** (binar). Time finalna slika nosi minimum → manja, brži pull, **manja površina za napad** (nema shella/kompajlera koje bi napadač iskoristio).

### Gotče

- `AS <ime>` imenuje fazu; `COPY --from=<ime>` vadi iz nje. Može i `COPY --from=0` (po indeksu faze).
- za Go treba **statički** binar (distroless `static`); ako binar ovisi o C-bibliotekama, koristi `distroless/base` ili `alpine` + odgovarajuće lib-ove.
- ako `gcr.io` nije dostupan, finalna faza može biti `alpine:3.20` — i dalje sitno naspram golang slike (ali veće od distrolessa).
- isti princip vrijedi za Java (build s Maven/JDK → run na JRE/distroless), Node (build → kopira se `dist/` + `node_modules`), itd.

---

# Skupina C — k8s dijagnostika (zad. 19–43)

> Klaster mora raditi: `minikube status` → ako Stopped/zatrovan, `minikube start --driver=podman` (ili `minikube delete --all` + `export MINIKUBE_HOME=$HOME/.minikube` + start). `kubectl get nodes` → `Ready`.
> Glavni alat dijagnostike: **`kubectl describe <resurs>`** (Events na dnu obično kažu uzrok) + **`kubectl logs`** (+`--previous`) + **`kubectl get events --sort-by=.lastTimestamp`**.

## Zadatak 19 — Pod zaglavljen u `Pending` (Insufficient memory)

**Kvar (manifest koji traži više nego čvor ima):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "100Gi"
```

**Razlaganje manifesta:**
- `apiVersion: v1` / `kind: Pod` — goli Pod (bez kontrolera)
- `metadata.name` — ime Poda
- `spec.containers` — lista kontejnera (množina!); `name`, `image`
- `resources.requests.memory: "100Gi"` — Pod GARANTIRANO traži 100 GB ← namjerni kvar

> `requests` = koliko resursa Pod garantirano treba; scheduler traži čvor koji to može dati.

**Dijagnoza (provjereno uživo):**
```bash
kubectl apply -f pending.yaml
kubectl get pod pending-pod          # STATUS: Pending (0/1)
kubectl describe pod pending-pod     # Events na dnu:
```
```
Warning  FailedScheduling  default-scheduler
0/1 nodes are available: 1 Insufficient memory.
preemption: ... No preemption victims found for incoming pod.
```

### Čitanje ispisa (za ispit)

- `FailedScheduling` od `default-scheduler` → scheduler nije mogao smjestiti Pod
- `0/1 nodes are available` → od 1 čvora, 0 prihvatljivih
- `Insufficient memory` → traženih 100 GB nijedan čvor ne može dati
- `No preemption victims` → ne može ni izbaciti drugi Pod da oslobodi mjesto

**Pending = scheduler ne nalazi mjesto.** Pod je validan i prihvaćen, ali nijedan čvor ne zadovoljava zahtjeve.

### Ispravak — tražiti realnu količinu (ili ukloniti `requests`)

```yaml
    resources:
      requests:
        memory: "64Mi"
```
```bash
kubectl delete pod pending-pod       # goli Pod je immutable za resources → delete+create
kubectl apply -f pending.yaml
kubectl get pod pending-pod          # ContainerCreating → Running (1/1)
```
**(provjereno uživo ✓)** — prijelaz `Pending → ContainerCreating → Running` potvrđuje da je scheduler odmah smjestio Pod čim su zahtjevi pali na realnih 64Mi.

### Zašto (srž za ispit) — tri tipična uzroka Pendinga

1. **Nedovoljno resursa** (ovaj slučaj) — `requests` veći od kapaciteta čvora → `Insufficient cpu/memory`.
2. **`nodeSelector`/affinity/taint** bez odgovarajućeg čvora → `node(s) didn't match node selector` (LO4 zad. 14).
3. **Nevezan PVC** — Pod čeka volumen koji ne postoji (zad. 28).

Dijagnoza je uvijek ista: `kubectl describe pod <ime>` → čitaj `Events:` na dnu.

### Gotče

- goli Pod je **immutable** za `resources`/`image` → `apply` izmjene padne; treba `delete` + `apply`. (Deployment bi sam zamijenio Pod preko template-a — LO4 zad. 15.)
- `requests` (garancija za scheduling) ≠ `limits` (tvrda gornja granica u runtime-u; proboj memorije → OOMKilled, zad. 23).
- alternativa `describe`-u: `kubectl get events --sort-by=.lastTimestamp` (kronološki).

---

## Zadatak 20 — YAML uvlaka/typo (manifest se ne parsira)

**Kvar (loša uvlaka `image:`):**
```yaml
    spec:
      containers:
      - name: nginx
       image: nginx:1.25      # ← uvučen za 1 razmak previše (nije poravnat s name)
```

**Dijagnostički alat — `cat -An <file>`:**
- `cat` — ispiši datoteku
- `-n` — numeriraj retke
- `-A` — prikaži NEVIDLJIVE znakove: tabovi kao `^I`, kraj retka kao `$`
- → tako su vidljivi razmaci/tabovi koji se inače ne primjećuju (YAML ne dopušta tabove i osjetljiv je na uvlake)

`cat -An broken.yaml` je pokazao (provjereno uživo):
```
16   - name: nginx$
17    image: nginx:1.25$
```
`- name:` na jednoj razini, `image:` uvučen dublje — moraju biti na ISTOJ razini (oba su polja istog stavka liste). `$` potvrđuje da nema skrivenih znakova → problem je čista uvlaka.

**Greška pri `apply` (provjereno uživo):**
```
error: error parsing broken.yaml: error converting YAML to JSON: yaml: line 16: did not find expected key
```

### Čitanje ispisa (za ispit)

- `error converting YAML to JSON` → kubectl prvo parsira YAML→JSON; pukne **prije** slanja serveru (čisto lokalna greška)
- `line 16` → parser zapne tu (gleda se taj redak i okolo)
- `did not find expected key` → zbog uvlake parser je očekivao ključ na toj razini, a nije ga našao

### Ispravak — poravnaj `image:` s `name:`

```yaml
      containers:
      - name: nginx
        image: nginx:1.25      # isti broj razmaka kao name (poravnato ispod "- ")
```
```bash
kubectl apply -f broken.yaml --dry-run=client   # provjera: "created (dry run)" = YAML sad valja
kubectl apply -f broken.yaml                     # pravi apply: deployment.apps/web created
kubectl get deployment web                       # READY 2/2
kubectl get pods -l app=web                      # 2× web-... Running 1/1
```
**(provjereno uživo ✓)**

### Zašto (srž za ispit)

YAML strukturu nosi ISKLJUČIVO uvlaka (razmaci). Polja istog objekta moraju imati isti broj razmaka; jedan višak/manjak razmaka mijenja (ili razbija) strukturu. kubectl parsira YAML→JSON prije slanja, pa se takve greške vide lokalno kao `error converting YAML to JSON` s brojem retka. Alat za lov: `cat -An` (vidi razmake/tabove) + `--dry-run=client` (parsiraj bez stvaranja).

### Gotče

- **NIKAD tabovi u YAML-u** — samo razmaci. `cat -An` pokaže tab kao `^I`.
- `--dry-run=client` = parsira i validira lokalno, NE šalje serveru (`(dry run)` u ispisu). `--dry-run=server` šalje serveru na validaciju ali ne snima (stroža provjera — zad. 40).
- broj retka u grešci je orijentir, ne uvijek točan redak — problem zna biti redak iznad/ispod.
- predigra za zad. 40 (`--dry-run=server` / `--validate=true` za hvatanje typo-a polja prije applya).

---

## Zadatak 21 — ImagePullBackOff (slika/tag ne postoji)

**Kvar (nepostojeći tag):**
```yaml
      containers:
      - name: app
        image: nginx:nepostojeci-tag      # ← tag ne postoji u registru
```

> ⚠️ Usput nas je dočekao i **selector mismatch** (ostatak kopiranja: `matchLabels: app: badimg` ali `template.labels: app: web`) → server odbija: `selector does not match template labels`. To je sadržaj zad. 26 — ovdje samo poravnaj labele (`app: badimg` na oba mjesta) da nastavimo s ImagePullBackOffom.

**Dijagnoza (provjereno uživo):**
```bash
kubectl apply -f badimage.yaml
kubectl get pods -l app=badimg       # 0/1 ImagePullBackOff
kubectl describe pod -l app=badimg   # -l = izaberi Pod po labeli (bez nasumičnog sufiksa)
```
Events:
```
Normal   Scheduled  default-scheduler  Successfully assigned default/badimg-... to minikube
Normal   Pulling    kubelet            Pulling image "nginx:nepostojeci-tag"
Warning  Failed     kubelet            Failed to pull image ...: manifest for nginx:nepostojeci-tag not found: manifest unknown
Warning  Failed     kubelet            Error: ErrImagePull
Warning  Failed     kubelet            Error: ImagePullBackOff
Normal   BackOff    kubelet            Back-off pulling image "nginx:nepostojeci-tag"  (x20 over 5m)
```

### Čitanje ispisa (za ispit)

- `Scheduled ... assigned ... to minikube` → Pod je UREDNO raspoređen → problem NIJE scheduling (razlika od Pendinga, zad. 19)
- `Pulling` → kubelet pokušava povući sliku
- `manifest ... not found: manifest unknown` → **korijenski uzrok**: registar nema taj tag/sliku
- `ErrImagePull` → prvi neuspjeli pokušaj; `ImagePullBackOff` → prelazak u backoff
- `BackOff ... (x20 over 5m)` → kubelet ponavlja sve rjeđe da ne davi registar

**ImagePullBackOff = problem POVLAČENJA slike, ne scheduling-a.** `Scheduled` na vrhu to potvrdi.

### Ispravak — ispravan tag

```bash
kubectl set image deployment/badimg app=nginx:1.25
kubectl get pods -l app=badimg       # novi Pod: Running 1/1
```
**(provjereno uživo ✓)** — novi Pod ima NOVI pod-template-hash (rolling update; promjena slike → novi RS, LO4 zad. 4), stari (Backoff) se ukloni.

**Razlaganje `kubectl set image`:**
- `set image` — imperativno promijeni sliku kontejnera (pokrene rolling update)
- `deployment/badimg` — na kojem objektu
- `app=nginx:1.25` — **`ime_kontejnera=nova_slika`** (kontejner `app`, ispravan tag `1.25`)

### Zašto (srž za ispit) — tri tipična uzroka ImagePullBackOff

1. **Krivi tag** (ovaj) → `manifest unknown` / `not found`.
2. **Typo u imenu/registru** → `repository does not exist` ili sličan „not found".
3. **Privatni registar bez `imagePullSecret`** → `unauthorized` / `authentication required` (LO4 zad. 36 — `imagePullSecrets`).

Dijagnoza ista: `kubectl describe pod` → `Events:` → pročitaj točnu poruku iza „Failed to pull".

### Gotče

- `ErrImagePull` (trenutno) vs `ImagePullBackOff` (nakon ponovljenih neuspjeha + backoff) — isti korijen, druga faza.
- `Scheduled` u Events odmah razlučuje ImagePullBackOff (slika) od Pending (scheduling).
- `-l app=badimg` u `describe`/`get` = rad po labeli, ne treba znati nasumični sufiks Poda.

---

## Zadatak 22 — CrashLoopBackOff (`kubectl logs --previous`)

**Kvar (naredba koja namjerno padne):**
```yaml
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "echo pokrecem se; sleep 2; echo padam sad; exit 1"]
```

**Razlaganje (ključni redak):**
- `image: busybox:1.36` — slika POSTOJI (povuče se bez problema) → ovo NIJE ImagePullBackOff
- `command: ["sh","-c","..."]` — exec forma; shell pokrene niz: `echo pokrecem se` → `sleep 2` → `echo padam sad; exit 1` ← **exit 1 = namjerni pad** (≠0 = greška)

**Dijagnoza (provjereno uživo) — prijelaz kroz faze:**
```
crashy-... 0/1 Error            1 (7s ago)   9s
crashy-... 0/1 CrashLoopBackOff 3 (29s ago)  83s
```
`RESTARTS` RASTE (1 → 3), STATUS prelazi `Error` → `CrashLoopBackOff` (k8s čeka sve dulje između restarta).

```bash
kubectl logs -l app=crashy --previous
```
```
pokrecem se
padam sad
```

### Čitanje ispisa (za ispit)

- **`RESTARTS` raste** = potpis CrashLoopBackOffa. (Kod ImagePullBackOffa je 0 — kontejner se NIKAD ne digne; ovdje se digne pa padne.)
- `CrashLoopBackOff` = kontejner se uporno ruši, k8s ga restarta sa sve dužim razmacima (backoff).
- `logs --previous` pokazuje log **PRETHODNE, srušene** instance → tu je poruka pada (`padam sad`).

**Zašto `--previous` (`-p`):** kontejner se upravo srušio i restarta, pa „trenutni" log zna biti prazan/nov. `--previous` čita zadnju srušenu instancu — bez njega se često uhvati novi pokušaj prije ičega.

### Ispravak — naredba koja ostane živa

```yaml
        command: ["sh", "-c", "echo radim; sleep infinity"]
```
```bash
gedit crash.yaml            # command se mijenja kroz MANIFEST (nema imperativni "set command")
kubectl apply -f crash.yaml
kubectl get pods -l app=crashy   # Running 1/1, RESTARTS prestaje rasti (ostaje 0)
```
**(provjereno uživo ✓)** — `... exit 1` → `sleep infinity`: proces ostaje živ → kontejner radi → nema petlje. (Isti princip kao zad. 4 — busybox treba dugotrajan proces.) Novi pod-template-hash jer se promijenio `command`.

> ⚠️ `kubectl set image` ovdje NE pomaže — problem nije slika nego `command`. `command`/`args` se mijenjaju kroz manifest (`apply`/`edit`), ne imperativnim `set image`.

### Zašto (srž za ispit)

CrashLoopBackOff = slika je OK i Pod je raspoređen, ali glavni proces stalno izlazi (kod ≠ 0, ili odmah završi). k8s ga po `restartPolicy: Always` (default kod Deploymenta) restarta, pa opet padne → petlja s rastućim backoffom. Korijenski uzrok se čita iz **`logs --previous`** (aplikacijska greška, kriva konfiguracija, fali env/datoteka, ili proces koji jednostavno završi).

### Gotče

- `RESTARTS` raste → CrashLoop; `RESTARTS 0` + `ImagePullBackOff`/`Pending` → kontejner se nije ni pokrenuo (drugi razred problema).
- `Error` (trenutni status pada) vs `CrashLoopBackOff` (nakon više padova + backoff) — isti korijen.
- ubuntu/busybox bez dugotrajne naredbe → `Completed`/CrashLoop jer proces odmah završi (zad. 29).
- za neuhvatljiv pad pri startu: `kubectl describe pod` (zadnji `State`/`Last State`, `Exit Code`) + `logs --previous`.

---

## Zadatak 23 — OOMKilled (premali `limits.memory`)

> k8s ekvivalent Podmanovog `Exited (137)` iz zad. 6 — isti SIGKILL mehanizam.

**Kvar (aplikacija traži više nego limit dopušta):**
```yaml
      containers:
      - name: app
        image: python:3.11-slim
        command: ["python", "-c", "x = bytearray(200*1024*1024); import time; time.sleep(600)"]
        resources:
          limits:
            memory: "32Mi"
```

**Razlaganje (ključni dijelovi):**
- `image: python:3.11-slim` — slika postoji (povuče se)
- `command` — Python koji `bytearray(200*1024*1024)` = alocira **200 MB**, pa `sleep(600)` (ostane živ 10 min)
- `resources.limits.memory: "32Mi"` — **tvrdi strop 32 MB** ← namjerni kvar (200 MB >> 32 MB)

> `limits` (≠ `requests` iz zad. 19) = tvrda gornja granica u runtime-u. Probije li je kontejner → jezgra ga UBIJE (OOM).

**Dijagnoza (provjereno uživo):**
```bash
kubectl apply -f oom.yaml
kubectl get pods -l app=oomy          # RESTARTS raste; OOMKilled/CrashLoopBackOff
kubectl describe pod <oomy-pod> | grep -A5 "Last State"
```
```
Last State:   Terminated
  Reason:     OOMKilled
  Exit Code:  137
  Started:    Tue, 23 Jun 2026 17:55:13 +0200
  Finished:   Tue, 23 Jun 2026 17:55:13 +0200
```

### Čitanje ispisa (za ispit)

- `Reason: OOMKilled` → jezgra ubila kontejner zbog probijanja memorijskog limita
- `Exit Code: 137` → **128 + 9 (SIGKILL)** — IDENTIČNO Podmanovom `Exited (137)` (zad. 6)
- `Started` = `Finished` (isti tren) → kontejner alocira 200 MB, probije 32 MB strop, ubijen ISTOG časa
- `grep -A5 "Last State"` → `-A5` = redak + 5 redaka iza (After), da se uhvati Reason + Exit Code

### Ispravak — digni `limits.memory` iznad stvarne potrebe

```yaml
        resources:
          limits:
            memory: "256Mi"      # 200 MB stane ispod 256 MB stropa
```
```bash
kubectl apply -f oom.yaml
kubectl get pods -l app=oomy          # Running 1/1, RESTARTS 0
```
**(provjereno uživo ✓)**

### Zašto (srž za ispit)

`limits.memory` je tvrdi cgroup strop koji jezgra provodi (kao `--memory` u Podmanu). Kad proces zatraži više, kernel OOM-killer ga ubije (SIGKILL → exit 137), a kubelet to upiše u `Last State: Terminated, Reason: OOMKilled` i restarta po `restartPolicy`. Rješenje: ili podigni limit na stvarne potrebe, ili popravi aplikaciju da troši manje. Razlika od običnog CrashLoopa: `Reason` je izričito `OOMKilled`, ne aplikacijski exit.

### Dva immutable pravila koja su nas dočekala (VAŽNO za ispit)

1. **Goli Pod: `resources` je immutable** → `apply` izmjene padne; treba `delete` + `apply` (zad. 19).
2. **Deployment: `spec.selector` je immutable** → mijenja li se selector postojećeg Deploymenta, `apply` javi `field is immutable`. Rješava se `kubectl delete deploy <ime>` pa ponovni `apply`. (Razlog: selector određuje koje Podove kontrolira; mijenjati ga usred rada bi osirotilo postojeće Podove.)

### Gotče

- `-l app=X` filtrira po labeli — ako je labela kriva (npr. ostala `app: crashy` od kopiranja), `get pods -l app=oomy` vrati „No resources found" iako Pod POSTOJI. Provjerava se `get pods` BEZ filtra, ili `cat manifest` da se vidi stvarna labela.
- ime labele nije bitno za rad — bitno je da se `selector.matchLabels` i `template.labels` **poklapaju** (inače validacijska greška, zad. 26).
- typo u imenu Poda (`...868-tsm29` vs `...868d-tsm29`) → `NotFound`; lakše je `-l <labela>` ili kopirati puno ime iz `get pods`.
- `describe pod` polja `Last State` + `Exit Code` su zlato za sve restart-uzroke (OOM vs aplikacijski pad).

---

## Zadatak 24 — Pod nikad ne postane Ready (readiness probe pada → `0/1`)

**Kvar (proba gađa krivi port):**
```yaml
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 8000          # ← nginx sluša na 80, proba gađa 8000
          initialDelaySeconds: 3
          periodSeconds: 5
```

**Razlaganje (ključni dio):**
- `image: nginx:1.25` + `containerPort: 80` — nginx radi i sluša na **80**
- `readinessProbe` — provjera „je li Pod spreman primati promet"
  - `httpGet: path: / port: 8000` — HTTP GET na **8000** ← **kvar: nitko ne sluša na 8000**
  - `initialDelaySeconds: 3` — čekaj 3 s prije prve provjere
  - `periodSeconds: 5` — ponavljaj svakih 5 s

> **readiness ≠ liveness:** readiness odlučuje smije li Pod primati promet (ulazi/izlazi iz Service **Endpoints**). Pad readinessa → Pod ostaje `0/1` i **ispada iz Endpointsa**, ali se NE restarta (restart bi radio liveness).

**Dijagnoza (provjereno uživo):**
```bash
kubectl get pods -l app=readyfail
kubectl describe pod -l app=readyfail | grep -A3 -i readiness
```
```
readyfail-...  0/1  Running  0  32s

Readiness:  http-get http://:8000/ delay=3s timeout=1s period=5s #success=1 #failure=3
Warning  Unhealthy  3s (x9 over 43s)  kubelet
Readiness probe failed: Get "http://10.244.0.45:8000/": ... connection refused
```

### Čitanje ispisa (za ispit)

- **`0/1` + `Running` + `RESTARTS 0`** = potpis readiness-fail: kontejner RADI (nije pao, nije restartan), ali nije ready.
- `http-get http://:8000/` → proba gađa 8000, nginx sluša na 80 → `connection refused`.
- `#failure=3` → nakon 3 neuspjeha proba je „failed"; `(x9 over 43s)` → broj promašaja raste.
- posljedica: Pod `0/1` → **izbačen iz Service Endpointsa** (Servis ga preskače pri load-balansingu), ali se NE restarta.

### Ispravak — uskladi probu na port gdje app sluša (80)

```yaml
        readinessProbe:
          httpGet:
            path: /
            port: 80
```
```bash
kubectl apply -f readiness.yaml      # obična izmjena template-a (ne dira selector)
kubectl get pods -l app=readyfail    # READY 1/1, Running
```
**(provjereno uživo ✓)** — prijelaz `0/1 → 1/1` čim proba na port 80 prolazi.

### Zašto (srž za ispit)

Readiness probe je k8s-ov način da pita „smije li ovaj Pod primati promet". Dok ne prođe, Pod je `0/1` i Servis ga **ne uvrštava u Endpoints** (promet ga zaobilazi). Za razliku od liveness probe (koja na pad **restarta** kontejner), readiness samo isključuje Pod iz prometa. Kvarovi: kriva `port`/`path`, prekratak `initialDelaySeconds` (app se još diže), ili app stvarno nije spreman. Dijagnoza: `describe pod` → `Readiness:` linija + `Unhealthy` event.

### Gotče

- **readiness** (pad → izbačen iz Endpoints, NE restarta) vs **liveness** (pad → restarta kontejner). Pomiješati ih je klasična greška.
- `0/1 Running RESTARTS 0` → readiness; `RESTARTS raste` → liveness/CrashLoop. Broj restarta ih razlikuje.
- prekratak `initialDelaySeconds` → proba krene prije nego se app digne → lažni padovi na startu (za spor start vidi startup probe, zad. 52 u LO4).
- `connection refused` (nitko ne sluša na portu) ≠ `404` (sluša, ali putanja kriva) — poruka kaže koji je od dva.

---

## Zadatak 25 — Servis vraća prazno (selector/label mismatch → Endpoints `<none>`)

**Kvar (Servisov selector ne odgovara labeli Poda):**
```yaml
# Deployment: Podovi imaju label app: websvc
---
apiVersion: v1
kind: Service
metadata:
  name: websvc
spec:
  selector:
    app: pogresna-labela      # ← nijedan Pod nema tu labelu (Podovi su app: websvc)
  ports:
  - port: 80
    targetPort: 80
```

**Razlaganje:**
- `---` razdvaja dva objekta u istoj datoteci (Deployment + Service)
- Service `selector: app: pogresna-labela` — bira Podove s tom labelom ← **kvar**
- `port: 80` — port na kojem Servis sluša; `targetPort: 80` — port na Podu kamo prosljeđuje

> Servis NE zna izravno za Podove. Preko svog **selectora** stalno traži Podove s odgovarajućom labelom i njihove IP:port upisuje u **Endpoints**. Kriva labela → prazan Endpoints → Servis nema kamo slati.

**Dijagnoza (provjereno uživo) — ključ je `get endpoints`:**
```bash
kubectl get pods -l app=websvc        # 2× websvc-... Running 1/1 (Podovi OK!)
kubectl get endpoints websvc          # ENDPOINTS: <none>  ← dokaz problema
```

### Čitanje ispisa (za ispit)

- Podovi su `Running` (s njima sve OK), ali `get endpoints` → **`<none>`** = Servis nema nijedan backend IP.
- Uzrok: Servisov `selector` (`app: pogresna-labela`) ≠ labela Poda (`app: websvc`) → Endpoints controller ne nalazi nijedan Pod.
- `<none>` u `get endpoints` = klasičan potpis **selector/label mismatcha** između Servisa i Podova.

### Ispravak — uskladi selector Servisa s labelom Poda

```bash
kubectl patch svc websvc -p '{"spec":{"selector":{"app":"websvc"}}}'
kubectl get endpoints websvc
```
Prije → poslije **(provjereno uživo ✓)**:
```
websvc   <none>
websvc   10.244.0.49:80,10.244.0.50:80
```
Čim je selector usklađen, Endpoints odmah dobije IP:port oba Poda.

**Razlaganje `kubectl patch`:**
- `patch svc websvc` — izmijeni postojeći Servis na licu mjesta
- `-p '{...}'` — JSON s onim što se mijenja (`spec.selector.app` = `websvc`)
- Servisov selector JE promjenjiv (za razliku od Deployment selektora, zad. 23) → patch prolazi bez delete-a
- (alternativa: `gedit svc.yaml` → `pogresna-labela` u `websvc` → `kubectl apply`)

### Zašto (srž za ispit)

Servis i Podovi su povezani ISKLJUČIVO preko labela: Servisov `selector` mora odgovarati `labels` Podova. Endpoints (ili EndpointSlice) je „živi popis" Pod IP:port koje selector trenutno hvata — controller ga automatski održava. Prazan Endpoints (`<none>`) znači da selector ne hvata nijedan (ready) Pod: ili kriva labela (ovaj slučaj), ili Podovi nisu ready (readiness fail, zad. 24), ili ih uopće nema. Zato je `get endpoints` prva naredba kad „Servis ne radi a Podovi su živi".

### Gotče

- `get endpoints <svc>` → `<none>` = mismatch ILI nema ready Podova. Treba provjeriti i labele (`get pods --show-labels`) i readiness.
- readiness fail (zad. 24) također prazni Endpoints — Pod mora biti **ready** da uđe u Endpoints, ne samo Running.
- `targetPort` mora biti port na kojem app sluša; kriv `targetPort` → Endpoints postoji ali promet ne prolazi (drugi kvar).
- veza: zad. 41 (sistematska dijagnoza „app ne doseže Servis": pod→svc→endpoints→DNS→port) i zad. 48 iz LO4 (isti lanac).

---

## Zadatak 26 — Deployment: `selector` ≠ `template.labels` (validacijska greška)

> Već viđeno UŽIVO u zad. 21 (dočekalo nas slučajno). Ovdje rezoniranje + taj stvarni nalaz.

**Kvar (manifest):**
```yaml
spec:
  selector:
    matchLabels:
      app: badimg          # selector traži Podove s app=badimg
  template:
    metadata:
      labels:
        app: web           # ali template lijepi app=web  ← NESKLAD
```

**Stvarna greška servera (iz zad. 21, provjereno uživo):**
```
The Deployment "badimg" is invalid: spec.template.metadata.labels:
Invalid value: map[string]string{"app":"web"}: `selector` does not match template `labels`
```

### Greška i uzrok — Deployment ne smije imati neusklađen selector/template

- `selector.matchLabels` = koje Podove Deployment **smatra svojima**.
- `template.metadata.labels` = labele koje **lijepi na Podove koje stvara**.
- Ako se ne poklapaju, Deployment bi stvarao Podove koje SAM ne bi prepoznao kao svoje → stvarao bi ih u beskonačno. Zato k8s to **odbije već pri validaciji** (`apply` puca), prije nego išta stvori.

### Razlika od zad. 25 (BITNO za ispit)

- **zad. 25 (Service):** Service `selector` ≠ Pod `labels` → Servis se STVORI, ali Endpoints prazan (`<none>`). Tihi kvar, otkriva `get endpoints`.
- **zad. 26 (Deployment):** Deployment `selector` ≠ `template.labels` → `apply` ODMAH puca, ništa se ne stvori. Glasna validacijska greška.

Razlog: Deployment MORA imati usklađen selector/template (njegova srž), pa k8s provjerava unaprijed. Servis smije imati selector koji trenutno ništa ne hvata (Podovi možda tek dolaze) — to je legitimno.

### Ispravak — uskladi `template.labels` sa `selector.matchLabels`

```yaml
spec:
  selector:
    matchLabels:
      app: badimg
  template:
    metadata:
      labels:
        app: badimg          # ← isto kao selector
```

### Zašto (srž za ispit)

Kod Deploymenta su `selector` i `template.labels` dvije strane iste veze: selector definira skup, template proizvodi članove tog skupa. k8s zahtijeva da se poklapaju (inače bi kontroler gradio Podove koje ne kontrolira). Validacija to hvata pri `apply` — za razliku od Servisa, gdje je prazan selector dopušten.

### Gotče

- `selector does not match template labels` pri `apply` → uskladi labele na oba mjesta.
- čest izvor: kopiranje manifesta pa promjena imena na jednom mjestu, a ne na drugom (točno kako nas je dočekalo u zad. 21).
- Deployment `selector` je k tomu i **immutable** nakon stvaranja (zad. 23) — pa ako se mijenja selector postojećeg, treba `delete` + `apply`.

---

## Zadatak 27 — Pending: `requests` > kapacitet čvora (`Insufficient cpu`)

> Rođak zad. 19, ali drukčiji u dvije stvari: vrtimo **CPU** (ne memoriju) i broj je *uvjerljiv* (`8`), ne apsurdan kao `100Gi` — pa treba **provjeriti stvarni kapacitet čvora**, ne procijeniti okom. Glavni novi alat: `kubectl describe node`.

**Pokvareni manifest:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-pending
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: "8"          # ← traži 8 cijelih jezgri; node ima 4
```

**Razlaganje manifesta:**
- `kind: Pod` — goli Pod, bez kontrolera (kao zad. 19)
- `resources.requests.cpu: "8"` — Pod **garantirano traži 8 cijelih jezgri** ← namjerni kvar
  - jedinica CPU-a: `"8"` = 8 cores = `8000m` (milicores). `500m` = pola jezgre, `1` = cijela.

**Dijagnoza (provjereno uživo):**
```bash
kubectl apply -f pending-cpu.yaml
kubectl get pod cpu-pending          # STATUS: Pending (0/1)
kubectl describe pod cpu-pending     # Events na dnu:
```
```
Warning  FailedScheduling  default-scheduler
0/1 nodes are available: 1 Insufficient cpu.
preemption: ... No preemption victims found for incoming pod.
```

### Čitanje ispisa (za ispit)

- `FailedScheduling` od `default-scheduler` → Pod neraspoređen (validan je, ali nema mjesta)
- `Insufficient cpu` → traženi CPU nijedan čvor ne može dati (kod zad. 19 je pisalo `Insufficient memory` — isti potpis, druga dimenzija)
- `No preemption victims` → ne može ni izbaciti tuđi Pod da oslobodi jezgre

### Glavni alat — `kubectl describe node minikube`

```bash
kubectl describe node minikube       # kapacitet čvora + koliko je već rezervirano
```
Tri polja koja čitamo (stvarni ispis s VM-a):
```
Capacity:     cpu: 4
Allocatable:  cpu: 4
Allocated resources:   cpu  750m (18%)
```
- **`Capacity:`** — ukupni hardver koji čvor prijavljuje (ovdje **4 jezgre**)
- **`Allocatable:`** — Capacity **minus** rezervirano za sustav/kubelet → **ovo scheduler stvarno gleda** (ovdje isto 4, jer minikube ništa ne rezervira)
- **`Allocated resources:`** — zbroj `Requests` svih Podova **već na čvoru** + postotak → koliko je preostalo

Onih **750m (18%)** dolazi od control-plane Podova u `kube-system` (apiserver 250m, controller-manager 200m, coredns/etcd/scheduler po 100m). Leftover Podovi iz zad. 24–25 (`readyfail`, `websvc`) traže **0m** → ne troše ništa (zato `0%`).

### Ključni račun (srž zadatka)

Scheduler NE uspoređuje s `Capacity`, nego sa **slobodnim** `Allocatable`:
```
slobodno = Allocatable − (Requests svih Podova na čvoru)
         = 4000m − 750m = 3250m
```
Naš Pod traži **8000m**. 8000 > 4000 → ne stane ni na prazan čvor, a kamoli u slobodnih 3250m.

### Ispravak — spusti `cpu` da stane u slobodni Allocatable

```yaml
    resources:
      requests:
        cpu: "500m"      # pola jezgre — komotno stane u 3250m
```
```bash
kubectl delete pod cpu-pending       # goli Pod je IMMUTABLE za resources → delete+create (kao zad. 19)
kubectl apply -f pending-cpu.yaml
kubectl get pod cpu-pending          # ContainerCreating → Running (1/1)
```
**(provjereno uživo ✓)** — `Pending → Running 1/1` čim je request (`500m`) pao ispod slobodnog Allocatable.

### Zašto (srž za ispit)

`requests` je garancija koju scheduler mora namiriti PRIJE nego smjesti Pod. Usporedba ide protiv **slobodnog Allocatable** = `Allocatable` čvora minus zbroj `Requests` Podova koji su već gore — **ne** protiv sirovog `Capacity`. Zato request manji od ukupnog čvora svejedno može pući ako je veći od preostatka. Dijagnoza: `describe pod` → `Events: Insufficient cpu/memory`, pa `describe node` → `Capacity` / `Allocatable` / `Allocated resources` da se vidi koliko je stvarno slobodno.

### Gotče

- **`Capacity` ≠ `Allocatable` ≠ slobodno.** Scheduler gleda **slobodno** (`Allocatable − već rezervirano`). `Allocated resources: ... (NN%)` odmah kaže postotak zauzeća.
- CPU jedinice: `"8"` = 8 jezgri = `8000m`; `500m` = pola jezgre; `1` = cijela. `m` = milicores (tisućinka jezgre).
- isti potpis kao zad. 19, samo `cpu` ↔ `memory`. Treća varijanta Pendinga: nevezan PVC (zad. 28).
- `requests` (scheduling-garancija) ≠ `limits` (runtime gornja granica; proboj memorije → OOMKilled 137, zad. 23).
- goli Pod immutable za `resources` → `apply` izmjene padne, treba `delete`+`apply`. Deployment to riješi sam preko template-a (LO4 zad. 15).
- Podovi koji traže `0m` (kao naši leftoveri) ne ulaze u račun zauzeća — `0 (0%)` u `Allocated`. Tek Podovi s pravim `requests` jedu slobodni Allocatable.

---

## Zadatak 28 — Pending: Pod čeka volumen koji ne postoji (nevezan PVC)

> **Treća varijanta Pendinga**: 19 = nedovoljno resursa, 27 = requests > kapacitet, **28 = nevezan volumen**. Pod traži PVC koji ne postoji → scheduler ga ne može smjestiti dok volumen nije spreman.

**Pojmovi (treba za čitanje ispisa):**
- **PV** (PersistentVolume) — komad stvarnog skladišta (disk).
- **PVC** (PersistentVolumeClaim) — *zahtjev* Poda za skladištem („trebam 1Gi"). Pod montira **PVC**, ne PV izravno.
- veza: **Pod → PVC → (binda se na) PV**. PVC ne postoji ili se ne može vezati → Pod visi u Pending.

**Pokvareni manifest:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pending
spec:
  containers:
  - name: app
    image: nginx:1.25
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: ne-postoji        # ← taj PVC ne postoji
```

**Razlaganje manifesta:**
- `volumeMounts` — gdje se *unutar kontejnera* montira volumen
  - `name: data` — referenca na volumen definiran niže (mora se podudarati)
  - `mountPath: /usr/share/nginx/html` — putanja u kontejneru gdje se volumen pojavljuje
- `volumes` — *definicija* volumena na razini Poda
  - `name: data` — ime (veže se s `volumeMounts.name`)
  - `persistentVolumeClaim.claimName: ne-postoji` — montiraj PVC tog imena ← **OVDJE greška**

**Dijagnoza (provjereno uživo):**
```bash
kubectl apply -f pvc-pending.yaml
kubectl get pod pvc-pending          # STATUS: Pending (0/1)
kubectl describe pod pvc-pending     # Events na dnu:
```
```
Warning  FailedScheduling  default-scheduler
0/1 nodes are available: persistentvolumeclaim "ne-postoji" not found.
preemption: ... Preemption is not helpful for scheduling.
```

### Čitanje ispisa (za ispit)

- isti `FailedScheduling` od istog schedulera kao 19/27, **ali poruka je drukčija**: `persistentvolumeclaim "ne-postoji" not found` — ne `Insufficient`. **Poruka doslovno imenuje PVC koji fali** → sama je dijagnoza.
- `Preemption is not helpful` (a NE `No preemption victims` kao kod resursa) → izbacivanje tuđih Podova tu ne pomaže jer problem nije manjak resursa nego nepostojeći volumen. Suptilna razlika u tekstu razlikuje tip Pendinga.

### Drugi dokaz — `kubectl get pvc`

```bash
kubectl get pvc                      # popis svih PVC-a u namespaceu
```
Na VM-u zatečena tri **Bound** PVC-a iz LO4 (Redis StatefulSet):
```
data-redis-0   Bound   pvc-...   100Mi   RWO   standard   19h
data-redis-1   Bound   pvc-...   100Mi   RWO   standard   19h
data-redis-2   Bound   pvc-...   100Mi   RWO   standard   19h
```
- **`Bound`** = PVC uspješno vezan na PV → zdravo stanje.
- u popisu **nema `ne-postoji`** → potvrda da PVC kojeg Pod traži stvarno ne postoji (drugi dokaz uz Events).
- StatefulSet svakom Podu radi vlastiti PVC po šabloni `volumeClaimTemplates`, ime `data-<sts>-<ordinal>` (LO4).

### Ispravak — dvije legitimne opcije

**Opcija A — Pod ne treba trajno skladište** (uzeto u vježbi): efemerni `emptyDir`.
```yaml
  volumes:
  - name: data
    emptyDir: {}            # prazan privremeni direktorij; ne treba PVC ni PV; nestaje s Podom
```
```bash
kubectl delete pod pvc-pending       # goli Pod immutable za volumene → delete+create
kubectl apply -f pvc-pending.yaml
kubectl get pod pvc-pending          # ContainerCreating → Running 1/1
```
**(provjereno uživo ✓)** — `Pending → Running 1/1` čim je nepostojeći PVC zamijenjen `emptyDir`-om.

**Opcija B — Pod stvarno treba trajni volumen**: napravi PVC koji fali (drugi dokument u istoj datoteci, odvojen `---`):
```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ne-postoji         # točno ime koje Pod traži (claimName)
spec:
  accessModes:
  - ReadWriteOnce          # RWO — jedan čvor čita/piše (kao kod redisa)
  resources:
    requests:
      storage: 100Mi
```
StorageClass `standard` (minikubeov default provisioner) tada **automatski** napravi PV i veže PVC → Pod krene (dynamic provisioning).

### Zašto (srž za ispit)

Pod ne montira PV izravno — traži ga preko **PVC-a** (`claimName`). Ako PVC ne postoji, scheduler odbija smjestiti Pod (volumen mora biti razriješen prije bind-a Poda na čvor) → `Pending` s porukom `persistentvolumeclaim "..." not found`. Razlika od 19/27: tamo je `Insufficient cpu/memory` (manjak resursa); ovdje je volumen. Tekst Eventa razlikuje: `Preemption is not helpful` (volumen) vs `No preemption victims` (resursi). Fix ovisi o namjeri: ne treba li trajnost → `emptyDir`; treba li → stvoriti PVC (StorageClass `standard` dinamički isporuči PV).

### Gotče

- **`persistentvolumeclaim "X" not found`** u Events = Pod traži nepostojeći PVC. Brza provjera: `get pvc` (je li `X` u popisu i je li `Bound`).
- PVC statusi: **`Bound`** (vezan, OK) / **`Pending`** (čeka PV ili provisioning) / **`Lost`** (PV nestao). PVC `Pending` → i Pod ostaje `Pending`.
- razlika u poruci: `Insufficient cpu/memory` (zad. 19/27, resursi) vs `persistentvolumeclaim not found` (zad. 28, volumen) — oba su `FailedScheduling`, ali tekst kaže koji tip.
- **PVC NE briše `delete pod` ni `delete deploy`** — perzistencija je namjerna; ostaje dok ga eksplicitno ne obrišeš (`kubectl delete pvc <ime>`). Zato su `data-redis-*` preživjeli sve dosadašnje čistke.
- `emptyDir` ≠ PVC: emptyDir je vezan za životni vijek **Poda** (nestaje s njim); PVC/PV preživljava Pod (prava perzistencija).
- StatefulSet PVC-i imaju predvidiva imena (`data-<sts>-<ordinal>`) — za razliku od Deploymenta koji volumene dijeli ili koristi emptyDir.

---

## Zadatak 29 — kontejner bez stalnog procesa (`Completed` → CrashLoopBackOff)

> **Srž:** kontejner živi **samo dok mu PID 1 (glavni proces) traje**. `ubuntu` po defaultu pokrene `bash`; `bash` bez TTY-a i bez skripte odmah izađe (kod 0) → kontejner „završi". Pod kontrolerom s `restartPolicy: Always` → diže se iznova = CrashLoop. **Veza: zad. 4 (Podman, isti mehanizam) + zad. 22 (CrashLoop), ali tu je suprotan exit kod.**

**Pokvareni manifest:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nostart
spec:
  containers:
  - name: app
    image: ubuntu:22.04        # ← nema dugotrajnog procesa
```

**Razlaganje manifesta:**
- `image: ubuntu:22.04` — čista Ubuntu slika ← **OVDJE problem**
  - nema `command`/`args` → koristi default iz slike (`bash`)
  - `bash` u neinteraktivnom kontejneru (bez `-it`, bez skripte) nema što čitati → izađe s kodom **0**
- nema `restartPolicy` → default **`Always`** (Pod restartuje kontejner koji izađe, bez obzira na exit kod)

**Dijagnoza (provjereno uživo):**
```bash
kubectl apply -f nostart.yaml
kubectl get pod nostart              # snimi 2-3x; status se mijenja u vremenu
kubectl describe pod nostart         # State / Last State + Events
```
`get pod` u vremenu:
```
ContainerCreating → CrashLoopBackOff (RESTARTS raste 1 → 2 ...)
```
`describe pod` → blok kontejnera (najčišći prikaz koncepta):
```
State:        Terminated
  Reason:     Completed
  Exit Code:  0            ← uredan izlazak, NIJE pad
  Started:    19:09:34
  Finished:   19:09:34     ← ista sekunda → proces traje milisekundu
Last State:   Terminated
  Reason:     Completed
  Exit Code:  0
Restart Count: 2
```
Events:
```
Pulled / Created / Started  (x3 over 30s)    ← kubelet ga 3x digao
Warning  BackOff  Back-off restarting failed container app in pod nostart
```

### Čitanje ispisa (za ispit) — kontrast sa zad. 22

- **`Exit Code: 0` + `Reason: Completed`** = proces je **uspješno završio**, samo nije bio trajan.
- `Started` == `Finished` (ista sekunda) → kontejner živi tren, odmah izađe.
- **Isti simptom (CrashLoopBackOff), suprotan uzrok** — exit kod ih razlikuje:
  - `Exit Code: 0` → kontejner radi točno što mu je rečeno, ali posao kratko traje (ovaj zad.)
  - `Exit Code: 1` → proces stvarno pao (zad. 22: `exit 1`)
  - `Exit Code: 137` → OOMKilled (zad. 23)
- k8s ga u Events zove **„failed container"** iako je exit 0 — jer pod `restartPolicy: Always` očekuje da trajni Pod **stalno radi**; „failed" tu znači „nije ostao gore", ne „srušio se". **Poruka zna zavarati — istinu kaže `Exit Code`.**

### Ispravak — dvije varijante (po namjeri Poda)

**Varijanta A — Pod TREBA stalno raditi** (dugotrajni servis): daj mu proces koji ne izlazi.
```yaml
  containers:
  - name: app
    image: ubuntu:22.04
    command: ["sleep", "infinity"]
```
- `command: [...]` — gazi default (`bash`); k8s ekvivalent docker `ENTRYPOINT`/`CMD` (prvi element = program, ostali = argumenti)
- `sleep infinity` — spava zauvijek → PID 1 nikad ne izađe → kontejner ostaje `Running`
```bash
kubectl delete pod nostart           # goli Pod immutable → delete+create
kubectl apply -f nostart.yaml
kubectl get pod nostart              # Running 1/1, RESTARTS 0 i OSTAJE
```
**(provjereno uživo ✓)** — `Running 1/1`, RESTARTS 0; trajni PID 1 drži kontejner gore.

**Varijanta B — posao je STVARNO kratak i to je OK** (jednokratan task): k8s-u se kaže da ne restartuje.
```yaml
spec:
  restartPolicy: OnFailure       # ili Never
  containers:
  - name: app
    image: ubuntu:22.04
    command: ["echo", "gotovo"]
```
- `restartPolicy: Never` — kontejner izađe, Pod ostane **`Completed`** (NE CrashLoop) → legitiman kraj jednokratnog Poda
- `OnFailure` — restartuje **samo** ako exit ≠ 0; kod 0 → `Completed`
- ovo je točno logika **Joba** iz LO4 (Job koristi `OnFailure`/`Never`); goli Pod s defaultom `Always` zato „crashloopa" na završenom poslu.

### Zašto (srž za ispit)

Kontejner nije „mašina koja stoji upaljena" — to je **omotač oko jednog procesa (PID 1)**. Kad taj proces izađe (svejedno uspješno ili s greškom), kontejner je gotov. Trajni servis MORA imati proces koji ne izlazi (server koji sluša, `sleep infinity`…). `restartPolicy` (default `Always`) određuje hoće li k8s dizati kontejner iznova: za servis se to želi, za jednokratni task ne (`OnFailure`/`Never`). CrashLoopBackOff je samo „kontejner izlazi pa ga stalno dižem" — `Exit Code` u `Last State` kaže je li izašao uspješno (0, fali mu trajan posao) ili pao (≠0, prava greška).

### Gotče

- **`Exit Code: 0` + CrashLoop** = kontejner radi posao pa izađe, a `Always` ga diže — fali mu trajan proces (ovaj zad.). **`Exit Code: ≠0` + CrashLoop** = stvarni pad (zad. 22 = 1, OOM = 137). Exit kod je razlika.
- `Last State` / `State` blok u `describe pod` (`Reason` + `Exit Code` + `Started`/`Finished`) je prvo mjesto za CrashLoop dijagnozu; `Started`==`Finished` → proces ne traje.
- Events zovu kontejner „failed" i kad je exit 0 — ne valja vjerovati riječi „failed", provjerava se `Exit Code`.
- `restartPolicy` vrijednosti: **`Always`** (default, za servise) / **`OnFailure`** (restart samo na grešku) / **`Never`** (nikad). Za jednokratni posao biraju se zadnja dva → Pod završi u `Completed`.
- `command:` (k8s) = `ENTRYPOINT` (docker); `args:` (k8s) = `CMD` argumenti. `sleep infinity` / `sleep 3600` / `tail -f /dev/null` su tipični „drži me gore" trikovi za debug-kontejnere.
- isti mehanizam u Podmanu (zad. 4): `podman run ubuntu` odmah `Exited (0)` — kontejner je proces, ne VM.

---

# Skupina C — alati dijagnostike (zad. 30–33)

> Prijelaz s „pokvareni manifest → popravi" na **naredbe kojima istražuješ klaster**. Grade jedan na drugi: 30 (kronologija cijelog klastera, pogled IZVANA) → 31 (uđi U Pod, DNS iznutra) → 32 (uđi u Pod koji NEMA shell, ephemeral debug) → 33 (privremeni Pod za test povezanosti).

## Zadatak 30 — `kubectl get events --sort-by=.lastTimestamp` (kronologija klastera)

> **Zašto postoji:** `describe pod` da događaje **jednog** objekta. Kad nije poznato ni gdje je problem, treba **kronološka traka svega** na klasteru. `get events` po defaultu NIJE poredan po vremenu → zato `--sort-by`.

**Priprema „prometa" da events ne bude prazan** (nakon čistke klastera):
```bash
kubectl run busy --image=busybox:1.36 --restart=Never -- sh -c "echo radim; exit 3"
```
**Razlaganje `kubectl run` flag po flag:**
- `kubectl run busy` — imperativno stvara **goli Pod** `busy` (bez manifesta)
- `--image=busybox:1.36` — slika
- `--restart=Never` — `restartPolicy: Never` → ne crashloopa; izađe i ostane `Error`/`Completed` (čistije za events; usp. zad. 29)
- `--` — **kraj kubectl-zastavica**; sve iza je naredba za kontejner
- `sh -c "echo radim; exit 3"` — shell, ispisuje, izlazi kodom 3 (namjeran „pad")

**Glavna naredba (provjereno uživo):**
```bash
kubectl get events --sort-by=.lastTimestamp     # najstariji gore → najnoviji dolje
kubectl get events                              # za usporedbu: NEporedano (izmiješano)
```
**Razlaganje:**
- `get events` — svi Event objekti u namespaceu
- `--sort-by=.lastTimestamp` — sortiraj po polju `lastTimestamp` (JSONPath put do polja u Event objektu) → kronološki niz
- bez sortiranja redoslijed je nasumičan (po internom ključu) → promakne ti tok

### Čitanje ispisa (za ispit)

- **sortirano** → uredna kronološka traka (npr. čisti niz `4h22m → 3h44m`), vidljiv je tok `Scheduled → Pulling → Pulled → Created → Started → (BackOff/Failed)`
- **golo `get events`** → ages skaču gore-dolje (nepouzdano za rekonstrukciju toka)
- na ovom VM-u events su čuvali **cijelu povijest sesije** (svi LO4+LO5 objekti) → korisno za „što se uopće događalo"

### Dvije kvake (BITNO — ispravak naivne predodžbe)

1. **TTL nije fiksno 1h.** Apiserver čisti događaje nakon `--event-ttl` (**zadano 1h**), ali je **konfigurabilno**; na ovom klasteru je očito dulje (vidljivi događaji stari 4h+). Zaključak: events NISU trajni log, ali rok ovisi o postavci klastera.
2. **`--sort-by=.lastTimestamp` zna izgledati neuredno.** Noviji događaji (Events API) drže vrijeme u `.eventTime`, a `.lastTimestamp` im je prazan → ispadnu iz reda. **Robusne zamjene:**
   - `kubectl events --sort-by=.lastTimestamp` — noviji **namjenski** alat (k8s 1.23+), podržava `--for <objekt>`, `--watch`
   - `kubectl get events --sort-by=.metadata.creationTimestamp` — sortira po vremenu stvaranja (uvijek popunjeno)

### Zašto (srž za ispit)

Events su kratkotrajni zapisi koje generiraju kontroleri/kubelet/scheduler o tome što rade s objektima. `describe <objekt>` filtrira događaje tog objekta; `get events` daje **sve** u namespaceu. Za dijagnozu „nešto se dogodilo a ne zna se gdje" potrebni su **kronološki** — `--sort-by=.lastTimestamp` (ili robusnije `.metadata.creationTimestamp` / `kubectl events`). Valja pamtiti da istječu (`--event-ttl`) → nisu zamjena za trajni logging.

### Gotče

- **`get events` bez sortiranja = nasumičan redoslijed.** Uvijek dodaj `--sort-by` za tok.
- `--sort-by=.lastTimestamp` može varati (prazan `.lastTimestamp` kod novih događaja) → koristi se `.metadata.creationTimestamp` ili `kubectl events`.
- events su **namespaced** (`-n <ns>` za drugi namespace; `-A` za sve namespaceove) i **istječu** (TTL konfigurabilan, default 1h).
- `--` u `kubectl run`/`exec` odvaja kubectl-zastavice od naredbe kontejnera (sve iza `--` ide kontejneru).
- `--field-selector` filtrira (npr. `--field-selector type=Warning` → samo upozorenja; korisno kad je liste previše).
- veza: `describe pod` (zad. 19–29) = events jednog Poda; ovo = events cijelog klastera.

---

## Zadatak 31 — `exec -it` + DNS iznutra (`resolv.conf`, `nslookup`)

> **Zašto postoji:** events/`describe` kažu što k8s vidi IZVANA. „App ne nalazi servis/bazu" je često **DNS iznutra Poda** — vidljiv je samo ako se uđe u Pod i pita njegov resolver. Alat: `kubectl exec`. **Nastavak na zad. 30** (tamo pogled izvana, ovdje iznutra).

**Postavljanje mete i alata (imperativno, bez manifesta):**
```bash
kubectl create deployment web --image=nginx:1.25          # meta: app+Pod
kubectl expose deployment web --port=80                   # Servis 'web' (ClusterIP) → DNS ime
kubectl run netshoot --image=busybox:1.36 --restart=Never -- sleep infinity   # Pod za ući (busybox ima nslookup)
```
- `expose deployment web --port=80` — napravi **Servis** `web` koji cilja Deployment → time se dobiva ime `web` za razriješiti
- `run netshoot ... -- sleep infinity` — goli Pod koji ostaje `Running` (usp. zad. 29); busybox jer ima ugrađen `nslookup`

**Glavni dio — uđi u Pod i pitaj DNS:**
```bash
kubectl exec -it netshoot -- sh
```
**Razlaganje `exec` flag po flag:**
- `exec` — pokreće naredbu u **POSTOJEĆEM** kontejneru (≠ `run` koji stvara novi Pod)
- `-it` — `-i` (drži stdin) + `-t` (TTY) → interaktivni shell (isti `-it` kao `podman run`/`docker exec`)
- `netshoot` — u koji Pod; `--` — kraj kubectl-zastavica; `sh` — busybox nema `bash`, ima `sh`

**Unutar Poda (prompt `/ #`):**
```sh
cat /etc/resolv.conf
nslookup web
nslookup kubernetes
nslookup nepostojeci-servis
exit
```

**Stvarni ispis s VM-a (provjereno uživo):**
```
# cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local dns.podman vua.cloud
options ndots:5

# nslookup web
Server:  10.96.0.10
Name:    web.default.svc.cluster.local
Address: 10.101.245.89          # = ClusterIP Servisa 'web' ✓

# nslookup kubernetes  →  kubernetes.default.svc.cluster.local  Address: 10.96.0.1
# nslookup nepostojeci-servis  →  samo NXDOMAIN (nema uspješne linije)
```

### Čitanje ispisa (za ispit)

- **`nameserver 10.96.0.10`** = ClusterIP CoreDNS-a (servis `kube-dns`); konvencija `.10` (jer `kubernetes` = `.1`).
- **`search ...`** = search domene; kratko ime `web` radi jer se proširi na `web.default.svc.cluster.local`.
- **otkrivanje servisa po imenu radi**: `web` → ClusterIP Servisa (`10.101.245.89`).
- **FQDN servisa**: `<svc>.<namespace>.svc.cluster.local`.
- `options ndots:5` = ako ime ima < 5 točaka, prvo proba sa search domenama (tek onda apsolutno).

### Dvije „prave" kvake uhvaćene uživo (BITNO)

1. **Search domene imaju VIŠAK naslijeđen s hosta.** Uz 3 klasterske (`default.svc...`, `svc...`, `cluster.local`) pojavile su se i **`dns.podman` i `vua.cloud`** — to su domene iz VM-ovog `/etc/resolv.conf` (`vua.cloud` = Algebra/VUA cloud). Kubelet (zadani `dnsPolicy: ClusterFirst`) **dopiše čvorove search domene iza klasterskih**. Zato su vidljivi pokušaji `web.dns.podman`, `web.vua.cloud`. → **Pod-ov search list = klasterske domene + domene čvora.**
2. **Zid NXDOMAIN-a kod busybox `nslookup` = bučnost, NE greška.** Resolver prošeta SVAKU search domenu × (IPv4 + IPv6) → gomila „can't find" + duplikati. **Uspjeh je zakopan** — tražiš redak **bez** NXDOMAIN (`Name:`/`Address:`). Kod busyboxa ne paničariš na crveno.
   - `exit` može dati `command terminated with exit code 1` jer goli `exit` vrati status **zadnje** naredbe (ovdje neuspio `nepostojeci-servis`). Bezopasno.

### Zašto (srž za ispit)

Svaki Pod dobije `/etc/resolv.conf` koji k8s ubrizga: `nameserver` pokazuje na **CoreDNS** (ClusterIP `kube-dns` servisa), a `search` domene omogućuju kratka imena (`web` umjesto punog FQDN-a). Otkrivanje servisa = DNS A-zapis `<svc>.<ns>.svc.cluster.local` → ClusterIP. `kubectl exec -it <pod> -- sh` ubacuje u kontejner radi provjere iznutra (resolv.conf + nslookup) — ključno kad „app ne vidi drugi servis": je li DNS kriv, ili je problem dalje (Endpoints/port). Valja paziti na busybox nslookup bučnost i na search domene naslijeđene s čvora.

### Gotče

- `exec -it <pod> -- sh` = uđi u **postojeći** kontejner (≠ `run` koji stvara novi). busybox/alpine = `sh` (nemaju `bash`).
- `/etc/resolv.conf` u Podu: `nameserver` = CoreDNS ClusterIP (10.96.0.10), `search` = klasterske + čvorove domene, `ndots:5`.
- kratko ime servisa radi **samo unutar istog namespacea**; za drugi namespace treba `<svc>.<ns>` (ili pun FQDN).
- busybox `nslookup` je bučan (prošeta sve search domene, A+AAAA) → nađi redak bez NXDOMAIN. Za čišći ispis: `nslookup <svc>.default.svc.cluster.local` (pun FQDN → manje pokušaja).
- `nslookup` razriješi ime → IP, ali NE testira povezanost (to je zad. 33: `wget`/`nc` na ClusterIP:port). DNS OK + nema odgovora = problem je dalje (Endpoints prazan zad. 25, kriv port, NetworkPolicy).
- ako Pod NEMA shell (distroless) → `exec -- sh` padne → treba `kubectl debug` ephemeral (zad. 32).

---

## Zadatak 32 — `kubectl debug` (ephemeral container za sliku BEZ shella)

> **Bridge s 31:** `exec -- sh` radi jer busybox **ima** shell. Produkcijske **distroless** slike (samo binarka + ovisnosti, bez `sh`/`ls`/ičega) → `exec -- sh` **padne**. Rješenje: `kubectl debug` prikvači **privremeni (ephemeral) kontejner s alatima** na živi Pod, dijeleći mu namespace. Pod se NE restarta.

**Distroless meta** (`pause` — vrti se zauvijek, nema shell, sićušan, već keširan):
```bash
kubectl run distroless --image=registry.k8s.io/pause:3.9 --restart=Never
```
- `pause` sama spava zauvijek → Pod `Running` bez `command`; ime kontejnera = ime Poda = `distroless`

**Korak 1 — `exec` padne (nema shella) — provjereno uživo:**
```bash
kubectl exec -it distroless -- sh
```
```
OCI runtime exec failed: ... exec: "sh": executable file not found in $PATH
command terminated with exit code 126
```
- `/bin/sh` doslovno ne postoji u slici. `exit code 126` = „naredbu nije moguće izvršiti".

**Korak 2 — `kubectl debug` prikvači ephemeral kontejner — provjereno uživo:**
```bash
kubectl debug -it distroless --image=busybox:1.36 --target=distroless -- sh
```
**Razlaganje flag po flag:**
- `debug` — pokreće debug-sesiju; **dodaje ephemeral kontejner u POSTOJEĆI Pod** (bez restarta, bez novog Poda)
- `-it` — interaktivno + TTY
- `distroless` — **ciljani Pod** (kamo dodajemo debug-kontejner)
- `--image=busybox:1.36` — slika **ephemeral kontejnera** (IMA `sh`, `ps`, `wget`, `nslookup`)
- `--target=distroless` — **dijeli namespace s tim kontejnerom** (PID + ostalo) → vidljivi su procesi/datoteke mete
- `-- sh` — naredba u ephemeral kontejneru

Stvarni ispis iznutra:
```
Targeting container "distroless"... Defaulting debug container name to debugger-m74xx
/ # ps -ef
PID   USER     TIME  COMMAND
  1   65535    0:00  /pause      ← proces METE (distroless Poda)!
 13   root     0:00  sh          ← naš debug kontejner
 19   root     0:00  ps -ef
/ # ls /proc/1/root
ls: /proc/1/root: Permission denied
```

### Čitanje ispisa (za ispit)

- **`PID 1 /pause` vidljiv iz debug-kontejnera** → `--target` radi, **PID namespace se dijeli** ✓. (`USER 65535` = „nobody" uid pod kojim pause vrti.)
- k8s automatski imenuje ephemeral kontejner (`debugger-xxxxx`) i **doda ga u živi Pod bez restarta**.
- **`ls /proc/1/root` → Permission denied**: vidjeti **procese** (`ps`) ne treba privilegije, ali **zaviriti u filesystem** druge mete traži `CAP_SYS_PTRACE`. Default debug kontejner je **neprivilegiran** (+ ovaj minikube je ugniježđen u rootless podman → remapirani uid-evi) → odbije.
  - za browse mete: `kubectl debug ... --profile=sysadmin` (daje privilegije). Glavni cilj (vidjeti procese distroless mete) ipak radi.

### Zašto (srž za ispit)

Distroless slike namjerno nemaju shell/alate (manja površina napada, manja slika) — ali onda u njih nije moguć `exec -- sh`. **Ephemeral kontejneri** rješavaju to: `kubectl debug` ubacuje privremeni kontejner (sa slikom punom alata) **u postojeći Pod**, dijeleći mu mrežu i (uz `--target`) PID namespace. Tako debugiraš metu „izvana iznutra": `ps` njene procese, `wget`/`nslookup` s njene mreže, a uz elevated profil i njen filesystem. Bez restarta i bez mijenjanja originalne slike. Ephemeral kontejneri su GA od k8s 1.25.

### Gotče

- distroless / `scratch` / `pause` → nema `sh` → `exec -- sh` padne (`executable file not found`, exit 126/127). Tada `kubectl debug`.
- `kubectl debug -it <pod> --image=<alati> --target=<kontejner> -- sh` = ephemeral kontejner, dijeli namespace, **bez restarta Poda**.
- bez `--target`: debug kontejner dijeli **mrežu** Poda (isti localhost/IP), ali NE vidi procese mete (zaseban PID namespace). Za procese/FS mete treba `--target`.
- `--target` ovisi o podršci runtimea (poruka to i kaže); ovdje docker → radi.
- za pristup **filesystemu** mete treba elevated profil: `--profile=sysadmin` (default je neprivilegiran).
- ephemeral kontejner se NE može maknuti iz Poda (ostaje u spec-u dok Pod živi); nestaje kad Pod nestane. Ne troši ništa nakon `exit`.
- veza: `exec` (zad. 31) za slike SA shellom; `debug` (ovaj) za one BEZ. Oba služe za ulazak i pregled iznutra.

---

## Zadatak 33 — `run tmp --rm -it` (privremeni Pod za test povezanosti)

> **Zašto postoji:** `nslookup` (zad. 31) potvrdi DNS, ali ne i da nešto **odgovara** na portu. Za end-to-end test diže se **Pod-jednokratka**, spaja se na servis, pa ga sam obriše. Nula smeća. **Zatvara dijagnostičku skupinu: name→IP (31) → uđi u distroless (32) → stvarna povezanost (33).**

Metu (`web` Deployment + Servis) imamo iz zad. 31 — nema nove pripreme.

```bash
kubectl run tmp --image=busybox:1.36 --rm -it --restart=Never -- sh
```
**Razlaganje flag po flag:**
- `run tmp` — novi goli Pod `tmp`
- `--image=busybox:1.36` — slika s alatima (`wget`, `nc`, `nslookup`)
- `--rm` — **auto-obriši Pod čim se izađe** (isti `--rm` kao `podman run`, zad. 5) → throwaway
- `-it` — interaktivno + TTY
- `--restart=Never` — nužno uz interaktivni `--rm` (Pod ne smije restartati)
- `-- sh` — shell

**Unutar Poda (provjereno uživo):**
```sh
wget -qO- http://web
wget -T 3 -qO- http://web:9999
exit
```
- `wget -qO- http://web` — `-q` tiho, `-O-` ispis u **stdout** (`-`=stdout); `http://web` → DNS→ClusterIP→port 80→nginx
  → ispiše **pun nginx welcome HTML** = end-to-end radi (DNS + Service + Endpoints + Pod)
- `wget -T 3 -qO- http://web:9999` — `-T 3` timeout 3s; port 9999 → `wget: download timed out`

### Čitanje ispisa (za ispit) — `timeout` vs `connection refused` (KLJUČNO za zad. 41)

- **uspjeh** = HTML se ispiše → cijeli lanac zdrav.
- **`download timed out`** (a NE „refused"): Servis `web` definira samo port 80; spoj na ClusterIP:**9999** → kube-proxy **nema pravilo** za taj port → paketi u prazno (blackhole) → **istek**.
- **`connection refused`** bi bio drukčiji signal: host dosegnut, port aktivno odbija (RST) — nešto je tamo i kaže „ne".
- oba znače „ime OK, ne mogu razgovarati", ali **smjer dijagnoze se razlikuje**:
  - **timeout** → ruta / Service-pravilo / NetworkPolicy (paket nigdje ne stiže)
  - **refused** → dosegnuo si host, ali port zatvoren / nitko ne sluša

### Tri ishoda `wget`/`nc` testa (cheat-sheet za ispit)

| Ishod | Što znači | Gdje gledati dalje |
|-------|-----------|--------------------|
| HTML / odgovor | sve radi | — |
| **NXDOMAIN** (nslookup) | ime se ne razriješi | **DNS**: krivo ime, krivi namespace, CoreDNS (zad. 31) |
| **connection refused** | host OK, port zatvoren | app ne sluša na `targetPort`, krivi port |
| **timeout** | paket nigdje ne stiže | prazni Endpoints (zad. 25), kriv Service port, NetworkPolicy |

### Zašto (srž za ispit)

Otkrivanje servisa ima dva sloja: **razlučivanje imena** (DNS → ClusterIP) i **povezanost** (stvarno spajanje na ClusterIP:port → Endpoints → Pod). `nslookup` testira samo prvi; `wget`/`nc` iz Pod-jednokratke testira **cijeli lanac**. `kubectl run tmp --rm -it` je standardni „digni-testiraj-baci" alat: čist Pod s mrežnim alatima, koji se sam ukloni. Razlika `timeout` vs `refused` u odgovoru već sužava gdje je kvar (ruta/pravilo vs zatvoren port).

### Gotče

- `--rm` + `-it` traži `--restart=Never` (inače kubectl odbije — `--rm` radi samo s Never).
- `--rm` obriše Pod tek na **čist izlaz** iz shella (`exit`); ako se prekine drugačije, Pod zna ostati → `kubectl delete pod tmp`.
- busybox `wget`: `-qO-` (tiho, na stdout), `-T <s>` (timeout). Za samo provjeru porta bez HTTP-a: `nc -zv web 80` (`-z` skeniraj, `-v` verbose).
- `http://web` (kratko ime) radi samo u **istom namespaceu**; inače `http://web.<ns>` ili FQDN.
- veza: ovo + zad. 25 (Endpoints) + zad. 31 (DNS) = pune komponente lanca dijagnoze u zad. 41.

---

# Skupina C — nastavak (zad. 34+): miks hands-on i rezoniranja

## Zadatak 34 — čvor `NotReady` (kubelet / disk / CNI / runtime)

> **Rezoniranje** (ne rušimo kubelet na jednočvornom klasteru — ubilo bi ga). Naslanja se na `Conditions:` blok iz `describe node` (zad. 27).

**Mehanizam — kako čvor postaje (Not)Ready:**
- svaki čvor vrti **kubelet** koji šalje **heartbeat** API serveru (node lease + status).
- uvjet **`Ready`**: `True` = zdrav, prima Podove; `False`/`Unknown` = **NotReady**.
- kubelet prestane slati otkucaje (pad, mrežni prekid) → node-controller nakon ~40s označi `Ready=Unknown`.
- Podovi na NotReady čvoru: nakon ~5 min (`node.kubernetes.io/unreachable` tolerancija 300s) kontroler ih **iseli/preraspoređuje** (ako su pod kontrolerom; goli Pod ostaje zaglavljen).

**Zdravi baseline (provjereno uživo) — `kubectl describe node minikube | grep -A8 "Conditions:"`:**
```
Type             Status   Reason                       Message
MemoryPressure   False    KubeletHasSufficientMemory   ← OK
DiskPressure     False    KubeletHasNoDiskPressure     ← OK
PIDPressure      False    KubeletHasSufficientPID      ← OK
Ready            True     KubeletReady                 ← OK
```
- `grep -A8 "Conditions:"` — redak `Conditions:` + 8 redaka iza (`-A` = after) → cijela tablica bez ostatka opisa

### Čitanje ispisa (za ispit) — logika „obrni i imaš kvar"

- **`Ready True` + svi pressure `False`** = zdravo (prima Podove).
- **`Ready False`** = kubelet javlja da NIJE spreman (CNI nije gotov, runtime pao…) → `Reason`/`Message` kažu što.
- **`Ready Unknown`** = kubelet **uopće ne javlja** (otkucaj izgubljen ~40s) → mrežni prekid ili kubelet mrtav.
- bilo koji pressure **`True`** = resource pressure; ozbiljan `DiskPressure` sam obara čvor u NotReady.

**Dva polja koja potvrđuju živost:**
- **`LastHeartbeatTime`** — kad je kubelet zadnji put javio (svjež = živ; zamrznut u prošlosti = pao).
- **`LastTransitionTime`** — kad se status zadnji put **promijenio** (fiksan dok je stanje stabilno).

### Pet tipičnih uzroka NotReady (srž za ispit)

1. **kubelet pao/zaustavljen** — servis ne radi / kriva konfiguracija → `systemctl status kubelet`, `journalctl -u kubelet`
2. **mrežni prekid** — čvor ne doseže API server → heartbeat izgubljen → `Ready=Unknown`
3. **resource pressure** — pun disk (`DiskPressure`), memorija, PID → `df -h` na čvoru
4. **CNI nije spreman** — mrežni plugin neinstaliran/pao → `NetworkReady=false` / `cni plugin not initialized` → **klasika na svježem klasteru**
5. **container runtime pao** — docker/containerd crashao → kubelet ne upravlja kontejnerima

### Put dijagnoze

```
kubectl get nodes                    → STATUS NotReady (prvi signal)
kubectl describe node <node>         → Conditions: + Reason/Message; je li heartbeat zastao
   ↳ na čvoru: systemctl status kubelet / journalctl -u kubelet / df -h / status runtimea
```

> **Napomena PDF ↔ VM (proturječje):** PDF (zad. 34) za inspekciju na minikubeu navodi `sudo minikube logs`. Na ovom VM-u **`sudo` ne radi** (rootless okruženje, vidi vrh skripte i `## VM-specifične zamke`), pa `sudo minikube logs` nije izvediv. Ekvivalent bez `sudo`: `minikube logs` (bez prefiksa) ili `minikube ssh` pa unutra `journalctl -u kubelet`. Spomen zadržan jer ga PDF traži; izvedba usklađena s VM-stvarnošću.

### Zašto (srž za ispit)

Čvor je „spreman" dok kubelet redovito javlja zdrav status API serveru. NotReady znači jedno od: kubelet ne radi/ne javlja (heartbeat zastao → `Unknown`), ili javlja problem (`Ready False` s konkretnim Reasonom — CNI/runtime/pressure). Dijagnoza ide odozgo (`get nodes` → `describe node` → `Conditions`/heartbeat) pa, ako treba i ako postoji pristup čvoru, na sam čvor (kubelet log, disk, runtime). Razlika `False` vs `Unknown` odmah kaže je li kubelet živ-ali-nezadovoljan ili posve nedostupan.

### Gotče

- **`Ready False`** (kubelet živ, javlja problem) vs **`Ready Unknown`** (kubelet nedostupan, heartbeat zastao) — različit smjer dijagnoze.
- na NOVOM klasteru NotReady je najčešće **CNI još nije primijenjen** (`NetworkReady=false`) — rješenje je instalacija CNI-ja, ne paljenje vatre.
- goli Pod na palom čvoru ostaje zaglavljen (nema kontrolera da ga preseli); Deployment/StatefulSet Pod se nakon ~5 min preraspodijeli (ako ima drugi čvor).
- `kubectl get nodes -o wide` daje i InternalIP, OS, kernel, runtime — korisno za brzi pregled flote.
- veza: `describe node` `Conditions` je isti blok koji smo gledali u zad. 27 (tamo za scheduling kapacitet, ovdje za zdravlje čvora).

---

## Zadatak 35 — Pod zaglavljen u `Terminating` (finalizeri, force delete + rizici)

> **Hands-on.** Pod koji odbija umrijeti. Konceptualno bitno: finalizeri + rizici prisilnog brisanja.

**Mehanizam — što je finalizer:**
- **finalizer** = string-ključ u `metadata.finalizers`. Kad obrišeš objekt, k8s mu postavi `deletionTimestamp` i stavi ga u **`Terminating`**, ali ga **NE makne iz etcd-a dok se svi finalizeri ne uklone**.
- finalizere uklanja **kontroler** koji ih je postavio — tek kad odradi svoj **cleanup prije brisanja** (otkvači volumen, odjavi s LB-a, obriše cloud resurs).
- kontroler ne postoji/mrtav → objekt zauvijek visi u `Terminating`.

**Manifest (izmišljen finalizer koji nitko ne uklanja):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: stuck
  finalizers:
  - example.com/blokada        # ← izmišljen finalizer; nijedan kontroler ga ne miče
spec:
  containers:
  - name: app
    image: nginx:1.25
```
- `metadata.finalizers` — popis; mora biti **kvalificirano ime** (`domena/naziv`)

**Tijek (provjereno uživo):**
```bash
kubectl apply -f stuck.yaml
kubectl get pod stuck                 # Running 1/1
kubectl delete pod stuck              # ⚠️ VISI (čeka da objekt nestane) → ~3s pa Ctrl+C
kubectl get pod stuck                 # STATUS: Completed/Terminating (vidi napomenu)
kubectl get pod stuck -o yaml | grep -A2 finalizers
```
Ispis potvrde zaglavljenosti:
```
deletionGracePeriodSeconds: 0         # objekt JE označen za brisanje (grace istekao)
finalizers:
- example.com/blokada                 # ali finalizer ga drži u etcd-u
```

**Fix — ukloni finalizer (objekt odmah nestane):**
```bash
kubectl patch pod stuck -p '{"metadata":{"finalizers":null}}' --type=merge
kubectl get pod stuck                 # Error: pods "stuck" not found  ✓
```
**Razlaganje `patch`:**
- `-p '{"metadata":{"finalizers":null}}'` — postavi `finalizers` na `null` → ukloni ih
- `--type=merge` — merge patch; `null` briše polje
- čim finalizer ode, a `deletionTimestamp` već postoji → API server **odmah** makne Pod

### Čitanje ispisa (za ispit) — suptilni detalj sa STATUS-om

- nakon Ctrl+C status je bio **`Completed`**, ne nužno `Terminating`. Razlog: kolona STATUS pokazuje stanje **kontejnera** (nginx dobio SIGTERM, uredno stao → `Completed`), dok je **Pod-objekt** zarobljen u brisanju.
- pravi dokaz zaglavljenosti NIJE STATUS nego **`deletionGracePeriodSeconds: 0` + popunjeni `finalizers`** (objekt naručen za brisanje, finalizer ga drži).
- na klasteru s više prometa STATUS bi pisao `Terminating`; ovdje je nginx prebrzo stao → `Completed`. Svejedno zarobljen.

### Rizici — zašto se finalizeri NE smiju olako micati

Finalizer = kontrolerov cleanup prije brisanja. `finalizers:null` taj cleanup **preskoči**:
- volumen ostane „prikvačen" u oblaku (dangling disk)
- cloud LB / DNS zapis ostane registriran (curenje resursa, plaćaš ga)
- vanjski zapis o objektu ostane siroče

→ `finalizers:null` je **zadnja opcija**, tek kad je **poznato** da je kontroler mrtav i cleanup nepotreban (ili ručno odrađen). Kod izmišljenog finalizera nema pravog resursa → bezopasno; u produkciji oprezno.

### Dva različita „Terminating zaglavljen" (NE pomiješati) — KLJUČNO

| Uzrok | Prepoznavanje | Ispravan fix |
|-------|---------------|--------------|
| **Finalizer** (ovaj) | `finalizers:` popunjen + `deletionTimestamp` | ukloni finalizer (`patch ... finalizers:null`); **`--force` NE pomaže** |
| **Nedostupan čvor** (kubelet ne potvrđuje) | čvor `NotReady` (zad. 34), Pod na njemu visi | `kubectl delete pod X --force --grace-period=0` |

- `--force --grace-period=0` rješava **samo** drugi slučaj (makne Pod iz API-ja bez čekanja kubeleta); na finalizer **ne djeluje**.
- rizik force-delete s nedostupnog čvora: ako čvor oživi, kontejner možda i dalje radi („zombie") → kod StatefulSeta mogu nakratko postojati **dvije kopije** (stabilni identitet narušen).

### Zašto (srž za ispit)

`Terminating` znači „brisanje je naručeno (`deletionTimestamp` postavljen), ali još nije dovršeno". Dva su razloga zašto zapne: (1) **finalizer** blokira micanje iz etcd-a dok ga kontroler ne ukloni — fix je ukloniti finalizer; (2) **kubelet ne može potvrditi** brisanje (čvor pao) — fix je force-delete. Pomiješati ih = pogrešan alat: force ne miče finalizer, a patch finalizera ne treba kad je problem samo nedostupan čvor. Force-delete uvijek nosi rizik dvostrukog pokretanja.

### Gotče

- STATUS `Terminating` (ili `Completed`/`Running` uz postavljen `deletionTimestamp`) + popunjeni `finalizers` = zaglavljeno na finalizeru.
- `kubectl patch ... -p '{"metadata":{"finalizers":null}}' --type=merge` (ili JSON patch `[{"op":"remove","path":"/metadata/finalizers"}]`) miče finalizer.
- `--force --grace-period=0` NE miče finalizere — samo preskače čekanje kubeleta (za Pod s palog čvora).
- isto se događa i **namespaceu** zaglavljenom u `Terminating` (čest slučaj) — uzrok su finalizeri resursa unutra; ista logika (riješiti/ukloniti finalizere).
- finalizeri su legitiman, koristan mehanizam (npr. `kubernetes.io/pv-protection`, `foregroundDeletion`) — ne micati naslijepo.
- veza: zad. 34 (NotReady čvor → drugi tip zaglavljenog Terminatinga).

---

## Zadatak 36 — krivi `apiVersion`/`kind` par (`no matches for kind`)

> Brz hands-on + alat za otkrivanje ispravnog para. Svaki `kind` živi u točno određenoj **API grupi/verziji**.

**Pozadina:** `Pod` je u **`v1`** (core grupa); `Deployment`/`ReplicaSet`/`StatefulSet`/`DaemonSet` su u **`apps/v1`**. Zamijeniš li ih, server ne poznaje par.

**Pokvareni manifest (namjerno krivi par):**
```yaml
apiVersion: apps/v1        # ← Pod NIJE u apps/v1, nego u v1
kind: Pod
metadata:
  name: krivoverz
spec:
  containers:
  - name: app
    image: nginx:1.25
```

**Dijagnoza (provjereno uživo):**
```bash
kubectl apply -f krivoverz.yaml
```
```
error: resource mapping not found for name: "krivoverz" ...
no matches for kind "Pod" in version "apps/v1"
ensure CRDs are installed first
```

### Čitanje ispisa (za ispit)

- `no matches for kind "Pod" in version "apps/v1"` → server u grupi `apps/v1` **nema** `Pod`. Greška dolazi sa **servera** (discovery), ne lokalno.
- **`ensure CRDs are installed first`** — server pretpostavi da možda tražiš **custom** tip (CRD) koji nije instaliran. **Ista poruka iskoči i kad stvarno fali CRD** (npr. `kind: Certificate` bez cert-managera). Dakle „no matches for kind" = ili **krivi apiVersion** (ovaj slučaj) **ili nedostaje CRD**.

**Alat za otkrivanje ispravnog para — `kubectl api-resources`:**
```bash
kubectl api-resources | grep -iE 'pods|deployments'
```
```
pods          po       v1         true   Pod
deployments   deploy   apps/v1    true   Deployment
```
- `api-resources` — svi tipovi koje klaster poznaje: `NAME · SHORTNAMES · APIVERSION · NAMESPACED · KIND`
- kolona **APIVERSION** = autoritativan odgovor (`Pod → v1`, `Deployment → apps/v1`)
- bonus: SHORTNAMES (`po`, `deploy`) = kratice za `kubectl get po`/`get deploy`

**Fix — `apiVersion: v1`:**
```bash
# apiVersion: apps/v1 → v1
kubectl apply -f krivoverz.yaml          # pod/krivoverz created
kubectl get pod krivoverz                # Running 1/1  ✓
```

### Referentna tablica kind → apiVersion (najčešći — srž za ispit)

| kind | apiVersion |
|------|------------|
| Pod, Service, ConfigMap, Secret, PVC, Namespace, Node | **`v1`** (core, bez grupe) |
| Deployment, ReplicaSet, StatefulSet, DaemonSet | **`apps/v1`** |
| Job, CronJob | **`batch/v1`** |
| Ingress, NetworkPolicy | **`networking.k8s.io/v1`** |
| Role, RoleBinding, ClusterRole, ClusterRoleBinding | **`rbac.authorization.k8s.io/v1`** |

**Logika za upamtiti:** „goli" osnovni objekti (Pod, Service, Config…) → **`v1`**; sve što **upravlja Podovima** (Deployment, StatefulSet, DaemonSet, ReplicaSet) → **`apps/v1`**.

### Zašto (srž za ispit)

k8s API je grupiran u verzionirane grupe; `kind` + `apiVersion` zajedno jednoznačno određuju tip. Pogrešna kombinacija → server pri discovery-ju ne nađe mapping → `no matches for kind`. To je **serverska** greška (traži klaster), ne lokalni YAML problem. Ista poruka pokriva i nedostajući CRD. Lijek je znati ili **pronaći** točan par: `kubectl api-resources` (popis svih) ili `kubectl explain <kind>` (verzija + polja).

### Gotče

- **`no matches for kind X in version Y`** = krivi (kind, apiVersion) par ILI nedostaje CRD za custom tip.
- razlika od zad. 20: tamo nevaljan YAML → **lokalna** `error converting YAML to JSON`; ovdje valjan YAML, neprepoznat par → **serverska** `no matches for kind`. Različit sloj.
- `kubectl api-resources | grep -i <kind>` → APIVERSION kolona; `kubectl explain <kind>` → `VERSION:` + struktura polja (npr. `kubectl explain deployment.spec.strategy`).
- zastarjele verzije: starije `apiVersion` (npr. `extensions/v1beta1` za Deployment/Ingress) su uklonjene → na novom klasteru daju istu grešku; koristi aktualne (`apps/v1`, `networking.k8s.io/v1`).
- `kubectl api-versions` (bez `-resources`) = popis samih grupa/verzija (bez kindova).

---

## Zadatak 37 — `CreateContainerConfigError` (fali ključ u ConfigMapu)

> **Hands-on.** Novo stanje: slika je OK, ali kontejner se ne može **konfigurirati** jer mu fali podatak iz ConfigMapa.

**Mehanizam:** Pod env-varijablu vuče iz ConfigMapa preko `configMapKeyRef` (ime + **ključ**). ConfigMap postoji ali ključ ne → kubelet ne može sastaviti config kontejnera → `CreateContainerConfigError`. Pod raspoređen, slika povučena, ali kontejner NE krene.

**Priprema — ConfigMap s jednim ključem:**
```bash
kubectl create configmap appconfig --from-literal=BOJA=plava
```
- `--from-literal=BOJA=plava` — par ključ=vrijednost; **jedini ključ je `BOJA`**

**Pokvareni manifest (traži ključ koji ne postoji):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configfail
spec:
  containers:
  - name: app
    image: nginx:1.25
    env:
    - name: VELICINA
      valueFrom:
        configMapKeyRef:
          name: appconfig
          key: VELICINA        # ← ConfigMap ima samo BOJA, ne VELICINA
```
- `valueFrom.configMapKeyRef` — vrijednost iz ConfigMapa: `name` (koji), `key` (koji ključ) ← greška

**Dijagnoza (provjereno uživo):**
```bash
kubectl apply -f configfail.yaml
kubectl get pod configfail              # STATUS: CreateContainerConfigError (0/1)
kubectl describe pod configfail
```
Ključni dijelovi `describe`-a:
```
Status: Pending
Containers: app:
  State:   Waiting
  Reason:  CreateContainerConfigError
  Environment:
    VELICINA: <set to the key 'VELICINA' of config map 'appconfig'>  Optional: false
Events:
  Warning  Failed  (x3 over 28s)  Error: couldn't find key VELICINA in ConfigMap default/appconfig
```

### Čitanje ispisa (za ispit)

- **dvije razine stanja** (kao zad. 29): `Status: Pending` (faza Poda, jer nijedan kontejner ne radi) + kontejner `State: Waiting`, `Reason: CreateContainerConfigError` (razlog čekanja). PodScheduled/Initialized `True`, Ready `False`.
- **`Optional: false`** = ključ je **obavezan** → fali li, kontejner blokiran. Da je `optional: true` u `configMapKeyRef`, **nedostajući ključ bi se preskočio** (varijabla se ne postavi, kontejner krene). Način da env bude „nice-to-have" bez rušenja.
- Events poruka **imenuje ključ I ConfigMap** (`VELICINA` u `default/appconfig`) → odmah je jasno što popraviti.
- **`(x3 over 28s)` = kubelet PONAVLJA** → zato se Pod sam oporavi čim popunimo ključ.

**Fix — dodaj ključ (Pod se SAM oporavi, bez recreate) — provjereno uživo:**
```bash
kubectl patch configmap appconfig -p '{"data":{"VELICINA":"velika"}}'
kubectl get pod configfail -w           # -w = watch (prati promjene); Ctrl+C za izlaz
```
```
configfail  0/1  CreateContainerConfigError  ...
configfail  1/1  Running                     ...   ← ~5s nakon patcha, SAM
```
- spec Poda je cijelo vrijeme valjan; falio je **vanjski podatak** → popuni se ConfigMap → kubelet u idućem pokušaju pokrene kontejner. **Ne treba `delete`+`apply`.**

### Zašto (srž za ispit)

`CreateContainerConfigError` = slika je tu, Pod je raspoređen, ali kubelet ne može **sastaviti konfiguraciju kontejnera** — najčešće referenca na ConfigMap/Secret ključ koji ne postoji (ili na ConfigMap/Secret kojeg uopće nema). Razlikuj od `ImagePullBackOff` (problem **slike**, zad. 21) i `CrashLoopBackOff` (kontejner **krene pa padne**, zad. 22/29). Pošto je greška u **podatku** a ne u spec-u Poda, ispravak podatka (patch ConfigMapa) izaziva **samostalan oporavak** — kubelet ionako ponavlja pokušaj. To je razlika od izmjene **spec-a** golog Poda (immutable → delete+apply, zad. 19/27).

### Gotče

- **`CreateContainerConfigError`** = fali ConfigMap/Secret ili **ključ** u njemu. Events imenuju što. (Ako fali cijeli ConfigMap: `configmap "X" not found`.)
- `Optional: false` (default) = obavezan ključ → blokira; `optional: true` = preskoči ako fali (kontejner krene bez te varijable).
- isto vrijedi za **Secret** preko `secretKeyRef` (ista greška, fali ključ/Secret).
- greška u **podatku** (ConfigMap/Secret) → Pod se sam oporavi nakon ispravka; greška u **spec-u** golog Poda → delete+apply.
- razlikuj stanja: `ImagePullBackOff` (slika) / `CreateContainerConfigError` (config prije starta) / `RunContainerError` (runtime ne može pokrenuti) / `CrashLoopBackOff` (krene pa izađe).
- `kubectl get ... -w` (watch) = prati prijelaze stanja uživo; `Ctrl+C` za izlaz.
- ConfigMap montiran kao **volumen** (ne env) se osvježava sam (s kašnjenjem); ConfigMap kao **env** se čita pri startu kontejnera (promjena ne ulazi u već pokrenuti kontejner — ali ovdje kontejner još nije ni krenuo, pa retry pokupi novo).

---

## Zadatak 38 — Secret/ConfigMap preko granice prostora imena (namespace)

> **Nadovezuje se na zad. 37:** ista klasa greške (`CreateContainerConfigError`), ali korijen je **zid prostora imena (namespace)**.

**Ključni koncept:** ConfigMap i Secret su **namespaced** objekti. Pod **vidi samo** ConfigMap/Secret iz **vlastitog** prostora imena. Referenciraš li Secret koji postoji u drugom prostoru imena → „not found" (kao da ne postoji). Nema gledanja preko granice.

**Postavi Secret u `default` i napravi prostor imena `drugi`:**
```bash
kubectl create secret generic apisecret --from-literal=API_KEY=12345
kubectl create namespace drugi
```
- `create secret generic apisecret` — Secret tipa `generic` (proizvoljni ključ/vrijednost)
- `--from-literal=API_KEY=12345` — par ključ=vrijednost; Secret samo **base64-kodira**, NE enkriptira
- `create namespace drugi` — novi prostor imena; Secret `apisecret` u njemu NE postoji (stvoren je u `default`)

**Pokvareni manifest (Pod u `drugi` traži Secret iz `default`):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secretfail
  namespace: drugi          # ← Pod je u prostoru imena 'drugi'
spec:
  containers:
  - name: app
    image: nginx:1.25
    env:
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: apisecret    # traži 'apisecret' u VLASTITOM prostoru imena (drugi) — a nje tu nema
          key: API_KEY
```

**Dijagnoza (provjereno uživo):**
```bash
kubectl apply -f secretfail.yaml
kubectl get pod secretfail -n drugi          # -n drugi = gledaj u tom prostoru imena!
kubectl describe pod secretfail -n drugi
```
Ključni dijelovi:
```
State:  Waiting   Reason: CreateContainerConfigError
Environment:
  API_KEY: <set to the key 'API_KEY' in secret 'apisecret'>  Optional: false
Events:
  Scheduled  → drugi/secretfail to minikube        # format: namespace/pod
  Warning  Failed  → Error: secret "apisecret" not found
```

### Čitanje ispisa (za ispit)

- identičan obrazac kao zad. 37 (`Waiting` / `CreateContainerConfigError` / `Optional: false` / kubelet ponavlja), ali Events imenuju **drugi korijen**: `secret "apisecret" not found`.
- `drugi/secretfail` (format `namespace/pod`) potvrđuje da je Pod u prostoru imena `drugi`.
- Secret **postoji** (u `default`), ali ga kubelet traži u `drugi` → za Pod je to isto kao da ne postoji. **Dokaz zida prostora imena.**
- `-n drugi` je **obavezan** na svakoj naredbi; bez njega kubectl gleda `default` i javio bi „pod not found".

**Fix — napravi Secret u TOM prostoru imena (Secret se NE dijeli; kopira se u svaki) — provjereno uživo:**
```bash
kubectl create secret generic apisecret --from-literal=API_KEY=12345 -n drugi
kubectl get pod secretfail -n drugi -w       # Running (self-heal kao zad. 37)
```
- `-n drugi` na `create secret` → Secret sad postoji u `drugi` → kubelet ga pri idućem pokušaju nađe → Pod krene **sam**, bez delete/apply.

### Zašto (srž za ispit)

Namespace je granica imenovanja i izolacije: objekti unutar njega referenciraju se kratkim imenom, ali **samo unutar istog namespacea**. ConfigMap/Secret/PVC/ServiceAccount su namespaced — Pod ih vidi isključivo u svom namespaceu. Nema sintakse za „Secret iz drugog namespacea" u `secretKeyRef`/`configMapKeyRef`/volumenu/`imagePullSecrets`. Treba li ista tajna u N namespaceova → stvara se u svih N (kopija po namespaceu). Zato `CreateContainerConfigError` ovdje znači „referenca OK, ali traženi objekt nije u MOM namespaceu".

### Gotče

- **nema cross-namespace reference** za ConfigMap/Secret/PVC. Kopiraj objekt u svaki namespace koji ga treba.
- `-n <namespace>` na SVAKOJ naredbi za ne-default namespace; inače kubectl gleda `default` (→ „not found"). `-A`/`--all-namespaces` = svi odjednom (za pregled).
- `secret "X" not found` (cijeli Secret fali / krivi namespace) vs `couldn't find key K in Secret X` (Secret tu, ključ fali, zad. 37) — različite poruke, različit fix.
- Secret samo **base64-kodira** (nije enkripcija); `kubectl get secret X -o jsonpath='{.data.API_KEY}' | base64 -d` vraća čistu vrijednost. Prava enkripcija = encryption-at-rest na etcd-u (zasebna konfiguracija).
- namespaced vs cluster-scoped: Pod/Service/ConfigMap/Secret/PVC = namespaced; Node/PV/Namespace/ClusterRole/StorageClass = cluster-scoped (bez namespacea). (`kubectl api-resources` kolona NAMESPACED, zad. 36.)
- self-heal vrijedi jer je greška u **vanjskom podatku** (Secret), ne u spec-u Poda (kao zad. 37).

---

## Zadatak 39 — `ProgressDeadlineExceeded` (rollout zaglavi)

> **Hands-on.** Signal na razini **Deploymenta** da je rollout zapeo — nova verzija se ne uspijeva podići u zadanom roku.

**Mehanizam:** Deployment ima `progressDeadlineSeconds` (default **600s** = 10 min). Tijekom rollouta, ako novi ReplicaSet ne napreduje (Podovi ne postanu dostupni) u tom roku, uvjet **`Progressing`** prijeđe u `False` s razlogom **`ProgressDeadlineExceeded`**. Tipičan uzrok: nova slika je pokvarena (crashloopa / ne povlači se). Skraćujemo rok na 10s da ne čekamo 10 minuta.

**Manifest (zdrav, skraćen rok):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rollstuck
spec:
  progressDeadlineSeconds: 10      # skraćeno s 600 na 10s (da brzo vidimo kvar)
  replicas: 2
  selector:
    matchLabels:
      app: rollstuck
  template:
    metadata:
      labels:
        app: rollstuck
    spec:
      containers:
      - name: app
        image: nginx:1.25
```

**Tijek (provjereno uživo) — zdrav, pa namjerno pokvaren rollout:**
```bash
kubectl apply -f rollstuck.yaml
kubectl get deployment rollstuck             # READY 2/2 (zdravo)
kubectl set image deployment/rollstuck app=nginx:nepostojeci-tag   # LOŠ rollout (ImagePullBackOff, kao zad. 21)
kubectl rollout status deployment/rollstuck  # ⚠️ prati ~10s pa javi GREŠKU (ne visi zauvijek)
```
```
Waiting for ... rollout to finish: 1 out of 2 new replicas have been updated...
error: deployment "rollstuck" exceeded its progress deadline
```
```bash
kubectl get deployment rollstuck             # READY 2/2 (stari!), UP-TO-DATE 1
kubectl describe deployment rollstuck
```
Ključni dijelovi `describe`-a:
```
Replicas: 2 desired | 1 updated | 3 total | 2 available | 1 unavailable
RollingUpdateStrategy: 25% max unavailable, 25% max surge
Conditions:
  Available    True    MinimumReplicasAvailable      ← app živ
  Progressing  False   ProgressDeadlineExceeded      ← rollout zapeo
OldReplicaSets: rollstuck-67ffc64c4c (2/2 created)   ← stari RS, drži 2 zdrava
NewReplicaSet:  rollstuck-76f569b8bf (1/1 created)   ← novi RS, 1 pokvaren Pod
```

### Čitanje ispisa (za ispit)

- `rollout status` prati pa nakon isteka roka ispiše `exceeded its progress deadline` (i vraća **ne-nula izlazni kod** → korisno u CI/CD da se otkrije neuspjeli deploy). Ne visi zauvijek.
- **READY 2/2 ali UP-TO-DATE 1**: app radi (2 STARA Poda), ali samo 1 Pod je nove verzije (i taj visi) → rollout nedovršen.
- **`3 total` = 2 stara + 1 surge**: `25% max unavailable, 25% max surge`. Na 2 replike: maxUnavailable 25% → zaokruži **dolje na 0** (2 stara ostaju gore), maxSurge 25% → zaokruži **gore na 1** (1 novi). Objašnjava `3 total`.
- **dva uvjeta zajedno** su srž: `Available True` (korisnici OK) **+** `Progressing False / ProgressDeadlineExceeded` (nova verzija ne ide). Deployment istovremeno kaže oboje.
- Events `Scaled up ...-67ffc64c4c to 2` (stari) i `...-76f569b8bf to 1` (novi) = lanac Deployment→RS (LO4).

**Fix — vrati na prethodnu (radnu) reviziju — provjereno uživo:**
```bash
kubectl rollout undo deployment/rollstuck
kubectl get deployment rollstuck             # READY 2/2, UP-TO-DATE 2  ✓
```
- `rollout undo` — vrati na **prethodnu reviziju** (radni `nginx:1.25`); pokvareni RS spusti na 0.
- (alternativa: `kubectl set image deployment/rollstuck app=nginx:1.25` ručno.)

### Zašto (srž za ispit)

`progressDeadlineSeconds` je „štoperica" za rollout: ako novi RS ne dovede Podove do dostupnih unutar roka, Deployment se sam označi `Progressing=False / ProgressDeadlineExceeded`. To je **upozorenje, ne akcija** — **nema automatskog rollbacka**; stari Podovi nastave raditi (zaštita rolling updatea, bez izpada), a ti ručno `rollout undo` ili popraviš sliku. Razlika od pojedinačnog Pod-kvara (zad. 21 ImagePullBackOff): tamo gledaš jedan Pod; ovdje Deployment **agregira** stanje rollouta i daje jedan jasan signal „nova verzija ne napreduje".

### Gotče

- **`ProgressDeadlineExceeded`** = rollout ne napreduje u roku (`progressDeadlineSeconds`, default 600s). NE vraća automatski.
- `READY` (stara verzija drži app) ≠ `UP-TO-DATE` (koliko ih je nove verzije). Stuck rollout: READY pun, UP-TO-DATE manjak.
- `kubectl rollout status <deploy>` = prati + ne-nula kod na neuspjeh (CI/CD gate); `kubectl rollout undo <deploy>` = vrati prethodnu reviziju; `kubectl rollout history <deploy>` = popis revizija; `kubectl rollout undo --to-revision=N` = na točnu reviziju.
- stari Podovi rade jer `maxUnavailable` (default 25%) na malom broju replika zaokruži na 0 → Deployment ne ruši staro dok novo nije ready. Zato „stuck rollout" obično NE znači izpad.
- uzrok stuck rollouta je gotovo uvijek na novim Podovima (ImagePullBackOff zad. 21, CrashLoop zad. 22/29, CreateContainerConfigError zad. 37, readiness fail zad. 24) — `describe`/`get pods` novog RS-a kaže koji.
- veza: LO4 rolling update / revizije / RS lanac; zad. 21 (ImagePullBackOff kao čest uzrok).

---

## Zadatak 40 — `--dry-run=server` / `--validate` (provjera prije stvaranja)

> **Predigra bila zad. 20** (`--dry-run=client`). Dvije razine „probe" prije pravog `apply`.

**Pozadina:**
- **`--dry-run=client`** (zad. 20): kubectl parsira/validira **lokalno**, **ne šalje** serveru, ništa ne snima. Hvata YAML; **može propustiti** grešku u polju.
- **`--dry-run=server`**: **pošalje** serveru na **punu** obradu — validacija + **admission** (webhookovi, quota, policy) + **defaulting** + provjere stanja — ali **ne snima**. Najstroža „hoće li stvarno proći".

**Pokvareni manifest (typo u nazivu polja):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dryrun
spec:
  containers:
  - name: app
    image: nginx:1.25
    imagePullPolicyy: Always      # ← typo: treba imagePullPolicy (dvostruko 'y')
```

**Korak 1 — usporedi obje provjere (provjereno uživo):**
```bash
kubectl apply -f dryrun.yaml --dry-run=client     # → pod/dryrun created (dry run)   ← PROPUSTIO typo!
kubectl apply -f dryrun.yaml --dry-run=server     # → strict decoding error ...       ← UHVATIO!
```
```
# client:
pod/dryrun created (dry run)
# server:
Error from server (BadRequest): ... Pod in version "v1" cannot be handled as a Pod:
strict decoding error: unknown field "spec.containers[0].imagePullPolicyy"
```

### Čitanje ispisa (za ispit) — KLJUČNA razlika

- **client je PROPUSTIO** typo (`created (dry run)`) — lokalna provjera ovdje nije bila stroga prema nepoznatim poljima.
- **server je UHVATIO** (`strict decoding error: unknown field "spec.containers[0].imagePullPolicyy"`) + **točan put do polja**.
- → **server validacija je stroža od client.** Ono što client propusti, server odbije. Zato je `--dry-run=server` prava „hoće li stvarno proći" provjera prije `apply`.
- `get pod dryrun` → `NotFound` → nijedan dry-run nije ništa stvorio (ne dira klaster).

**Korak 2 — defaulting (popravi typo, server vrati objekt s defaultima) — provjereno uživo:**
```bash
# imagePullPolicyy → imagePullPolicy
kubectl apply -f dryrun.yaml --dry-run=server -o yaml | head -40
```
Polja koja NISMO napisali, a server ih je dodao (8-redni manifest → puni objekt):
```
imagePullPolicy: Always           # (za nginx:1.25 bez navođenja → IfNotPresent)
resources: {}
terminationMessagePath: /dev/termination-log
volumeMounts: kube-api-access-... # automatski montiran ServiceAccount token
dnsPolicy: ClusterFirst           # puni /etc/resolv.conf (zad. 31)
restartPolicy: Always             # default koji je „crashloopao" goli Pod (zad. 29)
schedulerName: default-scheduler  # javljao FailedScheduling (zad. 19/27/28)
serviceAccountName: default
terminationGracePeriodSeconds: 30
tolerations: not-ready/unreachable ... 300s   # 5-min iseljenje s palog čvora (zad. 34)
```

### Zašto (srž za ispit)

Dvije „probe": `client` = brza lokalna provjera (sintaksa, shema iz cachea) bez kontakta sa serverom; `server` = puna obrada kao da stvaraš (validacija + admission + defaulting + provjere stanja) **bez snimanja**. Server je stroži — hvata što client propusti (ovaj typo) i pokazuje **stvarni** objekt s popunjenim defaultima. Mnogo ponašanja iz LO5 (DNS `ClusterFirst`, `restartPolicy: Always`, scheduler, 5-min tolerancije) dolazi iz tih **tiho dodanih defaulta** — `--dry-run=server -o yaml` ih sve pokaže na jednom mjestu.

### Gotče

- **`--dry-run=client` MOŽE propustiti grešku u polju** (ovisno o verziji/strogosti); **`--dry-run=server` je stroži** (admission + strict decoding). Za pravu provjeru koristi **server**.
- `--dry-run=server -o yaml` = prikazuje objekt sa svim server-defaultima PRIJE snimanja (provjera „što će stvarno biti").
- `--validate=strict` (default od k8s 1.25) odbija nepoznata/duplicirana polja; `--validate=warn` (upozori, ne blokira); `--validate=ignore` (preskoči). Strogost client-strane ovisi i o ovome.
- `strict decoding error: unknown field "..."` = typo/nepoznato polje; put u zagradi (`spec.containers[0]...`) kaže gdje.
- dry-run NIKAD ne snima — `get` poslije pokaže `NotFound`. Siguran za vježbu na živom klasteru.
- veza: zad. 20 (`--dry-run=client` za YAML typo) — par: client za sintaksu, server za sve ostalo.

---

## Zadatak 41 — lanac dijagnoze „app ne doseže Servis" (pod → svc → endpoints → DNS → port)

> **Metoda, ne pojedinačni kvar.** Kad „aplikacija ne može do servisa", prolazi se **redom niz lanac** od 5 karika; svaka isključi jedan sloj. Sinteza: zad. 24 (readiness), 25 (Endpoints), 31 (DNS), 33 (povezanost).

**Scenarij s namjernim kvarom — krivi `targetPort`:**
```bash
kubectl create deployment shop --image=nginx:1.25
kubectl expose deployment shop --port=80 --target-port=8080
```
- `--port=80` = port Servisa (gdje klijenti kucaju); `--target-port=8080` = port na Podu (kamo Servis prosljeđuje) ← **kvar** (nginx unutra sluša na **80**, ne 8080)

**Lanac — 5 karika (provjereno uživo):**
```bash
# 1. POD — backend Running i READY?
kubectl get pods -l app=shop          # shop-... 1/1 Running ✓
# 2. SERVICE — postoji, koji portovi?
kubectl get svc shop                  # ClusterIP 10.109.232.164, 80/TCP ✓
# 3. ENDPOINTS — backend IP-evi, NA KOJEM PORTU?
kubectl get endpoints shop            # 10.244.0.68:8080  ← TRAG! popunjeno ali :8080
# 4+5. DNS + POVEZANOST — iz Pod-jednokratke
kubectl run tmp --image=busybox:1.36 --rm -it --restart=Never -- sh
#   nslookup shop                     → shop.default.svc.cluster.local → 10.109.232.164 ✓
#   wget -q -O - http://shop          → can't connect ... Connection refused
```

### Čitanje lanca (za ispit) — što svaka karika isključuje

- **(1) Pod** `1/1 Running` → backend živ i ready; `0/1` → readiness fail (zad. 24), dalje nema smisla.
- **(2) Service** postoji, ima `PORT(S)` → objekt OK.
- **(3) Endpoints** — **ključ**: prazno (`<none>`) → selector mismatch / Podovi ne-ready (zad. 25); **popunjeno → gleda se NA KOJEM PORTU**. Ovdje `:8080` = trag (Servis šalje na port gdje nitko ne sluša).
- **(4) DNS** `nslookup shop` → ClusterIP → DNS OK (zad. 31).
- **(5) Povezanost** `wget http://shop` → padne unatoč svemu „OK" → problem je **port**.

**Connection refused vs timeout (potvrda iz zad. 33):**
- ovdje **`Connection refused`** (ne timeout): Servis JE proslijedio na `podIP:8080`, kontejner dosegnut, ali na 8080 nitko ne sluša → kernel aktivno odbija (RST). Paket **stigne do Poda** → refused.
- (krivi Service port bez ikakvog backenda bi dao timeout/blackhole — zad. 33.)

**Fix — ispravi `targetPort` na 80 — provjereno uživo:**
```bash
kubectl patch svc shop -p '{"spec":{"ports":[{"port":80,"targetPort":80}]}}'
kubectl get endpoints shop            # 10.244.0.68:80  ← Endpoints odmah prepisan na :80
# ponovni wget iz tmp → nginx welcome HTML ✓
```
- `targetPort: 80` → Endpoints controller odmah prepiše endpoint na `:80` (gdje nginx stvarno sluša) → promet prolazi.

Prije/poslije na istom scenariju:
```
targetPort 8080:  endpoints :8080  →  wget  →  Connection refused
targetPort 80:    endpoints :80    →  wget  →  HTML ✓
```

### Zašto (srž za ispit)

„App ne doseže Servis" ima više mogućih uzroka na različitim slojevima; umjesto nagađanja prolazi se lanac redom i svaka karika eliminira jedan sloj: **Pod ready?** → **Servis postoji?** → **Endpoints popunjen i na pravom portu?** → **DNS razriješi?** → **povezanost prolazi?**. Prvi korak koji „padne" lokalizira kvar. Ovdje su 1–4 izgledali OK, ali Endpoints na `:8080` + `Connection refused` na 5 pokazali su da je `targetPort` kriv. `targetPort` mora biti port na kojem app **stvarno sluša**; Endpoints to ogledaju (`podIP:targetPort`), pa je `get endpoints` često najbrži trag.

### Gotče (uklj. 3 ulovljene uživo)

- **redoslijed lanca:** Pod → Service → Endpoints → DNS → port. Prvi pad lokalizira sloj. `get endpoints` je najinformativnija pojedinačna naredba.
- **Endpoints popunjen ALI krivi port** (`:8080`) = `targetPort` ne odgovara portu aplikacije. Prazan Endpoints (`<none>`) = selector mismatch / ne-ready Podovi (zad. 25). Različita kvara, oba kroz `get endpoints`.
- **`Connection refused`** (paket stigao do Poda, port zatvoren) vs **`timeout`** (paket nigdje ne stigao, zad. 33) — refused znači da je meta bliže.
- **busybox `wget` zamka (ULOVLJENO):** ovaj build ne voli spojeno `-qO-` → `invalid option -- '0'`. Razdvoji: **`wget -q -O - http://shop`** (razmak oko `-O -`). Timeout flag: `--timeout=3` (ne uvijek `-T`).
- **typo u imenu (ULOVLJENO):** `http://show` umjesto `http://shop` → `bad address 'show'` (DNS ne nađe). Paste ne radi → tipfeleri; provjerava se ime.
- `--port` (Servis) vs `--target-port` (Pod) kod `expose` — lako zamijeniti; `--target-port` mora pogoditi port aplikacije.
- veza: ovo objedinjuje zad. 24 (readiness → Pod ready), 25 (Endpoints/selector), 31 (DNS), 33 (wget/timeout-refused). Ako je potrebna jedna „master" vježba za mrežu — ovo je ona.

---

## Zadatak 42 — `jsonpath` (vađenje `state` / `lastState` / `restartCount`)

> Ta polja se vide u `describe` (zad. 22, 29). Sada se **precizno vade** s `-o jsonpath` — za skripte, brze provjere, monitoring, bez skrolanja kroz cijeli `describe`.

**Pod koji crashloopa (da imamo restartCount > 0 i lastState s exit kodom):**
```bash
kubectl run jptest --image=busybox:1.36 -- sh -c "echo start; sleep 3; exit 1"
```
- `kubectl run jptest` — goli Pod; bez `--restart` → default `restartPolicy: Always`
- `-- sh -c "...; exit 1"` — radi 3s pa padne s kodom **1** → CrashLoopBackOff (kao zad. 22). Pričekaj ~20s.

**Vađenje polja (provjereno uživo) — `{"\n"}` umjesto `echo` za novi red:**
```bash
kubectl get pod jptest -o jsonpath='{.status.containerStatuses[0].restartCount}{"\n"}'
kubectl get pod jptest -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}{"\n"}'
kubectl get pod jptest -o jsonpath='{.status.containerStatuses[0].state}{"\n"}'
```
Stvarni ispis:
```
5
1
{"waiting":{"message":"back-off 2m40s restarting failed container=jptest ...","reason":"CrashLoopBackOff"}}
```

**Razlaganje jsonpath sintakse:**
- `-o jsonpath='{...}'` — ispiši po **JSONPath** izrazu (umjesto cijelog objekta)
- `{.status.containerStatuses[0].restartCount}` — navigacija: `status` → `containerStatuses` (array) → `[0]` prvi kontejner → `restartCount`
- `{"\n"}` — umetni novi red (jsonpath ne dodaje newline; bez ovoga ispis se zalijepi za prompt)
- može se miješati doslovni tekst i polja: `'Restarts={...} LastExit={...}{"\n"}'` → `Restarts=5 LastExit=1`

### Čitanje ispisa (za ispit) — `state` vs `lastState`

- **`restartCount` → `5`** = broj restarta, jedan broj, bez describe-a.
- **`state`** = gdje je kontejner **SAD** (`running` / `waiting` / `terminated`). Ovdje `waiting` + `CrashLoopBackOff` (+ koliko backoff traje).
- **`lastState`** = gdje je bio **PRIJE** zadnjeg restarta. Ovdje `terminated` + **`exitCode: 1`** (naš `exit 1`).
- za CrashLoop dijagnozu gledaš **`lastState.terminated.exitCode`** — trenutni `state` je samo „čeka", a **zašto** pada krije se u `lastState` (analogno `logs --previous`, zad. 22).

### Zašto (srž za ispit)

`-o jsonpath` kirurški izvuče točno polje iz bilo kojeg objekta — neprocjenjivo za skripte, monitoring i brze provjere, umjesto kopanja po `describe`/`-o yaml`. Struktura statusa kontejnera: `status.containerStatuses[i]` ima `restartCount`, `state` (trenutno: running/waiting/terminated) i `lastState` (prethodno). `state` kaže gdje je sad, `lastState` zašto je tu (zadnji exit). Isti princip vrijedi za sva polja objekta.

### Gotče (uklj. ulovljeno uživo)

- **ULOVLJENO:** `echo` mora ići u **zaseban redak** (Enter pa `echo`), inače ga kubectl shvati kao argument naredbe → `pods "echo" not found`. Čišće: ugradi `{"\n"}` u sam jsonpath.
- **`state`** (trenutno) vs **`lastState`** (prethodno) — za „zašto crashloopa" gledaš `lastState.terminated.exitCode` (exit kod), ne trenutni `state` (samo „waiting").
- `containerStatuses[0]` = prvi kontejner; za Pod s više kontejnera mijenjaj indeks ili `[*]` (svi) / filtriraj po imenu.
- korisni jsonpath izrazi: `{.status.podIP}`, `{.spec.containers[0].image}`, `{.status.phase}`, `{.data.API_KEY}` (Secret → `| base64 -d`), `{.items[*].metadata.name}` (sva imena iz liste), `{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}` (petlja po listi).
- alternative: `-o custom-columns=NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount` (tablica); `-o json | jq '...'` (puna jq snaga).
- jsonpath se piše u **jednostrukim** navodnicima u bashu (da `$`/`{}` ne dira shell).

---

## Zadatak 43 — OpenShift Route vs Kubernetes Ingress (teorija)

> **Čista teorija** — na ovom VM-u nema OpenShifta ni `oc`-a, pa se ne izvodi. (Zapravo LO6-tip pitanja.)

**Oba rješavaju isti problem:** izložiti HTTP(S) servis prema van preko hostname-a (`shop.primjer.com` → Servis → Podovi). Sjede **iznad** Servisa — Servis radi L4 (ClusterIP/NodePort), Route/Ingress dodaju **L7 HTTP routing** (host/path pravila, TLS).

**Ingress (standardni Kubernetes):**
- API objekt koji **opisuje pravila** (host/path → koji Servis, koji port) + TLS preko Secreta
- **sam po sebi NE radi ništa** — treba **Ingress Controller** (nginx-ingress, Traefik, HAProxy…) koji gleda Ingress objekte i konfigurira pravi proxy. Bez controllera Ingress je samo papir.
- **prenosiv** — radi na svakom k8s klasteru (vendor-neutralan standard)

**Route (OpenShift):**
- OpenShiftov **vlastiti** objekt, **stariji od Ingressa**
- **router ugrađen** (HAProxy) — ne instaliraš ništa, radi odmah
- bogatiji TLS „out-of-the-box": **edge** (TLS prekida router, plain do Poda), **passthrough** (TLS ravno do Poda), **re-encrypt** (router prekine pa ponovo enkriptira do Poda)
- **samo OpenShift** (nije prenosiv na čisti k8s)

**Kako koegzistiraju (ključno):** OpenShift podržava **oba**. Napraviš li na OpenShiftu **Ingress**, Ingress Operator ga **automatski pretvori u Route** ispod haube. Route = nativni, bogatiji oblik; Ingress = prenosivi standard koji se na OpenShiftu svodi na Route.

| | **Ingress** (k8s) | **Route** (OpenShift) |
|--|--|--|
| Podrijetlo | k8s standard | OpenShift-nativ (stariji) |
| Controller | zaseban, instaliraš ga | ugrađen HAProxy router |
| TLS | edge (preko Secreta) | edge / passthrough / re-encrypt |
| Prenosivost | svaki k8s | samo OpenShift |
| Na OpenShiftu | pretvori se u Route | nativni |

**Obranjive teze (svaka je obranjiva):**
- *„Biram Ingress"* → prenosivost: isti manifest na bilo kojem k8s, bez vezivanja za vendora.
- *„Route je moćniji na OpenShiftu"* → ugrađen router (nula setupa) + bogatiji TLS (passthrough/re-encrypt) koji standardni Ingress povijesno nije imao.
- *„Nije ili-ili"* → na OpenShiftu pišeš Ingress radi prenosivosti, a platforma ga realizira kao Route.

### Zašto (srž za ispit)

Servis sam izlaže Podove na L4 (ClusterIP unutar klastera, NodePort/LoadBalancer prema van) — bez HTTP svijesti. Za HTTP routing po hostu/putanji i TLS treba sloj iznad: **Ingress** (k8s standard, treba controller, prenosiv) ili **Route** (OpenShift-nativ, ugrađen router, bogatiji TLS, neprenosiv). OpenShift premosti razliku: Ingress objekt → automatski Route. Promet na kraju uvijek teče istim lancem: Ingress/Route → **Servis → Endpoints → Pod** (zad. 41).

### Gotče / veze

- Ingress objekt bez **Ingress Controllera** = ništa se ne događa (česta zamka). Route ima router ugrađen.
- Route/Ingress su **iznad** Servisa (ClusterIP/NodePort/LoadBalancer iz LO4), ne zamjena za njega.
- na rootless minikubeu `LoadBalancer` ostaje `<pending>` (nema `minikube tunnel`) → za HTTP izlaganje na pravom klasteru ide Ingress + controller (ili `minikube addons enable ingress`).
- TLS terminacija: edge (najčešće), passthrough (end-to-end TLS do Poda), re-encrypt (oboje) — Route ih ima nativno; Ingress ovisi o controlleru.
- veza: lanac dijagnoze (zad. 41) vrijedi i ispod Ingressa/Routea — ako „app nedostupna izvana", provjeravaju se i Ingress/Route pravila I donji lanac (svc/endpoints/port).

---

## Brza referenca

**Podman (Skupina A) — dijagnoza pogrešnog `run`:**
- `podman ps -a` — i zaustavljeni/izašli kontejneri (`STATUS` kaže `Exited (kod)`).
- `podman logs <ime>` — stdout/stderr; često doslovno imenuje uzrok pada.
- `podman port <ime>` — mapiranja porta (prazno uz `--network host`).
- `Exited (0)` = uredan izlazak (fali dugotrajan proces) · `Exited (137)` = OOM (premalo memorije).
- host port uvijek **≥ 1024** (rootless ne smije privilegirane) · `-p host:kontejner` (desno = port gdje proces sluša).
- DNS po imenu radi tek na **custom** mreži (`podman network create`), ne na default.
- bind-mount: apsolutna putanja + `:Z` (SELinux relabel; na ovom VM-u radi i bez, ali `:Z` je prenosiv odgovor).

**Containerfile (Skupina B):**
- `apt-get update && apt-get install ... && rm -rf /var/lib/apt/lists/*` — sve u **istom** `RUN` (sloju).
- `CMD ["bin","arg"]` (exec forma) za ispravne signale; shell forma stavlja `sh` kao PID 1.
- manifest (`package*.json` / `requirements.txt`) u zaseban sloj **iznad** instalacije → cache.
- `ENV PATH=/app/bin:$PATH` (proširi, ne pregazi) · `EXPOSE` samo dokumentira, port objavljuje `-p`.
- `USER` što kasnije (tik prije `CMD`); privilegirani build koraci kao root.
- multi-stage (`FROM ... AS build` + `COPY --from=build`) → finalna slika nosi samo artefakt.

**Kubernetes (Skupina C) — glavni alati:**
- `kubectl describe <resurs>` → `Events:` na dnu obično kažu uzrok.
- `kubectl logs <pod> --previous` → log srušene instance (CrashLoop).
- `kubectl get events --sort-by=.metadata.creationTimestamp` (ili `kubectl events`) → kronologija.
- `kubectl get endpoints <svc>` → `<none>` (selector/label mismatch) ili krivi port.
- `kubectl exec -it <pod> -- sh` (ima shell) · `kubectl debug -it <pod> --image=busybox --target=<c> -- sh` (distroless).
- `kubectl apply --dry-run=server` (stroža provjera, hvata typo polja) vs `--dry-run=client` (lokalno, blaže).
- `kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'` → exit kod.

**Statusi i njihov potpis:**
- `Pending` — scheduler ne nalazi mjesto: `Insufficient cpu/memory` (resursi) ili `persistentvolumeclaim not found` (volumen).
- `ImagePullBackOff` — slika/tag ne postoji, typo, ili privatni registar bez `imagePullSecret`; `RESTARTS 0`.
- `CrashLoopBackOff` — kontejner krene pa izađe; `RESTARTS` raste; uzrok u `lastState`/`logs --previous`.
- `OOMKilled` — `Reason: OOMKilled`, `Exit Code 137`; podići `limits.memory` ili popraviti aplikaciju.
- `0/1 Running RESTARTS 0` — readiness probe pada (Pod ispada iz Endpoints, ne restarta se).
- `Completed` + CrashLoop — `Exit Code 0`, fali dugotrajan proces (ili `restartPolicy: Never/OnFailure`).
- `CreateContainerConfigError` — fali ConfigMap/Secret ili ključ u njemu; Pod se sam oporavi nakon ispravka podatka.
- `Terminating` zaglavljen — finalizer (`patch ... finalizers:null`) ili nedostupan čvor (`--force --grace-period=0`).
- `ProgressDeadlineExceeded` — rollout ne napreduje u roku; `kubectl rollout undo`.
- `NotReady` (čvor) — kubelet/disk/CNI/runtime; `Ready False` (javlja problem) vs `Unknown` (heartbeat zastao).

**kind → apiVersion:** Pod/Service/ConfigMap/Secret/PVC = `v1` · Deployment/StatefulSet/DaemonSet/ReplicaSet = `apps/v1` · Job/CronJob = `batch/v1` · Ingress/NetworkPolicy = `networking.k8s.io/v1`.

**VM-napomene (rootless, bez sudo):** `minikube tunnel` ne radi → LoadBalancer ostaje `<pend