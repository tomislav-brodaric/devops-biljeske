# DevOps — LO3 (isporuka aplikacija, mrežna arhitektura, sigurnost komponenata)

> **Što je ovo:** kompletan LO3, svih 23 zadatka, grupirano po konceptu (A–F). Za svaki zadatak: tekst → naredbe → **svaka zastavica razložena** → ⚠️ začkoljice → sažetak u 3 rečenice.
>
> **Status:** pisano za čitanje i potom izvođenje na VM-u. Naredbe nisu sve istjerane na faks-VM-u. Sve gdje VM zna odstupati označeno je s **⚠️ ZA PROVJERU** uz PREFLIGHT listu odmah ispod.
>
> **Okruženje:** rootless Podman, minikube, kubectl, bez `sudo`.
>
> **Ispit:** praktičan (pisani, uz otvorene bilješke). Dozvoljena je službena dokumentacija + GitHub bilješke. Cilj je tečnost u izvođenju.

## Rječnik kratica

- **Pod** — najmanja jedinica raspoređivanja; grupa kontejnera koji dijele dio Linux prostora imena (mrežu, IPC, UTS).
- **DNS** (Domain Name System) — sustav koji prevodi ime u IP adresu.
- **IP** (Internet Protocol) — mrežna adresa kontejnera/računala.
- **PID** (Process ID) — identifikator procesa; svaki kontejner ima vlastiti PID 1.
- **IPC** (Inter-Process Communication) — međuprocesna komunikacija (dijeljena memorija, semafori).
- **UTS** (Unix Time-Sharing) — prostor imena koji nosi hostname.
- **TCP** (Transmission Control Protocol) — protokol za pouzdane mrežne veze.
- **SQL** (Structured Query Language) — jezik za upite nad bazom.
- **DB** (database) — baza podataka.
- **SELinux** (Security-Enhanced Linux) — sigurnosni podsustav koji traži oznaku (`:Z`/`:z`) na bind mountovima.
- **YAML** (YAML Ain't Markup Language) — format za deklarativne manifeste.
- **LB** (Load Balancer) — raspoređivač prometa preko više replika.
- **PV / PVC** (PersistentVolume / PersistentVolumeClaim) — trajni svezak / zahtjev za trajnim sveskom (k8s).
- **k8s** — Kubernetes.
- **VM** (Virtual Machine) — virtualni stroj na kojem se izvode zadaci.
- **UID** (User ID) — identifikator korisnika unutar kontejnera.

---

## PREFLIGHT — provjera ovoga na VM-u prije svega (5 naredbi)

Prvo se izvodi ovo, radi uvida u zatečeno stanje:

```bash
podman compose version        # 1. radi li compose uopće? (kritično za Grupu D)
podman kube generate --help   # 2. radi li "kube generate" (novi) ili treba "generate kube" (stari)
podman secret --help          # 3. radi li secret podsustav (Grupa C, zad. 5)
podman network ls             # 4. koje mreže već postoje
podman images                 # 5. koje slike imaš lokalno
```

**Što očekivati / na što paziti:**
- **`podman compose version`** — ako vrati verziju, u redu je. Ako javi *"unrecognized command"* ili traži providera → Grupa D treba `podman-compose` ili `docker-compose` instaliran. Ako padne, to je blokada za Grupu D.
- **`kube generate` vs `generate kube`** — Podman je preimenovao naredbu. Novi oblik: `podman kube generate`. Stari (i dalje radi kao alias): `podman generate kube`. U skripti se koristi **novi**, a stari se spominje uz svaki zadatak.
- **`--cpus` rootless** — ⚠️ **nepotvrđeno da radi.** Rootless `--cpus` zna pasti ("cpu cgroup controller" nedostupan bez delegacije). Ne valja se oslanjati na to — ako padne, zamjena je samo `--memory` / `mem_limit`. Provjerava se na licu mjesta.
- **SELinux** — CentOS ima SELinux uključen → svaki **bind mount** treba `:Z` ili `:z` na kraju ili slijedi *"permission denied"*. Imenovani volumeni se relabeliraju sami (ne trebaju to).
- **DNS** — ⚠️ **default mreža NEMA DNS, custom mreža IMA.** To je najveća začkoljica cijelog LO3.

---

## OBNOVA OKRUŽENJA NAKON RESETA

VM se resetira između sesija — obrišu se radne mape i većina slika, odjava s registara. To je normalno. Obnova:

```bash
mkdir -p ~/devops && cd ~/devops
podman version
podman pull docker.io/library/nginx
podman pull docker.io/library/postgres:16
podman pull docker.io/library/alpine
podman pull docker.io/library/adminer
```

- `mkdir -p ~/devops` — napravi radnu mapu. `-p` = napravi i roditeljske mape ako fale i **ne buni se** ako mapa već postoji.
- `&& cd ~/devops` — ulazak u nju, ali tek ako je `mkdir` uspio (`&&` = pokreće desno samo ako lijevo prođe).
- `podman version` — provjerava da motor zdravo odgovara.
- `podman pull ...` — skini javne slike s registra (`docker.io`) na lokalni disk. Za javne slike **ne treba login**. (`gitea/gitea` skida se u Grupi E.)

⚠️ Ako pull padne s *"unauthorized"* ili *rate limit* → tada slijedi prijava na Docker Hub (`cat ~/docker/token | podman login docker.io -u tomislavb16 --password-stdin`), ali prvo se pokušava bez.

**Slike koje se koriste kroz cijeli LO3:** `nginx` (app/web, ima bash → `/dev/tcp` trik radi), `postgres:16` (baza), `alpine` (sićušni pomoćni kontejner), `adminer` (web DB-admin), `gitea/gitea` (dvoslojni stack u Grupi E).

> ⚠️ **Zašto `postgres:16`, a ne samo `postgres`?** `postgres` (= `postgres:latest`) je sada **verzija 18**, koja je **promijenila gdje očekuje podatke**: stari postgres (15/16/17) koristi `/var/lib/postgresql/data`, a 18 očekuje `/var/lib/postgresql` (pa unutra sam pravi poddirektorij po verziji). Cijela ova skripta montira volumen na **staru** putanju `/var/lib/postgresql/data`, pa s `latest` (18) postgres **odbije start i kontejner padne** ("Exited (1)", a u `podman logs` piše da su podaci na "unused mount/volume"). Rješenje: **kroz cijelu skriptu koristi se `postgres:16`** — verzija 16 koristi staru putanju, pa svi zadaci rade kako su napisani. Bonus: prikivanje verzije je upravo **reproducibilnost** (LO2 pojam) — `latest` se mijenja pod nogama, prikovana verzija ne.

---
---

# GRUPA A — PODOVI (zadaci 1, 18)

## Temelj grupe: prostor imena + Pod

**Prostor imena (namespace)** = Linux mehanizam izolacije. Kernel procesu da vlastiti *pogled* na neki resurs pa proces misli da je sam na stroju. Relevantni tipovi:

- **network namespace** — vlastita mrežna sučelja, vlastiti `localhost` (loopback), vlastiti IP. Ovdje je sva magija poda.
- **PID namespace** — vlastito numeriranje procesa; svaki kontejner ima svoj **PID 1**.
- **mount namespace** — vlastiti pogled na datotečni sustav.
- **IPC namespace** — vlastita međuprocesna komunikacija (dijeljena memorija, semafori).
- **UTS namespace** — vlastiti hostname.

**Kontejner** = proces zamotan u svoj set ovih prostora imena → misli da je izolirani stroj.

**Pod** = grupa kontejnera koji **dijele** dio prostora imena. U Podmanu dijele **network + IPC + UTS**, a **NE dijele** (po defaultu) **PID + mount**.

Ključ: jer dijele **network namespace**, dva kontejnera u istom podu vide se preko **`localhost`-a** — kao dva programa na istom računalu. Zato se port objavljuje **na podu**, ne na pojedinom kontejneru.

**Infra kontejner** — kad se napravi pod, Podman pokrene jedan sićušni dodatni kontejner ("infra") koji **drži dijeljene prostore imena na životu** dok app-kontejneri dolaze i odlaze. Zato `podman pod ps` pokaže jedan kontejner više nego što je dodano.

> 🪲 **LO6 hook:** ovaj "pod" nije Podmanova izmišljotina — to je isti koncept kao **Kubernetes Pod**, namjerno kopiran. Rečenica za LO6 esej: *"Pod je najmanja jedinica raspoređivanja — grupa kontejnera koji dijele mrežu i nalaze se preko localhosta."*

---

## Zadatak 1 — Pod s objavljenim portom, app + baza, dokaz da se vide preko localhosta

```bash
podman pod create --name webpod -p 8080:80
```
- `podman pod create` — napravi novi pod (infra kontejner + dijeljeni prostori imena).
- `--name webpod` — ime poda je `webpod`.
- `-p 8080:80` — objavi port **na podu**. Mapira host:8080 → pod:80. Format je `hostPort:podPort`. **Port ide na POD, ne na kontejner.**

```bash
podman run -d --pod webpod --name web nginx
```
- `podman run` — napravi + pokreće kontejner.
- `-d` — *detached* (pozadina), odmah vrati prompt.
- `--pod webpod` — ubaci ovaj kontejner **u pod** `webpod` (dijeli njegov network namespace).
- `--name web` — ime kontejnera je `web`.
- `nginx` — slika (web server koji sluša na portu 80).

⚠️ **Začkoljica:** kad kontejner ulazi u pod, **`-p` se NE stavlja na kontejner** — pod već drži portove. Ako se pokuša `-p` na kontejneru u podu, Podman pukne ("cannot set port bindings...").

```bash
podman run -d --pod webpod --name db -e POSTGRES_PASSWORD=tajna postgres:16
```
- `-d` — pozadina.
- `--pod webpod` — i baza ide u **isti** pod (dijeli isti localhost s `web`).
- `--name db` — ime `db`.
- `-e POSTGRES_PASSWORD=tajna` — postavi env varijablu. **Postgres slika OBAVEZNO traži ovo** ili odbije startati. `tajna` = lozinka (demo vrijednost).
- `postgres:16` — slika baze (prikovana verzija; `latest` je sada 18 koja mijenja putanju podataka i pada — vidi napomenu o slikama na vrhu).

**Dokaz da se vide preko localhosta** (bez ijedne instalacije — koristi bash builtin `/dev/tcp`):
```bash
podman exec web bash -c "echo > /dev/tcp/localhost/5432 && echo OPEN || echo CLOSED"
```
- `podman exec web` — pokreće naredbu unutar kontejnera `web`.
- `bash -c "..."` — pokreće string kao bash naredbu.
- `echo > /dev/tcp/localhost/5432` — bash trik: `/dev/tcp/host/port` nije prava datoteka nego pseudo-uređaj; otvaranje znači "pokušaj TCP spojiti se na taj host:port". Ovdje se testira `localhost:5432` (postgres).
- `&& echo OPEN || echo CLOSED` — ako spajanje uspije → ispiši `OPEN`, ako padne → `CLOSED`.

→ Mora ispisati **`OPEN`**. To znači: iz `web` kontejnera vidljiv je postgres na `localhost:5432`, jer **dijele network namespace**. To je dokaz.

**Dodatno — vidljiv infra kontejner (+1):**
```bash
podman pod ps
podman ps --pod
```
- `podman pod ps` — popis podova; stupac s brojem kontejnera pokaže **3** (web, db, infra).
- `podman ps --pod` — popis kontejnera **sa stupcem poda**; vidljivi su `web`, `db` i `…-infra`.

**Sažetak (3 rečenice):** Pod objavljuje port na sebi (`pod create -p`), a kontejneri se ubacuju s `--pod`. Jer dijele network namespace, `web` doseže `db` preko `localhost:5432` — dokazano `/dev/tcp` trikom bez ikakve instalacije. `pod ps` pokaže kontejner više od dodanog jer infra kontejner drži dijeljene prostore imena.

---

## Zadatak 18 — Inspect poda: koje prostore imena dijeli (network, IPC), koje ne (PID)

```bash
podman pod inspect webpod
```
- `podman pod inspect webpod` — ispiši puni JSON opis poda. Traži se polje **`SharedNamespaces`** → tu piše `net`, `ipc`, `uts` (to su dijeljeni).

Brzo izdvajanje samo toga:
```bash
podman pod inspect webpod | grep -A6 SharedNamespaces
```
- `| grep -A6 SharedNamespaces` — proslijedi ispis u `grep`, nađi redak `SharedNamespaces` i pokaži **6 redaka iza** (`-A6` = *after 6*).

**Dokaz da PID NIJE dijeljen** (svaki kontejner ima svoj PID 1):
```bash
podman exec web cat /proc/1/comm
podman exec db cat /proc/1/comm
```
- `cat /proc/1/comm` — ispiši ime procesa koji je **PID 1** unutar tog kontejnera. `/proc/1/comm` je uvijek dostupno, ne treba `ps`.
- `web` → `nginx`, `db` → `postgres`. **Različiti PID 1 = odvojeni PID namespace.** Da ga dijele, drugi kontejner ne bi imao svoj PID 1.

(Dokaz da network JEST dijeljen već postoji iz zad. 1 — `localhost:5432` je radio.)

⚠️ **Začkoljica:** ne treba očekivati `ps -ef` u nginx slici — minimalne slike često nemaju `procps`. Zato `cat /proc/1/comm`, koji uvijek radi.

**Sažetak (3 rečenice):** `podman pod inspect` u polju `SharedNamespaces` pokaže da pod dijeli network, IPC i UTS. Da PID nije dijeljen dokazuje `/proc/1/comm` — `web` ima PID 1 = nginx, `db` ima PID 1 = postgres, dakle odvojeni PID prostori imena. Dijeljenu mrežu već potvrđuje localhost-doseg iz prošlog zadatka.

---
---

# GRUPA B — MREŽE I DNS (zadaci 2, 19, 20)

## Temelj grupe: most (bridge) mreža + ugrađeni DNS

**Most (bridge) mreža** = virtualni switch koji Podman napravi; kontejneri spojeni na isti most mogu pričati međusobno.

**DNS** = sustav koji prevodi **ime → IP**. Bez DNS-a potreban je IP; s DNS-om dovoljno je ime.

⚠️ **NAJVAŽNIJA začkoljica cijelog LO3:**
- **Default mreža** (zove se `podman`) → **NEMA DNS.** Kontejneri se NE mogu naći po imenu. (Plus: `inspect '{{.NetworkSettings.IPAddress}}'` na default rootless mreži vraća **prazno** — IP je pod `.Networks.<ime>.IPAddress`.)
- **Custom mreža** (`podman network create`) → **IMA DNS** (aardvark-dns). Ime kontejnera = DNS ime.
- DNS radi **samo unutar iste mreže.**

> 🪲 **LO6 hook:** segmentacija mreže = princip najmanjih privilegija. Baza skrivena iza proxyja = manja površina napada. Materijal za LO6 esej o sigurnosti kontejnerskih mreža.

---

## Zadatak 2 — Custom mreža, app + postgres, dokaz razrješavanja baze po imenu

```bash
podman network create appnet
```
- `podman network create appnet` — napravi custom most mrežu `appnet`. **Custom = ima ugrađeni DNS.**

```bash
podman run -d --network appnet --name db -e POSTGRES_PASSWORD=tajna postgres:16
podman run -d --network appnet --name web nginx
```
- `-d` — pozadina.
- `--network appnet` — spoji kontejner na `appnet` (ne na default). **Na `appnet` DNS radi.**
- `--name db` / `--name web` — imena su ujedno DNS imena.
- `-e POSTGRES_PASSWORD=tajna` — obavezna lozinka za postgres.

**Dokaz da `web` razrješava `db` po imenu** (bez instalacije — `getent` je u glibc slici poput nginxa):
```bash
podman exec web getent hosts db
```
- `podman exec web` — pokreće unutar `web`.
- `getent hosts db` — pitaj sustav za razrješenje imena `db`. Ako DNS radi → ispiše **IP adresu** baze. To je direktan dokaz da ime → IP funkcionira.

**Sažetak (3 rečenice):** Custom mreža (`network create`) ima ugrađeni DNS pa se kontejneri nalaze po imenu. `getent hosts db` iz `web` kontejnera ispiše IP baze = dokaz razrješavanja imena. Da su na default mreži, ovo bi vratilo ništa.

---

## Zadatak 19 — Razrješavanje imena PADA na različitim mrežama, pa popravak zajedničkom mrežom

```bash
podman network create net1
podman network create net2
podman run -d --network net1 --name app1 nginx
podman run -d --network net2 --name app2 nginx
```
- `network create net1` / `net2` — dvije odvojene custom mreže.
- `app1` na `net1`, `app2` na `net2` → **različite mreže, odvojeni DNS opsezi.**

**Prikaz da pada:**
```bash
podman exec app1 getent hosts app2
```
- `getent hosts app2` iz `app1` → **ne vrati ništa** (izlazni kod ≠ 0). `app2` je u drugom DNS opsegu, `app1` ga ne vidi.

**Popravak — `app1` se spaja i na `net2` uživo:**
```bash
podman network connect net2 app1
```
- `podman network connect` — spoji **već pokrenuti** kontejner na **dodatnu** mrežu, bez restarta.
- `net2 app1` — spoji kontejner `app1` na mrežu `net2`.

**Sad radi:**
```bash
podman exec app1 getent hosts app2
```
- Sad je `app1` na `net2` (kao i `app2`) → dijele DNS opseg → ime se razriješi → ispiše IP. **Popravljeno.**

**Sažetak (3 rečenice):** DNS radi samo unutar iste mreže, pa kontejneri na različitim mrežama ne nalaze jedan drugoga po imenu. `getent hosts` to pokaže (prazno → IP nakon popravka). `podman network connect` spaja kontejner na dodatnu mrežu uživo bez restarta.

---

## Zadatak 20 — Jedan kontejner na dvije mreže (frontend + backend), korist segmentacije

```bash
podman network create frontend
podman network create backend
podman run -d --network backend --name db -e POSTGRES_PASSWORD=tajna postgres:16
podman run -d --network frontend --name proxy nginx
podman network connect backend proxy
```
- `network create frontend` / `backend` — dvije zone.
- `db` samo na `backend` → skriven, ništa s frontenda ga ne doseže direktno.
- `proxy` na `frontend`, pa `network connect backend proxy` → proxy je **na obje** mreže.
- **Samo `proxy` dodiruje obje** → kontrolirani most.

**Dokaz da proxy doseže bazu:**
```bash
podman exec proxy getent hosts db
```
- `getent hosts db` iz `proxy` → ispiše IP (proxy je na `backend`). Kontejner samo na `frontend` ovo NE bi mogao.

**Korist segmentacije:** baza je izolirana na `backend`; jedini put do nje je preko `proxy`. Time se **smanjuje površina napada** — napadač na frontendu ne vidi bazu direktno.

> 🪲 **LO6 hook:** ovo je "network segmentation" / least-privilege networking — ide u LO6 esej o sigurnosti.

**Sažetak (3 rečenice):** Kontejner se s `network connect` može vezati na više mreža i postati kontrolirani most između zona. Baza ostaje na backend mreži, dosežna samo proxyju, ne i frontendu. Tako se površina napada smanjuje jer postoji samo jedna kontrolirana točka pristupa bazi.

---
---

# GRUPA C — PERZISTENCIJA, VOLUMENI, TAJNE (zadaci 4, 5, 11, 12, 22)

## Temelj grupe: zapisivi sloj vs perzistencija

Kontejner ima **zapisivi sloj** (writable layer) — sve što upiše unutra živi tu i **nestane kad se kontejner obriše** (`rm`). To je *efemerno*.

Za podatke koji moraju preživjeti potrebno ih je staviti **izvan zapisivog sloja**:
- **Imenovani volumen (named volume)** — spremnik kojim upravlja Podman (živi pod `~/.local/share/containers/storage/volumes/`). Prenosiv, nije vezan za putanju na hostu, preživi `rm` kontejnera.
- **Bind mount** — mapa s **hosta** ugurana u kontejner. Vezana za konkretnu putanju, uređuje se izvana.

**Tajna (secret)** = osjetljiv podatak (lozinka) koji ne treba biti u `inspect`-u ni u env listi → Podman ga drži u zasebnom spremištu, izvan konfiguracije kontejnera.

> 🪲 **LO6 hook:** efemerni sloj vs perzistentni volumen = stateless vs stateful. U k8s to su **PersistentVolume / PersistentVolumeClaim**. Materijal za LO6.

---

## Zadatak 4 — Imenovani volumen preživi `rm` + ponovnu izradu baze

```bash
podman volume create pgdata
podman run -d --name db -e POSTGRES_PASSWORD=tajna -v pgdata:/var/lib/postgresql/data postgres:16
```
- `podman volume create pgdata` — napravi imenovani volumen `pgdata`.
- `-d` — pozadina.
- `--name db` — ime.
- `-e POSTGRES_PASSWORD=tajna` — obavezna lozinka.
- `-v pgdata:/var/lib/postgresql/data` — montiraj **imenovani volumen** `pgdata` na postgresov direktorij podataka. Format: `imeVolumena:putanjaUKontejneru`.

**Upis podatka:**
```bash
podman exec -it db psql -U postgres -c "CREATE TABLE test(id int); INSERT INTO test VALUES (42);"
```
- `-it` — interaktivno + terminal (da psql lijepo radi).
- `psql -U postgres` — postgres klijent kao korisnik `postgres` (default superuser).
- `-c "SQL"` — izvrši jednu SQL naredbu: napravi tablicu `test`, ubaci `42`.

**Uništenje i ponovna izrada kontejnera:**
```bash
podman rm -f db
podman run -d --name db -e POSTGRES_PASSWORD=tajna -v pgdata:/var/lib/postgresql/data postgres:16
```
- `podman rm -f db` — prisilno ukloni kontejner (`-f` = force, ubije ako radi). **Zapisivi sloj NESTAJE.** Ali `pgdata` ostaje.
- Drugi `run` — nov kontejner, **isti volumen** montiran.

**Dokaz da su podaci preživjeli:**
```bash
podman exec -it db psql -U postgres -c "SELECT * FROM test;"
```
- Vrati `42` → podaci su preživjeli u volumenu, neovisno o tome što je kontejner uništen.

⚠️ **Začkoljice:**
- Treba pričekati koju sekundu prije `SELECT` — postgres se inicijalizira; prebrzo i javi "starting up".
- Na ponovnoj izradi `POSTGRES_PASSWORD` se **ignorira** za postojeću bazu (lozinka je već zapisana u volumenu pri prvoj inicijalizaciji).

**Sažetak (3 rečenice):** Zapisivi sloj kontejnera je efemeran i nestaje s `rm`, pa podaci baze idu u imenovani volumen (`-v pgdata:...`). Nakon `rm -f` i ponovne izrade kontejnera s istim volumenom, `SELECT` i dalje vraća `42`. To dokazuje da volumen živi neovisno o životu kontejnera.

---

## Zadatak 5 — Podman secret umjesto `--env` za lozinku

```bash
printf 'tajna' | podman secret create pgpw -
```
- `printf 'tajna'` — ispiši lozinku **bez** završnog newlinea (`echo` bi dodao `\n` i pokvario lozinku; `printf` ne dodaje).
- `| podman secret create pgpw -` — napravi tajnu imena `pgpw`, čitajući vrijednost sa **stdina** (završni `-` znači "čitaj sa stdina").

```bash
podman run -d --name db --secret pgpw,type=env,target=POSTGRES_PASSWORD postgres:16
```
- `-d` — pozadina.
- `--secret pgpw` — ubaci tajnu `pgpw`.
- `,type=env` — izloži je kao **env varijablu** (default je `type=mount`, tj. datoteka).
- `,target=POSTGRES_PASSWORD` — ime env varijable koje postgres očekuje.

**Sigurnosna korist — dokaz da je skrivena:**
```bash
podman inspect db --format '{{.Config.Env}}'
```
- `--format '{{.Config.Env}}'` — ispiši samo env listu. **Vrijednost lozinke NIJE tu** (kod `--env` bi bila vidljiva u `inspect`-u). Tajne se drže u zasebnom spremištu, ne u konfiguraciji kontejnera.

⚠️ **Začkoljica (nijansa za bodove):** `type=env` je zgodan, ali env zna iscuriti kroz `/proc/PID/environ`. Sigurnija varijanta je **datoteka** + postgresov `_FILE` mehanizam:
```bash
podman run -d --name db \
  --secret pgpw,target=/run/secrets/pgpw \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/pgpw \
  postgres:16
```
- `--secret pgpw,target=/run/secrets/pgpw` — montiraj tajnu kao datoteku na tu putanju.
- `-e POSTGRES_PASSWORD_FILE=...` — postgres čita lozinku iz datoteke umjesto iz env varijable. **Lozinka nikad ne uđe u env listu.**

**Sažetak (3 rečenice):** Tajna se napravi `secret create` (čita sa stdina) i ubaci s `--secret` umjesto `--env`. Korist: lozinka se ne vidi u `podman inspect` ni u env listi, nego stoji u zasebnom spremištu. Varijanta s datotekom (`type=mount` + `POSTGRES_PASSWORD_FILE`) je još sigurnija jer izbjegava env potpuno.

---

## Zadatak 11 — Bind mount (konfiguracija) + imenovani volumen (podaci) u istom kontejneru

```bash
mkdir -p ~/devops/conf
echo "# moja konfiguracija" > ~/devops/conf/app.conf
podman volume create appdata
podman run -d --name app \
  -v ~/devops/conf:/etc/app:ro,Z \
  -v appdata:/var/lib/app \
  nginx
```
- `mkdir -p ~/devops/conf` — mapa na hostu za config (`-p` = napravi roditelje, ne buni se ako postoji).
- `echo "..." > .../app.conf` — napravi config datoteku.
- `podman volume create appdata` — imenovani volumen za podatke.
- `-d` / `--name app` — pozadina, ime.
- `-v ~/devops/conf:/etc/app:ro,Z` — **bind mount**: host-mapa → `/etc/app` u kontejneru.
  - `:ro` — *read-only* (app ne smije mijenjati konfiguraciju).
  - `:Z` — **SELinux** relabel (privatno, samo ovaj kontejner). **Na CentOS-u OBAVEZNO** ili "permission denied".
- `-v appdata:/var/lib/app` — **imenovani volumen** za podatke (Podman-upravljan, perzistentan).

**Zašto svaki:**
- **Bind mount za config** — datoteka se uređuje editorom na hostu, kontejner je vidi uživo; vezan za host putanju; idealno za konfiguraciju koja se piše ručno.
- **Imenovani volumen za podatke** — Podman-upravljan, prenosiv, nije vezan za host putanju, preživi uklanjanje kontejnera; idealno za stanje baze/aplikacije.

⚠️ **Začkoljica:** bind mount na SELinux sustavu treba `:Z` (privatno) ili `:z` (dijeljeno između više kontejnera). **Imenovani volumeni se relabeliraju automatski, bind mountovi NE** — zato baš njima treba `:Z`.

**Sažetak (3 rečenice):** U istom kontejneru bind mount nosi konfiguraciju s hosta (read-only, `:Z` zbog SELinuxa), a imenovani volumen nosi podatke. Bind mount služi kad se config piše ručno i mijenja uživo na hostu; imenovani volumen kad treba prenosiva, perzistentna pohrana stanja. Razlika je tko "posjeduje" podatke — host ili Podman.

---

## Zadatak 12 — Backup imenovanog volumena u tarball pomoćnim kontejnerom, pa restore u svježi volumen

```bash
podman volume create datavol
podman run --rm -v datavol:/data alpine sh -c "echo 'vazni podaci' > /data/file.txt"
```
- `podman volume create datavol` — volumen koji će se backupirati.
- `podman run --rm -v datavol:/data alpine sh -c "..."` — napuni ga jednokratnim alpine kontejnerom.
  - `--rm` — ukloni kontejner čim završi.
  - `-v datavol:/data` — montiraj volumen na `/data`.
  - `alpine` — sićušna pomoćna slika.
  - `sh -c "echo ... > /data/file.txt"` — upiši datoteku u volumen.

**Backup:**
```bash
podman run --rm \
  -v datavol:/data:ro \
  -v ~/devops:/backup \
  alpine \
  tar czf /backup/datavol-backup.tar.gz -C /data .
```
- `--rm` — ukloni pomoćni kontejner nakon završetka.
- `-v datavol:/data:ro` — montiraj volumen koji se backupira, **read-only** (original se ne dira).
- `-v ~/devops:/backup` — bind mount host-mape koja prima tarball.
- `alpine` — pomoćna slika.
- `tar czf /backup/datavol-backup.tar.gz -C /data .` — napravi gzip-tarball:
  - `c` = *create* (stvara arhivu),
  - `z` = *gzip* (komprimiraj),
  - `f` = *file* (slijedi ime datoteke),
  - `-C /data` = prvo uđi u `/data` (pa su putanje u arhivi relativne, ne `/data/...`),
  - `.` = sve iz tog direktorija.

**Restore u svježi volumen:**
```bash
podman volume create datavol-restored
podman run --rm \
  -v datavol-restored:/data \
  -v ~/devops:/backup:ro \
  alpine \
  tar xzf /backup/datavol-backup.tar.gz -C /data
```
- `volume create datavol-restored` — nov, prazan volumen.
- `-v datavol-restored:/data` — montiraj ga (zapisivo).
- `-v ~/devops:/backup:ro` — backup-mapa read-only.
- `tar xzf ... -C /data` — raspakiraj: `x` = *extract*, `z` = gzip, `f` = file, `-C /data` = u taj direktorij.

**Dokaz:**
```bash
podman run --rm -v datavol-restored:/data alpine cat /data/file.txt
```
- Ispiše `vazni podaci` → restore uspio.

⚠️ **Začkoljica:** `-C /data .` (s točkom) je bitan — tarira **sadržaj**, ne roditeljsku mapu, pa pri restoreu datoteke padnu ravno u korijen novog volumena. Ovaj obrazac (pomoćni kontejner montira volumen + bind mount za tarball) je **standardni idiom** za backup volumena u Podmanu/Dockeru.

**Sažetak (3 rečenice):** Volumen se backupira jednokratnim alpine kontejnerom koji montira volumen (read-only) i bind-mount mapu za izlaz, pa `tar czf -C /data .` napravi arhivu. Restore je isti obrazac obrnuto: nov volumen + `tar xzf -C /data`. Dokaz je `cat` datoteke iz vraćenog volumena.

---

## Zadatak 22 — Dijeljenje konfiguracije između dvije replike preko zajedničkog volumena + rizik

```bash
podman volume create shared-conf
podman run --rm -v shared-conf:/conf alpine sh -c "echo 'shared=true' > /conf/app.conf"
podman run -d --name app1 -v shared-conf:/conf:ro nginx
podman run -d --name app2 -v shared-conf:/conf:ro nginx
```
- `volume create shared-conf` — zajednički volumen.
- prvi `run --rm` — napuni ga konfiguracijom (jednokratni alpine).
- `app1` i `app2` — dvije replike, obje montiraju **isti** volumen `:ro` (read-only) → dijele istu konfiguraciju.

**Rizik dijeljenog read-WRITE storagea:** ako bi obje replike montirale volumen **zapisivo** i istovremeno pisale iste datoteke → utrke (*race conditions*) i korupcija (npr. dva procesa pišu isti log/SQLite → ispremiješan, pokvaren zapis). Većina podatkovnih engina pretpostavlja **jednog pisca**. Zato se config montira `:ro`, a stateful podaci traže ili jednog pisca ili pravi klasterirani store.

> 🪲 **LO6 hook:** u k8s to su access modovi **ReadWriteOnce vs ReadWriteMany** kod PersistentVolume-a. Direktno ovaj koncept → LO6.

**Sažetak (3 rečenice):** Dvije replike mogu dijeliti isti imenovani volumen i tako čitati istu konfiguraciju (`:ro`). Rizik nastaje kod dijeljenog zapisivog storagea: istovremeni zapisi iste datoteke vode u utrke i korupciju jer engini pretpostavljaju jednog pisca. Zato config ide read-only, a dijeljeno zapisivanje treba klasterirani sustav ili jednog pisca.

---
---

# GRUPA D — COMPOSE (zadaci 6, 7, 8, 9, 10, 17, 21, 23)

## Najprije provjera da compose radi:
```bash
podman compose version
```
Ako padne → blokada za nastavak Grupe D. Compose je deklarativni način: cijeli stack se opiše u jednoj YAML datoteci umjesto pamćenja hrpe `run` naredbi.

## Temelj grupe: compose.yaml

- **`services:`** — vrhovni ključ; svaki podključ je jedan servis = jedan kontejner.
- Compose sam napravi **default mrežu** za projekt → servisi se nalaze po imenu servisa (DNS radi, jer je to custom mreža).
- `up` digne sve, `down` sruši sve.

---

## Zadatak 6 — compose.yaml za app + bazu, `up -d`, oba rade

Datoteka `~/devops/compose.yaml`:
```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: tajna
  app:
    image: nginx
    ports:
      - "8080:80"
```
- `services:` — popis servisa.
- `db:` → `image: postgres:16`, env `POSTGRES_PASSWORD`.
- `app:` → `image: nginx`, `ports: - "8080:80"` = objavi host:8080 → kontejner:80.

```bash
cd ~/devops
podman compose up -d
```
- `cd ~/devops` — gdje je `compose.yaml`.
- `podman compose up` — napravi + pokreće sve iz datoteke.
- `-d` — *detached* (pozadina).

```bash
podman compose ps
```
- `podman compose ps` — pokaže oba servisa kako rade.

**Sažetak (3 rečenice):** Compose datoteka deklarira `db` i `app` servis u jednom YAML-u. `podman compose up -d` digne cijeli stack u pozadini, a oba servisa dijele compose-ovu projektnu mrežu. `podman compose ps` potvrdi da rade.

---

## Zadatak 7 — Baza na internoj mreži bez objavljenog porta, samo app izložen; dokaz da host ne doseže bazu

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: tajna
    networks:
      - backend
  app:
    image: nginx
    ports:
      - "8080:80"
    networks:
      - backend
networks:
  backend:
```
- `db` **nema `ports:`** → nije objavljen hostu.
- oba na `backend` → app doseže bazu interno (po imenu `db`).
- host ne doseže bazu (nema objavljenog porta).

**Dokaz s hosta:**
```bash
podman compose ps
ss -tln | grep 5432
```
- `podman compose ps` — kod `db` nema host-port mapiranja.
- `ss -tln` — pokaži TCP portove koji slušaju na hostu: `-t` = TCP, `-l` = listening, `-n` = brojevi (ne razrješavaj imena). `grep 5432` → **ništa** = baza nije dosežna s hosta.

⚠️ **Začkoljica:** za još jaču izolaciju dodaje se `internal: true` mreži (`backend: \n   internal: true`) — tada mreža nema ni izlaz prema vani. Ali za "host ne doseže bazu" dovoljno je **ne objaviti port**.

**Sažetak (3 rečenice):** Baza bez `ports:` nije dosežna s hosta, ali app je doseže interno preko zajedničke mreže. `ss -tln | grep 5432` na hostu ne vraća ništa = dokaz izolacije. `internal: true` na mreži dao bi još jaču izolaciju (bez izlaza prema vani).

---

## Zadatak 8 — Healthcheck + `depends_on: service_healthy`, demonstracija redoslijeda

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: tajna
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
  app:
    image: nginx
    depends_on:
      db:
        condition: service_healthy
```
- `healthcheck.test: ["CMD-SHELL", "pg_isready -U postgres"]` — provjera zdravlja: `pg_isready` ispituje prima li postgres konekcije. `CMD-SHELL` = pokreće kroz shell.
- `interval: 5s` — provjeravaj svakih 5 s.
- `timeout: 3s` — svaka provjera traje najviše 3 s.
- `retries: 5` — 5 neuspjeha = nezdravo.
- `depends_on: db: condition: service_healthy` — app **NE startuje** dok `db` nije **zdrav** (ne samo pokrenut).

**Demonstracija redoslijeda:**
```bash
podman compose up
```
- (bez `-d`, radi gledanja logova) — `db` starta, healthcheck se vrti, i **tek kad postane zdrav** krene `app`.

⚠️ **Začkoljica (za bodove):** obični `depends_on: [db]` čeka samo da `db` **startuje** (kontejner pokrenut), NE da bude **spreman**. `condition: service_healthy` je ono što čeka stvarnu spremnost. ⚠️ **Za provjeru na VM-u:** poštuje li compose provider `service_healthy` — neki ga ignoriraju.

**Sažetak (3 rečenice):** Healthcheck (`pg_isready`) Podmanu javlja kad je baza stvarno spremna, a ne samo pokrenuta. `depends_on: condition: service_healthy` čini da app čeka to zdravlje prije starta. Razlika prema običnom `depends_on` (koji čeka samo start) je ključna i nosi bodove.

---

## Zadatak 9 — `env_file` za kredencijale baze umjesto inline vrijednosti

Datoteka `~/devops/db.env`:
```
POSTGRES_PASSWORD=tajna
POSTGRES_USER=appuser
POSTGRES_DB=appdb
```
compose:
```yaml
services:
  db:
    image: postgres:16
    env_file:
      - db.env
```
- `env_file: - db.env` — učitaj env varijable iz datoteke `db.env` umjesto da se pišu inline u `compose.yaml`. Drži kredencijale izvan compose datoteke.

⚠️ **Začkoljica:** `env_file` **NIJE isto što i secret** — datoteka je plaintext na disku i varijable se i dalje vide u `podman inspect`. To je **organizacija/odvajanje**, ne sigurnost. (Usporedi sa zad. 5, gdje je tajna stvarno skrivena.)

**Sažetak (3 rečenice):** `env_file` izdvaja kredencijale u zasebnu datoteku pa `compose.yaml` ostaje čist. To je organizacijska pogodnost, ne sigurnosna mjera — datoteka je plaintext i varijable se vide u `inspect`-u. Za pravu tajnost koristi se secret (zad. 5).

---

## Zadatak 10 — Skaliranje servisa na više replika, kako se raspoređuju zahtjevi

```bash
podman compose up -d --scale app=3
```
- `--scale app=3` — pokreće **3 replike** servisa `app`.

⚠️ **VELIKA začkoljica:** ako `app` ima **fiksni host port** (`ports: - "8080:80"`), skaliranje na 3 **PADA** — 3 kontejnera ne mogu sva vezati host:8080 (sukob porta). Za skaliranje potrebno je ili maknuti fiksni host port, ili koristiti raspon, i staviti **reverse proxy / load balancer** ispred.

**Kako se raspoređuju zahtjevi:** compose-ov interni DNS radi **round-robin** preko replika kad ih se doseže po **imenu servisa** (interno). Ali da bi se **host promet** raspoređivao, treba **nginx** ispred (veže se na zad. 13). Bez LB-a, samo skaliranje **ne** raspoređuje host promet.

**Sažetak (3 rečenice):** `--scale app=3` digne tri replike servisa. Ako servis ima fiksni host port, skaliranje pada zbog sukoba porta — treba ga maknuti i staviti proxy ispred. Raspoređivanje ide round-robin preko internog DNS-a po imenu servisa; za host promet treba reverse proxy (nginx).

---

## Zadatak 17 — Per-service CPU/mem limiti u compose, provjera s `podman stats`

```yaml
services:
  app:
    image: nginx
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 128M
```
- `deploy.resources.limits.cpus: "0.50"` — najviše pola CPU-a.
- `memory: 128M` — najviše 128 MB RAM-a.

⚠️ **Začkoljica:** `deploy:` dolazi iz Swarma; **hoće li ga compose provider poštovati ovisi o provideru** — neki `deploy` ignoriraju. Sigurnija alternativa koju Podman pouzdano primjenjuje su ključevi na razini servisa:
```yaml
services:
  app:
    image: nginx
    mem_limit: 128m
    cpus: 0.50
```
- `mem_limit: 128m` — tvrdi limit memorije.
- `cpus: 0.50` — limit CPU-a.

**⚠️ Za provjeru:** koji oblik provider primjenjuje (vidi `podman stats`).

**Provjera:**
```bash
podman stats --no-stream
```
- `podman stats` — uživo CPU/mem po kontejneru.
- `--no-stream` — jedan snimak pa izlaz (ne kontinuirano). Stupac **MEM LIMIT** → mora pokazati 128M.

**Sažetak (3 rečenice):** Limiti se postave u compose (`deploy.resources.limits` ili `mem_limit`/`cpus`). Provider možda ignorira `deploy`, pa postoji i zamjenski oblik. `podman stats --no-stream` u stupcu MEM LIMIT potvrdi da je limit primijenjen.

---

## Zadatak 21 — `compose logs` + `compose ps` za nadzor i troubleshooting

```bash
podman compose ps
podman compose logs
podman compose logs db
podman compose logs -f
```
- `podman compose ps` — status svih servisa (radi/izašao/zdravlje) → vidljivo tko je pao.
- `podman compose logs` — objedinjeni logovi svih servisa, obojeni po servisu.
- `podman compose logs db` — samo logovi servisa `db`.
- `podman compose logs -f` — `-f` = *follow*, prati logove uživo.

**Tok troubleshootinga:** `ps` da se vidi tko je pao → `logs <servis>` da se vidi zašto.

**Sažetak (3 rečenice):** `compose ps` pokaže koji servis ne radi ili se restarta. `compose logs <servis>` daje razlog (greška u startu, krivi env...). `-f` prati logove uživo tijekom dijagnostike.

---

## Zadatak 23 — `compose down` + imenovani volumeni + `--volumes` zastavica

```bash
podman compose down
```
- `podman compose down` — zaustavi i **ukloni** kontejnere + default mrežu koju je compose napravio. Po defaultu **imenovani volumeni OSTAJU** (podaci prežive).

```bash
podman compose down --volumes
```
- `--volumes` (ili `-v`) — **TAKOĐER ukloni** imenovane volumene deklarirane u compose datoteci. **Podaci se UNIŠTAVAJU.**

⚠️ **Začkoljica:** `down` sam je siguran za podatke (volumeni prežive); `down --volumes` je **destruktivan**. Ključna ispitna poanta: imenovani volumeni prežive `down`, ginu s `down --volumes`.

**Sažetak (3 rečenice):** `compose down` ukloni kontejnere i mrežu, ali imenovane volumene **ostavi** — podaci prežive. `compose down --volumes` dodatno briše i volumene, čime se podaci uništavaju. Zato je `--volumes` opasan i koristi se namjerno.

---
---

# GRUPA E — DVOSLOJNI STACKOVI I POMOĆNI KONTEJNERI (zadaci 3, 13, 14)

## Temelj grupe: dvoslojna arhitektura + pomoćni kontejneri

**Dvoslojni stack** = aplikacija (app, često stateless) + baza (stateful) koje surađuju preko mreže. App nađe bazu **po imenu** (custom-net DNS). **Pomoćni kontejneri** (reverse proxy, DB-admin) dodaju se na istu mrežu da prošire/promatraju sustav, ne mijenjajući app.

> 🪲 **LO6 hook:** dvoslojna arhitektura (stateless app + stateful db) → u k8s to su **Deployment + StatefulSet**. Reverse proxy → **Ingress**. Materijal za LO6.

---

## Zadatak 3 — Dvoslojni stack (NE Drupal/Joomla/WordPress), first-run setup

Odabir: **Gitea + PostgreSQL** (lagan, jasan first-run). `~/devops/gitea-compose.yaml`:
```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: gitea
      POSTGRES_PASSWORD: gitea
      POSTGRES_DB: gitea
    volumes:
      - giteadb:/var/lib/postgresql/data
  gitea:
    image: gitea/gitea:latest
    environment:
      GITEA__database__DB_TYPE: postgres
      GITEA__database__HOST: db:5432
      GITEA__database__NAME: gitea
      GITEA__database__USER: gitea
      GITEA__database__PASSWD: gitea
    ports:
      - "3000:3000"
    depends_on:
      - db
    volumes:
      - giteadata:/data
volumes:
  giteadb:
  giteadata:
```
- `db` servis: postgres s `POSTGRES_USER/PASSWORD/DB` = gitea, podaci u imenovanom volumenu `giteadb`.
- `gitea` servis:
  - `GITEA__database__*` — Gitea čita config iz env varijabli (dvostruka donja crta = razina ugnježđenja). `HOST: db:5432` = baza po **imenu servisa** `db`.
  - `ports: - "3000:3000"` — Gitea web na host:3000.
  - `depends_on: - db` — kreni nakon baze.
  - `giteadata:/data` — perzistentni podaci Gitee.

```bash
cd ~/devops
podman compose -f gitea-compose.yaml up -d
```
- `-f gitea-compose.yaml` — koristi tu datoteku (jer nije default `compose.yaml`).
- `up -d` — digni u pozadini.

**First-run setup:** otvori `http://<host>:3000` → Gitea pokaže instalacijsku stranicu (DB postavke su predpopunjene iz env-a) → klik na "Install Gitea" → napravi admin korisnika. Gotovo.

⚠️ **Začkoljica:** Gitea bazu doseže po imenu `db` (ime servisa → DNS na compose mreži). Treba pričekati da baza bude spremna; `depends_on` pomaže (za čvršće, dodaje se healthcheck kao u zad. 8).

**Sažetak (3 rečenice):** Dvoslojni stack (Gitea + PostgreSQL) preko compose-a: app nađe bazu po imenu servisa, oba imaju perzistentne volumene. `up -d` digne stack, a first-run setup završava se u pregledniku na portu 3000. Gitea je lagan i prikladan za rootless VM (lakši od Ghosta/Redminea).

---

## Zadatak 13 — Nginx reverse-proxy ispred app kontejnera, host promet kroz nginx do appa

```bash
podman network create proxynet
podman run -d --network proxynet --name app nginx
```
- `network create proxynet` — custom mreža (treba DNS da proxy nađe `app` po imenu).
- `app` kontejner (nginx kao stand-in app) na `proxynet`, **bez objavljenog porta**.

Konfiguracija proxyja `~/devops/proxy/default.conf`:
```bash
mkdir -p ~/devops/proxy
cat > ~/devops/proxy/default.conf <<'EOF'
server {
    listen 80;
    location / {
        proxy_pass http://app:80;
    }
}
EOF
```
- `listen 80;` — proxy sluša na 80.
- `proxy_pass http://app:80;` — nginx prosljeđuje zahtjeve kontejneru `app` (razriješen po imenu preko `proxynet` DNS-a) na port 80.

```bash
podman run -d --network proxynet --name proxy \
  -p 8080:80 \
  -v ~/devops/proxy/default.conf:/etc/nginx/conf.d/default.conf:ro,Z \
  nginx
```
- `-d` / `--name proxy` — pozadina, ime.
- `--network proxynet` — proxy na istoj mreži kao `app` (da ga razriješi po imenu).
- `-p 8080:80` — **samo proxy** objavljen: host:8080 → proxy:80.
- `-v .../default.conf:/etc/nginx/conf.d/default.conf:ro,Z` — bind mount konfiguracije. `:ro` = read-only, `:Z` = SELinux relabel (CentOS).

**Test:**
```bash
curl localhost:8080
```
- host → proxy(8080) → app(80) → nginx welcome stranica. Promet ide kroz proxy do appa; `app` sam nije objavljen.

⚠️ **Začkoljica:** proxy MORA biti na istoj mreži kao `app` (`proxynet`) da `proxy_pass http://app` radi — to je custom-net DNS. Na default mreži bi razrješavanje imena palo (Grupa B začkoljica). Config mount na CentOS-u treba `:ro,Z`.

> 🪲 **LO6 hook:** reverse proxy ovdje = k8s **Ingress** u LO4/LO6.

**Sažetak (3 rečenice):** Nginx proxy i app stoje na istoj custom mreži; proxy prosljeđuje promet na `app` po imenu (`proxy_pass http://app:80`). Samo je proxy objavljen hostu, app je interan. Razrješavanje imena radi jer je mreža custom (DNS); config se montira read-only s `:Z` zbog SELinuxa.

---

## Zadatak 14 — DB-admin kontejner (adminer) na mreži, pregled baze

```bash
podman network create dbnet
podman run -d --network dbnet --name db \
  -e POSTGRES_PASSWORD=tajna -e POSTGRES_USER=appuser -e POSTGRES_DB=appdb postgres:16
podman run -d --network dbnet --name adminer -p 8081:8080 adminer
```
- `network create dbnet` — custom mreža (DNS).
- `db` — postgres s korisnikom `appuser`, bazom `appdb`, lozinkom `tajna`, na `dbnet`.
- `adminer` — lagani web DB-admin; u kontejneru sluša na 8080, objavljuje se na host:8081; na `dbnet` da doseže `db` po imenu.

**Korištenje:** otvori `http://<host>:8081` → Adminer login. Upiše se: **System:** PostgreSQL, **Server:** `db`, **Username:** `appuser`, **Password:** `tajna`, **Database:** `appdb` → spajanje i pregled tablica.

⚠️ **Začkoljica:** u Adminerovo polje **"Server"** upisuje se **ime kontejnera `db`**, ne `localhost` — jer Adminer doseže postgres preko zajedničke custom mreže po DNS imenu. **System** se postavlja na PostgreSQL (default je MySQL).

**Sažetak (3 rečenice):** Adminer je web DB-admin koji se doda na istu mrežu kao baza i objavi na hostu. Prijava ide s imenom kontejnera baze kao "Server" (ne localhost), jer se spaja interno preko custom-net DNS-a. Tako se pregledava baza bez ikakvog dodatnog klijenta na hostu.

---
---

# GRUPA F — MOST PREMA KUBERNETESU (zadaci 15, 16)

## Temelj grupe: Podman pod ↔ Kubernetes

Podmanovi podovi su namjerno **kompatibilni s Kubernetesom**. Moguće je:
- iz pokrenutog poda **izvesti** k8s YAML (`kube generate`),
- iz k8s YAML-a **pokrenuti** pod lokalno (`kube play`).

Isti YAML može ići i na pravi k8s klaster (`kubectl apply`). To je doslovan prijelaz LO3 → LO4: *definira se jednom, pokreće gdje god treba.*

> 🪲 **LO6 hook:** Podman↔Kubernetes kompatibilnost (zašto Podman koristi podove, OCI/k8s usklađenost) = jak materijal za LO6 esej "Docker/Podman vs Kubernetes".

> ℹ️ **Napomena o nazivima naredbi:** noviji oblici su `podman kube generate` i `podman kube play`; stariji aliasi su `podman generate kube` i `podman play kube`. Oba rade. PDF zadatak 15 koristi stariji oblik (`podman generate kube`), a zadatak 16 noviji (`podman kube play`); ovdje se dosljedno koristi noviji oblik, a stariji se navodi uz svaki zadatak.

---

## Zadatak 15 — Generiranje k8s YAML iz pokrenutog poda, objašnjenje mosta prema LO4

```bash
podman pod create --name kubepod -p 8080:80
podman run -d --pod kubepod --name web nginx
```
- napravi pod `kubepod` s portom + ubaci `web` (nginx) — nešto što će se izvesti.

```bash
podman kube generate kubepod -f kubepod.yaml
```
- `podman kube generate` — proizvedi **Kubernetes YAML** (Pod manifest) koji opisuje pokrenuti pod.
- `kubepod` — pod koji se izvozi.
- `-f kubepod.yaml` — zapiši u datoteku (može i `> kubepod.yaml`).

⚠️ **Napomena:** novi oblik je `podman kube generate`. Stari (i dalje radi kao alias): **`podman generate kube kubepod > kubepod.yaml`**. PDF zadatak izrijekom navodi stari oblik (`podman generate kube`). Ako jedan ne radi, koristi se drugi.

```bash
cat kubepod.yaml
```
- pregled: pravi k8s Pod spec (`apiVersion: v1`, `kind: Pod`, `containers`, `ports`).

**Most prema LO4:** taj isti YAML može se `kubectl apply -f`-ati na pravi Kubernetes klaster. Prototip lokalno Podmanom, pa deploy generiranog manifesta na k8s. To je literalni handoff LO3 → LO4.

**Sažetak (3 rečenice):** `podman kube generate` iz pokrenutog poda izvuče Kubernetes Pod manifest. Taj YAML je standardni k8s objekt koji se može primijeniti i na pravi klaster (`kubectl apply`). Zato Podman služi kao lokalna odskočna daska prema Kubernetesu (LO4).

---

## Zadatak 16 — Pokretanje poda iz k8s YAML-a (`kube play`), rušenje (`--down`)

```bash
podman kube play kubepod.yaml
```
- `podman kube play` — pročitaj Kubernetes YAML i **napravi** opisani pod/kontejnere lokalno. (Stari alias: `podman play kube`.)

```bash
podman pod ps
```
- pod iz YAML-a sad radi.

```bash
podman kube play --down kubepod.yaml
```
- `--down` — zaustavi i ukloni sve što je YAML napravio (obrnuto od `play`).

⚠️ **Začkoljice:**
- `kube play` (novi) vs `play kube` (stari) — isto kao kod `generate`. **Za provjeru:** koji od njih radi.
- Ako je port iz prethodnog identičnog poda još zauzet, `play` padne s "port in use" → prvo se makne stari pod: `podman pod rm -f kubepod`.

**Sažetak (3 rečenice):** `podman kube play` iz k8s YAML-a digne pod lokalno, a `--down` ga uredno sruši. To dokazuje kružni tok: pod → YAML (zad. 15) → pod (zad. 16). Par `generate`/`play` je doslovni most prema Kubernetesu (LO4).

---
---

# ČIŠĆENJE IZMEĐU ZADATAKA

Da se ne sudaraju imena/portovi tijekom prolaska kroz zadatke:

```bash
podman rm -f $(podman ps -aq)          # ukloni SVE kontejnere
podman pod rm -f $(podman pod ps -q)   # ukloni SVE podove
podman network prune -f                # ukloni nekorištene mreže
```
- `$(podman ps -aq)` — ubaci popis ID-eva svih kontejnera (`-a` = svi, `-q` = samo ID). Prazan popis → bezopasna greška "requires at least 1 argument".
- `podman pod rm -f ...` — isto za podove.
- `podman network prune -f` — `-f` = bez pitanja, počisti nekorištene mreže.

⚠️ **Volumeni se NE prune-aju automatski** radi zadržavanja podataka iz zad. 4/12. Volumeni se brišu ručno (`podman volume rm <ime>`) tek kad je sigurno.

---

## Brza referenca

| Tema | Naredba |
|---|---|
| Pod + port | `podman pod create --name X -p 8080:80` |
| Kontejner u pod | `podman run -d --pod X --name web nginx` |
| Inspect poda (prostor imena) | `podman pod inspect X` → `SharedNamespaces` |
| Custom mreža | `podman network create appnet` |
| Spoji uživo na mrežu | `podman network connect net2 app1` |
| Test DNS (bez instalacije) | `podman exec web getent hosts db` |
| Imenovani volumen | `podman volume create pgdata` → `-v pgdata:/path` |
| Bind mount (CentOS) | `-v /host:/cont:ro,Z` |
| Secret | `printf 'pw' \| podman secret create pgpw -` → `--secret pgpw,type=env,target=...` |
| Backup volumena | `podman run --rm -v vol:/data:ro -v ~/devops:/backup alpine tar czf /backup/x.tar.gz -C /data .` |
| Compose gore/dolje | `podman compose up -d` / `podman compose down [--volumes]` |
| Skaliranje | `podman compose up -d --scale app=3` |
| Statistika/limiti | `podman stats --no-stream` |
| Pod → k8s YAML | `podman kube generate X -f x.yaml` (stari: `generate kube`) |
| k8s YAML → pod | `podman kube play x.yaml` / `--down` |

---

# ŠTO DALJE

1. **Prvo:** PREFLIGHT (5 naredbi) + obnova okruženja. Ako `podman compose version` padne — to je blokada za Grupu D.
2. **Izvedba grupa A → F** redom. Gdje VM odstupi od skripte (⚠️ ZA PROVJERU mjesta) — to se popravlja uživo i upisuje u skriptu.
3. **Kad LO3 prođe hands-on** → ovo postaje finalna, verificirana skripta.
4. **Pa LO4** (čisti Kubernetes, minikube je na VM-u). Najveći zalogaj, ali `generate`/`play` iz Grupe F već daje osjećaj za k8s YAML.

**LO6 hookovi pokupljeni usput** (za esej): pod = najmanja jedinica raspoređivanja; ephemeral vs persistent (PV/PVC); ReadWriteOnce vs ReadWriteMany; network segmentation / least privilege; reverse proxy = Ingress; Podman↔k8s kompatibilnost. To je već 6 talking