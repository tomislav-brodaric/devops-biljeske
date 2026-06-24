# DevOps — bilješke za ispit (Intro to DevOps)

Osobne bilješke za kolegij Intro to DevOps (Algebra Bernays). Pokrivaju svih šest ishoda (LO1 do LO6), hands-on provjereno na ispitnom računalu.

**Dopušteno na ispitu:** otvorene bilješke (ovaj repozitorij), službena dokumentacija (kubernetes.io, docs.podman.io, Docker, minikube, Red Hat) i osobni GitHub. Alati umjetne inteligencije nisu dopušteni.

---

## Karta datoteka

| Datoteka | Što pokriva |
|---|---|
| `DevOps-Temeljni-pojmovi.md` | Kreni odavde. Osnovni pojmovi od nule: Linux, terminal, kontejner, slika, Docker, Podman, Kubernetes. |
| `Terminal-Podman-cheatsheet.md` | Brzi pregled: Linux ljuska i sve glavne Podman naredbe i zastavice. |
| `DevOps-LO1-skripta.md` | Uporaba kontejnera (Podman): pokretanje, portovi, restart, ograničenja, varijable okoline, mreže, pregledavanje, oznake, Quadlet. 23 zadatka. |
| `DevOps-LO2-skripta.md` | Izrada i upravljanje slikama: Containerfile, višefazna izrada, oznake, HEALTHCHECK, ENTRYPOINT i CMD, ADD i COPY, sažetak (digest), slojevi. 23 zadatka. |
| `DevOps-LO3-skripta.md` | Isporuka aplikacija, mreže, sigurnost: podovi, prilagođene mreže i imena, Compose, tajne, trajni volumeni, obrnuti posrednik. 23 zadatka. |
| `DevOps-LO4-skripta.md` | Kubernetes: Deploymenti, rolling update, StatefulSet, DaemonSet, Job, CronJob, ConfigMap, Secret, Servisi, sonde (probes). 53 zadatka. |
| `DevOps-LO5-skripta.md` | Rješavanje kvarova: pokvarene podman naredbe, pokvareni Containerfile, dijagnostika u Kubernetesu. 43 zadatka. |
| `DevOps-LO6-skripta.md` | Teorija: usporedbe sustava za orkestraciju, obranjive teze za i protiv, plus okvir za odlučivanje. 23 pitanja. |
| `DevOps-Cesto-zamijenjeno.md` | Parovi pojmova koje ispitivači rado provjeravaju, s jasnom razlikom (port i ciljani port, zahtjevi i granice, Deployment i StatefulSet, sonde, tipovi Servisa…). |

---

## Brzo pronalaženje kvarova (LO5) — pretraži simptom

Puno objašnjenje svakog je u `DevOps-LO5-skripta.md`, kod navedenog zadatka.

### Pokvarene `podman run` naredbe

| Simptom | Uzrok | Rješenje | Zadatak |
|---|---|---|---|
| Stranica nedostupna na očekivanom portu | Port je obrnut (`80:8080`) | Ispravan redoslijed je port-računala dvotočka port-kontejnera | 1 |
| Kontejner baze odmah završi pri pokretanju | Lozinka zadana bez vrijednosti | Daj vrijednost varijabli, ili dopusti praznu lozinku | 2 |
| Zastavica za port se ne primjenjuje | Mreža je postavljena na `host` | U načinu `host` portovi se ne objavljuju zasebno | 3 |
| Kontejner odmah u stanju Exited (0) | Nema dugotrajnog procesa | Daj naredbu koja traje, na primjer spavanje | 4 |
| Pregled zapisa ne radi nakon pokretanja | Samouklanjanje uz pozadinski rad | Bez samouklanjanja ako trebaš zapise | 5 |
| Baza nikad ne postane zdrava | Premala granica memorije | Podigni granicu memorije | 6 |
| Imena se ne razlučuju među kontejnerima | Zadana mreža nema razlučivanje imena | Stvori vlastitu mrežu i spoji ih na nju | 7 |
| Datoteke se ne vide u kontejneru | Nedostaje oznaka za dijeljenje pri povezivanju mape | Dodaj oznaku za dijeljenje (`:Z`) | 8 |

### Pokvareni Containerfile

| Simptom | Uzrok | Rješenje | Zadatak |
|---|---|---|---|
| Izgradnja ne nalazi paket | Nedostaje osvježavanje popisa paketa prije instalacije | Spoji osvježavanje i instalaciju u istom koraku | 10 |
| Aplikacija se ne zaustavlja čisto | Naredba u obliku ljuske ne prosljeđuje signale | Koristi oblik s popisom (exec oblik) | 11 |
| Svaka promjena koda pokreće punu instalaciju | Loš redoslijed koraka kvari predmemoriranje | Prvo kopiraj popis ovisnosti, instaliraj, pa kopiraj kod | 12 |
| Naredbe se ne nalaze nakon postavljanja putanje | Postavljanje putanje pregazilo postojeću | Dodaj na postojeću putanju umjesto pregaženja | 13 |
| Ništa ne odgovara na portu | Najavljen port ne znači objavljen port | Najava porta je samo dokumentacija; objavi port pri pokretanju | 15 |
| Slika je golema | Čišćenje nije u istom koraku kao instalacija | Instaliraj i očisti u istom koraku | 16 |
| Greške zabrane pristupa | Prebacivanje na običnog korisnika prije kopiranja | Pazi na redoslijed i vlasništvo datoteka | 17 |
| Konačna slika je oko jedan gigabajt | Izgradnja i pokretanje u istoj slici | Koristi višefaznu izradu | 18 |

### Stanja i kvarovi u Kubernetesu

| Simptom (stanje) | Uzrok | Rješenje | Zadatak |
|---|---|---|---|
| Pending, nedovoljno memorije | Zahtjev za memorijom prevelik | Smanji zahtjev | 19 |
| Greška pri primjeni opisa | Pogrešna uvlaka u YAML datoteci | Ispravi uvlake (samo razmaci) | 20 |
| ImagePullBackOff | Kriva slika ili oznaka, ili privatni registar | Ispravi ime ili oznaku, dodaj tajnu za registar | 21 |
| CrashLoopBackOff, izlazni kod 1 | Proces pada s greškom | Pročitaj prethodne zapise, popravi uzrok | 22 |
| OOMKilled, kod 137 | Probijena granica memorije | Podigni granicu memorije | 23 |
| Podovi nikad spremni | Sonda spremnosti pada | Ispravi sondu ili aplikaciju | 24 |
| Servis ne vraća ništa | Oznake Servisa i Poda se ne poklapaju | Uskladi oznake; provjeri popis krajnjih točaka | 25 |
| Selektor ne odgovara predlošku | Oznake u selektoru i predlošku različite | Uskladi oznake na oba mjesta | 26 |
| Pending, nedovoljno procesora | Zahtjev za procesorom premašuje računalo | Smanji zahtjev; provjeri kapacitet računala | 27 |
| Pending, volumen ne postoji | Zahtjev za nepostojećim trajnim sveskom | Stvori traženi svezak, ili koristi privremeni | 28 |
| Completed ili CrashLoop iz čiste slike | Nema dugotrajne naredbe | Dodaj naredbu koja traje | 29 |
| Treba pregled svega što se dogodilo | Kronologija događaja | Popis događaja poredan po vremenu | 30 |
| Aplikacija ne nalazi drugi servis | Razlučivanje imena (DNS) | Uđi u Pod, provjeri imena i postavke imena | 31 |
| Treba ući u sliku bez ljuske | Distroless slika nema ljusku | Privremeni dijagnostički kontejner | 32 |
| Provjera povezanosti unutar klastera | Treba probni Pod | Privremeni Pod koji se sam obriše | 33 |
| Računalo NotReady | Usluga na čvoru, disk ili mreža | Provjeri uslugu čvora, prostor na disku, mrežu | 34 |
| Pod zaglavljen u Terminating | Finalizer blokira brisanje | Ukloni finalizer (oprezno) | 35 |
| Neprepoznat tip objekta | Pogrešan par verzije i tipa | Ispravi verziju i tip (provjeri popis tipova) | 36 |
| CreateContainerConfigError | Nedostaje ključ u izvoru postavki | Dodaj ključ; Pod se sam oporavi | 37 |
| Tajna se ne nalazi | Tajna je u drugom prostoru imena | Stvori tajnu u istom prostoru imena | 38 |
| ProgressDeadlineExceeded | Novo izdanje ne napreduje | Vrati na prethodno izdanje, ili popravi sliku | 39 |
| Treba uhvatiti greške prije primjene | Tipfeleri u poljima | Provjera na poslužitelju prije primjene | 40 |
| Aplikacija ne doseže Servis | Prekinuta jedna karika u lancu | Provjeri redom: Pod, Servis, krajnje točke, ime, port | 41 |
| Treba uzrok ponavljanih restarta | Stanje i broj restarta | Izvuci stanje i broj restarta ciljanim ispisom | 42 |
| Usporedba načina izlaganja prema van | Route i Ingress | Teorijska usporedba | 43 |

---

## LO6 — brzi podsjetnik

Za pitanja koja traže preporuku rješenja (10, 11, 13, 15, 17, 18, 19, 20, 21, 22, 23), koristi **okvir za odlučivanje** na kraju `DevOps-LO6-skripta.md`: izvuci faktore (veličina tima, skala, otpornost, sigurnost, trošak), poveži s rješenjem s ljestvice, obrazloži kroz faktore i navedi cijenu druge strane.

Pošto je LO6 usmeni dio (ne možeš ga tražiti po bilješkama na licu mjesta), ispitne kuke nauči napamet i izgovori naglas.

---

## Računi i pristup za ispit

- **Quay.io** (korisnik bez dvojne provjere)
- **Docker Hub** (korisnik bez dvojne provjere)
- **Red Hat Academy**
- **Infoeduka**
- Provjeri **pristup mreži (VPN)** i ulaz u ispitno okruženje dan prije ispita.
