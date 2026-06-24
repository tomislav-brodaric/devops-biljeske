# DevOps — LO4: Kubernetes

Ubrzana isporuka višeslojnih aplikacija u Kubernetesu: Deployment→ReplicaSet→Pod, rollouti, Servisi, sonde, konfiguracija/tajne/svesci i ostali kontroleri.

Okruženje: rootless Podman, minikube, kubectl, bez `sudo` (VM se resetira između sesija).

## Rječnik kratica

- **Pod** = najmanja jedinica raspoređivanja; jedan ili više containera koji dijele mrežu i volumene
- **RS** = ReplicaSet — kontroler koji drži zadani broj replika Poda (broji i nadomješta)
- **Deployment** = upravlja RS-ovima i verzionira ih (rolling update, rollback)
- **pod-template-hash** = otisak Pod template-a; labela kojom RS prepoznaje svoje Podove
- **manifest** = YAML datoteka koja opisuje željeno stanje objekta
- **etcd** = baza klastera u kojoj žive svi objekti (preživljava restart komponenti)

---

## `vi` — preživljavanje editora (`kubectl edit` ga otvara po defaultu)

`kubectl edit <resurs>` otvara živi objekt u `vi`. Editor ima dva moda: **normal** (kretanje/komande) i **insert** (tipkanje teksta). Otvara se u NORMAL modu.

**Minimum za izvući se:**
- `i` — uđi u INSERT mode (sad se tipka normalno)
- `Esc` — natrag u NORMAL mode (iz insert moda)
- `:wq` + Enter — **w**rite + **q**uit = sprema i izlazi
- `:q!` + Enter — izlazi BEZ spremanja (odustaje od svih izmjena)

**Kretanje (u NORMAL modu):**
- strelice rade · `gg` = skoči na vrh datoteke · `G` = skoči na dno
- `/tekst` + Enter = traži `tekst` (npr. `/replicas` da nađeš polje u velikom YAML-u); `n` = sljedeći pogodak

**Uređivanje (u NORMAL modu):**
- `x` = obriši znak pod kursorom · `dd` = obriši cijeli redak
- `u` = poništi zadnju promjenu (undo) · `Ctrl+r` = ponovi (redo)

**Tipičan tijek za promjenu jednog broja (npr. `replicas`):**
1. `/replicas` + Enter → kursor skoči na redak
2. strelicama do broja · `i` (insert) · obriši stari broj, upiši novi
3. `Esc` · `:wq` + Enter

**Ako se sve zapetlja:** `Esc` (izlazak iz insert moda) → `:q!` + Enter (odustajanje) → ponovno pokretanje `kubectl edit`.

---

## Pregled — 53 zadatka po fazama

| Faza | Tema | Zadaci |
|------|------|--------|
| **1** | Kičma: Deployment→RS→Pod, labele, goli Pod | 1, 11, 15, 3 |
| **2** | Rolloutovi: rolling update, rollback, strategije, zaglavljeni rollout | 4, 5, 6, 7, 9, 10 |
| **3** | Podešavanje Poda: resursi/limiti, nodeSelector→Pending, sidecar/initContainer/emptyDir | 8, 13, 14, 22, 28 |
| **4** | Servisi: ClusterIP, NodePort, LoadBalancer, DNS, port/targetPort/nodePort, load-balancing | 12, 38–47, 49 |
| **5** | Probe + dijagnostika: liveness/readiness/startup, playbook "zašto pod ne odgovara" | 48, 50, 51, 52, 53 |
| **6** | Config/Secret/Volume: ConfigMap, Secret, PVC, image-pull Secret | 2, 23–26, 29–37 |
| **7** | Ostali kontroleri: StatefulSet, DaemonSet, Job, CronJob | 16, 17, 18, 19, 20, 21, 27 |

Napredak: **Faze 1–3 KOMPLETNE** ✓ · F1 (1, 11, 15, 3) · F2 (4, 5, 6, 7, 9, 10) · F3 (8, 13, 14, 22, 28) · **Faza 4 KOMPLETNA** ✓ → 12, 38, 39, 40, 42, 45, 46, 41, 43, 44, 47, 49 · **Faza 5 KOMPLETNA** ✓ → 48, 50, 51, 52, 53 · **Faza 6 KOMPLETNA** ✓ → 2, 23, 24, 25, 26, 29, 30, 31, 32, 33, 34, 35, 36, 37 · **Faza 7 KOMPLETNA** ✓ → 16, 17, 18, 19, 20, 21, 27 — **LO4 GOTOV 53/53** 🎯

**Izvan ovih 53** (relevantno za ispit, dodaje se kao zasebne sekcije kasnije):
Ingress (Servisi su pokriveni, Ingress nije) · k8s error-katalog za LO5 (ImagePullBackOff, CrashLoopBackOff, OOMKilled, Pending…) · LO6 teorija (Docker Swarm, k3s, OpenShift vs k8s, k8s arhitektura)

---

## Faza 0 — start klastera (uvijek prvo)

```bash
minikube start --driver=podman
kubectl get nodes      # čvor mora pisati Ready prije svega
```

- `minikube` = sićušan klaster (1 čvor = control-plane + worker u jednom)
- `--driver=podman` = rootless podman backend, bez sudo
- `kubectl get nodes` = STATUS mora biti `Ready` (ROLES: control-plane)

### Gotča: start padne s `permission denied` na `/root/.minikube`

Simptom (cijela greška):
```
Exiting due to HOST_HOME_PERMISSION: Failed to start host: provision:
Error getting config for native Go SSH: open /root/.minikube/machines/minikube/id_rsa: permission denied
```

Uzrok: stari minikube profil pokazuje na `/root` (iz ranije root sesije na VM-u).
`$HOME` je `/home/student`, ali profil tvrdoglavo gađa `/root` → bez sudo = permission denied.
Provjera uzroka:
```bash
echo $HOME              # /home/student
echo $MINIKUBE_HOME     # prazno = nije postavljeno
ls -a ~/.minikube       # config postoji i ispravan je, ali profil gađa /root
```

Popravak (tri koraka):
```bash
minikube delete --all                      # makni zatrovani profil (NE traži sudo)
export MINIKUBE_HOME=$HOME/.minikube       # prikuj minikube na home
minikube start --driver=podman             # čist start
```

- ključ je u samom ispisu: minikube doslovno kaže `Running "minikube delete" may fix it`
- `MINIKUBE_HOME` = env varijabla koja forsira gdje minikube drži profil → referenca: minikube.sigs.k8s.io (na whitelisti)
- provjera uspjeha: start završi s `Done! kubectl is now configured...`, pa `kubectl get nodes` → Ready

### Gotča: `kubectl` daje `connection refused` (klaster "zaspao")

Simptom:
```
... dial tcp 192.168.49.2:8443: connect: connection refused
```

Dijagnoza — UVIJEK prvo provjeriti stanje:
```bash
minikube status
```
Tipičan ispis kad je klaster stao:
```
host: Running          # podman kontejner RADI
kubelet: Stopped       # ali mozak klastera STOJI
apiserver: Stopped     # API server (s kojim kubectl priča) STOJI
kubeconfig: Configured
```

Popravak:
```bash
minikube start --driver=podman    # ponovo digne apiserver + kubelet
kubectl get nodes                 # potvrdi Ready
```

- klaster NE preživljava restart VM-a / dužu pauzu / pritisak na RAM sam od sebe → treba ručni `minikube start`
- ovaj start je BRZ: u ispisu se vidi `Updating the running podman container` (ne `Creating`) jer host već stoji
- na ispitu: ako `kubectl` ne odgovara → prvo `minikube status`, pa `minikube start`

### Bitno: `minikube start` (budi) vs `minikube delete` (briše SVE)

- **`minikube start`** na zaspalom klasteru = samo probudi apiserver/kubelet. Objekti (Deployment, RS, Pod) žive u **etcd** i PREŽIVE → rad ostaje.
- **`minikube delete`** = briše cijeli klaster; sljedeći start je prazan (gube se svi Deploymenti). Koristiti SAMO za popravak zatrovanog profila.

### Gotča: `RESTARTS` skoči nakon buđenja (restart ≠ recreate)

Nakon `minikube start` (buđenja) Podovi pokazuju npr. `RESTARTS: 1`, a imena su ISTA:
```
web-d5496596c-7rwbn    1/1   Running   1 (3m ago)   28m
```
- kubelet je ponovo pokrenuo CONTAINER unutar postojećeg Poda (`restartPolicy`) → isto ime, isti IP, RESTARTS +1
- to NIJE recreate: da je RS recreirao cijeli Pod (npr. nakon `kubectl delete pod`), nastalo bi NOVO ime i `RESTARTS: 0`
- razlika za ispit: **restart** = container u istom Podu; **recreate** = nov Pod

---

## Faza 1 — Kičma: Deployment → ReplicaSet → Pod

Cilj faze: lanac Deployment→RS→Pod uđe u prste i može se napisati i dokazati. Četiri zadatka:
- **zad. 1** — stvaranje lanca (Deployment) + izvoz YAML-a
- **zad. 11** — labele kao ljepilo lanca (selektor)
- **zad. 15** — goli Pod vs upravljani Pod (zašto kontroler postoji)
- **zad. 3** — skaliranje, kako RS reagira na promjenu željenog broja

## Zadatak 1 — Deployment `web` (nginx:1.25, 3 replike) + izvoz YAML-a

**Cilj:** imperativno stvoriti Deployment u jednoj naredbi, pa izvesti njegov YAML manifest.

### Koncept: imperativno vs deklarativno

- **Imperativno** = naredbom se specificira točno ŠTO se hoće, odmah, u jednom redu. Brzo, ad-hoc, ne ide u git.
- **Deklarativno** = napiše se YAML pa `kubectl apply -f file.yaml`. Verzionirano, ide u git, ponovljivo.
- Ovaj zadatak spaja oboje: imperativno se brzo stvori objekt → izvuče YAML → dobije deklarativni manifest. To je most između dva pristupa.

### Naredba (imperativno stvaranje)

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
```

Razlaganje svakog dijela:
- `kubectl` — alat kojim se šalje željeno stanje API serveru klastera
- `create deployment` — imperativna podnaredba; stvara objekt tipa Deployment
- `web` — ime Deploymenta → `metadata.name`; iz njega se izvode imena RS-a i Podova
- `--image=nginx:1.25` — slika containera:
  - `nginx` = ime slike (default registry = Docker Hub, default namespace = `library`)
  - `:1.25` = tag (verzija); BEZ taga povlači `:latest`
- `--replicas=3` — željeni broj Podova; BEZ ovoga default je 1

Očekivani ispis:
```
deployment.apps/web created
```
- `deployment.apps` = puni tip objekta (resurs `deployment`, API grupa `apps`)

### Što se stvarno dogodilo (provjereni ispis s VM-a)

**1) Deployment** — `kubectl get deployment web`:
```
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    0/3     3            0           10s
```
- `NAME` = web
- `READY` = spremni / željeni Podovi (0/3 → postaje 3/3)
- `UP-TO-DATE` = broj Podova ažuriranih na najnoviju spec (3)
- `AVAILABLE` = Podovi spremni posluživati promet (0 → postaje 3 kad prođu readiness)
- `AGE` = koliko Deployment postoji
- NAPOMENA: `0/3` i `AVAILABLE 0` ODMAH nakon stvaranja je NORMALNO — Podovi se još dižu. Za par sekundi → `3/3`.

**2) ReplicaSet** — `kubectl get rs`:
```
NAME             DESIRED   CURRENT   READY   AGE
web-d5496596c    3         3         3       14s
```
- `NAME` = web-**d5496596c** → ime Deploymenta + **pod-template-hash**
- `DESIRED` = koliko Podova RS TREBA držati (3)
- `CURRENT` = koliko ih trenutno postoji (3)
- `READY` = koliko ih je spremno (3)
- pod-template-hash `d5496596c` = otisak Pod template-a; RS njime prepoznaje SVOJE Podove

**3) Podovi** — `kubectl get pods`:
```
NAME                   READY   STATUS    RESTARTS   AGE
web-d5496596c-7rwbn    1/1     Running   0          20s
web-d5496596c-mzbkg    1/1     Running   0          20s
web-d5496596c-n5znf    1/1     Running   0          20s
```
- ime = web (Deployment) + d5496596c (ISTI hash kao RS!) + random sufiks (7rwbn…)
- `READY` = spremni containeri / ukupno u Podu (1/1)
- `STATUS` = Running · `RESTARTS` = 0 (nije se rušio)

### Lanac uživo (poanta zadatka)

```
Deployment: web
      │  (stvara i verzionira)
      ▼
ReplicaSet: web-d5496596c            ← pod-template-hash veže RS uz Deployment
      │  (drži točno 3 replike)
      ▼
Pods: web-d5496596c-7rwbn / -mzbkg / -n5znf   ← isti hash = pripadaju ovom RS-u
```
- pod-template-hash `d5496596c` je ljepilo: ISTI je na RS-u i na sva 3 Poda
- to je vidljiv dokaz lanca Deployment→RS→Pod

### Izvoz YAML-a — dvije varijante (znati OBJE, ispitivač voli razliku)

```bash
# A) iz živog objekta (doslovno po zadatku) — ali nosi runtime smeće
kubectl get deployment web -o yaml > web-deployment.yaml
```
- `-o yaml` = ispiši objekt kao YAML
- `>` = preusmjeri u datoteku (snima se TU gdje terminal stoji → `~/devops/lo4`)
- živi YAML sadrži `status:`, `metadata.managedFields`, `uid`, `resourceVersion`, `creationTimestamp` → treba očistiti prije ponovnog `apply`

```bash
# B) čisto generiranje BEZ stvaranja (spreman za git)
kubectl create deployment web --image=nginx:1.25 --replicas=3 --dry-run=client -o yaml > web-deployment.yaml
```
- `--dry-run=client` = NE šalji serveru, samo lokalno sastavi i ispiši što bi se stvorilo
- daje čist manifest bez runtime polja

### Anatomija izvezenog manifesta (`cat web-deployment.yaml`)

```yaml
apiVersion: apps/v1        # API grupa/verzija — Deployment živi u "apps/v1"
kind: Deployment           # tip objekta
metadata:                  # podaci O Deploymentu
  labels: {app: web}       #   labela samog Deployment objekta
  name: web                #   ime → odavde imena RS-a i Podova
spec:                      # ŽELJENO stanje
  replicas: 3              #   koliko Podova
  selector:                #   KOJE Podove Deployment smatra svojima
    matchLabels: {app: web}
  template:                # KALUP po kojem se peku Podovi
    metadata:
      labels: {app: web}   #   labela koju svaki Pod dobije
    spec:
      containers:
      - image: nginx:1.25  #   slika
        name: nginx        #   ime containera
```

Tri ključne stvari:
1. **`selector.matchLabels` MORA = `template.metadata.labels`** (oboje `app: web`). Selector traži Podove s app=web, template peče Podove baš s tom labelom. Ne poklope li se → greška (vidi LO5 zad. 26).
2. **`template` = kalup za JEDAN Pod**; RS ga umnoži na `replicas`. Promjena u kalupu (npr. slika) → novi pod-template-hash → novi RS → rolling update.
3. **Ostaci `--dry-run`:** `creationTimestamp: null`, `strategy: {}`, `resources: {}`, `status: {}` = prazne ljušture, bezopasne, `apply` ih ignorira. (Živi `get -o yaml` ostavlja MNOGO više smeća: `uid`, `managedFields`, puni `status`.)

### Gotče

- bez `--replicas` → 1 replika (default)
- bez taga (`nginx`) → povlači `:latest` (nepredvidljivo)
- `READY 0/3` odmah nakon stvaranja je NORMALNO — Podovi se dižu; provjeriti opet za par sekundi → 3/3
- živi `get -o yaml` ima runtime smeće → za čist manifest koristiti `--dry-run=client`
- `>` snima u TRENUTNI folder → provjeriti `pwd` prije izvoza da se YAML ne izgubi
- pod-template-hash se mijenja kad se promijeni Pod template (npr. druga slika) → tada nastaje NOVI RS (bitno za rolling update, Faza 2)
- **`error: exactly one NAME is required, got N`** → izostavljena crtica kod zastavice (`o yaml` umjesto `-o yaml`); kubectl tada čita `o`/`yaml` kao dodatna imena. Provjeriti crtice: `-o` (jedna), `--dry-run=client` (dvije)
- **`cat: ... No such file`** → tipfeler u imenu datoteke. Dovršiti ime tabom radi izbjegavanja greške

### Provjera

```bash
kubectl get deployment web    # READY 3/3, UP-TO-DATE 3, AVAILABLE 3
kubectl get rs                # web-<hash>: DESIRED/CURRENT/READY = 3/3/3
kubectl get pods              # 3× web-<hash>-<rnd>, svi Running 1/1
kubectl get all               # cijeli lanac Deployment→RS→Pod odjednom
```

---

## Zadatak 11 — izlistaj Podove Deploymenta preko label selektora

**Cilj:** filtrirati Podove po labeli i razumjeti vezu Deployment → ReplicaSet → Pod preko selektora.

**Koncept:** lanac drže LABELE, ne imena. `kubectl create deployment web` stavi labelu `app=web` na Pod template. RS ima selektor `app=web` (+ pod-template-hash) → tako zna koje Podove broji i drži. Filtriranje po labeli = ručno korištenje istog mehanizma.

### Naredba (filter po labeli)

```bash
kubectl get pods -l app=web
```

- `-l` (malo L) — kratica za `--selector`; filter po labeli
- `app=web` — equality match; samo Podovi s točno tom labelom

### Vidi sve labele na Podovima

```bash
kubectl get pods --show-labels
```

Provjereni ispis s VM-a:
```
NAME                   READY   STATUS    ...   LABELS
web-d5496596c-7rwbn    1/1     Running   ...   app=web,pod-template-hash=d5496596c
web-d5496596c-mzbkg    1/1     Running   ...   app=web,pod-template-hash=d5496596c
web-d5496596c-n5znf    1/1     Running   ...   app=web,pod-template-hash=d5496596c
```

- `app=web` = labela iz `template.metadata.labels` (zad. 1); po njoj se filtrira
- `pod-template-hash=d5496596c` = ISTI hash kao na RS-u (`replicaset.apps/web-d5496596c`) → vidljivo ljepilo: RS-ov selektor gađa taj hash, svaki Pod ga nosi

### Veza u jednom pogledu

```
Deployment web  ──stvori──►  Podovi s labelom app=web
ReplicaSet web-d5496596c  ──selektor app=web + hash──►  broji i drži te iste Podove
kubectl get pods -l app=web  ──►  ručno korištenje istog selektora
```

### Ostali oblici selektora (za referencu)

```bash
kubectl get pods -l 'app in (web,api)'    # set-based: app je web ILI api
kubectl get pods -l 'app!=web'            # negacija: sve osim app=web
kubectl get pods -l app                   # postoji labela app (bilo koja vrijednost)
kubectl get pods -L app                   # -L (veliko) = DODAJ stupac za labelu (NE filtrira)
kubectl get pods -l app=web,pod-template-hash=d5496596c   # zarez = logički I
```

### Gotče

- `-l` malo = **filter**; `-L` veliko = **dodatni stupac** (lako se zamijene!)
- labele case-sensitive: `app=web` ≠ `app=Web`
- promašena labela → `No resources found in default namespace.` (prazno, **NIJE greška**)
- razmaci u set-based selektoru → pod navodnike `'...'`
- **`the server doesn't have a resource type "allč"`** → zalutali znak u naredbi (npr. hrvatsko `č` umjesto `all`); kubectl ga čita kao nepostojeći tip resursa. Paziti na raspored tipkovnice.

### Provjera

```bash
kubectl get pods -l app=web          # 3 Poda
kubectl get pods --show-labels       # LABELS: app=web + pod-template-hash
kubectl get pods -l app=ne-postoji   # prazno → filter stvarno sužava
```

---

## Zadatak 15 — goli Pod (bez kontrolera) vs Deployment-Pod

**Cilj:** napraviti goli Pod, obrisati ga (ostaje obrisan), pa obrisati Deployment-Pod (RS ga zamijeni) — dokaz self-healinga.

**Koncept:** goli Pod nema kontrolera koji ga nadzire → obrisan, OSTAJE obrisan. Deployment-Pod čuva ReplicaSet (selektor `app=web`) → obrisan, RS ga zamijeni NOVIM (novo ime/IP, RESTARTS 0). To je self-healing deklarativnog modela uživo.

### Napravi goli Pod

```bash
kubectl run sleeper --image=busybox -- sleep 3600
```

- `kubectl` — klijent koji šalje željeno stanje API serveru
- `run` — podnaredba; stvara GOLI POD (jedan Pod, bez kontrolera). NAPOMENA: danas (k8s ≥1.18) `run` stvara isključivo Pod (nekad je znao Deployment)
- `sleeper` — ime Poda → `metadata.name`; bez kontrolera ime ostaje točno takvo (nema random sufiksa)
- `--image=busybox` — slika; busybox = sićušna Linux slika; bez taga → `:latest`
- `--` — razdjelnik: sve iza NIJE kubectl zastavica, nego naredba za container
- `sleep 3600` — naredba u containeru: spavaj 1h → drži Pod živim. Bez naredbe busybox pokrene `sh` koji odmah izađe → `CrashLoopBackOff`

Ispis: `pod/sleeper created`

### Eksperiment — goli Pod nestane zauvijek

```bash
kubectl get pods             # sleeper Running, RESTARTS 0, uz 3 web Poda
kubectl delete pod sleeper   # pod "sleeper" deleted
kubectl get pods             # sleeper NESTAO i ostaje nestao (samo 3 web Poda)
```

### Eksperiment — Deployment-Pod se zamijeni

```bash
kubectl delete pod web-d5496596c-7rwbn   # obrisati JEDAN web Pod (prekopirati stvarno ime)
kubectl get pods                          # NOVI web Pod s drugim sufiksom!
```

Provjereni ispis nakon brisanja:
```
NAME                   READY   STATUS    RESTARTS       AGE
web-d5496596c-fw8wv    1/1     Running   0              6s     ← NOVI (recreate): novo ime, RESTARTS 0
web-d5496596c-mzbkg    1/1     Running   1 (13m ago)    38m    ← stari (preživio buđenje: RESTARTS 1)
web-d5496596c-n5znf    1/1     Running   1 (13m ago)    38m    ← stari
```

- obrisan `7rwbn` → RS odmah napravio `fw8wv` da opet bude 3 (self-healing)
- hash `d5496596c` ostaje isti (Pod template nepromijenjen, samo jedan primjerak zamijenjen)

### restart vs recreate — vidljivo u istom ispisu (ZA ISPIT)

- `fw8wv` ima **novo ime** + `RESTARTS 0` → to je **recreate** (potpuno nov Pod, jer je stari obrisan)
- `mzbkg`/`n5znf` imaju **ista imena** + `RESTARTS 1` → to je bio **restart** (kubelet digao container u istom Podu nakon buđenja klastera)
- goli Pod ima `restartPolicy` (container se diže u mjestu) ALI nema kontrolera → obrisan se ne vraća; Deployment-Pod ima oboje

### Varijanta — goli Pod s drugom restartPolicy

```bash
kubectl run jednom --image=busybox --restart=Never -- echo hello
```

- `--restart=Never` → `restartPolicy: Never` (container se NE diže nakon izlaska)
- u `kubectl run`: `--restart` postavlja restartPolicy Poda (Always = default / OnFailure / Never)

Provjereni ishod — `kubectl get pods`:
```
NAME      READY   STATUS      RESTARTS   AGE
jednom    0/1     Completed   0          57s
```
- `echo hello` se izvrši u sekundi → container uredno izađe → `Completed`
- `READY 0/1` = 0 od 1 containera trenutno radi (jer je gotov) — to NIJE greška
- `restartPolicy: Never` → ne diže se ponovo (s `Always` bi se vrtio u krug i pokazao `RESTARTS` rast)
- počistiti ga: `kubectl delete pod jednom`

### Gotče

- `kubectl run` danas pravi **Pod**, NE Deployment (stari tutorijali pre-1.18 tvrde drukčije — ne vrijedi više)
- busybox bez duge naredbe → `sh` izađe odmah → `CrashLoopBackOff`. Zato `sleep`
- zamjenski Deployment-Pod ima **novo ime** (nov random sufiks) — ne očekivati isto ime natrag
- da se stvarno maknu svi Podovi Deploymenta → obrisati **Deployment** ili `kubectl scale --replicas=0`; brisanje pojedinog Poda samo okida zamjenu
- **`kubectl delete pod` na par sekundi "zamrzne" prompt** — to je NORMALNO: kubectl čeka graceful shutdown (zadani grace period 30s, obično završi prije). Nije bug ni spori VM
- **`Completed` status** (kod golog Poda s naredbom koja završi) = container uredno odradio i izašao → NIJE greška. `READY 0/1` jer više ne radi
- **`Error from server (AlreadyExists): pods "X" already exists`** → već postoji Pod tog imena u namespaceu; obrisati stari ili koristiti drugo ime

### Provjera

```bash
kubectl get pods -o wide     # imena + IP-jevi (novi Pod ima novi IP)
kubectl get all              # sleeper bi stajao sam; web Podovi vise pod RS-om
```

---

## Zadatak 3 — skaliranje `web` 3→5 na dva načina + razlika disk vs klaster

**Cilj:** promijeniti broj replika imperativno (`scale`) i deklarativno (`edit`/`apply`), te razumjeti zašto se datoteka na disku i živi objekt u klasteru mogu razići.

### Način A — imperativno (`kubectl scale`)

```bash
kubectl scale deployment web --replicas=5
```

- `kubectl` — klijent
- `scale` — podnaredba za promjenu broja replika
- `deployment web` — tip resursa (`deployment`) + ime (`web`)
- `--replicas=5` — novi željeni broj Podova

Provjereni ispis:
```
deployment.apps/web scaled
$ kubectl get rs
NAME            DESIRED   CURRENT   READY   AGE
web-d5496596c   5         5         5       60m
```
- 5 Podova (stara 3 + 2 nova sufiksa, RESTARTS 0)
- **RS se NE mijenja**: isti `web-d5496596c`, samo DESIRED skoči 3→5. Hash isti jer se Pod template (slika/labele) ne dira — mijenja se *koliko*, ne *kakvi*
- `metadata.generation` se poveća (Deployment broji svoje izmjene)

### Način B1 — deklarativno preko editora (`kubectl edit`)

```bash
kubectl edit deployment web
```

- `edit` — otvara ŽIVI objekt u editoru (zadano `vi`); spremanjem → kubectl odmah primijeni
- naći `replicas:`, promijeniti broj, spremiti
- `vi` komande: vidi sekciju "`vi` — preživljavanje editora" na vrhu skripte (ukratko: `i` insert, `Esc`, `:wq` spremi, `:q!` odustani)

GOTČA: `deployment.apps/web edited` se ispiše i kad NIJE efektivno ništa promijenjeno (npr. ostao isti broj). "edited" ≠ nužno "promijenjeno" — provjeriti s `get`.

### Način B2 — deklarativno preko datoteke (`kubectl apply`)

```bash
kubectl apply -f ~/devops/lo4/web-deployment.yaml
```

- `apply -f <file>` — namjesti klaster PREMA datoteci ("neka stanje bude ovakvo")
- pošto `web-deployment.yaml` kaže `replicas: 3`, a klaster je bio 5 → apply vrati na **3**

Provjereni ispis:
```
Warning: resource deployments/web is missing the kubectl.kubernetes.io/last-applied-configuration annotation ...
deployment.apps/web configured
$ kubectl get deployment web
web   3/3   ...
```
- `configured` (ne `created`) jer objekt postoji, samo je usklađen
- WARNING o `last-applied-configuration`: normalno jer je objekt stvoren s `create`, a sad prvi put diran s `apply`. Bezopasno, `will be patched automatically`

### SRŽ — disk vs klaster (najvažnije za ispit)

```bash
cat ~/devops/lo4/web-deployment.yaml | grep replicas   # replicas: 3  (datoteka ZAMRZNUTA u trenutku izvoza)
kubectl get deployment web -o yaml | grep replicas     # replicas: 5  (živi objekt — STVARNO stanje)
```

- `scale`/`edit` mijenjaju **živi objekt**; datoteka na disku ZAOSTAJE → klaster i datoteka se raziđu
- `apply -f` ide obrnuto: **datoteka je izvor istine**, klaster se namjesti prema njoj
- zato: `scale` na 5 + `apply` datoteke s `replicas: 3` = završetak na **3**

Bonus: živi `get -o yaml` otkriva default polja korisna za kasnije faze:
- `strategy.rollingUpdate: maxSurge 25%, maxUnavailable 25%` → Faza 2 (zad. 6)
- `revisionHistoryLimit: 10` → Faza 2 (zad. 9)
- `progressDeadlineSeconds: 600` → Faza 5 (zad. 39)
- `terminationGracePeriodSeconds: 30` → onaj grace period iz zad. 15

### Gotče

- RS se kod skaliranja NE mijenja (isti hash) — mijenja se samo broj; novi RS nastaje samo na promjenu Pod template-a (slika itd.)
- `edit` ispiše `edited` i bez stvarne promjene → uvijek provjeriti `get`
- `apply` warning o `last-applied-configuration` na objektu stvorenom s `create` → bezopasno
- skaliranje na 0 (`--replicas=0`) ugasi sve Podove ali Deployment/RS ostaju → način da se app "ugasi" bez brisanja
- **`the server doesn't have a resource type "ea"/"allč"`** → zalutali/krivi naziv tipa resursa; `kubectl get <tip>` traži točan naziv (`rs`, `all`, `pods`...)

### Provjera

```bash
kubectl get deployment web    # READY = traženi broj
kubectl get rs                # isti RS, DESIRED = traženi broj
kubectl get pods              # toliko Podova
```

---

## Faza 2 — Rolloutovi: rolling update, rollback, strategije, zaglavljeni rollout

Cilj faze: mijenjati aplikaciju koja već radi bez prekida, i vraćati se ako nešto pođe po zlu. Šest zadataka:
- **zad. 4** — rolling update (nginx:1.25 → 1.27) + `rollout status`
- **zad. 5** — rollout history + rollback + CHANGE-CAUSE
- **zad. 6** — strategija RollingUpdate: `maxSurge` / `maxUnavailable`
- **zad. 7** — strategija Recreate + kad je nužna
- **zad. 9** — `revisionHistoryLimit` (koliko starih RS-ova se čuva)
- **zad. 10** — slika na nepostojeći tag → zaglavljeni rollout

Temelj cijele faze (iz Faze 1): promjena Pod template-a (npr. slike) → novi pod-template-hash → NOVI RS. Rolling update = Kubernetes pravi novi RS i postupno seli replike sa starog na novi.

## Zadatak 4 — rolling update `web` (nginx:1.25 → 1.27) + `rollout status`

**Cilj:** promijeniti sliku aplikacije bez prekida prometa i pratiti napredak.

**Koncept:** rolling update mijenja Podove POSTUPNO (digne novi, ugasi stari, …) tako da uvijek ima dovoljno živih → nula downtimea. Ispod haube: promjena slike → novi Pod template → novi pod-template-hash → NOVI RS. Deployment seli replike: stari RS 3→0, novi RS 0→3. Stari RS ostaje PRAZAN (ne obrisan) radi rollbacka.

### Naredba (promjena slike)

```bash
kubectl set image deployment/web nginx=nginx:1.27
```

- `kubectl` — klijent
- `set image` — podnaredba koja mijenja SAMO sliku containera (imperativno)
- `deployment/web` — tip/ime: na Deploymentu `web`
- `nginx=nginx:1.27` — format `<ime-containera>=<nova-slika>`. Lijevo `nginx` = IME CONTAINERA (iz manifesta `name: nginx`), desno `nginx:1.27` = nova slika

ZAMKA: lijevo od `=` ide IME CONTAINERA, ne ime slike. Ako je container nazvan drukčije, lijevi dio je to ime.

Ispis: `deployment.apps/web image updated`

### Praćenje rollouta (odmah nakon set image)

```bash
kubectl rollout status deployment/web
```

- `rollout status` — prati napredak uživo i BLOKIRA dok ne završi
- ispisuje `Waiting for deployment "web" rollout to finish: N out of 3 new replicas updated...` → na kraju `deployment "web" successfully rolled out`

### Što se stvarno dogodilo (provjereni ispis s VM-a)

`kubectl get rs` — DVA ReplicaSeta:
```
NAME            DESIRED   CURRENT   READY   AGE
web-c7d75d445   3         3         3       91s    ← NOVI RS (nginx:1.27), drži svih 3
web-d5496596c   0         0         0       107m   ← STARI RS (nginx:1.25), ispražnjen na 0
```

- **`d5496596c`** = stari RS (ISTI hash kroz cijelu Fazu 1!), sad `DESIRED 0` — ispražnjen ali NIJE obrisan
- **`c7d75d445`** = novi RS, nastao jer je promjena slike dala novi pod-template-hash → drži 3 Poda (nginx:1.27)
- Deployment je prebacio replike: stari 3→0, novi 0→3
- Podovi imaju nova imena (`web-c7d75d445-...`), RESTARTS 0, svježi AGE; stari Podovi nestali

`kubectl describe deployment web | grep Image` → `nginx:1.27` (potvrda nove slike).

### Zašto stari RS ostaje prazan

Namjerno — to je mreža za **rollback** (zad. 5). Ako nova verzija zakaže, Deployment vrati replike na stari RS (`d5496596c`: 0→3). `revisionHistoryLimit` (zad. 9) određuje koliko se starih RS-ova čuva.

### Alternativni način (deklarativno)

```bash
kubectl edit deployment web        # promijeni image: nginx:1.27 u template, spremi
# ili izmijeni web-deployment.yaml pa: kubectl apply -f web-deployment.yaml
```

### Gotče

- lijevo od `=` u `set image` = IME CONTAINERA, ne slike
- rolling update NE mijenja Podove u mjestu → pravi NOVI RS i seli replike (stari → 0, novi → N)
- stari RS ostaje s `DESIRED 0` (ne briše se) → omogućuje rollback
- `rollout status` blokira terminal dok update ne završi (to je namjerno, ne zamrznuće)
- novi pod-template-hash nastaje SAMO na promjenu Pod template-a (slika, env, labele…), ne na skaliranju

### Provjera

```bash
kubectl rollout status deployment/web              # successfully rolled out
kubectl get rs                                     # stari RS DESIRED 0, novi RS DESIRED N
kubectl get pods                                   # nova imena (novi hash), RESTARTS 0
kubectl describe deployment web | grep Image       # nova slika (nginx:1.27)
```

---

## Zadatak 5 — rollout history + rollback (undo) + CHANGE-CAUSE

**Cilj:** pogledati povijest revizija, vratiti se na prethodnu, i zabilježiti razlog promjene.

**Koncept:** stari RS koji rolling update ostavi prazan (zad. 4) omogućuje rollback — `rollout undo` prebaci replike natrag na njega. Revizije se vode kao rastući brojevi.

### Dio 1 — povijest revizija

```bash
kubectl rollout history deployment/web
```

- `rollout history` — ispiše sve revizije
- ispis (prije bilježenja razloga):
```
REVISION   CHANGE-CAUSE
1          <none>           ← početno (nginx:1.25)
2          <none>           ← rolling update (nginx:1.27)
```
- `CHANGE-CAUSE` = `<none>` jer razlog nije bilježen (vidi Dio 3)

### Dio 2 — rollback (vrati na prethodnu reviziju)

```bash
kubectl rollout undo deployment/web
```

- `rollout undo` — vrati na PRETHODNU reviziju (bez argumenta = jedan korak unatrag)
- na određenu reviziju: `kubectl rollout undo deployment/web --to-revision=1`
- radi kao rolling update obrnuto: prebaci replike sa sadašnjeg RS-a natrag na stari

Ispis: `deployment.apps/web rolled back`

Provjereni rezultat — RS-ovi zamijenili uloge:
```
prije undo:  c7d75d445 DESIRED 3 (nginx:1.27),  d5496596c DESIRED 0
poslije:     c7d75d445 DESIRED 0,               d5496596c DESIRED 3 (nginx:1.25)
```
- `describe deployment web | grep Image` → opet `nginx:1.25`
- stari RS se nije obrisao baš zato da bi rollback bio moguć

### KLJUČNO — kako se revizije presložе nakon undo

```
prije undo:   REVISION 1, 2
poslije undo: REVISION 2, 3      ← rev 1 "nestao", pojavio se rev 3
```

- rollback NE vraća broj unatrag — uzme sadržaj rev 1 (nginx:1.25) i premjesti ga naprijed kao rev 3
- rev 1 nestane iz liste jer je njegov sadržaj sad rev 3
- **revizije UVIJEK samo rastu; najveći broj = najnovija akcija**

### Dio 3 — CHANGE-CAUSE (zabilježi razlog)

```bash
kubectl annotate deployment/web kubernetes.io/change-cause="update na nginx 1.27"
```

- `annotate` — dodaje anotaciju objektu
- `kubernetes.io/change-cause` — posebna anotacija koju `rollout history` čita kao CHANGE-CAUSE
- nakon toga `rollout history` pokazuje opis umjesto `<none>` za tu reviziju
- stariji način `--record` (npr. `kubectl set image ... --record`) je DEPRECATED → koristiti anotaciju

### Gotče

- `rollout undo` bez argumenta = prethodna revizija; `--to-revision=N` = točno određena
- dva uzastopna `undo` vrate na polazno (svaki undo prebaci na "ono prije sadašnjeg")
- revizije samo rastu — rollback "premjesti" staru reviziju naprijed kao novu
- rollback zahtijeva da stari RS postoji → ovisi o `revisionHistoryLimit` (zad. 9)
- CHANGE-CAUSE `<none>` znači da razlog nije anotiran — nije greška

### Provjera

```bash
kubectl rollout history deployment/web             # popis revizija + CHANGE-CAUSE
kubectl rollout undo deployment/web                # rolled back
kubectl rollout status deployment/web              # successfully rolled out
kubectl describe deployment web | grep Image       # slika prethodne revizije
kubectl get rs                                     # RS-ovi zamijenili DESIRED
```

---

## Zadatak 6 — strategija RollingUpdate: maxSurge / maxUnavailable

**Cilj:** postaviti `maxSurge: 1` i `maxUnavailable: 0` te razumjeti efekt na dostupnost tijekom update-a.

**Koncept:** rolling update seli Podove postupno; ova dva parametra određuju koliko agresivno:
- **`maxUnavailable`** = koliko Podova smije NEDOSTAJATI tijekom update-a. Niže = sigurnije, sporije.
- **`maxSurge`** = koliko Podova smije privremeno VIŠKA iznad željenog broja. Više = brže, traži više resursa.
- default (vidljiv u živom YAML-u) = `maxSurge: 25%, maxUnavailable: 25%`.

`maxSurge: 1, maxUnavailable: 0` znači: NIKAD ispod željenog broja (svih 3 stara žive dok se ne digne zamjena), uz najviše 1 višak → privremeno 4 Poda, smjena po jedan. Najsigurnije, malo sporije.

### Postavljanje (izmjena manifesta — NIJE imperativna naredba)

`maxSurge`/`maxUnavailable` su ugniježđena polja u `spec.strategy`, pa se mijenjaju kroz `edit`/`apply`:

```bash
kubectl edit deployment web
```

Dio za izmjenu:
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # bilo 25%
      maxUnavailable: 0    # bilo 25%
```

### Provjera (3 načina)

```bash
kubectl describe deployment web | grep -i "RollingUpdateStrategy"
# → RollingUpdateStrategy:  0 max unavailable, 1 max surge
kubectl describe deployment web | grep -i strategy -A2
kubectl get deployment web -o yaml | grep -A3 strategy
```

### Vidjeti strategiju na djelu

Promjena strategije se NE primjenjuje odmah — vrijedi na SLJEDEĆI rollout. Za demonstraciju:
```bash
kubectl set image deployment/web nginx=nginx:1.25
kubectl get pods -w        # -w = watch; gleda uživo, Ctrl+C za izlaz
```
S `maxSurge:1, maxUnavailable:0` nakratko bude 4 Poda (nikad ispod 3), smjena po jedan.
NAPOMENA: na sitnom klasteru (slika cacheana) update proleti u sekundi → watch zna uhvatiti tek krajnje stanje. Da se vidi međukorak, pokrenuti `-w` u jednom terminalu PRIJE `set image` u drugom.

### Gotče

- vrijednosti mogu biti broj (`1`) ILI postotak (`25%`) — oba rade
- `maxSurge` i `maxUnavailable` NE smiju oba biti 0 → update se ne bi mogao pomaknuti (zaglavi)
- promjena strategije ne pokreće rollout sama — primjenjuje se na sljedeći update
- vraćanje na već korišten template (npr. nginx:1.25) koristi POSTOJEĆI stari RS (`d5496596c`), ne pravi novi
- `kubectl get ... -w` (watch) drži terminal i osvježava na promjenu → Ctrl+C za izlaz

### Pomoćna vještina — `grep -A / -B / -C` (za čitanje describe/YAML ispisa)

```bash
grep -A3 strategy   # After:   pogođeni redak + 3 ISPOD
grep -B3 strategy   # Before:  pogođeni redak + 3 IZNAD
grep -C3 strategy   # Context: + 3 IZNAD I ISPOD
```
- broj = koliko redaka konteksta; korisno kad je vrijednost polja u susjednim retcima
- primjer: `kubectl get pod X -o yaml | grep -A5 resources`

### Provjera

```bash
kubectl describe deployment web | grep -i "RollingUpdateStrategy"   # potvrda brojeva
kubectl get pods -w                                                 # ponašanje tijekom update-a
```

---

## Zadatak 7 — strategija Recreate (+ zašto `apply` uspije gdje `edit` padne)

**Cilj:** prebaciti strategiju Deploymenta s RollingUpdate na Recreate i opisati konkretan scenarij u kojem je Recreate nužan.

**Koncept:** dvije strategije ažuriranja Deploymenta:
- **RollingUpdate** (default) = postupna smjena Podova; stara i nova verzija nakratko KOEGZISTIRAJU; **bez downtimea** (zad. 4, 6)
- **Recreate** = ugasi SVE stare Podove → tek onda digni nove. Postoji trenutak kad NIJEDAN Pod ne poslužuje → **kratak downtime**

Recreate je NUŽAN kad stara i nova verzija NE smiju raditi istovremeno:
- migracija sheme baze (stari kod nekompatibilan s novom shemom)
- ekskluzivni `ReadWriteOnce` volumen ili lock koji smije držati samo jedan Pod
- nekompatibilni protokoli / format podataka između verzija

RollingUpdate se bira kad je bitan uptime i verzije mogu koegzistirati.

### Postavljanje (izmjena manifesta)

`strategy` je polje u `spec`, pa se mijenja kroz datoteku + `apply` (ili `edit`). Dio manifesta:
```yaml
spec:
  strategy:
    type: Recreate
```
- `strategy:` — 2 razmaka uvlake (dijete od `spec`)
- `type: Recreate` — 4 razmaka (dijete od `strategy`)
- VAŽNO: uz `type: Recreate` NE smije postojati `rollingUpdate:` blok (vidi gotču)

Primjena iz datoteke:
```bash
kubectl apply -f ~/devops/lo4/web-deployment.yaml
```
- `apply` — pomiri živi objekt prema datoteci; datoteka = izvor istine
- `-f` — file; slijedi putanja do manifesta

Provjereni ispis s VM-a:
```
deployment.apps/web configured
```
- `configured` = objekt je postojao i izmijenjen je (za razliku od `created` / `unchanged`)

### Provjera da je strategija stvarno promijenjena

```bash
kubectl describe deployment web | grep -i strategy
```
- `describe deployment web` — ljudski čitljiv sažetak živog objekta
- `| grep -i strategy` — propusti retke s "strategy"; `-i` = ignoriraj velika/mala slova

Provjereni ispis s VM-a:
```
StrategyType:   Recreate
```
- SAMO taj redak → blok `rollingUpdate` je nestao sa živog objekta (čisto)
- da je ostao, pojavio bi se i redak `RollingUpdateStrategy: ...` uz Recreate → nesklad (dva oprečna polja)

### KLJUČNA GOTČA — `Forbidden` pri prelasku na Recreate kroz `edit`

Pokušaj prebacivanja na Recreate kroz `kubectl edit` (kad u živom objektu već postoji `rollingUpdate` blok) vrati grešku:
```
spec.strategy.rollingUpdate: Forbidden: may not be specified when strategy type is 'Recreate'
```
- uzrok: `edit` mijenja SAMO ono što se dira u editoru; ako se upiše `type: Recreate` a stari `rollingUpdate:` (maxSurge/maxUnavailable) ostane → server dobije oba oprečna polja istovremeno → odbija
- popravak: pri prelasku na Recreate MORA se obrisati cijeli `rollingUpdate:` blok; ostaje samo `type: Recreate`

Kako pročitati razlog kad `edit` odbije izmjenu:
- nakon odbijanja, na vrhu privremene datoteke `/tmp/kubectl-edit-*.yaml` `edit` upiše točan razlog kao komentar `# *`
- UVIJEK prvo pročitati taj `# *` redak — doslovno piše što ne valja

### Zašto `apply` uspije gdje `edit` padne

- **`apply`** šalje CIJELU datoteku kao željeno stanje. Datoteka nema `rollingUpdate` blok → apply ga obriše sa živog objekta → nema sukoba.
- **`edit`** je ručna izmjena na licu mjesta → sve što se ne dira ostaje → stari `rollingUpdate` preživi → sukob.
- pravilo za prste: **`apply` = što fali u datoteci briše se s objekta; `edit` = mijenja se samo dirano, ostalo ostaje kako je bilo.**

### Vidjeti Recreate na djelu (opcionalno — kontrast s RollingUpdateom)

Nije nužno za rješenje; služi da se downtime vidi uživo:
```bash
kubectl get pods -w                                    # terminal 1: watch (Ctrl+C za izlaz)
kubectl set image deployment/web nginx=nginx:1.27      # terminal 2: pokreni update
```
Očekivano (Recreate): SVI stari Podovi odu u `Terminating` → trenutak kad NIJEDAN ne radi → tek onda se dignu novi. Kontrast: RollingUpdate (zad. 6) nikad ne padne ispod željenog broja.

### Gotče

- uz `type: Recreate` NE smije stajati `rollingUpdate:` blok → inače `Forbidden`
- pri prelasku RollingUpdate→Recreate obrisati cijeli `rollingUpdate:` (maxSurge/maxUnavailable)
- `kubectl edit` koji odbije izmjenu: razlog je u `/tmp/kubectl-edit-*.yaml` na vrhu kao `# *` — pročitati prvo
- `apply` briše s objekta polja kojih nema u datoteci; `edit` ne dira što se ne mijenja → zato je `apply` ovdje prošao bez ručnog čišćenja
- Recreate = downtime; bira se SAMO kad stara i nova verzija ne smiju koegzistirati

### Provjera

```bash
kubectl apply -f ~/devops/lo4/web-deployment.yaml          # deployment configured
kubectl describe deployment web | grep -i strategy         # StrategyType: Recreate (bez RollingUpdateStrategy)
```

---

## Zadatak 9 — `revisionHistoryLimit: 3`

**Cilj:** postaviti `revisionHistoryLimit: 3` i razumjeti kako utječe na broj zadržanih starih RS-ova i na doseg rollbacka.

**Koncept:** `revisionHistoryLimit` = broj **starih (neaktivnih, DESIRED 0) RS-ova** koje Deployment ČUVA.
- aktivni RS se uvijek čuva i NE broji se u limit
- default = **10** (zato se stari RS-ovi ne brišu sami od početka)
- veza s rollbackom: svaki zadržani stari RS = jedna revizija na koju `rollout undo` može stići. Kad limit obriše stari RS → ta revizija nestane iz `rollout history` → na nju se VIŠE ne može rollback.
- niži limit = čišća povijest, ali plići rollback; viši limit = dublji rollback, više zadržanih RS-ova

### Polazno stanje (prije postavljanja)

```bash
kubectl get rs
kubectl rollout history deployment/web
```

Provjereni ispis s VM-a:
```
NAME            DESIRED   CURRENT   READY   AGE
web-c7d75d445   0         0         0       8h     ← stari, prazan (nginx:1.27)
web-d5496596c   3         3         3       10h    ← aktivni (nginx:1.25)

REVISION   CHANGE-CAUSE
4          <none>
5          uppdate na nginx 1.27
```
- 2 RS ↔ 2 revizije: svaki zadržani RS = jedna revizija u povijesti
- brojevi revizija su 4 i 5 (ne 1 i 2) — revizije samo rastu (zad. 5), najveći broj = najnovija akcija
- 1 aktivni RS + 1 stari RS

### Postavljanje (izmjena manifesta)

`revisionHistoryLimit` je polje na glavnom `spec` (svojstvo Deploymenta, NE Poda). Dodaje se poravnato s `replicas`/`selector`/`strategy` (2 razmaka):
```yaml
spec:
  replicas: 3
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: web
  strategy:
    type: Recreate
```
- redoslijed polja unutar `spec` nije bitan; bitna je RAZINA (isto uvučeno kao `replicas`)
- NE smije ići pod ugniježđeni `spec` u `template:` (tamo su `containers` — to je Pod, ne Deployment)

Primjena:
```bash
kubectl apply -f ~/devops/lo4/web-deployment.yaml      # deployment.apps/web configured
```

### Provjera da je polje sjelo na živi objekt

```bash
kubectl get deployment web -o yaml | grep revisionHistoryLimit
```
- `get deployment web -o yaml` — cijeli živi objekt u YAML-u
- `| grep revisionHistoryLimit` — propusti samo taj redak

Provjereni ispis s VM-a (DVA pogotka, oba pokazuju 3):
```
    {"apiVersion":"apps/v1",...,"revisionHistoryLimit":3,...}   ← anotacija last-applied-configuration
  revisionHistoryLimit: 3                                       ← stvarno polje na spec
```
- gornji dugi redak = anotacija `kubectl.kubernetes.io/last-applied-configuration`: JSON kopija zadnje primijenjene datoteke; po njoj `apply` zna što obrisati kad se nešto izbaci iz datoteke
- donji redak = stvarno polje na `spec` živog objekta → to je ono što se traži

### KLJUČNA GOTČA — limit NE briše proaktivno do svoje vrijednosti

Nakon postavljanja `revisionHistoryLimit: 3`, broj RS-ova se NIJE promijenio — i dalje 2 RS-a. To je očekivano, NIJE greška:
- starih (DESIRED 0) RS-ova ima samo **1**, a limit je **3** → 1 < 3 → nema što rezati
- limit je GORNJA GRANICA koja reže tek kad broj starih RS-ova PRIJEĐE 3
- npr. na 4. starom RS-u, najstariji ispadne i njegova revizija nestane iz `rollout history`
- limit ne čuva "točno 3" niti briše do 3 — samo sprječava da ih bude VIŠE od 3

### Gotče

- `revisionHistoryLimit` broji SAMO stare (DESIRED 0) RS-ove; aktivni se ne broji
- default je 10 → zato se stari RS-ovi ne brišu sami dok ih ne bude >10
- limit ne briše proaktivno: reže tek kad broj starih RS-ova prijeđe limit
- ide na glavni `spec` (Deployment), ne pod `template.spec` (Pod)
- obrisani stari RS = izgubljena revizija → na nju više nema rollbacka (povezano sa zad. 5)

### Provjera

```bash
kubectl apply -f ~/devops/lo4/web-deployment.yaml                       # configured
kubectl get deployment web -o yaml | grep revisionHistoryLimit         # revisionHistoryLimit: 3
kubectl get rs                                                          # broj RS-ova nepromijenjen (1 stari < limit 3)
```

---

## Zadatak 10 — nepostojeći tag → zaglavljeni rollout (ImagePullBackOff)

**Cilj:** postaviti sliku na nepostojeći tag, promotriti da se rollout zaglavi, dijagnosticirati uzrok i objasniti ponašanje (uz važan obrat zbog strategije Recreate).

**Koncept:** kad se zada nepostojeća slika, Deployment napravi NOVI RS i pokuša dignuti nove Podove. Oni ne mogu povući sliku → zaglave (`ErrImagePull` → `ImagePullBackOff`).
- **uz RollingUpdate**: Deployment NE gasi stare Podove dok novi nije Ready. Novi nikad ne postane Ready → stari ostaju i DALJE POSLUŽUJU (ugrađena zaštita, veza s `maxUnavailable`, zad. 6).
- **uz Recreate (OVAJ slučaj!)**: Recreate prvo ugasi SVE stare → tek onda diže nove. Pošto su novi pokvareni → **NIJEDAN Pod ne radi → potpuni downtime**. To je konkretna opasnost Recreatea iz zad. 7, vidljiva uživo.

### Akcija — postavi nepostojeći tag (imperativno, brže za demonstraciju kvara)

```bash
kubectl set image deployment/web nginx=nginx:9.99
```
- `set image` — promijeni sliku containera u Deploymentu
- `deployment/web` — ciljni objekt
- `nginx=nginx:9.99` — lijevo od `=` IME CONTAINERA (`nginx`), desno nova slika; tag `9.99` NE postoji na Docker Hubu

Provjereni ispis: `deployment.apps/web image updated`

Stanje Podova odmah zatim — `kubectl get pods`:
```
NAME                   READY   STATUS         RESTARTS   AGE
web-555cd6d9f4-tzwmp   0/1     ErrImagePull   0          11s
web-555cd6d9f4-w8rgm   0/1     ErrImagePull   0          11s
web-555cd6d9f4-xztk9   0/1     ErrImagePull   0          11s
```
- SVA TRI Poda iz NOVOG RS-a (`555cd6d9f4`), nijedan stari ne radi → potvrda Recreate downtimea
- `0/1` READY, status `ErrImagePull`

### Dijagnoza — pročitati Events

```bash
kubectl describe pod -l app=web | grep -A5 Events
```
- `describe pod -l app=web` — detaljan opis Podova s labelom `app=web`
- `-l app=web` — selekcija po labeli (umjesto punog imena Poda)
- `| grep -A5 Events` — redak `Events:` + 5 ispod

Provjereni ključni reci:
```
Normal   Pulling   ...   Pulling image "nginx:9.99"
Warning  Failed    ...   Failed to pull image "nginx:9.99": ... manifest for nginx:9.99 not found: manifest unknown
```
- `manifest ... not found: manifest unknown` = registry kaže da ta verzija NE postoji → srž problema je tag
- `(x3 over 64s)` = pokušao 3 puta → kubelet usporava ponavljanje (backoff)

### ErrImagePull vs ImagePullBackOff

- **`ErrImagePull`** = trenutni neuspjeh povlačenja (upravo se dogodio); vidljiv na tek nastalim Podovima
- **`ImagePullBackOff`** = kubelet odustao od brzog ponavljanja i čeka sve dulje između pokušaja (eksponencijalni backoff)
- ISTI uzrok, kasnija faza; ostavi li se Pod malo dulje, `ErrImagePull` prelazi u `ImagePullBackOff`

### Popravak — rollback na zadnju dobru reviziju

```bash
kubectl rollout undo deployment/web
```
- vrati na prethodnu reviziju (ispravna slika nginx:1.25)
- prebaci replike s pokvarenog RS-a (`555cd6d9f4`) natrag na zadnji dobri (`d5496596c`)

Provjereni oporavak — `kubectl get pods`:
```
web-d5496596c-kg2l2   1/1   Running   0   7s
web-d5496596c-lrqpq   1/1   Running   0   7s
web-d5496596c-mrm2l   1/1   Running   0   7s
```
- imena opet `d5496596c` → vraćeni na STARI DOBRI RS (nginx:1.25)
- ali sufiksi NOVI + `AGE 7s` → Recreate ugasio pokvarene i podigao SVJEŽE Podove iz dobrog RS-a (isti template, nove instance)

### Gotče

- nepostojeća slika NE ruši Deployment — zaglavi novi RS; stara verzija opstaje SAMO uz RollingUpdate
- uz **Recreate** loša slika = POTPUNI downtime (svi stari ugašeni prije nego se utvrdi da novi ne rade) → oprez s Recreateom u kombinaciji s nepouzdanom slikom
- `ErrImagePull` (trenutno) i `ImagePullBackOff` (nakon backoffa) = isti uzrok, faze; uzrok se vadi iz `Events`
- popravak je `rollout undo` (ne `set image` natrag) — undo koristi povijest revizija (ovisi o `revisionHistoryLimit`, zad. 9)
- dijagnoza uvijek: `kubectl describe pod ... | grep -A5 Events` → poruka registryja (`not found` = tag, `unauthorized` = privatni registry bez secreta)

### Provjera

```bash
kubectl set image deployment/web nginx=nginx:9.99      # image updated
kubectl get pods                                       # ErrImagePull / ImagePullBackOff
kubectl describe pod -l app=web | grep -A5 Events       # manifest ... not found
kubectl rollout undo deployment/web                    # rolled back
kubectl get pods                                       # 3x Running
```

---

## Faza 3 — Podešavanje Poda

Cilj faze: prelazak s "kako se Deployment ažurira" na "kako se Pod iznutra slaže". Pet zadataka:
- **zad. 8** — resursi: requests i limits (CPU/memorija)
- **zad. 13** — sidecar (drugi container u istom Podu)
- **zad. 14** — nodeSelector → Pod ostaje Pending ako nijedan čvor ne odgovara
- **zad. 22** — dva containera dijele `emptyDir` (jedan piše, drugi čita)
- **zad. 28** — initContainer pripremi podatke prije glavnog containera

## Zadatak 8 — resursi: requests i limits (CPU/memorija)

**Cilj:** dodati containeru requests i limits za CPU i memoriju, pa potvrditi da se pojavljuju u speci živog Poda.

**Koncept — requests vs limits:**
- **requests** = koliko Pod GARANTIRANO dobije. Scheduler po tome bira čvor (Pod se rasporedi samo ako čvor ima barem `requests` slobodno). "Rezervacija."
- **limits** = gornja granica koju container NE smije prijeći. "Strop."
  - CPU preko limita → container se PRIGUŠI (throttling), NE ubija se
  - memorija preko limita → container se UBIJE (`OOMKilled`) i restarta

**Jedinice:**
- CPU: `1` = jedna cijela jezgra = `1000m` (milicores); `500m` = pola, `100m` = desetina
- memorija: `Mi` = mebibajt, `Gi` = gibibajt (npr. `64Mi`, `128Mi`)

### Akcija — dodaj `resources` blok (izmjena manifesta)

Container je imao `resources: {}` (prazno). Zamijeniti tim blokom, sve ugniježđeno pod containerom:
```yaml
      containers:
      - image: nginx:1.25
        name: nginx
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 250m
            memory: 128Mi
```
- `resources:` na istoj uvlaci kao `name`/`image`
- `requests:`/`limits:` → 2 razmaka dublje od `resources:`
- `cpu`/`memory` → još 2 razmaka dublje
- OPREZ: uz strategiju Recreate ova izmjena mijenja Pod template → novi RS → kratak downtime + nova imena Podova

Primjena:
```bash
kubectl apply -f ~/devops/lo4/web-deployment.yaml
```
Provjereni ispis: `deployment.apps/web configured`

### Provjera — vrijednosti moraju biti u speci ŽIVOG Poda (to zadatak traži)

Nova imena Podova (novi RS, jer se template promijenio):
```bash
kubectl get pods        # 3x Running, npr. web-56bb6547fb-...
```

Izvući Limits/Requests iz jednog Poda:
```bash
kubectl describe pod web-56bb6547fb-45hns | grep -A6 Limits
```
- `describe pod <ime>` — detaljan opis tog Poda
- `| grep -A6 Limits` — redak `Limits:` + 6 ispod (obuhvati i Requests blok)

Provjereni ispis:
```
Limits:
  cpu:     250m
  memory:  128Mi
Requests:
  cpu:     100m
  memory:  64Mi
```
- poklapanje sa zadanim → vrijednosti su prošle lanac: datoteka → Deployment → RS → živi Pod

### Gotče

- requests = garancija/rezervacija (scheduler ih gleda); limits = strop (enforce na čvoru)
- CPU iznad limita → throttling; memorija iznad limita → `OOMKilled` + restart (ključna razlika, čest ispit)
- promjena `resources` mijenja Pod template → novi RS + (uz Recreate) downtime
- dokaz se traži na ŽIVOM Podu (`describe pod`), ne na Deploymentu — vrijednost mora sjesti do dna lanca
- bez `requests` scheduler pretpostavlja 0 → Pod može sletjeti na prenapučen čvor; bez `limits` container može pojesti sve do OOM-a na razini čvora

### Provjera

```bash
kubectl apply -f ~/devops/lo4/web-deployment.yaml          # configured
kubectl get pods                                           # nova imena (novi RS)
kubectl describe pod <ime-poda> | grep -A6 Limits          # Limits + Requests sjeli na živi Pod
```

---

## Zadatak 13 — sidecar (drugi container u istom Podu)

**Cilj:** dodati drugi container u Pod template i razumjeti kako dva containera u istom Podu dijele mrežu i volumene.

**Koncept:** Pod može imati VIŠE containera koji žive zajedno i dijele:
- **mrežu** — isti Pod IP, isti `localhost`; container A se javlja containeru B preko `localhost:<port>` (kao na istom stroju, bez Servisa)
- **volumene** — ako su montirani u oba, dijele datoteke (vidi zad. 22)

**Sidecar** = pomoćni container uz glavni, u istom Podu. Tipično: skupljanje logova, proxy/ambassador, sinkronizacija datoteka, dohvat konfiguracije.
Containeri u Podu DIJELE SUDBINU: zajedno se raspoređuju, dižu i gase (isti čvor, isti životni ciklus).

### Akcija — dodaj drugi container u `containers:` listu

```yaml
      containers:
      - image: nginx:1.25
        name: nginx
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 250m
            memory: 128Mi
      - image: busybox:1.36
        name: sidecar
        command: ["sh", "-c", "while true; do echo sidecar živ; sleep 30; done"]
```
- novi `- image:` poravnat s gornjim `- image:` (isti `-`, ista razina) → DRUGI element iste liste
- `name`/`command` 2 razmaka iza `-` (poravnati s `image`)
- `command: [...]` daje busyboxu posao da ne izađe odmah (busybox bez naredbe završi → Pod bi vrtio restarte)
- OPREZ: uz Recreate izmjena template-a = novi RS + kratak downtime

Primjena:
```bash
kubectl apply -f ~/devops/lo4/web-deployment.yaml      # configured
```

### Provjera 1 — READY pokazuje 2/2

```bash
kubectl get pods
```
```
web-76994d859f-djnn7   2/2   Running   0   16s
```
- `READY 2/2`: lijevi broj = koliko containera SPREMNO, desni = koliko ih UKUPNO u Podu
- `2/2` = oba rade; da sidecar puca → `1/2` + restarti

### Provjera 2 — oba containera dijele isti Pod i isti IP

```bash
kubectl get pod <ime-poda> -o jsonpath='{.spec.containers[*].name}{"\n"}'
kubectl get pod <ime-poda> -o jsonpath='{.status.podIP}{"\n"}'
```
- `jsonpath='...'` — precizno vadi polje iz objekta; `[*]` = svi elementi liste; `{"\n"}` = novi red (uredan ispis)

Provjereni ispis:
```
nginx sidecar          # dva containera u JEDNOM Podu
10.244.0.46            # JEDAN IP za cijeli Pod → oba ga dijele
```
- posljedica: nginx i sidecar komuniciraju preko `localhost:<port>`, bez Servisa

### Gotče

- `READY 2/2` = broj spremnih / ukupnih containera u Podu (ne broj Podova)
- container bez dugotrajne naredbe (busybox) odmah izađe → Pod vrti restarte → uvijek dati `command`/dugotrajni proces
- jedan Pod IP za sve containere → međusobno preko `localhost`; van Poda treba Servis
- containeri Poda dijele sudbinu (isti čvor, zajedno gore/dolje) — sidecar nije zaseban "servis"
- promjena broja/sadržaja containera = promjena Pod template-a → novi RS (+ Recreate downtime)

### Provjera

```bash
kubectl apply -f ~/devops/lo4/web-deployment.yaml                              # configured
kubectl get pods                                                              # READY 2/2
kubectl get pod <ime> -o jsonpath='{.spec.containers[*].name}{"\n"}'          # nginx sidecar
kubectl get pod <ime> -o jsonpath='{.status.podIP}{"\n"}'                     # jedan Pod IP
```

---

## Zadatak 14 — nodeSelector → Pending (dijagnoza + popravak)

**Cilj:** napisati Deployment (`httpd:2.4`, 2 replike, imenovani port) s `nodeSelector` koji nijedan čvor ne zadovoljava, promotriti `Pending`, dijagnosticirati i popraviti.

**Koncept — nodeSelector:**
- uvjet u Pod templateu: "rasporedi me SAMO na čvor s ovom labelom". Scheduler radi PODUDARANJE STRINGOVA labela, NE provjerava hardver.
- nijedan čvor nema traženu labelu → scheduler nema kamo → Pod ostaje `Pending` (nije greška kontejnera, nego raspoređivanja)
- realna uporaba: gurati Podove na određene čvorove (GPU, SSD, prod…)

Zaseban manifest (ne dira `web`): `~/devops/lo4/httpd-pending.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-pending
spec:
  replicas: 2
  selector:
    matchLabels:
      app: httpd-pending
  template:
    metadata:
      labels:
        app: httpd-pending
    spec:
      nodeSelector:
        disktype: ssd
      containers:
      - image: httpd:2.4
        name: httpd
        ports:
        - containerPort: 80
          name: http
```
- `nodeSelector: disktype: ssd` — traži čvor s labelom `disktype=ssd` (minikube je NEMA → Pending)
- `ports: - containerPort: 80, name: http` — IMENOVANI port: container sluša 80, portu ime `http` (Servis se kasnije može referirati po imenu)

### GOTČA — `unknown field "spec.selector.template"` (kriva uvlaka)

Prvi apply pao s:
```
Error from server (BadRequest): ... unknown field "spec.selector.template"
```
- uzrok: `template:` bio uvučen 2 razmaka PREVIŠE → "upao" pod `selector:`. A `selector` i `template` su BRAĆA (oba izravno pod `spec`).
- popravak: poravnati `template:` na istu razinu kao `selector:` (2 razmaka, ravno pod `spec`)
- LO5 lekcija: `unknown field "X.Y"` = polje `Y` je na krivom mjestu (ugniježđeno pod `X` umjesto gdje treba) → poruka doslovno pokazuje gdje

Primjena (nakon popravka): `kubectl apply -f ~/devops/lo4/httpd-pending.yaml` → `created`

### Dijagnoza — Pod visi u Pending

```bash
kubectl get pods -l app=httpd-pending        # STATUS Pending, READY 0/1
kubectl describe pod <ime> | grep -A5 Events
```
Provjereni ispis:
```
Warning  FailedScheduling  default-scheduler  0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector.
preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```
- `FailedScheduling` → problem raspoređivanja, ne kontejner
- `0/1 nodes are available` → od 1 čvora, 0 prikladnih
- `didn't match ... node selector` → razlog: čvor nema `disktype=ssd`
- `preemption ... not helpful` → izbacivanje drugog Poda ne bi pomoglo (problem je labela, ne resursi)

### Popravak — labeliraj čvor da zadovolji selektor

Provjera da labele nema:
```bash
kubectl get nodes                                                   # čvor: minikube
kubectl get node minikube --show-labels | tr ',' '\n' | grep -i disk   # ništa → nema disktype
```
Dodaj labelu:
```bash
kubectl label node minikube disktype=ssd        # node/minikube labeled
kubectl get pods -l app=httpd-pending           # Pending → Running (1/1)
```
- `kubectl label node minikube disktype=ssd` — zalijepi labelu `ključ=vrijednost` na čvor
- Pod se digne ODMAH, bez ponovnog `apply` — scheduler stalno motri Pending Podove i pri sljedećem prolazu nađe valjan čvor

### Ključ — zašto label rješava problem

- `nodeSelector` traži PODUDARANJE STRINGA labele, ne stvarni SSD. Label = proizvoljna oznaka koju admin održava; K8s joj vjeruje na riječ.
- prije: nema labele → 0 podudaranja → Pending; poslije: string sjeo → podudaranje → Running
- promijenjen je UVJET podudaranja, ne disk. Da se `disktype=ssd` stavi na HDD čvor, Pod bi se svejedno digao (K8s ne provjerava hardver).
- labele su za organizaciju i usmjeravanje raspoređivanja; istinitost je na adminu

Luk: `nodeSelector disktype=ssd` → čvor bez labele → FailedScheduling/Pending → `label node` → uvjet zadovoljen → Running.

### Gotče

- `Pending` + `FailedScheduling` = problem raspoređivanja (selector/resursi/taint), NE kontejner — dijagnoza u `describe ... Events`
- `unknown field "X.Y"` pri apply = kriva uvlaka; polje `Y` je ugniježđeno pod `X` umjesto da je samostalno
- `nodeSelector` = podudaranje labela, ne provjera hardvera; admin jamči istinitost labele
- labeliranje čvora djeluje ODMAH na Pending Podove (scheduler ih stalno motri), bez ponovnog deploya
- `selector` i `template` su braća pod `spec`; `selector.matchLabels` se mora poklapati s `template.metadata.labels`

### Provjera

```bash
kubectl apply -f ~/devops/lo4/httpd-pending.yaml                    # created
kubectl get pods -l app=httpd-pending                              # Pending
kubectl describe pod <ime> | grep -A5 Events                       # FailedScheduling, didn't match node selector
kubectl label node minikube disktype=ssd                          # node labeled
kubectl get pods -l app=httpd-pending                              # Running 1/1
```

---

## Zadatak 22 — dva containera dijele `emptyDir` (writer/reader)

**Cilj:** Pod s dva containera koji dijele `emptyDir` volumen — jedan upiše datoteku, drugi je pročita. Dokazati dijeljenje.

**Koncept — `emptyDir`:**
- privremeni volumen, nastaje PRAZAN pri stvaranju Poda, živi koliko i Pod, briše se kad Pod nestane
- montiran u VIŠE containera istog Poda → svi vide ISTE datoteke → dijeljenje kroz disk (ne samo mrežu)
- tipično: jedan container generira (datoteku/log), drugi obrađuje/poslužuje

Mehanizam ima dva dijela koja se spajaju IMENOM:
1. **`volumes:`** (razina Poda) — definira volumen i daje mu ime
2. **`volumeMounts:`** (u svakom containeru) — "montiraj volumen tog imena na ovu putanju"
Ime u `volumeMounts` mora se TOČNO poklapati s imenom u `volumes`.

Goli Pod (PDF traži Pod): `~/devops/lo4/shared-emptydir.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-emptydir
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: writer
    image: busybox:1.36
    command: ["sh", "-c", "echo 'pozdrav iz writera' > /data/poruka.txt; sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
```
- `volumes: - name: shared-data, emptyDir: {}` — jedan volumen, `{}` = default postavke
- writer: `command` upiše tekst u `/data/poruka.txt`, pa `sleep 3600` (Pod ostaje živ); montira `shared-data` na `/data`
- reader: samo `sleep 3600`; montira ISTI `shared-data` na ISTU putanju `/data`
- ključ: oba montiraju volumen ISTOG imena → dijele sadržaj

Primjena: `kubectl apply -f ~/devops/lo4/shared-emptydir.yaml` → `pod/shared-emptydir created`

### GOTČA 1 — inline `command: [...]` s ugniježđenim navodnicima zna puknuti

Prvi pokušaj (s drugačijim formatiranjem) pao:
```
error converting YAML to JSON: yaml: line 11: did not find expected ',' or ']'
```
- uzrok: YAML se zna zbuniti oko navodnika unutar inline (flow) liste
- napomena: jednostruki navodnici UNUTAR dvostrukih (`"... 'tekst' ..."`) su OK i rade; problem nastaje kod složenijih kombinacija
- sigurnija alternativa kad zapne — blok-lista (svaki argument u svoj redak):
```yaml
    command:
    - sh
    - -c
    - echo 'pozdrav iz writera' > /data/poruka.txt; sleep 3600
```

### GOTČA 2 — `Not found: "<ime>"` = nepoklapanje imena volumena

Drugi pokušaj pao:
```
The Pod "shared-emptydir" is invalid: spec.containers[0].volumeMounts[0].name: Not found: "shared-adata"
```
- uzrok: `volumeMounts` tražio `shared-adata`, a `volumes` definira `shared-data` (tipfeler, slovo razlike)
- ime se mora poklapati IDENTIČNO na tri mjesta: `volumes` + writer `volumeMounts` + reader `volumeMounts`
- popravak: uskladiti ime (u geditu Ctrl+H → Replace All)

### Provjera — dokaz dijeljenja

```bash
kubectl get pod shared-emptydir        # READY 2/2, Running
kubectl exec shared-emptydir -c reader -- cat /data/poruka.txt
```
- `kubectl exec shared-emptydir` — izvrši naredbu UNUTAR Poda
- `-c reader` — u kojem containeru (Pod ima dva → mora se birati; bez ovoga uzme prvi)
- `--` — granica: sve iza je naredba u containeru, ne kubectl zastavice
- `cat /data/poruka.txt` — ispiši datoteku s dijeljene putanje

Provjereni ispis: `pozdrav iz writera`
- reader čita ono što je writer napisao → volumen je stvarno dijeljen
- da nije dijeljen → `No such file or directory`

### Veza s drugim zadacima
- zad. 13 (sidecar): containeri Poda dijele MREŽU (isti IP, `localhost`)
- zad. 22: containeri Poda dijele i DISK (kroz volumen)
- dvije osi dijeljenja unutar Poda

### Gotče

- `emptyDir` živi koliko i Pod → nije za trajne podatke (za trajnost ide PVC, kasnije)
- veza volumena je IME: `volumes.name` ↔ `volumeMounts.name`, identično; razlika u slovu = `Not found`
- `kubectl exec` na više-container Podu TRAŽI `-c <container>`; `--` odvaja naredbu od zastavica
- inline `command: [...]` radi, ali kod složenih navodnika prijeći na blok-listu
- montiranje istog volumena na istu putanju u oba containera = dijeljenje; različite putanje su OK, ali isti volumen

### Provjera

```bash
kubectl apply -f ~/devops/lo4/shared-emptydir.yaml                       # created
kubectl get pod shared-emptydir                                         # READY 2/2
kubectl exec shared-emptydir -c reader -- cat /data/poruka.txt          # pozdrav iz writera
```

---

## Zadatak 28 — initContainer (priprema podataka prije glavnog containera)

**Cilj:** initContainer pripremi podatke u dijeljeni volumen PRIJE nego ih glavni container pročita.

**Koncept — initContainer:**
- container koji se izvrši DO KRAJA (exit 0) PRIJE nego glavni container(i) krenu; tek po uspjehu Pod pokreće glavne
- više njih → idu REDOM, svaki mora završiti prije sljedećeg
- razlika od sidecara (zad. 13): sidecar traje cijelo vrijeme uz glavni; initContainer ODRADI i ugasi se
- tipično: pripremi podatke/konfiguraciju, dohvati datoteke, pričekaj uvjet, migriraj shemu — sve PRIJE glavnog
- spoj s emptyDir (zad. 22): init i glavni dijele isti volumen; init UPIŠE i ugasi se, glavni se digne i ZATEKNE podatke

Goli Pod: `~/devops/lo4/init-demo.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  initContainers:
  - name: init-setup
    image: busybox:1.36
    command: ["sh", "-c", "echo 'pripremio init container' > /data/init.txt"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  containers:
  - name: main
    image: busybox:1.36
    command: ["sh", "-c", "cat /data/init.txt; sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
```
- `initContainers:` — NOVO polje, ISTA razina kao `containers:` (oba pod `spec`, 2 razmaka)
- `init-setup`: upiše tekst u `/data/init.txt`, pa ZAVRŠI (nema `sleep` — init MORA završiti)
- `main`: `cat /data/init.txt` (pročita init-ov rezultat), pa `sleep 3600` (ostane živ)
- oba montiraju isti `shared-data` → init upiše PRIJE, main pročita POSLIJE

Primjena: `kubectl apply -f ~/devops/lo4/init-demo.yaml` → `pod/init-demo created`

### GOTČA — `unknown field "spec.initContainers[0].containers"` (kriva uvlaka)

Apply pao:
```
unknown field "spec.initContainers[0].containers"
```
- uzrok: `containers:` bio uvučen 2 razmaka PREVIŠE → "upao" pod zadnji initContainer
- popravak: `volumes:`, `initContainers:`, `containers:` su TRI BRATA — sva tri 2 razmaka, ravno pod `spec:`

### GOTČA — `apply` čita DISK, ne editor

Nakon popravka uvlake apply je i dalje javljao istu grešku JEDNOM — jer datoteka nije bila spremljena (`Ctrl+S`). Editor je pokazivao ispravno, ali na disku je bila stara verzija.
- `kubectl apply` uvijek čita stanje na DISKU → uvijek spremiti prije apply
- ako se ispravan ekran ne slaže s greškom → provjeriti disk: `cat ~/devops/lo4/init-demo.yaml`

### Provjera — redoslijed Init → glavni

```bash
kubectl get pod init-demo
```
- moguća prijelazna stanja: `Init:0/1` / `PodInitializing` (init još radi), pa `Running 1/1`
- `READY 1/1` broji SAMO glavne containere; init se NE broji (već je odradio i otišao)

Dokaz da je init išao PRIJE glavnog:
```bash
kubectl logs init-demo -c main
```
- `logs init-demo` — stdout Poda; `-c main` — kojeg containera

Provjereni ispis: `pripremio init container`
- ispisao ga je GLAVNI container (`cat` u startu) → datoteka je već BILA tu → init je morao završiti prije

Potvrda init containera:
```bash
kubectl describe pod init-demo | grep -A3 "Init Containers"     # sekcija Init Containers: init-setup
kubectl get pod init-demo -o jsonpath='{.status.initContainerStatuses[0].state}{"\n"}'
# stvarni ispis (skraćeno): {"terminated":{"exitCode":0,"reason":"Completed","startedAt":"...","finishedAt":"..."}}
# reason=Completed + exitCode=0 = init odradio i uspješno izašao; startedAt≈finishedAt = trajao djelić sekunde
```

### Gotče

- initContainer mora ZAVRŠITI (exit 0) prije glavnih; ne smije imati `sleep`/dugotrajni proces (inače Pod zapne na init fazi)
- `READY n/m` broji samo glavne containere; init se vidi u `describe`/`initContainerStatuses`, ne u READY
- više initContainera = sekvencijalno, redom
- `unknown field "X.Y"` = polje `Y` predaleko uvučeno pa upalo pod `X` (treći put isti obrazac — vidi zlatno pravilo)
- `apply` čita disk → spremiti prije; ako se greška ne slaže s ekranom, `cat` datoteku

### Provjera

```bash
kubectl apply -f ~/devops/lo4/init-demo.yaml                                              # created
kubectl get pod init-demo                                                                # Running 1/1 (init ne broji)
kubectl logs init-demo -c main                                                           # pripremio init container
kubectl describe pod init-demo | grep -A3 "Init Containers"                              # Init Containers: init-setup
```

---

## Zlatno pravilo YAML uvlaka (LO5) — `unknown field "X.Y"`

Tri puta u Fazi 3 pala je ista vrsta greške; vrijedi izdvojiti.

Simptom (pri `kubectl apply`):
```
unknown field "X.Y"
```
Značenje: polje `Y` je ugniježđeno POD `X`, a tamo ne pripada — gotovo uvijek jer je uvučeno 2 razmaka previše pa je "upalo" pod susjeda umjesto da mu je BRAT.

Viđeni primjeri:
- `unknown field "spec.selector.template"` → `template` upao pod `selector` (trebali su biti braća pod `spec`) — zad. 14
- `unknown field "spec.initContainers[0].containers"` → `containers` upao pod `initContainers` (braća pod `spec`) — zad. 28

Popravak: poravnati polje na ISTU razinu kao njegov pravi roditelj/brata; poruka doslovno pokazuje GDJE je polje završilo (`X`) i ŠTO je (`Y`).
Prevencija: kod ručnog uređivanja YAML-a paziti da su polja iste razine na istom stupcu; spremiti pa `apply` (čita disk).

---

## Faza 4 — Servisi

Cilj faze: pristup Podovima preko stabilne apstrakcije. Podovi su PROLAZNI (IP se mijenja pri svakom recreate-u), pa se ne može pouzdati u Pod IP. **Servis** = stabilan IP (ClusterIP) + DNS ime ispred grupe Podova; Podove nalazi preko SELEKTORA LABELA.

Zadaci faze:
- **zad. 12** — `kubectl expose` (Servis iz Deploymenta, izveden selektor)
- **zad. 38** — ClusterIP + DNS razrješavanje iz drugog Poda
- **zad. 39** — NodePort + `minikube service`
- **zad. 40** — LoadBalancer (`<pending>` na minikube, `minikube tunnel`)
- **zad. 41** — headless Service (clusterIP: None) za StatefulSet
- **zad. 42** — Endpoints/EndpointSlice (kako se pune iz selektora)
- **zad. 43** — cross-namespace pristup preko FQDN
- **zad. 44** — `port-forward` na Servis vs na Pod
- **zad. 45** — razlika port / targetPort / nodePort
- **zad. 46** — provjera ClusterIP-a iz throwaway debug Poda
- **zad. 47** — jedan Servis load-balansira preko dva Deploymenta (zajednička labela)
- **zad. 49** — NodePort + provjera da radi

## Zadatak 12 — `kubectl expose` (Servis iz Deploymenta, izveden selektor)

**Cilj:** napraviti Servis iz `web` Deploymenta jednom naredbom; objasniti koji objekt je nastao i kako mu je izveden selektor.

**Koncept:**
- Podovi prolazni → mijenjaju IP pri recreate-u → ne oslanjati se na Pod IP
- **Servis** = stabilna apstrakcija: stalan IP (ClusterIP) + DNS ime, uvijek pokazuje na TRENUTNE Podove
- Servis nalazi Podove preko SELEKTORA LABELA (isti mehanizam koji drži Deployment→Pod)

### Akcija — expose (imperativno)

```bash
kubectl expose deployment web --port=80 --target-port=80 --name=web-svc
```
- `expose deployment web` — napravi Servis za Deployment `web` (uzmi njegov selektor)
- `--port=80` — port na kojem SERVIS sluša (ulaz prema klijentima)
- `--target-port=80` — port na PODU/containeru kamo Servis prosljeđuje (nginx sluša 80)
- `--name=web-svc` — ime Servisa (bez ovoga = `web`, kao Deployment)

Provjereni ispis: `service/web-svc exposed`

### Provjera 1 — koji objekt je nastao

```bash
kubectl get svc web-svc
```
Provjereni ispis:
```
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
web-svc   ClusterIP   10.98.162.190   <none>        80/TCP    2m6s
```
- `TYPE: ClusterIP` — DEFAULT (nije traženo drugo); IP dostupan SAMO unutar klastera (drugi Podovi da, izvana ne)
- `CLUSTER-IP: 10.98.162.190` — STALNI IP Servisa; NE mijenja se ni kad se svi Podovi iza recreate-aju → "stabilna vrata"
- `EXTERNAL-IP: <none>` — nema vanjskog IP-a (očekivano za ClusterIP)
- `PORT(S): 80/TCP` — Servis sluša na 80 (iz `--port`)

Ideja: klijent uvijek gađa `10.98.162.190:80`, Servis interno proslijedi na neki trenutni Pod. Klijent ne mora znati Pod IP-eve ni broj.

### Provjera 2 — kako je selektor IZVEDEN (srž zadatka)

```bash
kubectl get svc web-svc -o jsonpath='{.spec.selector}{"\n"}'
```
Provjereni ispis: `{"app":"web"}`

Odakle baš to:
1. `web` Deployment ima `spec.selector.matchLabels: app=web` (Faza 1)
2. `kubectl expose deployment web` PROČITA taj selektor s Deploymenta i KOPIRA ga u novi Servis
3. zato Servis dobije `selector: app=web` → automatski cilja TOČNO one Podove koje Deployment vodi (nije ručno upisano, IZVEDENO je iz Deploymenta)

Lanac:
```
web Deployment  spec.selector.matchLabels: app=web
   │ expose PROČITA selektor
   ▼
web-svc Service  spec.selector: app=web
   │ Servis po njemu traži Podove
   ▼
Podovi s labelom app=web (web-...-djnn7, mq22g, vr9gm)
```

Zašto je pametno: Servis i Deployment gledaju isti skup Podova preko ISTE labele. Recreate Poda (novi IP/ime) → i dalje ima `app=web` → Servis ga automatski uključi. Veza je LABELA, ne IP/ime.

### Gotče

- bez `--type` expose pravi `ClusterIP` (samo unutar klastera); za vanjski pristup treba NodePort/LoadBalancer (idući zadaci)
- bez `--name` Servis preuzme ime Deploymenta
- selektor Servisa nije ručni — `expose` ga nasljeđuje od Deploymenta (ovo PDF traži objasniti)
- `CLUSTER-IP` je stabilan; Pod IP-evi nisu → zato se gađa Servis, ne Pod
- `--target-port` mora odgovarati portu na kojem container STVARNO sluša (inače Servis prosljeđuje u prazno)

### Provjera

```bash
kubectl expose deployment web --port=80 --target-port=80 --name=web-svc      # exposed
kubectl get svc web-svc                                                     # ClusterIP + stabilan IP
kubectl get svc web-svc -o jsonpath='{.spec.selector}{"\n"}'                # {"app":"web"} (izveden iz Deploymenta)
```

---

## Zadatak 38 — ClusterIP + DNS razrješavanje iz drugog Poda

**Cilj:** pristupiti `web-svc` Servisu preko DNS IMENA iz drugog Poda i razumjeti format `<svc>.<ns>.svc.cluster.local`.

**Koncept — DNS u Kubernetesu:**
- svaki Servis automatski dobije DNS ime unutar klastera (interni DNS = CoreDNS)
- pun oblik (FQDN): **`<ime-servisa>.<namespace>.svc.cluster.local`**
  - `web-svc` = ime Servisa, `default` = namespace, `svc.cluster.local` = fiksni sufiks
- iz Poda u ISTOM namespaceu dovoljno kratko ime (`web-svc`); pun FQDN tek za druge namespaceove (zad. 43)
- poanta: ne treba znati ClusterIP — gađa se IME, DNS ga prevede u trenutni IP Servisa
- DNS imena Servisa postoje SAMO unutar klastera → za test treba Pod IZNUTRA

### Mentalni model — dnstest vs web-svc
- `dnstest` = Pod-alat (busybox, `sleep`); ODAKLE pitamo; NIJE iza Servisa (nema labelu `app=web`), samo je u istoj mreži klastera → može pitati DNS o bilo kojem Servisu
- `web-svc` = ŠTO pitamo; Servis koji cilja `web` Podove
- dvije odvojene stvari: "u mreži klastera" (→ može razriješiti DNS) vs "ima ciljanu labelu" (→ iza Servisa, prima promet). `dnstest` ima prvo, ne drugo.

### Akcija — privremeni Pod-alat za testiranje

```bash
kubectl run dnstest --image=busybox:1.36 --restart=Never -- sleep 3600
kubectl get pod dnstest        # Running 1/1
```
- `kubectl run dnstest` — napravi Pod imena `dnstest`
- `--image=busybox:1.36` — busybox (ima `nslookup`, `wget`)
- `--restart=Never` — bez ovoga `run` pravi Deployment; `Never` = GOLI Pod (jednokratan)
- `-- sleep 3600` — naredba: spavaj sat (Pod ostaje živ za testiranje)

### Provjera 1 — DNS upit kratkim imenom

```bash
kubectl exec dnstest -- nslookup web-svc
```
- `exec dnstest` — GDJE pitamo (uđi u Pod); `--` granica; `nslookup web-svc` — ŠTO pitamo (razriješi ime)

Provjereni (bitni) dio ispisa:
```
Server:   10.96.0.10                          # interni DNS (CoreDNS)
Name:     web-svc.default.svc.cluster.local
Address:  10.98.162.190                       # TOČNO ClusterIP iz zad. 12
```

### GOTČA — busyboxov nslookup je bučan (NXDOMAIN + exit 1, a radi!)

Uz uspješni redak gore, busybox ispiše i hrpu:
```
** server can't find web-svc.svc.cluster.local: NXDOMAIN
** server can't find web-svc.dns.podman: NXDOMAIN
... command terminated with exit code 1
```
- uzrok: kratko ime se isproba sa SVAKIM sufiksom iz search-liste (`/etc/resolv.conf`); jedan pogodi (`...default.svc.cluster.local`), ostali ne postoje → `NXDOMAIN`
- busybox ispisuje SVAKI pokušaj (i neuspjele) i vraća `exit 1` iako je glavni uspio
- NIJE greška: gleda se je li ispravni `*.svc.cluster.local` redak razriješen u ClusterIP
- čišća provjera: dati pun FQDN odmah (DNS ne probava sufikse)

### Provjera 2 — pun FQDN (čist ispis)

```bash
kubectl exec dnstest -- nslookup web-svc.default.svc.cluster.local
```
Provjereni ispis (bez NXDOMAIN šuma):
```
Server:   10.96.0.10
Name:     web-svc.default.svc.cluster.local
Address:  10.98.162.190
```

FQDN po dijelovima:
```
web-svc . default . svc . cluster.local
  ime     namespace  "servis"  fiksni sufiks
```

### Gotče

- Servis se gađa po IMENU; DNS vrati trenutni ClusterIP → ne treba pamtiti IP
- isti namespace → kratko ime dovoljno; drugi namespace → pun FQDN (zad. 43)
- busyboxov `nslookup` bučan (NXDOMAIN za krive sufikse + exit 1); bitan je ispravni FQDN redak
- `dnstest` može razriješiti Servis jer je u mreži klastera, NE jer je iza Servisa (nije — nema labelu)
- `--restart=Never` u `kubectl run` = goli Pod; bez toga nastaje Deployment

### Provjera

```bash
kubectl run dnstest --image=busybox:1.36 --restart=Never -- sleep 3600     # created
kubectl get pod dnstest                                                    # Running
kubectl exec dnstest -- nslookup web-svc.default.svc.cluster.local         # Address: <ClusterIP>
```

---

## Zadatak 39 — NodePort Servis (pristup izvana)

**Cilj:** napraviti NodePort Servis i pristupiti mu izvana — preko `minikube service --url` i preko `$(minikube ip):<nodePort>`.

**Koncept — NodePort:**
- otvara ISTI port na SVAKOM čvoru klastera; promet na `<IP-čvora>:<nodePort>` → Servis → Podovi
- za razliku od ClusterIP (samo iznutra), NodePort je dostupan IZVANA (preko IP-a čvora)
- nodePort se dodjeljuje iz raspona **30000–32767** (osim ako se izričito zada)
- gradi se NA ClusterIP-u: NodePort Servis i dalje IMA ClusterIP, plus otvara vanjski port

Tri porta (puna priča zad. 45): `port` (Servis, iznutra) · `targetPort` (Pod) · `nodePort` (čvor, vani, 30000–32767)

### Akcija — NodePort (zaseban Servis, ne dira web-svc)

```bash
kubectl expose deployment web --type=NodePort --port=80 --target-port=80 --name=web-np
```
- `--type=NodePort` — NOVO: tip NodePort (umjesto defaultnog ClusterIP) → otvara vanjski port na čvoru
- ostalo isto kao zad. 12 (naslijedi selektor `app=web`)

Provjereni ispis: `service/web-np exposed`

### Što je nastalo

```bash
kubectl get svc web-np
```
```
NAME     TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
web-np   NodePort   10.103.186.125   <none>        80:31629/TCP   2m27s
```
- `TYPE: NodePort` ✓
- `CLUSTER-IP: 10.103.186.125` — I DALJE ima ClusterIP → NodePort gradi NA ClusterIP-u (ima oba)
- `PORT(S): 80:31629/TCP` — `80` = port Servisa (iznutra), `31629` = nodePort (na čvoru, vani, iz 30000–32767)
- `EXTERNAL-IP: <none>` — normalno za NodePort; pristup preko IP-a ČVORA

### Pristup izvana — način 1: minikube service

```bash
minikube service web-np --url
```
- minikube nađe NodePort Servis i složi URL; `--url` = samo ispiši (ne otvaraj browser)

Provjereni ispis: `http://192.168.49.2:31629`  (`192.168.49.2` = IP čvora = `minikube ip`, `31629` = nodePort)

Dokaz da radi:
```bash
curl http://192.168.49.2:31629
```
→ nginxov HTML, `<title>Welcome to nginx!</title>` = lanac radi

### Pristup izvana — način 2: $(minikube ip):nodePort

```bash
curl $(minikube ip):31629
```
- `$(minikube ip)` — shell izvrši `minikube ip`, vrati `192.168.49.2` i ubaci na to mjesto (ne treba pamtiti IP)
→ isti nginxov HTML

Lanac:
```
curl izvana → 192.168.49.2:31629 → web-np Servis → web Pod → nginx → "Welcome to nginx!"
              (IP čvora)(nodePort)   (preko app=web)
```

Kontrast: ClusterIP (zad. 38) dohvatljiv SAMO iznutra (iz `dnstest`); NodePort dohvatljiv IZVANA (s VM-a).

### Gotče

- NodePort ima i ClusterIP (gradi na njemu); `EXTERNAL-IP <none>` je normalno → pristup preko IP-a čvora
- nodePort iz raspona 30000–32767, automatski dodijeljen (ovdje 31629); može se fiksirati poljem `nodePort:` u manifestu
- `$(minikube ip)` ubaci IP čvora bez prepisivanja; `minikube service <svc> --url` složi cijeli URL
- na pravom (multi-node) klasteru port je otvoren na SVAKOM čvoru, ne samo jednom

### Provjera

```bash
kubectl expose deployment web --type=NodePort --port=80 --target-port=80 --name=web-np   # exposed
kubectl get svc web-np                                                                   # PORT(S) 80:<nodePort>
minikube service web-np --url                                                            # http://<ip>:<nodePort>
curl $(minikube ip):<nodePort>                                                           # Welcome to nginx!
```

---

## Zadatak 40 — LoadBalancer Servis (+ minikube tunnel)

**Cilj:** napraviti LoadBalancer Servis, objasniti zašto `EXTERNAL-IP` ostane `<pending>` na minikubeu, i (pokušati) pristup preko `minikube tunnel`.

**Koncept — LoadBalancer:**
- Servis koji traži VANJSKI load balancer od cloud providera (AWS/GCP/Azure) → pravi javni IP
- gradi se slojevito: LoadBalancer ima ClusterIP **i** NodePort **i** traži vanjski IP
  - ClusterIP (sloj 1, iznutra) → NodePort (sloj 2, vani preko čvora) → LoadBalancer (sloj 3, javni IP)
- "load balancer" ima dva značenja, ne miješati:
  - raspoređivanje prometa po Podovima → radi SVAKI Servis (i ClusterIP)
  - tip `LoadBalancer` → pribavljanje VANJSKOG IP-a od clouda (to je novost)
- u cloudu: cloud-controller-manager vidi zahtjev → cloud napravi pravi LB (PLAĆENA usluga) → dodijeli IP
- na minikubeu: NEMA clouda → nitko ne ispuni zahtjev → `EXTERNAL-IP` ostaje `<pending>` (NIJE greška)
- trošak: pravi cloud LB plaća vlasnik cloud računa (tvrtka/pojedinac); minikube je besplatan (ništa se stvarno ne stvara)

### Akcija — LoadBalancer Servis (zaseban)

```bash
kubectl expose deployment web --type=LoadBalancer --port=80 --target-port=80 --name=web-lb
kubectl get svc web-lb
```
Provjereni ispis:
```
NAME     TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
web-lb   LoadBalancer   10.106.2.148   <pending>     80:31448/TCP   11s
```
- `CLUSTER-IP: 10.106.2.148` → ima ClusterIP (sloj 1)
- `PORT(S): 80:31448/TCP` → `31448` = nodePort → ima NodePort (sloj 2)
- `EXTERNAL-IP: <pending>` → traži vanjski IP (sloj 3), minikube ga ne može isporučiti → vječno čeka
- zaključak: LoadBalancer = ClusterIP + NodePort + zahtjev za vanjskim IP-em, sve u jednom

### GOTČA — tip je case-sensitive

Tipfeler `--type=LOadBalancer` pao:
```
The Service "web-lb" is invalid: spec.type: Unsupported value: "LOadBalancer":
supported values: "ClusterIP", "ExternalName", "LoadBalancer", "NodePort"
```
- vrijednosti tipa su case-sensitive; poruka SAMA izlista dopuštene → prepisati točno `LoadBalancer`

### GOTČA — minikube tunnel traži root rute, na rootless VM-u NE prolazi

`minikube tunnel` (u zasebnom terminalu, jer ostaje pokrenut) treba otvoriti mrežne rute kao root → traži sudo. Na ovom VM-u `student` nema to pravo:
```
router: error adding Route: Sorry, user student is not allowed to execute
'/sbin/ip route add 10.96.0.0/12 via 192.168.49.2' as root on vm69-61
```
- tunnel iznova pokušava → ponavlja upit za lozinku; zaustaviti s `Ctrl+C` u tunnel terminalu
- `loadbalancer emulator: no errors` + `route: 10.96.0.0/12 -> 192.168.49.2` = emulator krenuo, ali sustav odbio dodavanje rute
- na rootless VM-u tunnel NIJE dostupan → za stvarni vanjski pristup koristiti NodePort (zad. 39)

### Odgovor za ispit
"Na minikubeu `EXTERNAL-IP` LoadBalancer Servisa ostaje `<pending>` jer nema cloud providera koji bi isporučio vanjski IP. Normalno se rješava `minikube tunnel`-om (lokalno emulira cloud LB), koji na ovom rootless VM-u ne prolazi jer `student` nema root ovlasti za dodavanje mrežnih ruta."

### Gotče

- `<pending>` na minikubeu je OČEKIVANO za LoadBalancer (nema cloud LB), ne greška
- LoadBalancer sadrži i ClusterIP i nodePort (slojevit); na pravom cloudu dobije i javni IP
- pravi cloud LB je plaćena usluga → u praksi se ne radi LB po servisu, nego Ingress ispred više servisa
- `--type` vrijednosti case-sensitive (`LoadBalancer`, ne `LOadBalancer`)
- `minikube tunnel` ostaje pokrenut (zauzme terminal) i treba root → na rootless VM-u nedostupan

### Provjera

```bash
kubectl expose deployment web --type=LoadBalancer --port=80 --target-port=80 --name=web-lb   # exposed
kubectl get svc web-lb                                                                       # EXTERNAL-IP <pending>
# minikube tunnel (zaseban terminal) → na ovom VM-u pada: student nije ovlašten za /sbin/ip route add
```

---

## Zadatak 45 — `port` / `targetPort` / `nodePort` (razlika)

**Cilj:** razlikovati tri porta Servisa — `port`, `targetPort`, `nodePort` — i pročitati ih s živog NodePort Servisa.

**Koncept — tri porta, tri "gdje":**

Servis primi promet na jednim vratima i proslijedi ga na druga.

- **`port`** = vrata SERVISA (na njegovom ClusterIP-u); gađa ga netko IZNUTRA klastera (npr. drugi Pod kuca `web-svc:80`)
- **`targetPort`** = vrata PODA/containera; gdje aplikacija STVARNO sluša (nginx → 80); Servis prosljeđuje `port` → `targetPort`
- **`nodePort`** = vrata ČVORA, otvorena prema VANI (raspon 30000–32767); samo NodePort/LoadBalancer Servisi ga imaju; tu ulazi promet IZVANA klastera

Lanac (NodePort Servis):
```
<IP čvora>:nodePort   →   ClusterIP:port   →   targetPort (na Podu)
   (izvana)               (Servis)             (aplikacija)
```

### Akcija — pročitati tri porta s `web-np`

```bash
kubectl get svc               # pregled svih servisa (PORT(S) = port:nodePort)
kubectl describe svc web-np   # NodePort servis: Port / TargetPort / NodePort odvojeno
```

Provjereni ispis `kubectl get svc` (svi servisi preživjeli buđenje klastera):
```
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1        <none>        443/TCP        25h
web-lb       LoadBalancer   10.106.2.148     <pending>     80:31448/TCP   57m
web-np       NodePort       10.103.186.125   <none>        80:31629/TCP   133m
web-svc      ClusterIP      10.98.162.190    <none>        80/TCP         3h31m
```
- stupac `PORT(S)`: kratica `80:31629` = `port:nodePort`
- ClusterIP `web-svc` ima samo `80` (nema nodePort jer nije izložen vani); NodePort/LoadBalancer imaju oba

Provjereni ispis `kubectl describe svc web-np` (izvadak):
```
Selector:                 app=web
Type:                     NodePort
IP:                       10.103.186.125
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31629/TCP
Endpoints:                10.244.0.65:80,10.244.0.66:80,10.244.0.67:80
```
- `Port: 80` → vrata Servisa na ClusterIP-u `10.103.186.125` (sloj iznutra)
- `TargetPort: 80` → vrata na Podu gdje nginx sluša; Servis šalje `port` → `targetPort`
- `NodePort: 31629` → vrata na čvoru, prema vani
- `Endpoints: ...0.65:80, .66:80, .67:80` → tri Pod-IP-a (3 replike `web`), svaki na `targetPort` 80 → dokaz da je Servis preko `Selector: app=web` našao 3 zdrava Poda
- ovdje je `port = targetPort = 80` (Servis stvoren s `--port=80 --target-port=80`); to su SVEJEDNO dva različita pojma — poklapaju se samo slučajno

Živi lanac za `web-np`:
```
<IP čvora>:31629  →  10.103.186.125:80  →  targetPort 80 na svakom od:
   (nodePort)         (ClusterIP:port)        10.244.0.65:80
                                              10.244.0.66:80
                                              10.244.0.67:80
                                              (Endpoints = 3 Poda)
```

### GOTČA — `<unset>` znači bez IMENA, ne bez vrijednosti

U `describe` ispisu stoji `Port: <unset> 80/TCP` i `NodePort: <unset> 31629/TCP`:
- `<unset>` se odnosi na NAZIV porta (portovi se smiju imenovati), NE na vrijednost
- vrijednost JEST postavljena (80, 31629); samo nema dodijeljeno ime → zato „unset"
- imenovani portovi imaju smisla tek kad Servis ima više portova pa ih treba razlikovati

### GOTČA — krivi `targetPort` (veže se na LO5)

Ako `targetPort` ne pogodi port na kojem container STVARNO sluša:
- promet stigne do Servisa, ali ga Pod odbije → „nothing answers" / connection refused
- `Endpoints` mogu izgledati popunjeno, a veza svejedno pada jer je ODREDIŠNI port kriv
- krivi `targetPort` = jedna od stavki u dijagnostici „zašto ne dolazi do Servisa" (vidi zad. 48 — playbook)

### Gotče

- `port` = vrata Servisa (iznutra) · `targetPort` = vrata aplikacije na Podu · `nodePort` = vrata čvora (vani)
- u `get svc` stupac `PORT(S)`: `port:nodePort` (npr. `80:31629`); ClusterIP nema nodePort dio
- `<unset>` u `describe` = port nema IME; vrijednost je svejedno postavljena
- nodePort iz raspona 30000–32767; ClusterIP Servis ga uopće nema
- `port` i `targetPort` smiju biti različiti; poklapanje (80/80) je samo zgodan slučaj

### Provjera

```bash
kubectl get svc web-np                  # PORT(S) = port:nodePort (80:31629)
kubectl describe svc web-np             # Port / TargetPort / NodePort odvojeno + Endpoints
kubectl get svc web-np -o jsonpath='{.spec.ports[0].port}/{.spec.ports[0].targetPort}/{.spec.ports[0].nodePort}{"\n"}'   # 80/80/31629
```

---

## Zadatak 42 — Endpoints / EndpointSlice (odakle Servisu popis Podova)

**Cilj:** pokazati da Servis ne gađa Podove izravno, nego preko Endpoints popisa koji Kubernetes puni automatski iz selektora; dokazati da su adrese u Endpoints točno IP-evi `web` Podova.

**Koncept — selektor → Endpoints → Podovi:**
```
Servis ima Selector: app=web
        ↓  (k8s neprestano traži zdrave Podove s tom labelom)
nađe sve ZDRAVE Podove app=web
        ↓
upiše njihove PodIP:targetPort u Endpoints objekt
        ↓
promet na Servis dijeli se po tom popisu
```
- Servis i Podovi NISU izravno spojeni; Endpoints je „živi imenik" koji k8s sam osvježava
- Pod se digne s `app=web` → uđe u popis; Pod padne/promijeni IP → popis se prepiše
- zato Podovi smiju biti prolazni a Servis (ClusterIP) stabilan

**EndpointSlice** = novija izvedba istog:
- stari `Endpoints` = jedan veliki popis svih adresa; kod stotina Podova svaka promjena prepisuje cijeli popis (neefikasno)
- `EndpointSlice` razbija popis na kriške (~100 adresa) → manje prepisivanja
- `Endpoints` ostaje zbog kompatibilnosti; `EndpointSlice` je ono što k8s interno koristi
- za ispit: EndpointSlice zamjenjuje Endpoints radi skaliranja

### Akcija — usporediti Endpoints s Pod-IP-evima

```bash
kubectl get endpoints web-svc                                        # popis PodIP:port iza servisa
kubectl get endpointslice -l kubernetes.io/service-name=web-svc      # ista stvar, novi oblik (kriške)
kubectl get pods -l app=web -o wide                                  # IP-evi samih Podova (za uparivanje)
```
- `-l kubernetes.io/service-name=web-svc` = filtrira kriške koje pripadaju servisu `web-svc` (ta labela veže slice na servis)
- `-o wide` = doda stupac `IP` Podu

Provjereni ispis:
```
# kubectl get endpoints web-svc
NAME      ENDPOINTS                                      AGE
web-svc   10.244.0.69:80,10.244.0.70:80,10.244.0.72:80   3h53m

# kubectl get endpointslice -l kubernetes.io/service-name=web-svc
NAME            ADDRESSTYPE   PORTS   ENDPOINTS                             AGE
web-svc-g588v   IPv4          80      10.244.0.70,10.244.0.72,10.244.0.69   3h53m

# kubectl get pods -l app=web -o wide
NAME                    READY   STATUS    RESTARTS        AGE   IP            NODE
web-76994d859f-djnn7    2/2     Running   10 (102s ago)   14h   10.244.0.72   minikube
web-76994d859f-mq22g    2/2     Running   10 (102s ago)   14h   10.244.0.69   minikube
web-76994d859f-vr9gm    2/2     Running   10 (102s ago)   14h   10.244.0.70   minikube
```
Uparivanje (sve tri liste = isti skup adresa):
```
Pod web-76994d859f-mq22g  →  10.244.0.69  ┐
Pod web-76994d859f-vr9gm  →  10.244.0.70  ├─ = Endpoints {.69, .70, .72} = EndpointSlice
Pod web-76994d859f-djnn7  →  10.244.0.72  ┘
```
- Endpoints sadrži TOČNO IP-eve `web` Podova → dokazano
- razlika u zapisu: `Endpoints` lijepi `:80` uz svaku adresu; `EndpointSlice` drži port u zasebnom stupcu `PORTS`, adrese bez porta
- redoslijed adresa nije bitan (lista, ne niz)
- ime kriške `web-svc-g588v` automatski je generirano (servis + nasumični sufiks)

### GOTČA — tipfeler u labeli = „No resources found" (prazno, NE greška)

Prvi pokušaj s krivom labelom `kubertnetes.io/...` (višak `t`):
```
kubectl get endpointslice -l kubertnetes.io/service-name=web-svc
No resources found in default namespace.
```
- label selektor koji se ne poklapa NI s čim → vrati PRAZNO, bez ikakve greške
- opasna zamka u dijagnostici: izgleda kao da resurs ne postoji, a zapravo je selektor krivo napisan
- pravilo: „No resources found" → prvo provjeriti je li selektor/labela točno napisana

### GOTČA — Pod-IP-evi se mijenjaju (zato Servis postoji)

Isti Podovi su u zad. 45 imali Endpoints `.65, .66, .67`; sada su `.69, .70, .72`:
- `RESTARTS 10 (102s ago)` = Podovi su se restartali (klaster je drijemao pod RAM-om pa ih je kubelet podigao iznova)
- ime Poda ISTO (`web-76994d859f-*`), AGE i dalje `14h` → Pod objekt preživio; ali sandbox je nanovo složen pa je CNI dodijelio NOVE IP-eve
- Endpoints se SAM prepisao na nove adrese → ClusterIP servisa (`10.98.162.190`) ostao identičan
- pouka: Pod-IP nije stabilan (promijenio se i od običnog drijemeža); zato se nikad ne gađa Pod-IP izravno, nego stabilan Servis

### Gotče

- Servis → Podovi ide PREKO Endpoints; k8s puni Endpoints iz selektora automatski
- `EndpointSlice` = skalabilna zamjena za `Endpoints` (kriške ~100 adresa); oba postoje paralelno
- label selektor bez pogotka → „No resources found" (prazno, ne greška) → provjeriti labelu
- Pod-IP je prolazan (mijenja se na restart/recreate); ClusterIP Servisa je stabilan — zato Servis
- `EndpointSlice` adrese bez porta (port u stupcu `PORTS`); `Endpoints` lijepi `:port` uz adresu

### Provjera

```bash
kubectl get endpoints web-svc                                     # PodIP:80 × broj replika
kubectl get endpointslice -l kubernetes.io/service-name=web-svc   # iste adrese, oblik kriški
kubectl get pods -l app=web -o wide                               # IP stupac = iste adrese
```

---

## Zadatak 46 — debug Pod: kucnuti ClusterIP iznutra (dokaz cijelog lanca)

**Cilj:** iz privremenog Poda, IZNUTRA klastera, dohvatiti `web-svc` po imenu i dokazati da promet prođe cijeli lanac DNS → ClusterIP → kube-proxy → jedan od 3 nginx Poda. Dodatno: sirovi TCP test da je port otvoren na transportnom sloju.

**Koncept — zašto debug Pod:**
- ClusterIP je vidljiv SAMO unutar klastera (nema vanjski IP) → s VM-a se ne može direktno kucnuti
- rješenje: dignuti sićušan jednokratni Pod UNUTAR klastera koji odradi jedan zahtjev i nestane
- busybox = minijaturna slika s osnovnim mrežnim alatima (`wget`, `nc`, `sh`) → idealna za ad-hoc dijagnostiku
- isti obrazac (`kubectl run tmp --rm -it ... --restart=Never -- <naredba>`) je univerzalni alat: ubacivanje u klaster na sekundu, provjera nečega, izlaz

**Lanac koji se dokazuje:**
```
web-svc            (ime servisa)
   ↓  CoreDNS razriješi ime
10.98.162.190      (ClusterIP, stabilan)
   ↓  kube-proxy (iptables/ipvs pravila biraju jedan Pod)
jedan od Endpoints {.69:80, .70:80, .72:80}
   ↓
nginx u tom Podu  →  vrati „Welcome to nginx!"
```

### Naredba A — HTTP dohvat (cijeli lanac, L7)

```bash
kubectl run tmp --rm -it --image=busybox --restart=Never -- wget -qO- http://web-svc
```

- `kubectl run tmp` — stvara i pokreće Pod imena `tmp`
- `--rm` — obriši Pod automatski čim naredba završi (bez smeća iza sebe)
- `-it` — `-i` drži stdin otvoren + `-t` dodijeli TTY → ispis se vidi uživo
- `--image=busybox` — slika busybox (sitni alati: `wget`, `nc`, `sh`)
- `--restart=Never` — postavi `restartPolicy: Never` → goli jednokratni Pod (ne pokušava se ponovo dizati)
- `--` — granica: sve IZA je naredba koja se vrti UNUTAR Poda, ne više zastavica za `kubectl`
- `wget` — dohvat preko HTTP-a (naredba unutar busyboxa)
- `-q` — quiet, bez ispisa o napretku preuzimanja
- `-O-` — VELIKO `O` = „ispiši sadržaj u odredište"; `-` kao odredište = stdout → sadržaj ide na ekran
- `http://web-svc` — kratko ime radi jer je `tmp` Pod u istom namespaceu (`default`) → razriješi se u `web-svc.default.svc.cluster.local`

Provjereni ispis (skraćeno):
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
...
</html>
pod "tmp" deleted
```
- vraćen pun nginx HTML → DNS razriješio ime → ClusterIP → kube-proxy → jedan nginx Pod → odgovor; cijeli lanac prošao
- `pod "tmp" deleted` na kraju = `--rm` odradio čišćenje Poda

### Naredba B — sirovi TCP test (sloj ispod HTTP-a, L4)

```bash
kubectl run tmp --rm -it --image=busybox --restart=Never -- sh -c 'nc -zv web-svc 80'
```

- isti omotač kao Naredba A; samo je naredba unutra druga
- `sh -c '...'` — pokreće shell (`sh`) koji s `-c` izvrši naredbu zadanu u navodnicima. Za ovako jednostavnu naredbu nije nužan (može i `-- nc -zv web-svc 80` direktno), ali je dobra navika kad unutarnja naredba treba shell stvari (cijevi, varijable)
- `nc` — netcat, alat za otvaranje TCP/UDP veza
- `-z` — zero-I/O / scan mode: samo provjerava je li port otvoren (spaja se i javlja), bez slanja/primanja podataka
- `-v` — verbose: ispiši rezultat (open / refused)
- `web-svc 80` — host i port koji se testiraju

Provjereni ispis:
```
web-svc (10.98.162.190:80) open
pod "tmp" deleted
```
- `open` = TCP veza do ClusterIP-a na portu 80 prohodna na transportnom sloju
- `nc` u zagradi sam pokazuje da je ime `web-svc` razriješeno u ClusterIP `10.98.162.190` → potvrda da rade i DNS i port
- razlika slojeva: `wget` dokazuje cijeli HTTP put (L7, aplikacijski), `nc -z` dokazuje da TCP utičnica radi (L4, transportni) — kad nešto pukne, dijele dijagnostiku na „ne radi mreža" vs „ne radi aplikacija"

### GOTČA — `wget -O-` (VELIKO O) vs `-o-` (malo o)

Prvi pokušaj s `wget -qo-` (malo `o`) dao je SAMO `pod "tmp" deleted`, bez ijednog retka HTML-a:
- u busyboxu je `-o DATOTEKA` zastavica za LOG datoteku, NE za sadržaj
- preuzeti sadržaj se tiho spremio u datoteku unutar Poda (default ime izvedeno iz URL-a); `-q` (quiet) ugasio i log → na ekran ništa
- Pod se zatim obrisao (`--rm`) i odnio tu datoteku sa sobom → ostala samo poruka o brisanju
- VELIKO `-O DATOTEKA` = ispiši SADRŽAJ u to odredište; `-` kao odredište = stdout → stranica se ispiše
- pravilo: kod `wget`-a **veliko `O`** izbacuje sadržaj na ekran, **malo `o`** je log datoteka

### GOTČA — zaostali `tmp` Pod ako se prekine s Ctrl+C

- `--rm` čisti Pod kad naredba uredno ZAVRŠI; ako se prekine na pola (Ctrl+C dok visi), Pod zna ostati u stanju `Error`/`Completed`
- provjera: `kubectl get pods | grep tmp`; ručno čišćenje: `kubectl delete pod tmp`
- na ispitu: ako `kubectl run tmp` javi `pods "tmp" already exists` → zaostao je od prošlog puta → obrisati pa ponoviti

### Gotče

- ClusterIP je dohvatljiv samo IZNUTRA klastera → za test treba Pod unutar klastera (busybox jednokratni)
- `wget -qO-`: veliko `O` šalje sadržaj na stdout; malo `o` je log (tiha zamka — izgleda kao da ništa nije dohvaćeno)
- `wget` testira L7 (HTTP), `nc -zv` testira L4 (TCP) — kad nešto ne radi, suzuju gdje točno puca
- `--rm` čisti samo na urednom završetku; zaostali `tmp` Pod → `kubectl delete pod tmp`
- kratko ime servisa radi samo iz ISTOG namespacea; iz drugog namespacea treba FQDN (vidi zad. 43)

### Provjera

```bash
kubectl get svc web-svc                                                               # ClusterIP postoji, 80/TCP
kubectl run tmp --rm -it --image=busybox --restart=Never -- wget -qO- http://web-svc  # vrati nginx HTML
kubectl run tmp --rm -it --image=busybox --restart=Never -- sh -c 'nc -zv web-svc 80' # port open
```

---

## Zadatak 41 — headless servis (`clusterIP: None`)

**Cilj:** stvoriti headless servis i dokazati da DNS upit na njegovo ime vraća IP-eve SVIH Podova direktno (po jedan A-zapis za svaki Pod), za razliku od običnog ClusterIP servisa koji vraća jedan stabilan servisov IP.

**Koncept — običan ClusterIP vs headless:**
- **običan ClusterIP** (`web-svc`): ima vlastiti stabilan IP; kube-proxy dijeli promet po Podovima. DNS upit na ime → vrati JEDAN IP (servisov). Klijent priča s „bilo kojim zdravim" Podom.
- **headless** (`clusterIP: None`): NEMA vlastiti IP; kube-proxy se ne miješa. DNS upit na ime → vrati IP-eve SVIH Podova (A-zapis po Podu). Klijent vidi pojedinačne Podove i sam bira s kojim priča.
- čemu služi: kad klijent treba KONKRETAN Pod, ne „bilo koji" — tipično uz StatefulSet (baze, brokeri) gdje svaki član ima identitet (`db-0`, `db-1`…). Zato je 41 most prema Fazi 7.
- bitno: headless NIJE zaseban tip servisa. `TYPE` ostaje `ClusterIP`; jedina razlika je `clusterIP: None`.

### Naredba — stvaranje headless servisa

```bash
kubectl expose deployment web --name=web-headless --port=80 --target-port=80 --cluster-ip=None
kubectl get svc web-headless
```

- `kubectl expose deployment web` — stvara servis za Deployment `web` (preuzme njegov selektor `app=web`)
- `--name=web-headless` — ime novog servisa
- `--port=80` — port samog servisa (vrata na koja se priča servisu)
- `--target-port=80` — port na Podu gdje nginx sluša
- `--cluster-ip=None` — KLJUČNO: postavi `clusterIP: None` → headless, bez dodjele vlastitog IP-a

Provjereni ispis:
```
service/web-headless exposed

NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
web-headless   ClusterIP   None         <none>        80/TCP    12s
```
- `CLUSTER-IP: None` = potvrda headless servisa (običan bi imao `10.x.x.x`)
- `TYPE: ClusterIP` = headless nije zaseban tip, samo ClusterIP s `clusterIP: None`

### Akcija — DNS upit na headless vs običan servis

```bash
kubectl run tmp --rm -it --image=busybox --restart=Never -- nslookup web-headless
kubectl run tmp --rm -it --image=busybox --restart=Never -- nslookup web-svc
```

- jednokratni busybox Pod (kao zad. 46); naredba `nslookup <ime>` pita DNS koje IP-eve ima to ime

Provjereni ispis — headless (skraćeno, izbačen NXDOMAIN šum):
```
Server:    10.96.0.10
Address:   10.96.0.10:53

Name:   web-headless.default.svc.cluster.local
Address: 10.244.0.69
Name:   web-headless.default.svc.cluster.local
Address: 10.244.0.70
Name:   web-headless.default.svc.cluster.local
Address: 10.244.0.72
```

Provjereni ispis — običan `web-svc` (skraćeno):
```
Name:   web-svc.default.svc.cluster.local
Address: 10.98.162.190
```

Usporedba (poanta zadatka):
```
web-headless  →  10.244.0.69, .70, .72   (3 Pod-IP-a, direktno)
web-svc       →  10.98.162.190           (1 servisov ClusterIP)
```
- headless: tri adrese = tri nginx Poda (isti IP-evi kao u zad. 42)
- običan: jedna adresa = stabilan ClusterIP servisa; kube-proxy iza njega dijeli promet po Podovima

### Razlaganje DNS ispisa

- `Server: 10.96.0.10` (`:53`) = CoreDNS, interni DNS klastera (sluša na portu 53); svaki Pod pita njega za razrješavanje imena
- hrpa `** server can't find ...: NXDOMAIN` = normalan šum, NE greška: busybox `nslookup` redom proba sufikse iz `/etc/resolv.conf` (`.default.svc.cluster.local`, `.svc.cluster.local`, `.cluster.local`, `.dns.podman`, `.vua.cloud`); pogodi samo prvi (`.default.svc.cluster.local`), ostali daju NXDOMAIN i ignoriraju se

### GOTČA — `kubectl espose` (tipfeler) → kubectl sam predloži ispravak

Prvi pokušaj `kubectl espose ...`:
```
error: unknown command "espose" for "kubectl"

Did you mean this?
        expose
```
- kad `kubectl` ne prepozna naredbu, javi grešku ALI i predloži najbliži točan oblik
- navika: pročitati `Did you mean this?` prije ručnog ispravljanja

### GOTČA — `nslookup` Pod završi `terminated (Error)` iako je upit uspio

Na kraju ispisa stoji `pod default/tmp terminated (Error)`:
- `nslookup` vrati ne-nula izlazni kod ako MAKAR JEDAN pokušaj (od sufiksa iz search-liste) padne s NXDOMAIN — a padne ih više
- `Error` status dolazi od IZLAZNOG KODA alata, NE od neuspjeha razrješavanja → odgovor koji se tražio (adrese) JEST tu
- `--rm` svejedno počisti Pod (`pod "tmp" deleted`)
- pouka: kod busybox `nslookup`-a `Error` na izlazu ne znači da DNS nije odgovorio; gleda se jesu li tražene adrese ispisane

### Gotče

- headless = `clusterIP: None`; DNS vrati sve Pod-IP-eve direktno (A-zapis po Podu), bez servisovog IP-a i bez kube-proxyja
- `TYPE` ostaje `ClusterIP` — headless nije zaseban tip, prepoznaje se po `CLUSTER-IP: None`
- običan servis: 1 IP (servisov, stabilan); headless: N IP-eva (Podovi, prolazni)
- busybox `nslookup`: NXDOMAIN redci kroz search-listu su šum; `terminated (Error)` od izlaznog koda ne znači neuspjeh
- `kubectl` na nepoznatu naredbu predloži ispravak (`Did you mean this?`)

### Provjera

```bash
kubectl get svc web-headless                                                       # CLUSTER-IP = None
kubectl run tmp --rm -it --image=busybox --restart=Never -- nslookup web-headless  # 3 Pod-IP-a
kubectl run tmp --rm -it --image=busybox --restart=Never -- nslookup web-svc       # 1 ClusterIP
```

---

## Zadatak 43 — pristup servisu iz DRUGOG namespacea (FQDN)

**Cilj:** dokazati da se servis iz drugog namespacea NE dohvaća kratkim imenom, nego punim (FQDN) `<servis>.<namespace>.svc.cluster.local`.

**Koncept — namespace i razrješavanje imena:**
- **namespace** = logična pregrada unutar klastera (mape za grupiranje resursa). Servis `web-svc` živi u `default`.
- iz ISTOG namespacea: dovoljno kratko ime `web-svc` (DNS sam doda `.<vlastiti-ns>.svc.cluster.local`)
- iz DRUGOG namespacea: kratko ime promaši, jer ga DNS proba u VLASTITOM namespaceu Poda → treba FQDN s točnim namespaceom
- FQDN format: `web-svc.default.svc.cluster.local` = `<servis>.<namespace>.svc.cluster.local`

### Naredba — pripremi namespace pa kucni kratko (padne) vs FQDN (prođe)

```bash
kubectl create namespace test-ns
# kratko ime — namjerno padne:
kubectl run tmp --rm -it --image=busybox --restart=Never -n test-ns -- wget -qO- --timeout=3 http://web-svc
# FQDN — prođe:
kubectl run tmp --rm -it --image=busybox --restart=Never -n test-ns -- wget -qO- --timeout=3 http://web-svc.default.svc.cluster.local
```

- `kubectl create namespace test-ns` — stvara novi namespace `test-ns`
- `-n test-ns` — pokreće `tmp` Pod U namespaceu `test-ns` (ne u `default`)
- `--timeout=3` — `wget` čeka najviše 3 s pa odustane (da ne visi kad ime ne postoji)
- FQDN raščlanjen:
  - `web-svc` — ime servisa
  - `default` — namespace gdje servis živi (ključ: eksplicitno „traži u `default`, ne u `test-ns`")
  - `svc` — označava da je riječ o servisu
  - `cluster.local` — korijenska domena klastera

Provjereni ispis — kratko ime iz `test-ns` (padne):
```
wget: bad address 'web-svc'
pod "tmp" deleted
pod test-ns/tmp terminated (Error)
```
- `bad address 'web-svc'` = DNS u Podu probao `web-svc.test-ns.svc...` (vlastiti namespace) → tamo nema tog servisa → ne razriješi
- `Error` = izlazni kod `wget`-a (kao kod `nslookup`-a); ne znači ništa osim da dohvat nije uspio

Provjereni ispis — FQDN iz `test-ns` (prođe, skraćeno):
```
<!DOCTYPE html>
...
<h1>Welcome to nginx!</h1>
...
</html>
pod "tmp" deleted
```
- pun nginx HTML → preko granice namespacea servis JEST dohvatljiv, ali samo punim imenom

Lanac (zašto kratko padne, FQDN prođe):
```
kratko "web-svc" iz test-ns   →  DNS proba web-svc.test-ns.svc.cluster.local  →  nema → bad address
FQDN "web-svc.default.svc..." →  DNS ide ravno u default                       →  nađe ClusterIP → nginx HTML
```

### Gotče

- kratko ime servisa radi SAMO iz istog namespacea; preko granice treba FQDN `<servis>.<namespace>.svc.cluster.local`
- razlog: DNS kratko ime nadopuni VLASTITIM namespaceom Poda, ne ciljanim
- `--timeout=3` na `wget`-u korisno kad se ime možda neće razriješiti (inače dugo visi)
- `bad address` = problem razrješavanja IMENA (DNS); razlikovati od `connection refused` (ime se razriješilo, ali veza pala)
- isti `Error` izlazni status kao kod `nslookup`-a: gleda se sadržaj ispisa, ne samo status

### Provjera

```bash
kubectl run tmp --rm -it --image=busybox --restart=Never -n test-ns -- wget -qO- --timeout=3 http://web-svc.default.svc.cluster.local   # nginx HTML
```

Napomena (čišćenje): `test-ns` namespace ostaje za sada; obrisati na kraju Faze 4 zajedno s viškom servisa: `kubectl delete namespace test-ns`.

---

## Zadatak 44 — `kubectl port-forward` (pristup s VM-a, izvan klastera)

**Cilj:** otvoriti privremeni tunel s lokalnog stroja (VM) ravno do servisa u klasteru i dohvatiti ga s `localhost`, bez izlaganja servisa vani (bez NodePorta/LoadBalancera).

**Koncept — gdje port-forward sjeda:**
- dosad se servis kucao IZNUTRA (iz Poda u klasteru)
- `port-forward` probije tunel s VM-a do servisa/Poda → pristup preko `localhost:<port>` na samom VM-u
- namjena: debug i lokalni pristup tijekom razvoja; NE trajno izlaganje (tunel živi samo dok naredba radi)
- cilj tunela može biti `service/<ime>`, `pod/<ime>` ili `deployment/<ime>`

### Naredba — otvori tunel (terminal 1)

```bash
kubectl port-forward service/web-svc 8080:80
```

- `kubectl port-forward` — uspostavi tunel s lokalnog stroja do resursa u klasteru
- `service/web-svc` — cilj tunela je servis `web-svc`
- `8080:80` — `lokalniPort:ciljniPort`:
  - `8080` — port na VM-u (localhost) gdje tunel sluša
  - `80` — port servisa `web-svc` na koji se promet prosljeđuje

Provjereni ispis (naredba NE vraća prompt — drži tunel živim):
```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```
- dva retka: `127.0.0.1` (IPv4) i `[::1]` (IPv6 localhost)
- prompt ne dolazi natrag = ispravno; tunel radi dok je naredba živa → ostaviti terminal i otvoriti DRUGI za test

### Naredba — kucni tunel (terminal 2)

```bash
curl http://localhost:8080
```

- `curl` — HTTP dohvat s VM-a (ne treba Pod jer se kuca localhost, ne klaster-interno ime)
- `http://localhost:8080` — localhost = VM; `8080` = port koji `port-forward` sluša → kroz tunel do `web-svc:80`
- (ako `curl` ne postoji: `wget -qO- http://localhost:8080`)

Provjereni ispis (skraćeno):
```
<!DOCTYPE html>
...
<h1>Welcome to nginx!</h1>
...
</html>
```
- pun nginx HTML → zahtjev krenuo s VM-a (izvana klastera), prošao tunelom do servisa
- u terminalu 1 se istovremeno pojavi `Handling connection for 8080` (tunel bilježi svaki zahtjev)

Živi lanac:
```
curl localhost:8080 (VM)  →  kubectl tunel  →  web-svc ClusterIP :80  →  jedan nginx Pod  →  HTML natrag
```

### Zatvaranje tunela

```
Ctrl+C   # u terminalu 1 — prekida port-forward, vraća prompt
```
- tunel je privremen: postoji samo dok naredba radi; po prekidu pristup preko `localhost:8080` nestaje

### Gotče

- `port-forward` NE vraća prompt — to nije zaglavljivanje, nego znak da tunel radi; test ide iz drugog terminala
- `lokalniPort:ciljniPort` — lijevi je na VM-u, desni na servisu/Podu; ne moraju biti isti (npr. `9000:80`)
- pristup je samo s VM-a (`127.0.0.1`/`[::1]`), ne za vanjski svijet → za to služe NodePort/LoadBalancer
- `Handling connection for 8080` u terminalu tunela = dokaz da je zahtjev prošao kroz tunel
- tunel pukne ako Pod/servis nestane ili se `kubectl` prekine; samo ponovo pokrenuti naredbu

### Provjera

```bash
kubectl port-forward service/web-svc 8080:80     # terminal 1 — Forwarding from ... (drži otvoreno)
curl http://localhost:8080                       # terminal 2 — nginx HTML
# Ctrl+C u terminalu 1 za kraj
```

---

## Zadatak 47 — dva Deploymenta iza JEDNOG servisa (selektor po zajedničkoj labeli)

**Cilj:** dokazati da servis bira Podove ISKLJUČIVO po labeli iz selektora, a ne po Deploymentu kojem pripadaju. Ako Podovi dvaju različitih Deploymenta dijele istu labelu, jedan servis ih hvata sve i dijeli promet preko svih zajedno (temelj za blue-green / canary).

**Koncept:**
- servis ne zna ni za Deployment ni za RS — gleda samo labele Podova kroz svoj `selector`
- dva Deploymenta + ista labela na njihovim Podovima + servis koji gađa tu labelu = jedan ulaz, svi Podovi obje grupe
- praktično: dvije verzije aplikacije (v1, v2) iza istog servisa → postupno preusmjeravanje prometa

### Korak 1 — drugi Deployment + zašto labela na Deploymentu nije dovoljna

```bash
kubectl create deployment web2 --image=nginx:1.25 --replicas=2
kubectl label deployment web2 app=web --overwrite
kubectl get pods -l app=web -o wide
```

- `kubectl create deployment web2 --image=nginx:1.25 --replicas=2` — novi Deployment `web2`, 2 replike, isti nginx
  - `--replicas=2` — broj Podova (2, lako se razlikuje od originalna 3)
- `kubectl label deployment web2 app=web --overwrite` — promijeni labelu na Deployment OBJEKTU
  - `--overwrite` — dopusti promjenu postojeće labele (inače `kubectl` odbije)
- `kubectl get pods -l app=web -o wide` — provjera koliko Podova nosi `app=web`

Provjereni ispis: i dalje samo **3 Poda** (`web-*`), `web2` Podovi se NE vide pod `app=web`.

#### GOTČA — labela na Deploymentu ≠ labela na njegovim Podovima

- `kubectl label deployment web2 app=web` mijenja labelu na Deployment OBJEKTU, ne na Podovima
- Podovi labele dobivaju iz Deploymentovog `template`-a; `web2` je preko templatea svojim Podovima stavio `app=web2` (default kod `create deployment web2`)
- zato `web2` Podovi i dalje nemaju `app=web` → servis `web-svc` ih ne vidi
- dodatna kvaka: `web2` Podovima se NE može samo nalijepiti `app=web` jer je ključ `app` zaključan `web2`-ovim selektorom (`app=web2`) — jedan ključ, jedna vrijednost

### Korak 2 — zajednička labela s NOVIM ključem na Podovima obje grupe

```bash
kubectl get pods -l app=web2 -o wide
kubectl label pods -l app=web  group=weball
kubectl label pods -l app=web2 group=weball
kubectl get pods -l group=weball -o wide
```

- `kubectl get pods -l app=web2 -o wide` — potvrdi da su `web2` Podovi gore (2 kom., `app=web2`)
- `kubectl label pods -l app=web group=weball` — dodaj labelu Podovima:
  - `-l app=web` — KOJIM Podovima (filtar po postojećoj labeli)
  - `group=weball` — KOJU labelu dodati; `group` je NOVI ključ (ne sudara se s `app`) → ne treba `--overwrite`
- treća naredba isto za `app=web2` Podove
- `kubectl get pods -l group=weball -o wide` — provjera tko nosi `group=weball`

Provjereni ispis: **5 Podova** s `group=weball`:
```
web-76994d859f-djnn7    2/2   Running   10   15h     10.244.0.72   # web (sa sidecarom)
web-76994d859f-mq22g    2/2   Running   10   15h     10.244.0.69
web-76994d859f-vr9gm    2/2   Running   10   15h     10.244.0.70
web2-8ff696bb9-4p8gb    1/1   Running   0    8m45s   10.244.0.83   # web2 (samo nginx)
web2-8ff696bb9-vjc2j    1/1   Running   0    8m45s   10.244.0.82
```
- `web` Podovi: `2/2` (nginx + sidecar), `RESTARTS 10`, AGE 15h
- `web2` Podovi: `1/1` (samo nginx), `RESTARTS 0`, AGE 8m → lako se razlikuju

#### GOTČA — `kubectl label pods` djeluje na ŽIVE Podove, NE na template

- direktno labeliranje Podova vrijedi ODMAH, ali nije u Deploymentovom `template`-u
- ako se Pod kasnije recreira (rollout / `delete pod`), novi NEĆE imati `group=weball`
- trajna varijanta = mijenjati `template` (ali to pokreće rollout i dira selektor); za demo je direktno labeliranje čišće i bezopasno

### Korak 3 — servis na zajedničku labelu + provjera Endpoints

```bash
kubectl create service clusterip web-all --tcp=80:80
kubectl set selector service web-all 'group=weball'
kubectl get endpoints web-all
kubectl get endpoints web-all -o jsonpath='{.subsets[0].addresses[*].ip}{"\n"}'
```

- `kubectl create service clusterip web-all --tcp=80:80` — ClusterIP servis `web-all`
  - `clusterip` — tip; `--tcp=80:80` = `port:targetPort`
  - GOTČA: ovaj oblik stvori servis s DEFAULT selektorom `app=web-all` → zato ga odmah mijenjamo
- `kubectl set selector service web-all 'group=weball'` — prepiši selektor na `group=weball` (sad gađa svih 5)
  - navodnici oko selektora = zaštita od shella (dobra navika)
- `kubectl get endpoints web-all` — koje Pod-IP-eve je servis pokupio
- `-o jsonpath='{.subsets[0].addresses[*].ip}'` — ispiši SVE IP-eve bez skraćivanja (`get endpoints` skrati na „+N more...")

Provjereni ispis:
```
# kubectl get endpoints web-all
NAME      ENDPOINTS                                       AGE
web-all   10.244.0.69:80,10.244.0.70:80,10.244.0.72:80 + 2 more...   17s

# jsonpath (bez skraćivanja)
10.244.0.69 10.244.0.70 10.244.0.72 10.244.0.82 10.244.0.83
```
- 5 adresa = 3 (`web`) + 2 (`web2`) → jedan servis preko zajedničke labele hvata Podove iz DVA Deploymenta
- `web-svc` (gađa `app=web`) ostao netaknut → i dalje vidi samo svoja 3

#### GOTČA — `get endpoints` skrati ispis na „+N more...", jsonpath pokaže sve

- `kubectl get endpoints` u stupcu `ENDPOINTS` skraćuje dugačke popise (npr. `... + 2 more...`)
- za pun popis: `-o jsonpath='{.subsets[0].addresses[*].ip}'` ili `kubectl describe endpoints web-all`

### Gotče

- servis bira po LABELI iz selektora, ne po Deploymentu → ista labela na dva Deploymenta = jedan servis hvata oba
- labela na Deployment objektu ≠ labela na Podovima (Podovi je dobivaju iz `template`-a)
- ključ labele je zaključan selektorom Deploymenta (`app=web2`) — za međugrupnu labelu uzeti NOVI ključ (`group`)
- `kubectl label pods ...` djeluje na žive Podove, ne na template → recreate gubi labelu
- `create service clusterip` postavi default selektor `app=<ime>` → po potrebi prepisati s `set selector`
- `get endpoints` skraćuje (`+N more...`); pun popis kroz `jsonpath` / `describe`

### Provjera

```bash
kubectl get pods -l group=weball -o wide                                          # 5 Podova (3 web + 2 web2)
kubectl get endpoints web-all -o jsonpath='{.subsets[0].addresses[*].ip}{"\n"}'   # 5 IP-eva
```

Napomena (čišćenje): `web2` Deployment, servis `web-all` i labela `group` su demo višak; obrisati na kraju Faze 4 (`kubectl delete deployment web2`, `kubectl delete service web-all`).

---

## Zadatak 49 — NodePort od nule (deklarativno) + dohvat izvana

**Cilj:** stvoriti NodePort servis iz YAML manifesta (deklarativni put) s FIKSNIM `nodePort`-om i dokazati dohvat izvana preko `<IP-čvora>:<nodePort>`. Srodno zad. 39, ali ondje imperativno (`kubectl expose`), ovdje deklarativno (`apply -f`).

**Koncept (kratko):**
- NodePort otvori isti port na ČVORU (raspon 30000–32767) → servis dohvatljiv izvana preko `<IP čvora>:<nodePort>`
- ispod sebe uvijek ima ClusterIP (slojevitost: NodePort → ClusterIP → Podovi)
- razlika prema zad. 39: tamo `expose` (imperativno), ovdje manifest + `apply` (deklarativno, ide u git)

### Korak 1 — manifest

```bash
gedit ~/devops/lo4/web-nodeport.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-np2
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

- `apiVersion: v1` — Servisi su u core API grupi (`v1`), bez prefiksa
- `kind: Service` — tip objekta
- `metadata.name: web-np2` — ime (`np2` da se ne sudara s `web-np` iz zad. 39)
- `spec.type: NodePort` — otvara port na čvoru
- `spec.selector.app: web` — gađa Podove `app=web` (3 originalna)
- `spec.ports`:
  - `port: 80` — vrata servisa (ClusterIP sloj, iznutra)
  - `targetPort: 80` — vrata na Podu gdje nginx sluša
  - `nodePort: 30080` — FIKSNI port na čvoru, biran ručno (mora 30000–32767); bez ovog retka k8s dodijeli nasumičan

### Korak 2 — apply + provjera

```bash
kubectl apply -f ~/devops/lo4/web-nodeport.yaml
kubectl get svc web-np2
```

- `kubectl apply -f <putanja>` — čita manifest s DISKA i stvara/ažurira servis (`-f` = file)
- `kubectl get svc web-np2` — rezultat

Provjereni ispis:
```
NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
web-np2   NodePort   10.103.92.250   <none>        80:30080/TCP   9s
```
- `PORT(S): 80:30080/TCP` → lijevo `port`, desno NAŠ fiksni `nodePort` (za razliku od zad. 39 gdje je k8s dao nasumičnih `31629`)
- dobio i ClusterIP `10.103.92.250` → NodePort uvijek ima ClusterIP ispod sebe

### Korak 3 — dohvat izvana (preko čvora)

```bash
curl http://$(minikube ip):30080
minikube service web-np2 --url
```

- `$(minikube ip)` — shell supstitucija: `minikube ip` vrati IP čvora (`192.168.49.2`), uvrsti se u URL → `http://192.168.49.2:30080`
- `:30080` — naš nodePort na čvoru
- `minikube service web-np2 --url` — ispiše punu URL adresu servisa bez ručnog sastavljanja (kao zad. 39)

Provjereni ispis:
```
# curl http://$(minikube ip):30080
<!DOCTYPE html> ... <h1>Welcome to nginx!</h1> ... </html>

# minikube service web-np2 --url
http://192.168.49.2:30080
```
- nginx HTML preko `<IP čvora>:nodePort` → pravi vanjski put (ne tunel, ne iznutra)
- `--url` potvrđuje IP čvora `192.168.49.2` + nodePort `30080` (isto što je `$(minikube ip)` uvrstio)

### Gotče

- `nodePort` se smije fiksirati u manifestu (30000–32767); bez njega k8s dodijeli nasumičan
- deklarativno (`apply -f`) vs imperativno (`expose`): isti rezultat, ali manifest ide u git i ponovljiv je
- `apply` čita s DISKA → `Ctrl+S` u editoru prije `apply` (inače se primijeni stara verzija)
- NodePort uvijek ima ClusterIP ispod (slojevitost); `<IP čvora>:nodePort` je samo vanjski ulaz
- `$(minikube ip)` zgodno za sastavljanje URL-a; `minikube service <ime> --url` isto bez ručnog rada

### Provjera

```bash
kubectl get svc web-np2                      # PORT(S) = 80:30080/TCP
curl http://$(minikube ip):30080             # nginx HTML
minikube service web-np2 --url               # http://<IP čvora>:30080
```

---

## Faza 4 — završena

Svih 12 zadataka Faze 4 (Servisi) odrađeno i verificirano: 12, 38, 39, 40, 42, 45, 46, 41, 43, 44, 47, 49.

Pokriveno: ClusterIP (stabilan IP, samo iznutra) · NodePort (vani preko čvora, 30000–32767) · LoadBalancer (`<pending>` na minikubeu) · headless (`clusterIP: None`, DNS vrati sve Pod-IP-eve) · DNS/FQDN (kratko ime u istom ns, FQDN preko granice) · `port`/`targetPort`/`nodePort` · Endpoints/EndpointSlice (selektor → popis Pod-IP-eva) · debug Pod (`wget`/`nc` iznutra) · `port-forward` (tunel s VM-a) · dva Deploymenta iza jednog servisa (selektor po labeli).

**Čišćenje viška iz Faze 4** (pokrenuti prije Faze 5 da se klaster rastereti):
```bash
kubectl delete service web-headless web-all web-np2     # demo servisi (ostaju web-svc, web-np, web-lb)
kubectl delete deployment web2                          # demo Deployment
kubectl delete namespace test-ns                        # demo namespace
```
(Originalni `web` Deployment + `web-svc`/`web-np`/`web-lb` ostaju za Fazu 5.)

---

## Faza 5 — Probe + dijagnostika

Cilj faze: provjere zdravlja Poda (liveness / readiness / startup) i playbook „zašto pod ne odgovara". Pet zadataka: 48, 50, 51, 52, 53. Glavni dijagnostički alat cijele faze: **`kubectl describe pod` → sekcija `Events` na dnu** — tu piše ZAŠTO se nešto dogodilo.

## Zadatak 50 — HTTP livenessProbe (nginx) + uvod u readiness/startup

**Cilj:** na živom klasteru dokazati tri tvrdnje: (1) kad padne READINESS, Pod ide u `0/1` ali ostaje `Running` i biva izbačen iz Endpoints (promet stane); (2) kad padne LIVENESS, kubelet ubije i restarta container (`RESTARTS+1`); (3) nakon restarta Pod se sam oporavi kad aplikacija opet odgovara.

**Koncept — tri vrste probe:**
- **readinessProbe** — „je li Pod spreman PRIMATI promet?" Pada → Pod izbačen iz Endpoints servisa (promet mu staje), ali Pod I DALJE RADI (nije ubijen). Vratar za Endpoints.
- **livenessProbe** — „je li container ŽIV?" Pada → kubelet UBIJE i RESTARTA container (`RESTARTS+1`). Mehanizam samoizlječenja.
- **startupProbe** — „je li se container POKRENUO?" Za spore startere; dok traje, liveness i readiness MIRUJU (ne počnu dok startup ne prođe). Sprječava da liveness ubije app koji se još diže.

**Razlika u jednoj rečenici:** readiness odlučuje DOBIVA li Pod promet; liveness odlučuje hoće li se RESTARTATI.

**Parametri probe (vrijede za sve tri):**
- `httpGet.path` + `httpGet.port` — koji HTTP put i port kucati (200–399 = uspjeh; ≥400 = pad)
- `initialDelaySeconds` — koliko čekati nakon starta containera prije PRVE provjere
- `periodSeconds` — razmak između provjera
- `failureThreshold` (default **3**) — koliko UZASTOPNIH padova = probe se proglasi neuspjelom
- `timeoutSeconds` (default 1), `successThreshold` (default 1) — koliko čekati odgovor / koliko uspjeha za oporavak

### Manifest (Deployment + Service u istom fileu)

```bash
gedit ~/devops/lo4/probe-demo.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: probe-demo
  template:
    metadata:
      labels:
        app: probe-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: probe-demo-svc
spec:
  selector:
    app: probe-demo
  ports:
    - port: 80
      targetPort: 80
```

- `---` — razdjelnik: jedan YAML file smije sadržavati VIŠE objekata; jedan `apply` stvori oba (Deployment + Service)
- `readinessProbe` i `livenessProbe` su BRAĆA — moraju biti na istoj uvlaci (oba pod `- name: nginx`, poravnata s `ports:`)
- readiness 2/5 (brža), liveness 5/10 (sporija) → readiness prva reagira na kvar

### Apply + provjera da su OBJE probe aktivne

```bash
kubectl apply -f ~/devops/lo4/probe-demo.yaml
kubectl get pods -l app=probe-demo
kubectl describe pod -l app=probe-demo | grep -E 'Liveness|Readiness'
```

- `kubectl describe pod ... | grep -E 'Liveness|Readiness'` — brza provjera koje su probe STVARNO aktivne
  - `|` = cijev (izlaz lijeve → ulaz desne); `grep` = filtar redaka; `-E` = prošireni regex (omogućuje `|` kao „ili"); `'Liveness|Readiness'` = uzorak

Provjereni ispis (obje probe aktivne):
```
Liveness:   http-get http://:80/ delay=5s timeout=1s period=10s #success=1 #failure=3
Readiness:  http-get http://:80/ delay=2s timeout=1s period=5s  #success=1 #failure=3
```
- ako se vidi SAMO `Readiness:` redak (bez `Liveness:`) → liveness blok je ispao iz manifesta (vidi gotču dolje)

### Sabotaža — sruši `/` da probe počnu padati (403)

```bash
POD=$(kubectl get pod -l app=probe-demo -o jsonpath='{.items[0].metadata.name}')
echo $POD
kubectl exec $POD -- mv /usr/share/nginx/html/index.html /usr/share/nginx/html/index.bak
```

- `POD=$(...)` — spremi ime Poda u varijablu (znak `=` BEZ razmaka — `POD =` ili `POD $(...)` ne radi)
  - `-o jsonpath='{.items[0].metadata.name}'` — izvuče ime prvog Poda; izraz MORA biti u vitičastim zagradama `{...}`
- `kubectl exec $POD -- mv .../index.html .../index.bak` — preimenuj nginxov index UNUTAR containera → `/` više nema što servirati → vraća **403** → obje probe (kucaju `/`) počnu padati

### Posljedica uživo

```bash
kubectl get pods -l app=probe-demo -w
```

- `-w` = watch: ne izlazi, ispisuje SVAKU promjenu stanja u stvarnom vremenu

Tijek (svi redci iskaču sami):
1. ~15 s (3 × readiness 5 s): `READY 1/1 → 0/1`, ali `STATUS Running`, `RESTARTS 0` → readiness pala, Pod izbačen iz Endpoints, ali ŽIV
2. ~30 s (3 × liveness 10 s): `RESTARTS 0 → 1` → kubelet UBIO i restartao container (liveness)
3. odmah nakon: nginx se digne SVJEŽ (čisti image → `index.html` se vrati) → readiness prolazi → `READY 1/1`, Pod se SAM izliječio

Provjereni ispis (uhvaćeno restartano stanje):
```
NAME                          READY   STATUS    RESTARTS        AGE
probe-demo-7b7b5b966f-7gbkr   1/1     Running   1 (2m11s ago)   11m
```
- `RESTARTS 1` = liveness odradila restart; ISTO ime Poda + AGE 11m = restart CONTAINERA u istom Podu, NE recreate Poda

### Endpoints dokaz (readiness izbacuje iz prometa)

```bash
kubectl get endpoints probe-demo-svc
```
- zdrav Pod: `ENDPOINTS = 10.244.0.x:80`; nakon pada readiness: `ENDPOINTS = <none>` (prazno) → servis mu prestao slati promet

### Events dokaz (zašto je restartao) — GLAVNI alat Faze 5

```bash
kubectl describe pod -l app=probe-demo | grep -E 'Liveness|Killing|Unhealthy|Started|Created'
```

Provjereni ispis:
```
Warning  Unhealthy  (x8 over 3m52s)  Readiness probe failed: HTTP probe failed with statuscode: 403
Warning  Unhealthy  (x3 over 3m47s)  Liveness probe failed: HTTP probe failed with statuscode: 403
Normal   Killing                     Container nginx failed liveness probe, will be restarted
Normal   Created    (x2 over 12m)    Created container nginx
Normal   Started    (x2 over 12m)    Started container nginx
```
- `Readiness ... failed ... 403` ×8 (kuca svakih 5 s, najbrža) → prva izbacila Pod u `0/1`
- `Liveness ... failed ... 403` ×3 → na pragu `failureThreshold: 3` kubelet reagira
- `Killing ... failed liveness probe, will be restarted` = DOSLOVAN dokaz da restart dolazi OD LIVENESS probe
- `Created`/`Started (x2 over 12m)` = container pokrenut 2× (stvaranje + restart)
- pravilo: Pod restarta a uzrok nije poznat → `describe` → `Events` → tu piše `failed liveness probe`

### GOTČA — probe blok ISPADNE pri ručnom prepisivanju (uvlaka)

Prvi pokušaj: u manifestu je bio SAMO `readinessProbe`, `livenessProbe` je nestao pri prepisivanju → Pod nikad nije restartao (nema tko ubiti container):
- `describe` je pokazivao samo `Readiness:` redak, bez `Liveness:`; `Restart Count: 0`; u `Events` samo readiness
- dodatni trag: `Readiness:` redak je imao `delay=5s period=10s` — a to su bile LIVENESS vrijednosti (znak da su se blokovi stopili)
- pouka: kad ponašanje ne odgovara očekivanju → `cat` manifest / `describe` pokaže što je STVARNO primijenjeno
- popravak bez ručnog YAML-a: `kubectl patch` (dodaj liveness na živi Deployment jednom naredbom, bez uvlaka)

### GOTČA — shell zamke pri prepisivanju (dodjela varijable, jsonpath)

- `POD $(...)` (bez `=`) → `bash: POD: command not found`. Dodjela MORA biti `POD=$(...)`, znak `=` BEZ razmaka s obje strane
- `jsonpath='.items[0]...'` (bez vitičastih zagrada) → `echo` ispiše doslovno `.items[0].metadata.name}` umjesto vrijednosti. Izraz mora biti `'{.items[0].metadata.name}'`

### GOTČA — `-w` watch se ZAMRZNE pod RAM pritiskom

- na ovom VM-u watch konekcija zna tiho stati: ekran drži STARO stanje, stvarno stanje se promijenilo
- simptom: ništa se ne miče duže od ~40 s iako bi trebalo
- popravak: `Ctrl+C` → svježe jednokratno čitanje `kubectl get pods -l app=probe-demo` + `kubectl describe pod ...` (sekcija `Events`)

### Gotče

- readiness pada → `0/1`, `Running`, izbačen iz Endpoints (promet stane), NE restarta se
- liveness pada → container restartan (`RESTARTS+1`); ISTO ime Poda = restart containera, ne recreate Poda
- startup probe miruje liveness/readiness dok se app diže (za spore startere)
- `failureThreshold` default 3 → treba 3 uzastopna pada prije reakcije; vrijeme do reakcije = `failureThreshold × periodSeconds`
- probe blokovi (`readinessProbe`/`livenessProbe`) su braća na istoj uvlaci — najčešća greška je da jedan ispadne
- glavni alat: `describe pod` → `Events` (tu piše `failed liveness probe` i `Killing`)
- HTTP probe: 200–399 = zdravo, ≥400 = pad (zato 403 ruši `httpGet /`)

### Provjera

```bash
kubectl describe pod -l app=probe-demo | grep -E 'Liveness|Readiness'   # obje probe aktivne
kubectl get pods -l app=probe-demo -w                                   # readiness 0/1, pa liveness RESTARTS+1, pa 1/1
kubectl describe pod -l app=probe-demo | grep -E 'Killing|Unhealthy'    # Events: failed liveness probe → Killing
```

Napomena (čišćenje): `probe-demo` Deployment + `probe-demo-svc` ostaju za iduće zadatke Faze 5 (probe se još koriste); obrisati na kraju faze.

---

## Dijagnostički alat — `kubectl exec` (ulazak u kontejner, naredbe iznutra)

> Nije numerirani LO4 zadatak — dijagnostički alat. Uključeno jer ga F5 koristi; veže se na LO5 (exec za DNS debug = LO5 31, ephemeral debug container = LO5 32).

**Cilj:** izvršiti naredbe UNUTAR kontejnera — jednokratno i interaktivno (shell) — te razumjeti razliku Pod vs kontejner i izbor kontejnera s `-c`.

**Koncept — dva načina:**
- **jednokratno:** `kubectl exec POD -- <naredba>` — izvrši jednu naredbu u kontejneru, ispiše rezultat, izađe
- **interaktivno:** `kubectl exec -it POD -- sh` — „uđe" u shell unutar kontejnera, tipka se kao na tom stroju dok se ne napiše `exit`

**Pod vs kontejner (bitna distinkcija):**
- `exec` ulazi u KONTEJNER, ne u „Pod kao cjelinu". Pod nije stroj — Pod je OMOTAČ oko jednog/više kontejnera koji dijele mrežu i (dio) diska
- Pod s VIŠE kontejnera (npr. `web`: nginx + sidecar): `exec` ulazi u PRVI po defaultu; izbor s `-c <ime>`
- `hostname` unutar kontejnera = ime PODA (ne kontejnera), jer svi kontejneri u Podu dijele isti network namespace (zato i `localhost` komunikacija sidecara) — to NE znači „u Podu", nego da kontejner dijeli Podov mrežni identitet

### Jednokratni exec

```bash
POD=$(kubectl get pod -l app=probe-demo -o jsonpath='{.items[0].metadata.name}')
echo $POD
kubectl exec $POD -- ls /usr/share/nginx/html
```

- `POD=$(...)` — ime u varijablu (`=` bez razmaka; jsonpath u `{...}`)
- `kubectl exec $POD -- ls /usr/share/nginx/html`:
  - `--` = granica (sve iza je naredba u kontejneru)
  - `ls /usr/share/nginx/html` = ispiši nginxov html direktorij

Provjereni ispis:
```
50x.html
index.html
```
- `index.html` opet postoji (nema `index.bak`) → dokaz da se Pod izliječio: liveness restart digao SVJEŽ kontejner iz čistog image-a, sabotaža (`mv index.html index.bak`) nestala s restartom

### Interaktivni shell

```bash
kubectl exec -it $POD -- sh
```

- `-i` stdin otvoren + `-t` TTY → živi terminal; `-- sh` = pokreće shell koji ostaje i čeka tipkanje
- prompt se promijeni (`[student@vm69-61 ~]$` → `#`) = sada „unutra"

Naredbe iznutra (provjereno):
```sh
# hostname
probe-demo-7b7b5b966f-7gbkr     # ime Poda (dijeljeni network namespace), ne VM
# cat /usr/share/nginx/html/index.html
<!DOCTYPE html> ... <h1>Welcome to nginx!</h1> ... </html>
# whoami
root                            # nginx kontejner po defaultu vrti kao root
# exit                          # natrag na VM (prompt → [student@vm69-61 ~]$)
```

### Izbor kontejnera kad ih Pod ima više

```bash
kubectl exec -it $POD -c nginx -- sh       # u kontejner imena nginx
# kubectl exec -it <web-pod> -c <sidecar> -- sh   # u sidecar (Pod web ima 2 kontejnera)
```
- `-c <ime>` = bira kontejner; bez njega `exec` uđe u PRVI
- `probe-demo` ima samo jedan kontejner (`nginx`) → `-c` nije bio nužan

### Gotče

- `exec` ulazi u KONTEJNER; Pod je skupina kontejnera s dijeljenom mrežom; `-c` bira kontejner kad ih je više
- `hostname` u kontejneru = ime Poda (dijeljeni namespace), ne dokaz da si „u Podu"
- iz interaktivnog shella se izlazi s `exit` (ili Ctrl+D) — inače sve tipkano ide kontejneru
- `--` granica obavezna: sve iza je naredba u kontejneru, ne zastavica za `kubectl`
- restart kontejnera vraća datotečni sustav na čisti image (zato je `index.html` opet tu) — promjene napravljene s `exec` NISU trajne preko restarta

### Provjera

```bash
kubectl exec $POD -- ls /usr/share/nginx/html     # 50x.html, index.html
kubectl exec -it $POD -- sh                        # uđe u shell; hostname = ime Poda; exit za izlaz
```

---

## Dijagnostički alat — `kubectl logs` (čitanje logova kontejnera)

> Nije numerirani LO4 zadatak — dijagnostički alat. Veže se na LO5 (`logs --previous` za CrashLoop = LO5 22).

**Cilj:** čitati stdout/stderr kontejnera — cijeli log, zadnjih N redaka, živo praćenje (`-f`), i logove MRTVE instance prije restarta (`--previous`, ključ za CrashLoop dijagnostiku).

**Koncept:**
- `kubectl logs` pokazuje što je kontejner ispisao na stdout/stderr (kod nginxa = startup poruke + access-log svakog HTTP zahtjeva)
- logovi su prvo mjesto za gledanje kad app radi čudno
- kod Poda s više kontejnera: `-c <ime>` bira kontejner (kao kod `exec`)

### Cijeli log (generiraj promet pa čitaj)

```bash
kubectl run tmp --rm -it --image=busybox --restart=Never -- wget -qO- http://probe-demo-svc
kubectl logs $POD
```

- prvi red = jednokratni busybox kuca servis → nginx zabilježi jedan zahtjev
- `kubectl logs $POD` = ispiši sve dosad zapisano (`$POD` iz varijable; po potrebi ponovo `POD=$(kubectl get pod -l app=probe-demo -o jsonpath='{.items[0].metadata.name}')`)

Provjereni ispis — tri sloja:
```
# startup (nginx se diže nakon zadnjeg restarta)
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/06/22 12:37:22 [notice] 1#1: nginx/1.25.5
2026/06/22 12:37:22 [notice] 1#1: start worker process 29

# probe zahtjevi (readiness + liveness iz zad. 50)
10.244.0.1 - - [.../12:37:32 ...] "GET / HTTP/1.1" 200 615 "-" "kube-probe/1.31" "-"
10.244.0.1 - - [.../12:37:32 ...] "GET / HTTP/1.1" 200 615 "-" "kube-probe/1.31" "-"

# vlastiti wget (drugačiji user-agent + IP)
10.244.0.91 - - [...] "GET / HTTP/1.1" 200 615 "-" "Wget" "-"
```
- `"kube-probe/1.31"` user-agent (IP `10.244.0.1` = node/kubelet) = readiness/liveness probe → u logu se DOSLOVNO vidi kako kubelet kuca `/`
- probe dolaze u parovima na isti timestamp = readiness (5 s) + liveness (10 s) se povremeno poklope
- `"Wget"` user-agent (IP `10.244.0.x` Poda) = vlastiti zahtjev
- dijagnostički uvid: za provjeru rade li sonde, log to pokaže izravno

### Zadnjih N + živo praćenje

```bash
kubectl logs $POD --tail=5 -f
```

- `--tail=5` = samo zadnjih 5 redaka (kad je log golem)
- `-f` = follow: ostane i ispisuje NOVE retke uživo (kao `tail -f`); NE vraća prompt → `Ctrl+C` za izlaz

Provjereni ispis: zadnjih 5 redaka pa svakih ~5 s novi `kube-probe` redak uživo.

### Logovi MRTVOG kontejnera prije restarta

```bash
kubectl logs $POD --previous --tail=20
```

- `--previous` (ili `-p`) = logovi PRETHODNE instance kontejnera (one koja je umrla prije zadnjeg restarta)
- `--tail=20` = zadnjih 20 redaka

Provjereni ispis (zadnji uzdah ubijenog kontejnera iz zad. 50):
```
2026/06/22 12:37:22 [notice] 29#29: gracefully shutting down
2026/06/22 12:37:22 [notice] 1#1: signal 17 (SIGCHLD) received from 32
2026/06/22 12:37:22 [notice] 1#1: worker process 31 exited with code 0
2026/06/22 12:37:22 [notice] 1#1: exit
```
- nginx se uredno gasi (`gracefully shutting down` → `exit`) jer ga je kubelet ubio nakon pala liveness probe; vrijeme `12:37:22` = trenutak restarta iz zad. 50
- ZAŠTO je zlato: kod `CrashLoopBackOff` kontejner stalno restarta → `kubectl logs` (bez `-p`) pokazuje NOVI kontejner koji se diže; razlog pada je u ZADNJEM uzdahu STAROG → `--previous` ga daje
- (403 zapisi od sabotaže ispali izvan zadnjih 20 redaka; za njih povećati `--tail`)

### GOTČA — `previous terminated container ... not found`

- ako kontejner NIJE nikad restartao, ILI je stari log već počišćen (prošlo dosta vremena / pritisak) → `--previous` javi `not found`
- to NIJE greška u naredbi, nego nedostupan stari log

### GOTČA — `kube logs` umjesto `kubectl logs`

- `kube logs $POD` → `bash: kube: command not found`; naredba je `kubectl` (omakne se pri kraćenju)

### Gotče

- `kubectl logs POD` = stdout/stderr kontejnera; `-c <ime>` bira kontejner kod više njih
- `--tail=N` ograniči broj redaka; `-f` (follow) prati uživo (NE vraća prompt → `Ctrl+C`)
- `--previous`/`-p` = log mrtve instance prije restarta → ključ za `CrashLoopBackOff` (razlog pada je u starom logu)
- u nginx access-logu se vide i same probe (`kube-probe` user-agent) → potvrda da probe rade
- `--previous not found` = nema restarta ili je stari log počišćen (ne greška u naredbi)

### Provjera

```bash
kubectl logs $POD                      # cijeli log (startup + probe promet + vlastiti zahtjev)
kubectl logs $POD --tail=5 -f          # zadnjih 5 + živo (Ctrl+C za izlaz)
kubectl logs $POD --previous --tail=20 # log mrtvog kontejnera prije restarta
```

---

## Dijagnostički alat — `describe` + `Events` (živi kvar: ImagePullBackOff)

> Nije numerirani LO4 zadatak — dijagnostički alat. Veže se na LO5 (`describe pod` za Pending = LO5 19, `get events --sort-by` = LO5 30).

**Cilj:** na namjerno pokvarenom Podu pokazati dijagnostički obrazac: `STATUS` (iz `get`) kaže ŠTO ne valja, sekcija `Events` (iz `describe`) kaže ZAŠTO točno — bez nagađanja.

**Koncept:**
- `kubectl get pod` → stupac `STATUS` = simptom (npr. `ImagePullBackOff`, `CrashLoopBackOff`, `Pending`)
- `kubectl describe pod` → sekcija `Events` na dnu = kronologija s točnim uzrokom
- pravilo: Pod ne radi → `get` (koji simptom) → `describe`/`Events` (zašto)

### Izazovi kvar — Pod s nepostojećim imageom

```bash
kubectl run broken --image=nginx:does-not-exist-9999 --restart=Never
kubectl get pod broken
```

- `--image=nginx:does-not-exist-9999` — tag koji NE postoji na registriju → povlačenje mora pasti
- `--restart=Never` — goli Pod (da se ne recreira)

Provjereni prijelaz statusa:
```
NAME     READY   STATUS              RESTARTS   AGE
broken   0/1     ErrImagePull        0          8s     # prvi pokušaj povlačenja pao
broken   0/1     ImagePullBackOff    0          27s    # kubelet odustao od brzog ponavljanja, čeka
```
- `ErrImagePull` = upravo pao pokušaj povlačenja
- `ImagePullBackOff` = kubelet usporava (čeka sve duže prije idućeg pokušaja)

### Dijagnoza — `describe` → `Events`

```bash
kubectl describe pod broken
```

Provjereni `Events` (kronologija kvara):
```
Type      Reason      Age              From                Message
Normal    Scheduled   46s              default-scheduler   Successfully assigned default/broken to minikube
Normal    Pulling     4s (x3 over 46s) kubelet             Pulling image "nginx:does-not-exist-9999"
Warning   Failed      3s (x3 over 45s) kubelet             Failed to pull image "...": Error response from daemon:
                                                           manifest for nginx:does-not-exist-9999 not found: manifest unknown
Warning   Failed      3s (x3 over 45s) kubelet             Error: ErrImagePull
Normal    BackOff     16s (x2 over 44s) kubelet            Back-off pulling image "nginx:does-not-exist-9999"
Warning   Failed      16s (x2 over 44s) kubelet            Error: ImagePullBackOff
```
- KLJUČNI redak: `Failed to pull image ... manifest ... not found` = registry nema taj image/tag → uzrok crno na bijelo
- `Scheduled` = raspoređivanje prošlo (problem NIJE u schedulingu)
- `Pulling (x3 over 46s)` = kubelet pokušao 3× pa ušao u backoff
- poanta: `Events` IMENUJE uzrok, ne mora se nagađati

### Mini-mapa: najčešći `STATUS` → gdje gledati uzrok (LO5 error-katalog)

| STATUS | Značenje | Glavni trag |
|---|---|---|
| `ImagePullBackOff` / `ErrImagePull` | ne može povući image | `Events`: `Failed to pull ... not found` (krivo ime/tag, privatni registry bez Secreta) |
| `CrashLoopBackOff` | kontejner se digne pa stalno pada | `kubectl logs --previous` (razlog u zadnjem uzdahu starog kontejnera) |
| `OOMKilled` | premašen memorijski limit | `describe` → `State/Last State: Reason: OOMKilled` |
| `Pending` | ne može se rasporediti | `Events`: `FailedScheduling` (nema resursa / nodeSelector ne pogađa čvor) |
| `CreateContainerConfigError` | fali ConfigMap/Secret na koji se referira | `Events`: `Error: configmap "..." not found` |
| `ContainerCreating` (zapne) | volume/mount problem | `Events`: `FailedMount` / `FailedAttachVolume` |

### Gotče

- `STATUS` (iz `get`) = simptom; `Events` (iz `describe`) = uzrok → uvijek oba
- `ErrImagePull` (svjež pad) vs `ImagePullBackOff` (kubelet u backoff čekanju) — isti korijenski problem, različita faza
- `manifest ... not found` = registry nema image/tag (tipfeler u imenu ili nedostupan tag)
- `Events` se s vremenom GUBE (kratko se čuvaju) → kod kvara pogledati odmah
- goli Pod s krivim imageom NE prelazi sam u Running (image stvarno ne postoji) → počistiti ručno

### Provjera

```bash
kubectl run broken --image=nginx:does-not-exist-9999 --restart=Never   # izazovi kvar
kubectl get pod broken                                                  # STATUS: ImagePullBackOff
kubectl describe pod broken                                             # Events: Failed to pull ... not found
kubectl delete pod broken                                               # čišćenje
```

---

## Zadatak 48 — playbook „zašto pod ne odgovara" (sustavna dijagnostika pod→servis)

**Cilj:** sustavni redoslijed koraka koji povezuje cijelu Fazu 5 (i Fazu 4) u jedan dijagnostički tijek. Ići SLOJEVITO od Poda prema van: postoji li Pod → je li zdrav → vidi li ga servis → razrješava li se ime → stiže li promet. Svaki sloj ima svoju naredbu i tipičan kvar.

**Načelo:** kvar se traži odozdo prema gore. Prvi sloj koji „nije zelen" pokazuje gdje je problem; nema smisla provjeravati DNS ako Pod nije ni Running.

### Dijagnostički lanac (5 slojeva)

```
Sloj 1: Postoji li Pod i je li raspoređen?
   kubectl get pods -l <labela> -o wide
   ZDRAVO: STATUS Running · KVAR: Pending (FailedScheduling), ImagePullBackOff, ContainerCreating zapne
        ↓
Sloj 2: Je li Pod ZDRAV (ready)?
   kubectl get pods ...   (stupac READY)   +   kubectl describe pod <ime>   (Events)
   ZDRAVO: READY 1/1 · KVAR: 0/1 (readiness pada), CrashLoopBackOff (RESTARTS raste)
        ↓
Sloj 3: Vidi li SERVIS taj Pod?
   kubectl get endpoints <svc>
   ZDRAVO: ENDPOINTS = PodIP:port · KVAR: <none> (selektor ne pogađa labelu ILI Pod nije ready)
        ↓
Sloj 4: Razrješava li se IME servisa?
   kubectl run tmp --rm -it --image=busybox --restart=Never -- nslookup <svc>
   ZDRAVO: vrati ClusterIP · KVAR: bad address (krivo ime / krivi namespace → treba FQDN)
        ↓
Sloj 5: Stiže li PROMET kroz cijeli lanac?
   kubectl run tmp --rm -it --image=busybox --restart=Never -- wget -qO- --timeout=3 http://<svc>
   ZDRAVO: vrati app odgovor (HTML) · KVAR: connection refused (krivi targetPort / app ne sluša)
```

### Provjereno na zdravom sustavu (sva 3 sloja zelena)

```bash
kubectl get pods -l app=probe-demo -o wide
kubectl get endpoints probe-demo-svc
kubectl run tmp --rm -it --image=busybox --restart=Never -- wget -qO- --timeout=3 http://probe-demo-svc
```

Provjereni ispis:
```
# Sloj 1+2
NAME                          READY   STATUS    RESTARTS        AGE   IP            NODE
probe-demo-7b7b5b966f-7gbkr   1/1     Running   1 (34m ago)     43m   10.244.0.90   minikube

# Sloj 3
NAME             ENDPOINTS         AGE
probe-demo-svc   10.244.0.90:80    88m

# Sloj 4+5
<!DOCTYPE html> ... <h1>Welcome to nginx!</h1> ... </html>
```
- KLJUČNO uparivanje: Pod-IP iz `get pods` (`.90`) = IP u Endpoints (`.90:80`) → servis gađa pravi Pod
- kad se ta dva poklapaju I wget prolazi → zdravo na svim slojevima; kad NE → sloj neslaganja je mjesto kvara

### Tablica: simptom → sloj → naredba → čest uzrok

| Simptom | Sloj | Naredba | Čest uzrok |
|---|---|---|---|
| Pod nije ni gore | 1 | `get pods -o wide`, `describe` | `Pending` (nema resursa/nodeSelector), `ImagePullBackOff` (krivi image) |
| Pod gore ali `0/1` | 2 | `describe` (Events), `logs` | readiness pada, app ne starta |
| Pod stalno restarta | 2 | `logs --previous` | `CrashLoopBackOff` (app crkne pri startu) |
| Servis ne šalje promet | 3 | `get endpoints` | `<none>` = selektor ne pogađa labelu / Pod nije ready |
| Ime se ne razrješava | 4 | `nslookup` iz Poda | krivo ime / drugi namespace (treba FQDN) |
| Spaja se pa odbije | 5 | `wget`/`nc` iz Poda | krivi `targetPort`, app ne sluša na tom portu |

### Gotče

- dijagnoza ide ODOZDO (Pod) PREMA GORE (promet); prvi „ne-zeleni" sloj = mjesto kvara
- uparivanje Pod-IP (`get pods -o wide`) s Endpoints adresom = brza provjera da servis gađa pravi Pod
- `describe` → `Events` je univerzalni drugi korak na slojevima 1–2 (imenuje uzrok)
- `<none>` u Endpoints i `0/1` u READY su povezani: Pod koji nije ready ispada iz Endpoints (vidi zad. 50)
- `bad address` (sloj 4) vs `connection refused` (sloj 5): prvo je DNS, drugo je veza/port — različiti slojevi

### Provjera

```bash
kubectl get pods -l app=probe-demo -o wide                                                          # sloj 1+2
kubectl get endpoints probe-demo-svc                                                                # sloj 3
kubectl run tmp --rm -it --image=busybox --restart=Never -- nslookup probe-demo-svc                 # sloj 4
kubectl run tmp --rm -it --image=busybox --restart=Never -- wget -qO- --timeout=3 http://probe-demo-svc  # sloj 5
```

---

## Faza 5 — završena

Faza 5 pokriva LO4 zadatke **48** (playbook „zašto pod ne odgovara"), **50** (HTTP livenessProbe), **51** (tcpSocket readiness na redis), **52** (startup budget), **53** (default bez proba) — svaki sada pod svojim ISPRAVNIM brojem. Plus dijagnostički alati (`exec`/`logs`/`describe`) koji NISU numerirani LO4 zadaci (vežu se na LO5).

⚠️ Redoslijed u datoteci nije strogo numerički: u glavnom dijelu su zad. 50 pa zad. 48; zad. 51/52/53 su u sekciji „Faza 5 — dopune" niže. Ali svaki NASLOV odgovara svom PDF broju.

Pokriveno: liveness/readiness/startup probe (readiness→Endpoints, liveness→restart, dokaz na živom kvaru) · `kubectl exec` (jednokratno + interaktivni shell, Pod vs kontejner, `-c`) · `kubectl logs` (`--tail`, `-f`, `--previous` za CrashLoop) · `describe`+`Events` dijagnostika (ImagePullBackOff uživo + error-katalog) · slojeviti playbook „zašto pod ne odgovara".

**Čišćenje Faze 5** (pokrenuti prije Faze 6):
```bash
kubectl delete deployment probe-demo
kubectl delete service probe-demo-svc
```

---

## Faza 5 — dopune (zad. 51, 52, 53)

> PDF zadaci 51 (tcpSocket readiness na redis), 52 (startup budget) i 53 (default bez ijedne probe) — svaki pod svojim PRAVIM brojem. (Oznake u Fazi 5 usklađene: glavni dio ima zad. 50 = HTTP liveness i zad. 48 = playbook; alati `exec`/`logs`/`describe` nisu numerirani LO4 zadaci.)

## Zadatak 53 — default ponašanje BEZ ijedne probe

**Cilj:** objasniti što Kubernetes radi s containerom kad NIJEDNA proba nije definirana.

**Koncept:** bez proba jedini signal zdravlja je „glavni proces containera radi". Tri posljedice:

- **Nema `readinessProbe`** → container je Ready ČIM se proces pokrene. Odmah ulazi u Endpoints servisa → prima promet i prije nego je aplikacija unutra stvarno spremna. Rizik: promet udari u app koja se još diže (greške dok ne dovrši inicijalizaciju).
- **Nema `livenessProbe`** → container je „živ" dok mu glavni proces traje. Kubelet ga restarta SAMO ako proces izađe/padne (po `restartPolicy`, default `Always` za Deployment). Aplikacija koja se zaglavi/deadlocka ali drži proces živim NEĆE biti restartana — kubelet ne zna da je zapela.
- **Nema `startupProbe`** → ako liveness/readiness postoje, kreću odmah (nakon `initialDelaySeconds`); za spore startere preagresivan liveness može ubiti app prije nego se digne.

**Jedna rečenica:** bez proba = „proces postoji" je jedini znak zdravlja → Ready odmah, restart samo na izlaz procesa.

**Gotča:** zaglavljena (hung) aplikacija sa živim procesom je nevidljiva bez livenessProbe — to je glavni razlog zašto liveness postoji. Restart na crash je default; restart na „više ne odgovara" traži livenessProbe.

## Zadatak 52 — startup probe budget za ~2-min boot (konkretne brojke)

**Cilj:** konfigurirati startupProbe za aplikaciju koja se diže ~2 min (120 s) i obrazložiti odabrane brojke.

**Koncept:** startupProbe drži liveness/readiness „na čekanju" dok se app ne digne. Vrijeme koje će čekati prije nego odustane (i kubelet ubije container):

**budžet = `failureThreshold × periodSeconds`**

Budžet mora biti VEĆI od najgoreg vremena boota, s rezervom.

**Primjer (budžet 180 s za boot od ~120 s):**
```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 10
  failureThreshold: 18      # 18 × 10s = 180s
```

**Obrazloženje brojki** (zad. traži „justify the numbers"):
- `18 × 10 = 180 s` > 120 s boota → ~50% rezerve; ne reže se na tijesno (inače slučajna ubijanja zdravog appa na spore dane).
- `periodSeconds: 10` → čim app stvarno odgovori, „started" se prizna unutar 10 s. Manji period = brže priznanje + više provjera.
- `failureThreshold: 18` = strpljenje za spori boot; veći broj tolerira sporiji start, ali odgađa restart stvarno pokvarene aplikacije.
- Čim startup JEDNOM uspije → gasi se; preuzimaju liveness/readiness.

**Gotča:** startupProbe i liveness/readiness se NE preklapaju u vremenu — dok startup traje, druge dvije miruju. Budžet prekratak = kubelet ubije app koja se još normalno diže (lažni kvar).

**Provjera:**
```bash
kubectl describe pod <pod> | grep -A2 Startup     # prikaže konfiguriranu startup probu
```

## Zadatak 51 — tcpSocket readiness na redis:6379

**Cilj:** dodati readiness probu tipa `tcpSocket` redis containeru i potvrditi je kroz `describe`.

**Koncept — tcpSocket vs httpGet:** `tcpSocket` proba samo provjeri da se TCP veza na zadani port OTVORI; NE šalje HTTP zahtjev. Idealno za servise koji ne govore HTTP (baze: redis, postgres…). Otvorena veza = proces sluša = Ready; odbijena = Not Ready (Pod ispada iz Endpoints).

**Naredbe:**

Deployment (imperativno):
```bash
kubectl create deployment redis --image=redis:7
```
- `--image=redis:7` — slika redis, verzija 7; `create` automatski doda labelu `app=redis`, container imenom `redis`, 1 replika

Proba se dodaje preko `kubectl patch` (NE `edit`/YAML — patch je jedna linija bez uvlaka, ne može „ispasti blok" kao pri ručnom prepisivanju):
```bash
kubectl patch deployment redis --type=strategic -p '{"spec":{"template":{"spec":{"containers":[{"name":"redis","readinessProbe":{"tcpSocket":{"port":6379},"periodSeconds":5}}]}}}}'
```
- `patch deployment redis` — mijenja postojeći Deployment na licu mjesta
- `--type=strategic` — strateški merge: lista `containers` se spaja po ključu `name`; container `redis` dobije probu, ostatak (slika…) ostaje. Default tip patcha.
- `-p '...'` — payload (zakrpa) u JSON-u; putanja prati strukturu: `spec.template.spec.containers[redis].readinessProbe`
- `tcpSocket.port: 6379` — otvori TCP vezu na 6379; uspjeh = Ready
- `periodSeconds: 5` — provjera svakih 5 s (otisak da je naš patch sjeo)

Patch mijenja Pod template → rolling update → **novi Pod** (novi pod-template-hash).

**VM ispis (anotiran):**
```
# prije patcha
redis-7957d7fc9c-szh4m   1/1   Running   0   12s
# nakon patcha — NOVI hash (84bf6884c6) = patch promijenio template = novi RS/Pod
redis-84bf6884c6-47t7s   1/1   Running   0   5m4s
```
READY `1/1` = tcpSocket proba prolazi (redis sluša na 6379).

Dokaz da je proba u Podu:
```bash
kubectl describe pod -l app=redis | grep Readiness
```
```
Readiness:  tcp-socket :6379 delay=0s timeout=1s period=5s #success=1 #failure=3
```
- `tcp-socket :6379` — tip tcpSocket na portu 6379 ✓ (to smo tražili)
- `delay=0s` = `initialDelaySeconds` (nije postavljen → 0) · `timeout=1s` default · `period=5s` = NAŠ (otisak patcha) · `#success=1`/`#failure=3` = defaulti

**Gotča:** patch s `--type=strategic` na listu containera traži točan `name` da zna KOJI container mijenja; bez `name: redis` patch bi DODAO novi container umjesto da izmijeni postojeći.

**Provjera:**
```bash
kubectl get pods -l app=redis                              # READY 1/1
kubectl describe pod -l app=redis | grep Readiness         # tcp-socket :6379 ... period=5s
```

---

## Faza 6 — Config/Secret/Volume

ConfigMap (neztajna konfiguracija) i Secret (tajne) odvajaju konfiguraciju od slike. Razlika: ConfigMap = čisti tekst u etcd; Secret = base64 (NIJE enkripcija). Oboje se u Pod ubacuje kao **env varijable** ILI **montirano kao datoteke** (volume).

### Temelj — ConfigMap imperativno (NIJE numerirani zadatak; podloga za 24/25)

```bash
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod
kubectl get configmap app-config -o yaml
```
- `--from-literal=KEY=value` — par ključ=vrijednost ravno iz komandne linije; ponavlja se za svaki ključ
- u `-o yaml` ispisu podaci su u `data:` kao ČISTI TEKST (usporedi sa Secret zad. 31, gdje su base64)

VM ispis (`data:` dio):
```
data:
  APP_COLOR: blue
  APP_MODE: prod
```

## Zadatak 24 — montiraj ConfigMap kao volumen (svaki ključ → datoteka)

**Cilj:** montirati ConfigMap kao volumen tako da svaki ključ postane zasebna datoteka, pa provjeriti sadržaj iznutra.

**Koncept:** ConfigMap se „prikvači" na direktorij u containeru. Svaki **ključ → ime datoteke**, **vrijednost → sadržaj datoteke**. Npr. ključ `APP_COLOR=blue` → datoteka `/etc/config/APP_COLOR` u kojoj piše `blue`.

**Manifest** (`~/devops/lo4/cm-volume.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-volume
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config
```
- `command: ["sleep", "3600"]` — drži busybox živim 1 h (inače odmah izađe → CrashLoop)
- `volumeMounts.mountPath: /etc/config` — KAMO se volumen vidi unutar containera
- `volumes.configMap.name: app-config` — sadržaj volumena = naš ConfigMap
- **ljepilo:** `name: config` u `volumeMounts` = `name: config` u `volumes` (spaja mount s izvorom)
- uvlake (često mjesto greške pri ručnom prepisivanju): `volumeMounts:` poravnat s `image:`; `volumes:` poravnat s `containers:` (ravno pod `spec:`)

**Naredbe:**
```bash
kubectl apply -f ~/devops/lo4/cm-volume.yaml
kubectl exec cm-volume -- ls /etc/config
kubectl exec cm-volume -- cat /etc/config/APP_COLOR
kubectl exec cm-volume -- cat /etc/config/APP_MODE
```

**VM ispis (anotiran):**
```
# ls /etc/config — svaki ključ je zasebna datoteka
APP_COLOR
APP_MODE
# cat datoteka — sadržaj = vrijednost ključa
blue
prod
```

**Gotča:** sadržaj datoteke iz ConfigMapa NEMA završni newline (`blue` se zalijepi za prompt: `blue[student@...]`) — normalno, ConfigMap ne dodaje `\n`.

**Provjera:**
```bash
kubectl exec cm-volume -- ls /etc/config                  # APP_COLOR  APP_MODE
kubectl exec cm-volume -- cat /etc/config/APP_COLOR       # blue
```

## Zadatak 25 — montiraj JEDAN ključ na točnu putanju (`subPath`)

**Cilj:** montirati samo jedan ključ ConfigMapa kao jednu datoteku na zadanu putanju, ne prekrivajući ostatak ciljnog direktorija.

**Koncept:** obični mount (zad. 24) zauzme CIJELI `mountPath` direktorij (samo datoteke iz ConfigMapa, ostalo sakriveno). `subPath` montira **jedan ključ kao jednu datoteku** → ubacuje konfig pored postojećih datoteka u direktoriju, bez prekrivanja.

**Caveat (zad. izrijekom traži):** datoteka montirana preko `subPath` se NE osvježava kad se ConfigMap promijeni (obične mount-datoteke se s vremenom osvježe; subPath ne → treba restart Poda).

**Manifest** (`~/devops/lo4/cm-subpath.yaml`) — razlika od zad. 24 su DVA retka:
```yaml
    volumeMounts:
    - name: config
      mountPath: /etc/app/color.conf     # putanja do DATOTEKE, ne direktorija
      subPath: APP_COLOR                  # uzmi SAMO ovaj ključ
```
(ostatak — `volumes`, ljepilo `name: config` — isto kao zad. 24)
- `mountPath: .../color.conf` — pokazuje na datoteku, ne direktorij
- `subPath: APP_COLOR` — iz ConfigMapa uzmi samo taj ključ na tu putanju; ostali se ignoriraju
- uvlaka: `subPath:` poravnat s `name:` i `mountPath:` (svi pod istim `- ` u `volumeMounts`)

**Naredbe + VM ispis (anotiran):**
```bash
kubectl apply -f ~/devops/lo4/cm-subpath.yaml
kubectl exec cm-subpath -- cat /etc/app/color.conf       # blue
kubectl exec cm-subpath -- ls /etc/app                    # SAMO color.conf
```
```
blue
color.conf
```
- `cat .../color.conf` → `blue` (vrijednost ključa APP_COLOR kao baš ta datoteka)
- `ls /etc/app` → samo `color.conf`; `APP_MODE` se NIJE pojavio → dokaz da je ušao SAMO jedan ključ (poanta subPath-a)

**Trik (iskorišten uživo):** umjesto prepisivanja cijelog YAML-a od nule, kopirati provjereni `cm-volume.yaml` → `cm-subpath.yaml` i promijeniti samo 2 retka. Uvlake ostanu netaknute. Vrijedi za cijelu Fazu 6.

**Provjera:**
```bash
kubectl exec cm-subpath -- ls /etc/app                    # samo color.conf
```

## Zadatak 31 — generički Secret iz literala + dokaz base64 (NIJE enkripcija)

**Cilj:** napraviti Opaque Secret s DB user/pass i pokazati da su vrijednosti base64-kodirane, ne šifrirane.

**Koncept:** Secret = ConfigMap za tajne. Vrijednosti su **base64**, što NIJE enkripcija — svatko s `get secret` pravom dekodira u sekundi. Zaštita = RBAC (tko smije čitati), ne sam zapis. (Veže se na LO1: `auth.json` je također base64, ne šifriran.)

**Naredbe:**
```bash
kubectl create secret generic db-cred --from-literal=DB_USER=admin --from-literal=DB_PASS=s3cret
kubectl get secret db-cred -o yaml
```
- `generic` — generički tip; u YAML-u `type: Opaque`. Posebni tipovi: `tls` (zad. 33), `docker-registry` (zad. 36).
- `--from-literal=KEY=value` — par iz komandne linije

Dekodiranje (dokaz base64):
```bash
kubectl get secret db-cred -o jsonpath='{.data.DB_PASS}' | base64 -d ; echo
```
- `-o jsonpath='{.data.DB_PASS}'` — izvuci samo base64 vrijednost ključa DB_PASS; jsonpath izraz MORA biti u `'{...}'` (inače `echo` ispiše doslovan tekst)
- `| base64 -d` — dekodiraj (`d` = decode) → original
- `; echo` — doda prijelom reda (dekodirana vrijednost nema `\n`)

**VM ispis (anotiran):**
```
# get secret -o yaml — vrijednosti su base64, type Opaque
data:
  DB_PASS: czNjcmV0       # base64, NE čitljivo
  DB_USER: YWRtaW4=
type: Opaque
# base64 -d — vraća original
s3cret
```
Usporedba: ConfigMap (zad. 24) je imao golo `blue`; Secret ima `czNjcmV0` koji se trivijalno dekodira u `s3cret`.

**Gotča:** base64 ≠ enkripcija. `kubectl get secret ... -o jsonpath | base64 -d` vrati lozinku odmah. Secret štiti od slučajnog pogleda (u `-o yaml` ispisu nije plain tekst), NE od nekoga tko ima pravo čitanja.

**Provjera:**
```bash
kubectl get secret db-cred -o jsonpath='{.data.DB_USER}' | base64 -d ; echo    # admin
```

## Zadatak 35 — cijeli Secret kao env varijable (`envFrom` + `secretRef`)

**Cilj:** učitati SVE ključeve Secreta kao env varijable odjednom i dokazati ih u containeru.

**Koncept:** `env:` nabraja varijable jednu po jednu; **`envFrom`** učita sve ključeve izvora odjednom. Unutar containera vrijednosti su DEKODIRANE (plain), app ih čita kao obične varijable. (`secretRef` za Secret; `configMapRef` za ConfigMap — isti mehanizam.)

**Manifest** (`~/devops/lo4/secret-env.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    envFrom:
    - secretRef:
        name: db-cred
```
- `envFrom` → `- secretRef: name: db-cred` — svaki ključ Secreta postane env var istog imena
- uvlake: `envFrom:` poravnat s `command:`; `secretRef:` pod `- `; `name:` još uvučen pod `secretRef:`

**Naredbe:**
```bash
kubectl apply -f ~/devops/lo4/secret-env.yaml
kubectl exec secret-env -- env | grep DB_
```

**VM ispis:**
```
DB_PASS=s3cret
DB_USER=admin
```
Oba ključa odjednom, dekodirana (`s3cret`, ne base64).

**Gotče (obje uhvaćene uživo):**
- `command: ["sleep","3600"]` MORA biti — bez njega busybox odmah izađe → CrashLoop, nema `exec`. (Redak je „ispao" pri tipkanju → uvijek `cat` provjera prije `apply`.)
- `kubectl exec ... -- env` traži RAZMAK iza `--` (granica kubectl↔naredba). `--env` (spojeno) = „unknown flag".

**Provjera:**
```bash
kubectl exec secret-env -- env | grep DB_     # DB_USER=admin, DB_PASS=s3cret
```

## Zadatak 26 + 34 — Secret kao volumen (dekodirane datoteke, restriktivne dozvole, trade-off vs env)

**Cilj:** montirati Secret kao volumen → svaki ključ datoteka s DEKODIRANom vrijednošću i restriktivnim dozvolama (26); usporediti sigurnost s env varijablama (34).

**Koncept:** kao ConfigMap volumen (zad. 24), ali izvor je Secret. Datoteke sadrže DEKODIRANE vrijednosti (ne base64). `defaultMode` postavlja dozvole datoteka.

**Manifest** (`~/devops/lo4/secret-vol.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    volumeMounts:
    - name: creds
      mountPath: /etc/secret
      readOnly: true
  volumes:
  - name: creds
    secret:
      secretName: db-cred
      defaultMode: 0400
```
- ⚠️ za Secret volumen ide **`secret.secretName:`** (NE `name:` kao kod ConfigMapa `configMap.name:`)
- `defaultMode: 0400` — datoteke dobiju `r--------` (samo vlasnik čita)
- `readOnly: true` — mount samo-za-čitanje
- ljepilo: `name: creds` mora biti isto u `volumeMounts` i `volumes`

**Naredbe + VM ispis (anotiran):**
```bash
kubectl apply -f ~/devops/lo4/secret-vol.yaml
kubectl exec secret-vol -- cat /etc/secret/DB_PASS ; echo        # s3cret (dekodirano)
kubectl exec secret-vol -- ls -l -L /etc/secret                   # dozvole stvarnih datoteka
```
```
s3cret
total 8
-r--------    1 root  root  6 Jun 22 18:05 DB_PASS     # 0400, 6 bajta = "s3cret"
-r--------    1 root  root  5 Jun 22 18:05 DB_USER     # 0400, 5 bajta = "admin"
```
- `cat` → `s3cret` = vrijednost DEKODIRANA u datoteci (ne base64) → dokaz zad. 26
- `-r--------` = 0400 restriktivno → dokaz zad. 26; veličina (6/5 bajta) = duljina dekodirane vrijednosti

**Zad. 34 — trade-off Secret kao env vs kao volumen:**

| | env (zad. 35) | volumen (zad. 26) |
|---|---|---|
| curenje | u okruženju procesa → nasljeđuju child-procesi, lako u logove/crash-dump | samo procesi koji OTVORE datoteku; nije u environment |
| dozvole | nema kontrole | `defaultMode` (npr. 0400) |
| pohrana | memorija procesa | tmpfs (RAM, ne disk čvora) |
| osvježavanje | fiksno na startu → promjena Secreta traži restart Poda | auto-osvježi se (osim subPath) |
| jednostavnost | app čita env var (lakše) | app čita datoteku (malo više koda) |

Zaključak: env = praktičnije ali curljivije; volumen = sigurnije (tmpfs, dozvole, auto-update). Za prave tajne → volumen.

**Gotče:**
- Secret/ConfigMap datoteke su SIMLINKOVI na skrivenu `..data/` (Kubernetes tako radi atomske izmjene) → `ls -l` pokaže linkove (`lrwxrwxrwx`, simlinkovi nemaju vlastite dozvole); `ls -lL` „prati" link i pokaže prave dozvole (`0400`).
- `-l` (malo L = long/dozvole) vs `-1` (broj jedan = jedan unos po retku) izgledaju gotovo isto u monospace fontu — kad razlika nije jasna, razdvojiti: `ls -l -L`. Veliko `-L` = dereference simlinka.

**Provjera:**
```bash
kubectl exec secret-vol -- cat /etc/secret/DB_USER ; echo       # admin
kubectl exec secret-vol -- ls -l -L /etc/secret                  # -r-------- (0400)
```

## Zadatak 37 — Secret s više ključeva, montiraj SAMO jedan (`items`)

**Cilj:** iz Secreta s više ključeva montirati samo jedan odabrani ključ (uz mogućnost preimenovanja datoteke).

**Koncept:** bez `items` mount projektira SVE ključeve Secreta (zad. 26 → `DB_USER` i `DB_PASS`). Polje **`items`** bira točno koje ključeve montirati; `path` zadaje ime datoteke (može se razlikovati od imena ključa = preimenovanje).

**Manifest** (`~/devops/lo4/secret-one.yaml`):
```yaml
  volumes:
  - name: creds
    secret:
      secretName: db-cred
      items:
      - key: DB_PASS        # KOJI ključ uzeti (postojeći, iz zad. 31)
        path: password       # KAKO nazvati datoteku (slobodan izbor)
```
- `key: DB_PASS` — ključ koji POSTOJI u Secretu (`--from-literal=DB_PASS=s3cret` iz zad. 31)
- `path: password` — ime datoteke u mountu; mi biramo. Bez `path` datoteka bi se zvala kao ključ.
- uvlaka: `items:` poravnat sa `secretName:`; `- key:` pod `items`; `path:` poravnat s `key:`

**Naredbe + VM ispis (anotiran):**
```bash
kubectl apply -f ~/devops/lo4/secret-one.yaml
kubectl exec secret-one -- ls /etc/secret               # samo: password (NE DB_USER)
kubectl exec secret-one -- cat /etc/secret/password ; echo   # s3cret
```
```
password
s3cret
```
- `ls` → samo `password`; `DB_USER` se NE pojavi → selektivni mount (poanta `items`)
- `cat password` → `s3cret` = vrijednost dekodirana, datoteka preimenovana (ključ `DB_PASS` → datoteka `password`)

**Gotče (obje uhvaćene uživo, nevezane za k8s ali tipične):**
- `~/devops` (tilda + `/`) = domaći direktorij `/home/student/devops`. `~devops` (BEZ `/`) = home korisnika `devops`; takav ne postoji → shell ostavi doslovno `~devops` kao relativno ime mape → gedit otvori praznu datoteku na krivom mjestu. Pravilo: `~/` UVIJEK s kosom crtom.
- ekstenzija `.yamlk` (typo) → `apply` padne s „path does not exist" — pazi na zalutalo slovo na kraju.

**Provjera:**
```bash
kubectl exec secret-one -- ls /etc/secret               # samo password
```

## Zadatak 32 + 33 — Secret iz cert/key datoteka: generic (`--from-file`) vs tls tip

**Cilj:** od para certifikat+ključ napraviti (32) generički Secret iz datoteka i identificirati ključeve, te (33) tls-tipizirani Secret i objasniti gdje se troši.

**Priprema — self-signed cert+key:**
```bash
cd ~/devops/lo4
openssl req -x509 -newkey rsa:2048 -nodes -keyout tls.key -out tls.crt -days 365 -subj "/CN=demo"
```
- `-x509` — gotov self-signed certifikat (ne CSR) · `-newkey rsa:2048` — usput novi RSA ključ 2048 bita · `-nodes` — ne zaključavaj ključ lozinkom · `-keyout`/`-out` — kamo ključ/cert · `-days 365` — trajanje · `-subj "/CN=demo"` — popuni subjekt bez interaktivnih pitanja
- openssl sam da ključu dozvole `-rw-------` (privatno), certifikatu `-rw-r--r--`

**Zad. 32 — generički Secret iz datoteka (`--from-file`):**
```bash
kubectl create secret generic tls-files --from-file=tls.crt --from-file=tls.key
kubectl describe secret tls-files
```
```
Type:  Opaque
Data
====
tls.key:  1704 bytes
tls.crt:  1099 bytes
```
- `--from-file=tls.crt` — ključ u Secretu = IME DATOTEKE (`tls.crt`), vrijednost = sadržaj (base64)
- ključevi su `tls.crt`/`tls.key` jer su tako nazvane datoteke → kod `--from-file` ime ključa dolazi iz imena datoteke (za razliku od `--from-literal` gdje ga sam tipkaš)

**Zad. 33 — tls-tipizirani Secret (`kubectl create secret tls`):**
```bash
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
kubectl get secret tls-secret -o jsonpath='{.type}' ; echo      # kubernetes.io/tls
kubectl describe secret tls-secret
```
```
Type:  kubernetes.io/tls
Data
====
tls.crt:  1099 bytes
tls.key:  1704 bytes
```
- `create secret tls` — poseban podtip; `--cert`/`--key` su obavezni
- tip je **`kubernetes.io/tls`** (ne Opaque), ključevi FIKSNO `tls.crt`/`tls.key`, a k8s usput provjeri da cert i ključ čine valjan par

**Razlika 32 vs 33:**

| | generic `--from-file` (32) | tls `--cert/--key` (33) |
|---|---|---|
| tip | `Opaque` | `kubernetes.io/tls` |
| imena ključeva | bilo koje (= ime datoteke) | FIKSNO `tls.crt` + `tls.key` |
| validacija | nema | provjerava da cert/ključ čine par |

**Gdje se tls Secret troši:** najčešće **Ingress** za HTTPS terminaciju (TLS na rubu klastera); pošto tip jamči strukturu, potrošač zna gdje su cert i ključ. (Ingress nije među 53 zadatka — vidi napomenu na vrhu skripte.)

**Gotča:** neuravnotežen navodnik u `-o jsonpath='{.type}'` (fali zatvarajući `'`) → shell padne u nastavak (`>` prompt, čeka kraj niza) → `Ctrl+C` pa ponovi cijelu naredbu.

**Provjera:**
```bash
kubectl get secret tls-secret -o jsonpath='{.type}' ; echo      # kubernetes.io/tls
```

## Zadatak 23 — izlistaj StorageClasse, PV-ove i PVC-ove

**Cilj:** vidjeti pejzaž trajne pohrane klastera.

**Pojmovi:**
- **StorageClass** — recept za dinamičko stvaranje diska (PV) na zahtjev. minikube ima default `standard`.
- **PV (PersistentVolume)** — stvarni komad diska (resurs klastera).
- **PVC (PersistentVolumeClaim)** — zahtjev Poda za diskom; veže se na PV.
- lanac: Pod → PVC (zahtjev) → PV (disk); StorageClass po potrebi PV proizvede sam (dinamičko provisioniranje).

**Naredbe:**
```bash
kubectl get storageclass     # kratica: sc
kubectl get pv
kubectl get pvc
```

**VM ispis (anotiran):**
```
# storageclass
NAME                 PROVISIONER                RECLAIMPOLICY  VOLUMEBINDINGMODE  ALLOWVOLUMEEXPANSION
standard (default)   k8s.io/minikube-hostpath   Delete         Immediate          false
# pv
No resources found
# pvc
No resources found in default namespace.
```
- `standard (default)` — default StorageClass (PVC bez navedene klase dobije ovu)
- `PROVISIONER k8s.io/minikube-hostpath` — tko stvara disk (minikube hostpath)
- `RECLAIMPOLICY Delete` — kad se PVC obriše, PV (i podaci) se BRIŠU (vs `Retain` = ostaju)
- `VOLUMEBINDINGMODE Immediate` — PV se veže čim PVC nastane (vs `WaitForFirstConsumer` = čeka da ga Pod stvarno koristi)
- `ALLOWVOLUMEEXPANSION false` — PVC se ne može naknadno povećati
- PV i PVC prazni → nije greška: minikube ima spreman mehanizam, ali disk nastaje tek na zahtjev (PVC)

**Gotča:** „No resources found" = PRAZNO, ne greška (ista poruka kao label selektor bez pogotka).

**Provjera:**
```bash
kubectl get sc                # standard (default)
```

## Zadatak 29 — `readOnly: true` na mountu (pisanje odbijeno)

**Cilj:** dokazati da je mount s `readOnly: true` samo-za-čitanje (pisanje pada).

**Koncept:** `readOnly: true` na `volumeMount` čini taj mount read-only za container → pisanje pada s „Read-only file system". Koristi se kad container smije samo čitati volumen (npr. konfiguraciju), nikad mijenjati. Koristimo `emptyDir` (inače zapisiv) da izoliramo efekt zastavice.

**Manifest** (`~/devops/lo4/ro-test.yaml`):
```yaml
    volumeMounts:
    - name: data
      mountPath: /data
      readOnly: true
  volumes:
  - name: data
    emptyDir: {}
```
- `readOnly: true` — mount postaje read-only za container
- `emptyDir: {}` — prazan, inače zapisiv disk; `readOnly` ga zaključa

**Naredbe + VM ispis:**
```bash
kubectl apply -f ~/devops/lo4/ro-test.yaml
kubectl exec ro-test -- touch /data/test.txt
```
```
touch: /data/test.txt: Read-only file system
command terminated with exit code 1
```
- `touch` padne → mount je read-only, pisanje fizički nemoguće
- `exit code 1` = naredba neuspjela (to je DOKAZ, ne problem)

**Kontrast:** `readOnly` → pisanje ODMAH odbijeno; `sizeLimit` (zad. 30) → pisanje prođe pa kubelet naknadno izbaci Pod.

**Provjera:**
```bash
kubectl exec ro-test -- touch /data/x   # Read-only file system, exit code 1
```

## Zadatak 30 — `sizeLimit` na `emptyDir` (prekoračenje → izbacivanje Poda)

**Cilj:** ograničiti veličinu emptyDir i pokazati što se dogodi kad container prekorači granicu.

**Koncept:** `sizeLimit` je MEKA granica — NE blokira pisanje u letu (za razliku od tvrde kvote). Pisanje prođe; kubelet PERIODIČKI (otprilike svakih 10 s) provjerava potrošnju i kad je prekoračena → IZBACI Pod (eviction). Goli Pod (bez kontrolera) nakon izbacivanja nestane (nema RS-a da ga vrati). Kontrast: `readOnly` (zad. 29) odbije pisanje ODMAH; `sizeLimit` pusti pisanje pa naknadno izbaci.

**Manifest** (`~/devops/lo4/size-test.yaml`):
```yaml
  volumes:
  - name: data
    emptyDir:
      sizeLimit: 1Mi
```
- `emptyDir` više nije `{}` nego ima dijete `sizeLimit: 1Mi` (dvije razine uvučeno pod `- name: data`)
- mount BEZ `readOnly` (želimo da pisanje prođe)

**Naredbe:**
```bash
kubectl apply -f ~/devops/lo4/size-test.yaml
kubectl exec size-test -- dd if=/dev/zero of=/data/big bs=1M count=5     # zapiši 5 MB (5x preko 1 Mi)
kubectl get pod size-test -w
```
- `dd if=/dev/zero of=/data/big bs=1M count=5` — 5 blokova × 1 MB = 5 MB; `if=/dev/zero` = beskonačne nule, `of=` = odredište, `bs`/`count` = veličina/broj blokova

**VM ispis (anotiran):**
```
# dd USPIJE — pisanje prolazi, granica se NE provjerava pri pisanju
5+0 records out
5242880 bytes (5.0MB) copied, 0.003683 seconds

# watch — ~11 s kasnije Pod pao (jedan ciklus kubeletove provjere ≈10 s)
size-test  1/1  Running  0  42s
size-test  0/1  Error    0  53s
```
- pisanje 5 MB prošlo iako je granica 1 Mi → `sizeLimit` je MEKA granica
- 42s → 53s = ~jedan kubeletov ciklus → primijetio prekoračenje → ugasio Pod
- nakon izbacivanja `kubectl describe pod size-test` → „pods not found" jer je goli Pod već počišćen (nema kontrolera da ga vrati)

**Gotče:**
- STATUS u `get pod` pokaže `Error` (posljedica na container), ne `Evicted` — pravi razlog (Usage of EmptyDir exceeds limit) je u statusu/događajima, ali goli Pod nestane prebrzo da ga `describe` uhvati. Da se razlog vidi crno-na-bijelo, treba Pod iza kontrolera (Deployment) ili vrlo brz `describe`.
- `minikube start -driver=podman` (JEDNA crtica) → minikube čita `-d river...` → zbrka „invalid argument er=podman". Treba `--driver` (DVIJE crtice).
- `kubectl get pods` koji ODGOVORI (makar „No resources found") = klaster RADI; samo „connection refused" znači da treba `minikube start`.

**Provjera (ponašanje; Pod se počisti sam):**
```bash
# nakon dd preko granice: kubectl get pod size-test -w → STATUS ode u Error/Evicted, pa Pod nestane
```

## Zadatak 2 + 36 — image-pull Secret (docker-registry) + `imagePullSecrets`

**Cilj:** napraviti docker-registry Secret za autentikaciju na Docker Hub (zad. 2: koji resurs) i referencirati ga preko `imagePullSecrets` u Podu (zad. 36).

**Koncept:**
- **zad. 2** pita „koji resurs za autenticirano povlačenje s Docker Huba (da se izbjegne anonimni pull rate limit)?" → odgovor: **docker-registry Secret** (tip `kubernetes.io/dockerconfigjson`).
- **zad. 36** = napravi ga + poveži preko `imagePullSecrets`.

**Naredba — napravi docker-registry Secret:**
```bash
kubectl create secret docker-registry dockerhub-cred --docker-server=https://index.docker.io/v1/ --docker-username=tomislavb16 --docker-password='<LOZINKA>' --docker-email=mail@example.com
kubectl get secret dockerhub-cred -o jsonpath='{.type}' ; echo
```
- `create secret docker-registry dockerhub-cred` — poseban podtip, imenom `dockerhub-cred`
- `--docker-server=https://index.docker.io/v1/` — Docker Hub default endpoint
- `--docker-username` / `--docker-password` / `--docker-email` — kredencijali (email je legacy polje, može bilo što)
- tip: **`kubernetes.io/dockerconfigjson`** (sadrži `.dockerconfigjson` ključ s base64 auth podacima — ista priča kao LO1 `auth.json`)

**Manifest — referenca preko `imagePullSecrets`** (`~/devops/lo4/pull-test.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pull-test
spec:
  imagePullSecrets:
  - name: dockerhub-cred
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
```
- `imagePullSecrets` — na razini `spec`, BRAT od `containers:` (oba 2 razmaka pod `spec:`) — glavna zamka
- `- name: dockerhub-cred` — koristi naš docker-registry Secret

**VM ispis:**
```
kubernetes.io/dockerconfigjson
pod/pull-test created
NAME        READY   STATUS    RESTARTS   AGE
pull-test   1/1     Running   0          7s
```
Pod radi → povlačenje busyboxa išlo autenticirano (kao `tomislavb16`), ne anonimno.

**Kad je `imagePullSecrets` obavezan (zad. 2 + 36):**
- **privatni registry** — bez pull Secreta slika se NE može povući (`ImagePullBackOff`) → OBAVEZAN
- **Docker Hub + javne slike** — radi i bez njega, ALI anoniman si i podliježeš pull rate limitu (~100 povlačenja / 6 h po IP-u); autentikacija diže limit → poanta zad. 2

**Gotča:** tip Secreta i drugi kratki ispisi znaju „odscrolati" izvan ekrana nakon više naredbi — provjeriti zasebnom `get ... -o jsonpath='{.type}'` naredbom.

**Provjera:**
```bash
kubectl get secret dockerhub-cred -o jsonpath='{.type}' ; echo    # kubernetes.io/dockerconfigjson
kubectl get pod pull-test                                          # Running
```

---

## Faza 6 — završena

Svih 14 zadataka Faze 6 (Config/Secret/Volume) odrađeno i verificirano: 2, 23, 24, 25, 26, 29, 30, 31, 32, 33, 34, 35, 36, 37.

Pokriveno: **ConfigMap** (kao volumen=datoteke, `subPath` jedan ključ) · **Secret** (base64≠enkripcija, `envFrom`, kao volumen+dozvole+trade-off vs env, `items` selektivno, iz datoteka `--from-file`, `tls` tip) · **trajna pohrana** (SC/PV/PVC listanje, `readOnly` mount, `emptyDir sizeLimit`→eviction) · **image-pull** (docker-registry Secret + `imagePullSecrets`).

**Napomena o stanju klastera:** Algebra je tijekom faze resetirala okruženje (obrisani Podovi + `web` Deployment; jednom i cijeli minikube profil → `/root` permission denied). Fix za profil (Faza 0): `minikube delete --all` → `export MINIKUBE_HOME=$HOME/.minikube` → `minikube start --driver=podman`. Klaster je trenutno čist; demo objekti iz F6 (osim `dockerhub-cred` + `pull-test`) nestali su u resetu — bezopasno.

**Čišćenje Faze 6** (ako klaster preživi):
```bash
kubectl delete pod pull-test
kubectl delete secret dockerhub-cred db-cred tls-files tls-secret 2>/dev/null
kubectl delete configmap app-config 2>/dev/null
```

---

## Faza 7 — Ostali kontroleri

StatefulSet (stabilni Podovi s identitetom), DaemonSet (jedan Pod po čvoru), Job (odradi jednom i završi), CronJob (Job po rasporedu). Za razliku od Deploymenta (zamjenjivi Podovi), ovi kontroleri imaju posebna ponašanja.

## Zadatak 16 — StatefulSet redis:7 (3 replike) + headless servis (stabilna imena + per-pod DNS)

**Cilj:** napraviti StatefulSet s headless servisom i pokazati stabilna imena Podova te per-pod DNS zapise.

**Koncept — StatefulSet vs Deployment:** Deployment tretira Podove kao zamjenjive (nasumična imena, bilo koji red). StatefulSet daje **identitet**:
- **stabilna imena** — `redis-0`, `redis-1`, `redis-2` (ordinalni indeks, ne hash)
- **stabilan per-pod DNS** — svaki Pod adresabilan po imenu, preko headless servisa
- **redoslijed** — stvara/gasi po redu (zad. 21)

**Headless servis** (`clusterIP: None`) je preduvjet: nema VIP-a, DNS vraća A-zapis SVAKOG Poda zasebno (umjesto jednog zajedničkog IP-a).

**Manifest** (`~/devops/lo4/redis-sts.yaml`) — dva objekta u istoj datoteci:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  clusterIP: None          # headless
  selector:
    app: redis
  ports:
  - port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis       # veže StatefulSet na headless servis (per-pod DNS)
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
```
- `---` razdvaja dva objekta u istoj datoteci
- Servis: `clusterIP: None` = headless; `selector app: redis` pokriva Podove
- StatefulSet: `serviceName: redis` = veže se na headless servis; `selector.matchLabels` MORA odgovarati `template.metadata.labels`

**Naredbe + VM ispis (anotiran):**
```bash
kubectl apply -f ~/devops/lo4/redis-sts.yaml
kubectl get pods -l app=redis -w
```
```
service/redis created
statefulset.apps/redis created
NAME      READY  STATUS   RESTARTS  AGE
redis-0   1/1    Running  0         22s     # najstariji = prvi krenuo
redis-1   1/1    Running  0         15s     # ~7s kasnije (tek kad je redis-0 gore)
redis-2   1/1    Running  0         14s     # zadnji
```
- imena `redis-0/1/2` stabilna i predvidljiva (ne hash kao Deployment)
- AGE stupac dokazuje REDOSLIJED stvaranja (22s → 15s → 14s)

Per-pod DNS (iz jednokratnog busyboxa):
```bash
kubectl run dnstest --rm -it --image=busybox:1.36 --restart=Never -- sh
# unutar shella:
nslookup redis-0.redis.default.svc.cluster.local      # vrati IP baš Poda redis-0
nslookup redis.default.svc.cluster.local              # vrati SVE tri Pod-IP odjednom
```
```
# pojedini Pod:
Name:    redis-0.redis.default.svc.cluster.local
Address: 10.244.0.6

# sam headless servis — TRI A-zapisa (per-pod):
Address: 10.244.0.6   (redis-0)
Address: 10.244.0.7   (redis-1)
Address: 10.244.0.8   (redis-2)
```
- pojedini Pod ima vlastitu stabilnu adresu preko FQDN `<pod>.<servis>.<ns>.svc.cluster.local`
- headless servis (bez VIP-a) vraća A-zapis SVAKOG Poda → „per-pod A records"

**Gotče:**
- iz DRUGOG Poda kratko ime `redis-0.redis` daje **NXDOMAIN** → treba PUNI FQDN `redis-0.redis.default.svc.cluster.local` (namespace + domena). (Kratko ime radi samo unutar istog namespacea preko search-domena, ovisno o resolv.conf.)
- `metadata:` se lako zamijeni s `metalama:` (autocorrect/navika) → StatefulSet bez imena, `apply` padne. UVIJEK `cat` provjera.

**`vi` popravak jednog retka (naučeno uživo):**
```
vi <file>            # otvori (Normal mode — tipke su naredbe)
:12                  # skoči na redak 12 (ili strelicama)
j j                  # spusti kursor (j = dolje)
cw                   # change word: obriše riječ pod kursorom + uđe u Insert mode
metadata             # upiši ispravnu riječ (dvotočka ostaje)
Esc                  # natrag u Normal mode
:wq                  # write + quit (spremi i izađi)
```

**Provjera:**
```bash
kubectl get pods -l app=redis        # redis-0/1/2, svi Running
kubectl get statefulset redis        # READY 3/3
```

## Zadatak 21 — StatefulSet stvara I gasi Podove po redu (kontrast s Deploymentom)

**Cilj:** pokazati uređeno stvaranje i gašenje StatefulSet Podova; usporediti s Deploymentom.

**Koncept:**
- **stvaranje** — od najnižeg indeksa: `redis-0` → `redis-1` → `redis-2` (svaki tek kad je prethodni Running). Vidljivo u AGE stupcu (zad. 16).
- **gašenje** — OBRNUTO, od najvišeg indeksa: `redis-2` → `redis-1` → `redis-0`, jedan po jedan (čeka da se prethodni potpuno ugasi).
- **logika:** kod klasteriranih aplikacija `redis-0` je često „glavni"; replike se ruše prije njega.
- **Deployment:** NEMA redoslijed — gasi/diže Podove nasumično, često paralelno (zamjenjivi Podovi).

**Naredba:**
```bash
kubectl scale statefulset redis --replicas=0
kubectl get pods -l app=redis -w
```
- `scale statefulset redis --replicas=0` — broj replika na 0 (ugasi sve, StatefulSet ostaje)
- u watchu (kad ga RAM ne zamrzne): `redis-2 Terminating` → nestane → `redis-1` → `redis-0`

**VM ispis:**
```
statefulset.apps/redis scaled
# nakon gašenja:
No resources found in default namespace.
```

**Gotča:** `-w` (watch) se zamrzava pod RAM pritiskom → može preskočiti `Terminating` faze. Rezultat se ipak vidi (`get pods` → prazno); redoslijed je StatefulSet zajamčeno odradio (definirano ponašanje), čak i ako ga watch ne uhvati uživo. Ako watch zapne >~40 s → `Ctrl+C` + svjež `kubectl get pods`.

**Provjera:**
```bash
kubectl get pods -l app=redis        # prazno nakon scale na 0
kubectl scale statefulset redis --replicas=3   # vrati: stvara se redom redis-0 → 1 → 2
```

## Zadatak 27 — skaliranje StatefulSeta = jedan PVC po replici (+ sudbina PVC-a pri brisanju)

**Cilj:** pokazati da StatefulSet s `volumeClaimTemplates` stvara zaseban PVC po replici, i objasniti što biva s PVC-ima kad se StatefulSet obriše.

**Koncept:** `volumeClaimTemplates` je PREDLOŽAK PVC-a — StatefulSet iz njega napravi ZASEBAN PVC za svaki Pod (ne dijele disk). Ime: `<predložak>-<statefulset>-<indeks>` → `data-redis-0/1/2`. Skaliranje gore = novi PVC po novoj replici.

**Manifest** (`~/devops/lo4/redis-sts-pvc.yaml`) — dodaci na zad. 16:
```yaml
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
        volumeMounts:              # NOVO: container montira disk
        - name: data
          mountPath: /data
  volumeClaimTemplates:            # NOVO: predložak PVC-a (dijete spec-a, 2 razmaka)
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Mi
```
- `volumeClaimTemplates.metadata.name: data` veže se na `volumeMounts.name: data`
- `accessModes: ["ReadWriteOnce"]` — disk montira jedan čvor (RWO)
- ⚠️ `volumeClaimTemplates:` je dijete `spec:`-a → **2 razmaka** (poravnato s `template:`/`replicas:`), NE na stupcu 0

**Naredbe + VM ispis:**
```bash
kubectl apply -f ~/devops/lo4/redis-sts-pvc.yaml
kubectl get pvc
```
```
NAME           STATUS   CAPACITY   ACCESS MODES   STORAGECLASS
data-redis-0   Bound    100Mi      RWO            standard
data-redis-1   Bound    100Mi      RWO            standard
data-redis-2   Bound    100Mi      RWO            standard
```
- TRI PVC-a, jedan po replici; različiti `pvc-...` VOLUME hashevi = stvarno odvojeni diskovi

Sudbina PVC-a pri brisanju StatefulSeta:
```bash
kubectl delete statefulset redis
kubectl get pvc
```
```
statefulset.apps "redis" deleted
# PVC-ovi I DALJE postoje (Bound): data-redis-0/1/2
```
- brisanje StatefulSeta NE briše PVC-ove → podaci namjerno prežive (zamjena/nadogradnja). Za stvarno brisanje: `kubectl delete pvc data-redis-0 ...` RUČNO.

**Gotče:**
- `volumeClaimTemplates` blok lako „ispadne" na stupac 0 pri ručnom prepisivanju → `unknown field`. **`cat -An <file>`** (prikaže točan broj razmaka + tabove kao `^I`) je pouzdaniji od oka — koristiti kad uvlaka nije jasna.
- `sed` uzorak mora TOČNO pogoditi tekst — typo u uzorku (`volumeClaimTempaltes`) = prazan rezultat, ne greška.

**Provjera:**
```bash
kubectl get pvc        # data-redis-0/1/2, Bound; prežive brisanje StatefulSeta
```

## Zadatak 17 — DaemonSet (jedan Pod po čvoru)

**Cilj:** napraviti DaemonSet i pokazati da pokreće točno jedan Pod po čvoru.

**Koncept:** DaemonSet pokrene **točno jedan Pod na SVAKOM čvoru**. NEMA `replicas` polje — broj = broj čvorova, automatski (dodaš čvor → dobije Pod; makneš → Pod nestane). Za agente na razini čvora (logovi, monitoring, mrežni dodaci). Na minikubeu (1 čvor) → 1 Pod.

**Manifest** (`~/devops/lo4/daemonset.yaml`):
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: agent
spec:
  selector:
    matchLabels:
      app: agent
  template:
    metadata:
      labels:
        app: agent
    spec:
      containers:
      - name: agent
        image: busybox:1.36
        command: ["sleep", "3600"]
```
- **nema `replicas`** — ključna razlika od Deploymenta/StatefulSeta
- `selector.matchLabels` = `template.metadata.labels` (`app: agent`)

**Naredbe + VM ispis:**
```bash
kubectl apply -f ~/devops/lo4/daemonset.yaml
kubectl get pods -l app=agent -o wide
```
```
NAME          READY  STATUS   RESTARTS  AGE  IP           NODE
agent-vp6fk   1/1    Running  0         13s  10.244.0.18  minikube
```
- točno JEDAN Pod, stupac `NODE` = `minikube` (jedini čvor) → jedan po čvoru

**Gotča:** polje je `containers:` (množina), NE `container:` → `unknown field container`. (`sed -i 's/^        container:/        containers:/' file` brzo popravi.)

**Provjera:**
```bash
kubectl get daemonset agent      # DESIRED/CURRENT/READY = broj čvorova (1)
```

## Zadatak 18 — Job (odradi jednom i završi)

**Cilj:** napraviti Job koji nešto izračuna jednom, vidjeti COMPLETIONS i pročitati rezultat iz logova.

**Koncept:** za razliku od Deploymenta (drži Pod stalno gore), Job **odradi zadatak jednom i Pod završi** (`Complete`, ne restarta se). `COMPLETIONS 1/1` = uspješno.

**Naredbe (imperativno, bez YAML-a):**
```bash
kubectl create job pi --image=perl:5.36 -- perl -Mbignum=bpi -wle "print bpi(20)"
kubectl get job pi
kubectl wait --for=condition=complete job/pi --timeout=60s
kubectl logs job/pi
```
- `create job pi --image=perl:5.36 -- <naredba>` — Job `pi`, izvrši naredbu jednom
- naredba `perl -Mbignum=bpi -wle "print bpi(20)"` = π na 20 znamenki
- `wait --for=condition=complete job/pi` — pričekaj da Job završi (elegantnije od watcha)
- `logs job/pi` — pročitaj rezultat

**VM ispis:**
```
job.batch/pi created
NAME  STATUS    COMPLETIONS  DURATION  AGE
pi    Running   0/1          4s        4s
job.batch/pi condition met
3.1415926535897932385          # ← rezultat iz logova
# kasnije:
pi    Complete  1/1           27s       63s
```

**Gotča:** `--for=condition=complete` (ne `contidion`) — typo u uvjetu daje „unrecognized condition".

**Provjera:**
```bash
kubectl get job pi               # Complete, COMPLETIONS 1/1
kubectl logs job/pi              # rezultat
```

## Zadatak 19 — CronJob (po rasporedu; suspend; lista Jobova)

**Cilj:** napraviti CronJob koji ispisuje datum svake minute, pauzirati ga, izlistati Jobove koje stvara.

**Koncept:** CronJob je „tvornica Jobova" — po cron rasporedu **stvara novi Job** (koji stvori Pod). Jobovi koje stvori imaju ime `<cronjob>-<timestamp>`.

**Naredbe (imperativno):**
```bash
kubectl create cronjob datum --image=busybox:1.36 --schedule="* * * * *" -- date
kubectl get cronjob datum
# pričekati ~minutu da okine:
kubectl get jobs                                          # datum-28xxxxxxx, Complete 1/1
kubectl patch cronjob datum -p '{"spec":{"suspend":true}}'   # pauziraj
kubectl get cronjob datum                                 # SUSPEND True
```
- `--schedule="* * * * *"` — cron: svake minute (min/sat/dan/mjesec/dan-u-tjednu, sve „svaki")
- `-- date` — naredba svakog Joba
- `get jobs` → Jobovi koje je CronJob automatski stvorio (`datum-29702733`, `datum-29702734`…)
- `patch ... suspend:true` — pauziraj (postojeći Jobovi ostaju, novi se ne stvaraju); SUSPEND `False`→`True`

**VM ispis:**
```
NAME            STATUS    COMPLETIONS  DURATION  AGE
datum-29702733  Complete  1/1          3s        75s
datum-29702734  Complete  1/1          3s        15s
# nakon patcha: SUSPEND True
```

**Gotča:** JSON payload za patch: `'{"spec":{"suspend":true}}'` — `"spec"` u navodnicima (NE `{"spec:` s dvotočkom unutar) → inače „invalid character 's' after object key".

**Provjera:**
```bash
kubectl get cronjob datum        # SUSPEND True, LAST SCHEDULE pokazuje zadnje okidanje
```

## Zadatak 20 — ručno pokretanje Joba iz CronJoba (`--from=cronjob/...`)

**Cilj:** pokrenuti Job iz postojećeg CronJoba na zahtjev (test bez čekanja rasporeda).

**Koncept:** `--from=cronjob/<ime>` napravi jednokratni Job iz CronJob predloška — radi **i dok je CronJob suspendiran**. Tipično za „daj da testiram prije puštanja rasporeda" ili izvanredno pokretanje.

**Naredbe (imperativno):**
```bash
kubectl create job rucni --from=cronjob/datum
kubectl get jobs -l job-name=rucni
kubectl logs job/rucni
```

**VM ispis:**
```
job.batch/rucni created
NAME   STATUS    COMPLETIONS  DURATION  AGE
rucni  Complete  1/1          3s        11s
Mon Jun 22 21:37:57 UTC 2026          # ← datum, ručno pokrenuto dok je CronJob suspendiran
```

**Gotča:** sintaksa je `--from=cronjob/datum` (puni resource tip `cronjob`), NE `--from=cron/...` → „server doesn't have a resource type cron".

**Provjera:**
```bash
kubectl get jobs -l job-name=rucni   # Complete 1/1
kubectl logs job/rucni               # datum (čak i kad je CronJob suspendiran)
```

---

## Faza 7 — završena

Svih 7 zadataka Faze 7 (Ostali kontroleri) odrađeno i verificirano: 16, 17, 18, 19, 20, 21, 27.

Pokriveno: **StatefulSet** (stabilna imena redis-0/1/2, headless servis + per-pod DNS, uređeno stvaranje/gašenje, PVC-po-replici preko volumeClaimTemplates + sudbina PVC-a) · **DaemonSet** (jedan Pod po čvoru) · **Job** (odradi jednom + Complete, rezultat iz logova) · **CronJob** (raspored, suspend, Jobovi koje stvara, ručno okidanje `--from`).

**Čišćenje Faze 7** (ako klaster preživi):
```bash
kubectl delete daemonset agent
kubectl delete job pi rucni
kubectl delete cronjob datum
kubectl delete pvc data-redis-0 data-redis-1 data-redis-2 2>/dev/null
kubectl delete service redis 2>/dev/null
```

---

## Brza referenca

Sažeta karta naredbi za LO4 (sve naredbe pojavljuju se u zadacima iznad).

**Klaster (Faza 0):**
```bash
minikube start --driver=podman          # digni/probudi klaster
kubectl get nodes                        # čvor mora biti Ready
minikube status                          # dijagnoza kad kubectl ne odgovara
minikube delete --all                    # SAMO za zatrovani profil (briše sve)
```

**Deployment / ReplicaSet / Pod (Faza 1):**
```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl create deployment web --image=nginx:1.25 --replicas=3 --dry-run=client -o yaml > web.yaml
kubectl scale deployment web --replicas=5
kubectl get pods -l app=web --show-labels
kubectl run sleeper --image=busybox -- sleep 3600          # goli Pod
kubectl get all                                            # cijeli lanac
```

**Rollouti (Faza 2):**
```bash
kubectl set image deployment/web nginx=nginx:1.27          # lijevo = IME CONTAINERA
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web [--to-revision=N]
kubectl annotate deployment/web kubernetes.io/change-cause="..."   # --record je zastarjelo
```

**Podešavanje Poda (Faza 3):** `resources` (requests/limits), drugi container (sidecar), `nodeSelector`, `emptyDir`, `initContainers` — preko `kubectl edit` / `apply -f`.

**Servisi (Faza 4):**
```bash
kubectl expose deployment web --port=80 --target-port=80 --name=web-svc            # ClusterIP
kubectl expose deployment web --type=NodePort --port=80 --target-port=80 --name=web-np
kubectl expose deployment web --type=LoadBalancer ... --name=web-lb                # EXTERNAL-IP <pending> na minikubeu
kubectl expose deployment web --name=web-headless ... --cluster-ip=None            # headless (per-pod DNS)
kubectl get endpoints <svc>                                                        # PodIP:port iza Servisa
kubectl port-forward service/web-svc 8080:80                                       # tunel s VM-a
minikube service <svc> --url                                                       # vanjski URL NodePorta
# DNS iz Poda: <svc>.<ns>.svc.cluster.local
```

**Sonde + dijagnostika (Faza 5):**
```bash
kubectl describe pod <pod>            # sekcija Events = ZAŠTO se nešto dogodilo
kubectl logs <pod> [--previous] [-f] [--tail=N]   # --previous = mrtva instanca prije restarta
kubectl exec -it <pod> [-c <ctr>] -- sh
kubectl run tmp --rm -it --image=busybox --restart=Never -- wget -qO- http://<svc>
```
Redoslijed dijagnoze (zad. 48): Pod gore → ready → Endpoints → DNS → port.

**Config / Secret / Volume (Faza 6):**
```bash
kubectl create configmap app-config --from-literal=KEY=value
kubectl create secret generic db-cred --from-literal=DB_USER=admin --from-literal=DB_PASS=s3cret
kubectl get secret db-cred -o jsonpath='{.data.DB_PASS}' | base64 -d ; echo        # base64, NIJE enkripcija
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key                  # kubernetes.io/tls
kubectl create secret docker-registry dockerhub-cred --docker-server=... --docker-username=... --docker-password=...
kubectl get storageclass / pv / pvc
```

**Ostali kontroleri (Faza 7):**
```bash
kubectl create job pi --image=perl:5.36 -- perl -Mbignum=bpi -wle "print bpi(20)"
kubectl create cronjob datum --image=busybox:1.36 --schedule="* * * * *" -- date
kubectl patch cronjob datum -p '{"spec":{"suspend":true}}'                         # pauziraj
kubectl create job rucni --from=cronjob/datum                                      # resurs cronjob/<ime>, NE cron/
kubectl get statefulset / daemonset
```

**apiVersion/kind parovi:** Deployment / StatefulSet / DaemonSet → `apps/v1`; Job / CronJob → `batch/v1`; Pod / Service / ConfigMap / Secret / PersistentVolumeClaim → `v1`.

**`--dry-run`:** `--dry-run=client` (lokalno sastavi) · `--dry-run=server` (stroža provjera, server validira, hvata više).

---

## LO4 kompletan — svih 53/53 zadataka odrađeno i verificirano na VM-u.
