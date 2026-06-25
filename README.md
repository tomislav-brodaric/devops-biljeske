# DevOps — bilješke za ispit (Intro to DevOps)

Bilješke za kolegij Intro to DevOps (Algebra Bernays). Pokrivaju svih šest ishoda (LO1 do LO6), uz praktične provjere na ispitnom računalu.

**Dopušteno na ispitu:** otvorene bilješke (ovaj repozitorij, javno dostupan na GitHubu — github.com, raw.githubusercontent.com), službena dokumentacija (docs.docker.com, docs.podman.io, kubernetes.io, minikube.sigs.k8s.io, rha.ole.redhat.com) te dopušteni registri slika i paketa (Docker Hub, Quay.io, registry.k8s.io, PyPI, npm i ostali s popisa dopuštenih domena). Alati umjetne inteligencije nisu dopušteni.

---

## Karta datoteka

| Datoteka | Što pokriva |
|---|---|
| `DevOps-Temeljni-pojmovi.md` | Početna točka. Osnovni pojmovi od nule: Linux, terminal, kontejner, slika, Docker, Podman, Kubernetes. |
| `Terminal-Podman-cheatsheet.md` | Brzi pregled: Linux ljuska i sve glavne Podman naredbe i zastavice. |
| `DevOps-LO1-skripta.md` | Uporaba kontejnera (Podman): pokretanje, portovi, restart, ograničenja, varijable okoline, mreže, pregledavanje, oznake, Quadlet. 23 zadatka. |
| `DevOps-LO2-skripta.md` | Izrada i upravljanje slikama: Containerfile, višefazna izrada, oznake, HEALTHCHECK, ENTRYPOINT i CMD, ADD i COPY, sažetak (digest), slojevi. 23 zadatka. |
| `DevOps-LO3-skripta.md` | Isporuka aplikacija, mreže, sigurnost: podovi, prilagođene mreže i imena, Compose, tajne, trajni volumeni, obrnuti posrednik. 23 zadatka. |
| `DevOps-LO4-skripta.md` | Kubernetes: Deploymenti, rolling update, StatefulSet, DaemonSet, Job, CronJob, ConfigMap, Secret, Servisi, sonde (probes). 53 zadatka. |
| `DevOps-LO5-skripta.md` | Rješavanje kvarova: pokvarene podman naredbe, pokvareni Containerfile, dijagnostika u Kubernetesu. 43 zadatka. |
| `DevOps-LO6-skripta.md` | Teorija: usporedbe sustava za orkestraciju, obranjive teze za i protiv, plus okvir za odlučivanje. 23 pitanja. |
| `DevOps-Cesto-zamijenjeno.md` | Parovi pojmova koje ispitivači rado provjeravaju, s jasnom razlikom (port i ciljani port, zahtjevi i granice, Deployment i StatefulSet, sonde, tipovi Servisa…). |

---

## Brzo pronalaženje kvarova (LO5) — pretraga po simptomu

Puno objašnjenje svakog kvara je u `DevOps-LO5-skripta.md`, kod navedenog zadatka.

### Pokvarene `podman run` naredbe

| Simptom | Uzrok | Rješenje | Zadatak |
|---|---|---|---|
| Stranica nedostupna na očekivanom portu | Port je obrnut (`80:8080`) | Ispravan redoslijed je port-računala dvotočka port-kontejnera | 1 |
| Kontejner baze odmah završi pri pokretanju | Lozinka zadana bez vrijednosti | Zadati vrijednost varijabli ili dopustiti praznu lozinku | 2 |
| Zastavica za port se ne primjenjuje | Mreža je postavljena na `host` | U načinu `host` portovi se ne objavljuju zasebno | 3 |
| Kontejner odmah u stanju Exited (0) | Nema dugotrajnog procesa | Zadati naredbu koja traje, na primjer spavanje | 4 |
| Pregled zapisa ne radi nakon pokretanja | Samouklanjanje uz pozadinski rad | Bez samouklanjanja kada su zapisi potrebni | 5 |
| Baza nikad ne postane zdrava | Premala granica memorije | Podignuti granicu memorije | 6 |
| Imena se ne razlučuju među kontejnerima | Zadana mreža nema razlučivanje imena | Stvoriti vlastitu mrežu i spojiti kontejnere na nju | 7 |
| Datoteke se ne vide u kontejneru | Nedostaje oznaka za dijeljenje pri povezivanju mape | Dodati oznaku za dijeljenje (`:Z`) | 8 |

### Pokvareni Containerfile

| Simptom | Uzrok | Rješenje | Zadatak |
|---|---|---|---|
| Izgradnja ne nalazi paket | Nedostaje osvježavanje popisa paketa prije instalacije | Spojiti osvježavanje i instalaciju u istom koraku | 10 |
| Aplikacija se ne zaustavlja čisto | Naredba u obliku ljuske ne prosljeđuje signale | Koristiti oblik s popisom (exec oblik) | 11 |
| Svaka promjena koda pokreće punu instalaciju | Loš redoslijed koraka kvari predmemoriranje | Prvo kopirati popis ovisnosti, instalirati, pa kopirati kod | 12 |
| Naredbe se ne nalaze nakon postavljanja putanje | Postavljanje putanje pregazilo postojeću | Dodati na postojeću putanju umjesto pregaženja | 13 |
| Ništa ne odgovara na portu | Najavljen port ne znači objavljen port | Najava porta je samo dokumentacija; port se objavljuje pri pokretanju | 15 |
| Slika je golema | Čišćenje nije u istom koraku kao instalacija | Instalirati i očistiti u istom koraku | 16 |
| Greške zabrane pristupa | Prebacivanje na običnog korisnika prije kopiranja | Paziti na redoslijed i vlasništvo datoteka | 17 |
| Konačna slika je oko jedan gigabajt | Izgradnja i pokretanje u istoj slici | Koristiti višefaznu izradu | 18 |

### Stanja i kvarovi u Kubernetesu

| Simptom (stanje) | Uzrok | Rješenje | Zadatak |
|---|---|---|---|
| Pending, nedovoljno memorije | Zahtjev za memorijom prevelik | Smanjiti zahtjev | 19 |
| Greška pri primjeni opisa | Pogrešna uvlaka u YAML datoteci | Ispraviti uvlake (samo razmaci) | 20 |
| ImagePullBackOff | Kriva slika ili oznaka, ili privatni registar | Ispraviti ime ili oznaku, dodati tajnu za registar | 21 |
| CrashLoopBackOff, izlazni kod 1 | Proces pada s greškom | Pročitati prethodne zapise, popraviti uzrok | 22 |
| OOMKilled, kod 137 | Probijena granica memorije | Podignuti granicu memorije | 23 |
| Podovi nikad spremni | Sonda spremnosti pada | Ispraviti sondu ili aplikaciju | 24 |
| Servis ne vraća ništa | Oznake Servisa i Poda se ne poklapaju | Uskladiti oznake; provjeriti popis krajnjih točaka | 25 |
| Selektor ne odgovara predlošku | Oznake u selektoru i predlošku različite | Uskladiti oznake na oba mjesta | 26 |
| Pending, nedovoljno procesora | Zahtjev za procesorom premašuje računalo | Smanjiti zahtjev; provjeriti kapacitet računala | 27 |
| Pending, volumen ne postoji | Zahtjev za nepostojećim trajnim sveskom | Stvoriti traženi svezak ili koristiti privremeni | 28 |
| Completed ili CrashLoop iz čiste slike | Nema dugotrajne naredbe | Dodati naredbu koja traje | 29 |
| Treba pregled svega što se dogodilo | Kronologija događaja | Popis događaja poredan po vremenu | 30 |
| Aplikacija ne nalazi drugi servis | Razlučivanje imena (DNS) | Ući u Pod, provjeriti imena i postavke imena | 31 |
| Treba ući u sliku bez ljuske | Distroless slika nema ljusku | Privremeni dijagnostički kontejner | 32 |
| Provjera povezanosti unutar klastera | Treba probni Pod | Privremeni Pod koji se sam obriše | 33 |
| Računalo NotReady | Usluga na čvoru, disk ili mreža | Provjeriti uslugu čvora, prostor na disku, mrežu | 34 |
| Pod zaglavljen u Terminating | Finalizer blokira brisanje | Ukloniti finalizer (oprezno) | 35 |
| Neprepoznat tip objekta | Pogrešan par verzije i tipa | Ispraviti verziju i tip (provjeriti popis tipova) | 36 |
| CreateContainerConfigError | Nedostaje ključ u izvoru postavki | Dodati ključ; Pod se sam oporavi | 37 |
| Tajna se ne nalazi | Tajna je u drugom prostoru imena | Stvoriti tajnu u istom prostoru imena | 38 |
| ProgressDeadlineExceeded | Novo izdanje ne napreduje | Vratiti na prethodno izdanje ili popraviti sliku | 39 |
| Treba uhvatiti greške prije primjene | Tipfeleri u poljima | Provjera na poslužitelju prije primjene | 40 |
| Aplikacija ne doseže Servis | Prekinuta jedna karika u lancu | Provjeriti redom: Pod, Servis, krajnje točke, ime, port | 41 |
| Treba uzrok ponavljanih restarta | Stanje i broj restarta | Izvući stanje i broj restarta ciljanim ispisom | 42 |
| Usporedba načina izlaganja prema van | Route i Ingress | Teorijska usporedba | 43 |

---

## LO6 — brzi podsjetnik

Za pitanja koja traže preporuku rješenja (10, 11, 13, 15, 17, 18, 19, 20, 21, 22, 23) služi **okvir za odlučivanje** na kraju `DevOps-LO6-skripta.md`: izvuku se faktori (veličina tima, skala, otpornost, sigurnost, trošak), povežu s rješenjem s ljestvice, obrazloži se izbor kroz te faktore i navede cijena druge strane.

LO6 je teorijski ishod i piše se (kao i ostatak ispita) uz otvorene bilješke. Ispitne kuke su sažete rečenice za brzo i sigurno odgovaranje.

---

## Računi i pristup za ispit

- **Quay.io** (korisnik bez dvojne provjere)
- **Docker Hub** (korisnik bez dvojne provjere)
- **Red Hat Academy**
- **Infoeduka**
- Provjeriti **pristup mreži (VPN)** i ulaz u ispitno okruženje dan prije ispita.
- https://github.com/vknue/DOCUMENTATIONDEVOPS
