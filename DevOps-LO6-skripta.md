# DevOps — LO6 skripta (teorija: obranjive teze za i protiv)

> **LO6** ("Evaluate the use of selected container orchestration systems" — Procjena uporabe odabranih sustava za orkestraciju kontejnera) je teorijski ishod. Ima 23 pitanja iz službenog PDF-a (str. 10), istim redoslijedom.
> **Format svakog pitanja:** kontekst, pa argumenti za obje strane, pa **obranjiv stav** (stav koji se može obraniti argumentima), pa **ispitna kuka** (jedna rečenica za pamćenje).
> **Stil:** svaka kratica je objašnjena, svaki pojam definiran, sve napisano riječima (bez simbola).
> Napredak: sva pitanja 1–23 gotova, plus okvir za odlučivanje i pregledi Docker Swarma i k3s-a. Temeljni pojmovi su u zasebnoj datoteci `DevOps-Temeljni-pojmovi.md`.

---

## Rječnik kratica (za cijeli LO6)

- **DevOps** (Development and Operations, razvoj i operacije) — praksa spajanja razvoja softvera i njegova održavanja u radu.
- **Kubernetes**, kratica **k8s** (slovo „k", zatim osam slova, zatim „s") — sustav za orkestraciju kontejnera, to jest za automatsko pokretanje velikog broja kontejnera preko više računala.
- **Orkestracija** — automatsko raspoređivanje, pokretanje, skaliranje i oporavak kontejnera.
- **Kontejner** — izolirani paket s aplikacijom i svime što joj treba za rad.
- **Slika (image)** — zamrznuti predložak iz kojeg se pokreće kontejner.
- **Pod** — najmanja jedinica u Kubernetesu: omotač oko jednog ili više kontejnera koji dijele istu mrežu.
- **Docker** — alat za izradu i pokretanje kontejnera (radi preko stalne pozadinske usluge zvane daemon).
- **Podman** — alat sličan Dockeru, ali bez stalne pozadinske usluge (bez daemona).
- **OpenShift**, kratica **OCP** (OpenShift Container Platform) — Kubernetes s dodanim alatima, tvrtke Red Hat.
- **IP adresa** (Internet Protocol, internetski protokol) — broj koji jednoznačno označava računalo ili kontejner na mreži, poput kućne adrese.
- **NAT** (Network Address Translation, prijevod mrežnih adresa) — postupak gdje se privatna adresa prevodi u javnu pri izlasku na vanjsku mrežu.
- **DNS** (Domain Name System, sustav imena domena) — prevodi imena (na primjer „web") u IP adrese.
- **CNI** (Container Network Interface, sučelje za mrežu kontejnera) — dodatak koji Kubernetesu daje mrežu (izvedbe: Calico, Flannel, Cilium).
- **CSI** (Container Storage Interface, sučelje za pohranu kontejnera) — standard preko kojeg Kubernetes razgovara s različitim sustavima za pohranu.
- **PV** (PersistentVolume, trajni svezak) — komad stvarne pohrane (disk) u Kubernetesu.
- **PVC** (PersistentVolumeClaim, zahtjev za trajnim sveskom) — zahtjev aplikacije za pohranom; aplikacija koristi PVC, a ne disk izravno.
- **CRD** (Custom Resource Definition, definicija vlastitog tipa resursa) — način da se u Kubernetes doda vlastita vrsta objekta.
- **Kontroler** — program koji stalno gleda stanje i radi da stvarno stanje odgovara željenom.
- **Operator** — kombinacija vlastitog tipa objekta (CRD) i kontrolera koji automatizira upravljanje nekom složenom uslugom.
- **S2I** (Source-to-Image, iz izvornog koda u sliku) — OpenShiftova značajka koja iz izvornog koda sama izgradi sliku i pokrene aplikaciju.
- **GUI** (Graphical User Interface, grafičko sučelje) — sučelje s prozorima i gumbima, za razliku od tipkanja naredbi.
- **CI** (Continuous Integration, kontinuirana integracija) — automatsko spajanje i provjera koda pri svakoj promjeni.
- **CD** (Continuous Delivery ili Deployment, kontinuirana isporuka) — automatsko pakiranje i objava aplikacije.
- **SCC** (Security Context Constraints, ograničenja sigurnosnog konteksta) — OpenShiftova pravila o tome što Pod smije raditi.
- **RBAC** (Role-Based Access Control, kontrola pristupa temeljena na ulogama) — tko smije što, prema dodijeljenoj ulozi.
- **OAuth** (Open Authorization, otvoreni standard za autorizaciju) — standardni način prijave i davanja ovlasti.
- **HPA** (Horizontal Pod Autoscaler, vodoravni autoskaler Podova) — automatski dodaje ili oduzima Podove prema opterećenju.
- **Cluster Autoscaler** — automatski dodaje ili oduzima cijela računala (čvorove) prema potrebi.
- **IAM** (Identity and Access Management, upravljanje identitetima i pristupom) — sustav za identitete i ovlasti, tipično u oblaku.
- **VM** (Virtual Machine, virtualni stroj) — softverski simulirano računalo unutar pravog računala.
- **UID** (User Identifier, korisnički identifikator) — broj koji označava korisnika u sustavu.
- **CNCF** (Cloud Native Computing Foundation) — neutralna zaklada koja održava Kubernetes i srodne otvorene projekte.
- **Stateless** (bez stanja) — aplikacija koja ne pamti podatke između pokretanja.
- **Stateful** (sa stanjem) — aplikacija koja mora trajno pamtiti podatke (na primjer baza podataka).
- **Manifest** — tekstualna datoteka (obično YAML) koja opisuje željeni objekt u Kubernetesu.
- **Servis (Service)** — objekt koji daje jednu stalnu adresu i ime za skup Podova koji se stalno mijenjaju.

---

## 1. Mrežni model Kubernetesa u odnosu na Dockerovu mrežu

**Kontekst:** Mrežni model određuje kako kontejneri dobivaju adrese i kako međusobno razgovaraju. **Bridge mreža** (mrežni most) je virtualna mreža na jednom računalu preko koje kontejneri komuniciraju. **NAT** (Network Address Translation, prijevod mrežnih adresa) prevodi privatnu adresu u javnu pri izlasku na vanjsku mrežu. **Objava porta** znači da se port unutar kontejnera izloži na portu računala. **Overlay mreža** (mreža preko mreže) povezuje kontejnere preko više računala. **CNI** (Container Network Interface, sučelje za mrežu kontejnera) je dodatak koji Kubernetesu daje mrežu.

**Kako radi Docker:** Po zadanim postavkama Docker koristi bridge mrežu (mrežni most) na jednom računalu. Svaki kontejner dobije privatnu IP adresu, kontejneri međusobno razgovaraju preko prijevoda adresa (NAT), a prema vanjskom svijetu izlažu se objavom porta (port unutar kontejnera vidi se na portu računala). Da bi kontejneri razgovarali preko više računala, treba poseban sloj zvan overlay mreža (mreža preko mreže) ili ručno podešavanje.

**Kako radi Kubernetes:** Kubernetes koristi ravni model s jednom IP adresom po Podu. Svaki Pod dobije vlastitu adresu dosežljivu u cijelom klasteru, i svaki Pod može doći do svakog drugog Poda bez prijevoda adresa. To omogućuje CNI dodatak. Iznad toga stoji Servis, koji daje jednu stalnu adresu i ime za skup Podova koji se stalno gase i dižu.

**Argumenti za model Kubernetesa:** ravna mreža bez prijevoda adresa znači jednostavnije pronalaženje servisa; CNI dodatak je zamjenjiv; Servisi i imena (DNS) skrivaju činjenicu da se Podovi mijenjaju; postoji i **NetworkPolicy** (pravilo koje ograničava tko s kim smije komunicirati) za odvajanje prometa.

**Argumenti za Dockerov model:** jednostavniji je na jednom računalu; manje toga treba naučiti; most i objava porta dovoljni su za male postavke.

**Obranjiv stav:** Dockerov model je sasvim dobar na jednom računalu. Ravni model Kubernetesa, s jednom adresom po Podu, jest ono što omogućuje rad preko više računala i skalabilno pronalaženje servisa, uz cijenu da treba CNI dodatak i razumijevanje više pojmova. Kubernetes rješava pitanje kako velik broj Podova preko mnogo računala međusobno razgovara, što Dockerov most sam ne pokriva.

**Ispitna kuka:** Docker koristi most po računalu, prijevod adresa i objavu porta. Kubernetes koristi ravnu mrežu s jednom adresom po Podu, gdje je svaki Pod dosežljiv bez prijevoda adresa, preko CNI dodatka, sa Servisima i imenima iznad stalne izmjene Podova.

---

## 2. Pohrana u Kubernetesu (CSI, StorageClass, dinamičko provisioniranje) u odnosu na pohranu u Podmanu

**Kontekst:** **StorageClass** (razred pohrane) je predložak koji opisuje vrstu pohrane. **Dinamičko provisioniranje** znači da se trajni disk (PV) stvori automatski čim aplikacija zatraži pohranu (preko PVC-a), bez ručne pripreme.

**Kako radi Kubernetes:** Aplikacija traži pohranu zahtjevom zvanim PVC (PersistentVolumeClaim). Uz StorageClass i CSI upravljački program (CSI, Container Storage Interface), Kubernetes **sam stvori** trajni disk (PV, PersistentVolume) iz pozadinskog sustava (na primjer diska u oblaku ili mrežne pohrane). Pošto je CSI standard, isti pristup radi s mnogo različitih pozadinskih sustava.

**Kako radi Podman:** Podman koristi **volumene** (imenovane spremnike podataka) s lokalnim upravljačkim programom, uz mogućnost dodataka. To je jednostavno i vezano za jedno računalo; nema automatskog stvaranja pohrane na razini cijelog klastera.

**Argumenti za Kubernetes:** automatsko stvaranje pohrane na zahtjev; prenosivost preko različitih pozadinskih sustava (zahvaljujući standardu CSI); svaka aplikacija traži svoju pohranu; za StatefulSet (skup Podova sa stalnim identitetom) svaki Pod dobije svoj PVC; aplikacija je odvojena od detalja pohrane.

**Argumenti za Podman:** jednostavnije, manje režije, dovoljno za pohranu na jednom računalu; imenovani volumen preživi i kad se kontejner obriše pa ponovo stvori.

**Obranjiv stav:** Za aplikacije sa stanjem (stateful) na više računala ili u velikoj mjeri, pristup Kubernetesa (CSI uz dinamičko provisioniranje) jest **znatno fleksibilniji** jer se pohrana traži opisno, neovisno o pozadinskom sustavu, i ona se stvori sama. Podmanovi volumeni su jednostavniji, ali vezani za jedno računalo i bez automatskog stvaranja pohrane preko klastera. Zato Kubernetes pobjeđuje u fleksibilnosti za aplikacije sa stanjem, dok je Podman dovoljan za jedno računalo.

**Ispitna kuka:** Kubernetes koristi opisni zahtjev za pohranom (PVC) i razred pohrane (StorageClass) sa standardom CSI, pa se disk stvara automatski i neovisno o pozadinskom sustavu. Podman koristi lokalne volumene na jednom računalu, jednostavne ali bez automatskog stvaranja preko klastera. Kubernetes je fleksibilniji za aplikacije sa stanjem.

---

## 3. Operator (vlastiti tip objekta i kontroler) u odnosu na same Kubernetes manifeste

**Kontekst:** **Manifest** je tekstualna datoteka koja opisuje željeni objekt. **Kontroler** je program koji stalno usklađuje stvarno stanje sa željenim. **Operator** je kombinacija vlastitog tipa objekta (CRD, Custom Resource Definition) i kontrolera koji automatizira upravljanje nekom uslugom. **Day-2 operacije** su poslovi nakon prvog puštanja u rad: održavanje, nadogradnje, sigurnosne kopije, oporavak nakon kvara.

**Kako radi sa samim manifestima:** Opis željenog stanja piše se ručno te se ručno održava i mijenja.

**Kako radi s operatorom:** Definira se vlastiti tip objekta, na primjer „PostgresCluster" (klaster baze Postgres), a kontroler iza njega zna kako tu uslugu postaviti, raditi sigurnosne kopije, preuzeti vodstvo pri kvaru i nadograditi je. Operativno znanje čovjeka tako je upisano u program.

**Argumenti za operatore:** automatiziraju složene poslove održavanja (oporavak pri kvaru, sigurnosne kopije, skaliranje, nadogradnja verzije); nudi se jednostavno, opisno sučelje na višoj razini; manje ručnog rada za složene usluge sa stanjem.

**Argumenti za same manifeste:** jednostavnije, nema dodatnog kontrolera koji treba napisati i održavati; sve je vidljivo i pregledno; dovoljno za jednostavne aplikacije ili one bez stanja; operatore je zahtjevno napisati i održavati.

**Obranjiv stav:** Za **složene usluge sa stanjem** (baze podataka, sustavi za poruke) operator se isplati jer automatizira poslove održavanja koje sami manifesti ne mogu. Za **jednostavne aplikacije ili one bez stanja** sami manifesti su jednostavniji, a operator bi bio nepotrebno složen. Glavna napetost je: više automatizacije i kontrole nasuprot većem trudu.

**Ispitna kuka:** Operator je vlastiti tip objekta i kontroler koji upisuje poslove održavanja (oporavak, sigurnosne kopije, nadogradnje), pa nudi više automatizacije uz veći trud izrade. Sami manifesti su jednostavan opis željenog stanja, dovoljan dok složenost održavanja ne naraste.

---

## 4. Iskustvo developera: čisti Kubernetes (naredbe i manifesti) u odnosu na OpenShift S2I i razvojnu konzolu

**Kontekst:** **Iskustvo developera** znači koliko je razvojnom programeru lako i brzo doći od koda do pokrenute aplikacije. **kubectl** je naredbeni alat za upravljanje Kubernetesom. **S2I** (Source-to-Image, iz izvornog koda u sliku) je OpenShiftova značajka koja iz izvornog koda sama izgradi sliku i pokrene aplikaciju. **Builder slika** je posebna slika koja zna izgraditi aplikaciju.

**Kako radi čisti Kubernetes:** Opisi (manifesti) pišu se ručno, a slike se grade i šalju ručno; sve ide preko naredbenog alata. Fleksibilno, ali u više koraka.

**Kako radi OpenShift S2I i konzola:** Predaju se izvorni kod i builder slika, a OpenShift **sam izgradi sliku i pokrene aplikaciju**. Postoji i grafičko sučelje (GUI, grafičko sučelje) za razvojne programere.

**Argumenti za čisti Kubernetes:** puna kontrola, sve je vidljivo, prenosivo na bilo koju platformu, bez vezivanja za određenog proizvođača; razumljiv je svaki sloj.

**Argumenti za OpenShift S2I i konzolu:** brže uvođenje novih razvojnih programera (kod se pošalje i aplikacija radi), ugrađena izgradnja slike, grafičko sučelje snižava prag ulaska; usmjeren je na brzinu razvoja.

**Obranjiv stav:** Čisti Kubernetes je usmjeren na **kontrolu, fleksibilnost i prenosivost**; OpenShift S2I je usmjeren na **brzinu razvoja i niži prag** jer skriva izgradnju slike i pokretanje. Tim koji je nov u kontejnerima ili kojem je važna brzina dobit će više od OpenShifta; tim koji želi kontrolu i prenosivost odabrat će čisti Kubernetes.

**Ispitna kuka:** Naredbe i manifesti daju kontrolu i prenosivost, ali traže više ručnog rada. OpenShift S2I i konzola daju brzinu razvoja (od koda do pokrenute aplikacije) i niži prag, uz cijenu vezanosti za tu platformu.

---

## 5. Selidba aplikacije s OpenShifta na Kubernetes

**Kontekst:** OpenShift je Kubernetes s **dodanim** stvarima (vlastitim objektima i alatima). Selidba na obični (vanilla) Kubernetes znači zamijeniti te OpenShiftove posebnosti standardnim ekvivalentima.

**Što treba prevesti pri selidbi:**
- **Route** (OpenShiftov objekt za izlaganje aplikacije prema van) zamijeniti **Ingressom** (standardni Kubernetes objekt za isto).
- **DeploymentConfig** (stariji OpenShiftov način pokretanja) zamijeniti običnim **Deploymentom**.
- **ImageStream i S2I** (OpenShiftova izgradnja slika) zamijeniti **vanjskim alatom za izgradnju i registrom slika**.
- **SCC** (Security Context Constraints, OpenShiftova sigurnosna pravila) zamijeniti standardnim mehanizmom **PodSecurity** i sigurnosnim postavkama Poda.
- Posebne značajke OpenShifta i njegov alat se gube; strože sigurnosne postavke treba ponovno postaviti ručno.

**Argumenti za selidbu:** miče se ovisnost o jednom proizvođaču; niži trošak licenci; prenosivost na bilo koji Kubernetes.

**Argumenti protiv (cijena):** stvaran trud prevođenja OpenShiftovih objekata; gube se ugrađeni alati (izgradnja iz koda, grafička konzola, ugrađena integracija); ono što je OpenShift davao gotovo treba ponovno izgraditi; sigurnosno učvršćivanje postaje vlastiti posao.

**Obranjiv stav:** Selidba je **izvediva** jer je OpenShift u jezgri Kubernetes, ali **nije jednostavna** jer se posebni OpenShiftovi objekti (Route, DeploymentConfig, ImageStream, SCC) i ugrađeni alati moraju zamijeniti. Isplati se zbog troška i prenosivosti, ali je skupa u inženjerskom trudu i izgubljenoj udobnosti.

**Ispitna kuka:** Put s OpenShifta na Kubernetes je moguć jer je OpenShift Kubernetes s dodacima, ali treba zamijeniti Route Ingressom, DeploymentConfig Deploymentom, izgradnju iz koda vanjskim alatom i registrom, a OpenShiftova sigurnosna pravila standardnim mehanizmom. Udobnost proizvođača mijenja se za prenosivost i niži trošak.

---

## 6. Agenti za izgradnju (CI/CD) unutar Kubernetesa u odnosu na namjenska računala (VM)

**Kontekst:** **CI/CD** (Continuous Integration and Continuous Delivery, kontinuirana integracija i isporuka) je automatsko spajanje, provjera i objava koda. **Agent** ili **runner** je radnik koji obavlja te poslove izgradnje. Primjeri alata su Jenkins, GitLab i Tekton. **VM** (Virtual Machine, virtualni stroj) je softverski simulirano računalo. **Noisy-neighbor** (bučni susjed) je pojava kada jedan zahtjevan posao na dijeljenom računalu uspori druge.

**Kako radi unutar Kubernetesa:** Agenti se pokreću kao Podovi, povremeno i po potrebi.

**Kako radi na namjenskim računalima:** Agenti rade na stalnim, fiksnim računalima (virtualnim strojevima).

**Argumenti za pokretanje u Kubernetesu:** elastično skaliranje (agenti se dignu kad treba, a kad nema posla mogu pasti na nulu); učinkovito iskorištenje resursa; svaka izgradnja ide u svjež Pod (izolacija po poslu); sve na jednoj platformi.

**Argumenti za namjenska računala:** jača granica izolacije (virtualni stroj odvaja bolje nego kontejner); predvidive performanse; nema bučnog susjeda na dijeljenim računalima; jednostavnije za teške ili povlaštene izgradnje.

**Obranjiv stav:** Agenti u Kubernetesu pobjeđuju u **elastičnom skaliranju, učinkovitosti i svježoj izolaciji po poslu**, ali je kontejnerska izolacija slabija od one virtualnog stroja, a bučni susjed i borba za procesor na dijeljenim računalima stvaran su rizik za teške izgradnje. Zato je Kubernetes primjeren za promjenjiv, navalni rad izgradnje, a namjenska računala kad treba jaka izolacija, predvidive performanse ili povlaštene izgradnje.

**Ispitna kuka:** Agenti u Kubernetesu znače skaliranje, učinkovitost i svježu izolaciju po poslu. Namjenska računala znače jaču izolaciju i predvidive performanse, bez bučnog susjeda. Kubernetes za promjenjive navale, namjenska računala za teške ili izolacijski osjetljive poslove.

---

## 7. Kako se Kubernetes povezuje s oblakom i dinamičkim dodavanjem kapaciteta

**Kontekst:** **Oblak (cloud)** su tuđa računala koja se unajmljuju preko interneta. **Čvor (node)** je jedno računalo u klasteru. **Upravljana usluga (managed service)** znači da pružatelj oblaka održava dio sustava umjesto korisnika.

**Kroz što se Kubernetes povezuje s oblakom:**
- **cloud-controller-manager** (upravitelj povezan s oblakom) — kad se zatraži javno dostupan Servis, on stvori pravi uređaj za raspodjelu prometa u oblaku; povezuje i diskove te identitete (IAM, upravljanje identitetima i pristupom).
- **Cluster Autoscaler** (autoskaler klastera) — **dodaje cijela računala** kad Podovi ne mogu stati (stanje pokriveno u LO5, zadatak 27, kad pohlepan zahtjev premaši kapacitet), a kad ima viška, prazna računala uklanja.
- **HPA** (Horizontal Pod Autoscaler, vodoravni autoskaler Podova) — **dodaje Podove** kad opterećenje raste, prema mjerenjima (na primjer procesoru).
- **Upravljani Kubernetes** (na primjer kod velikih pružatelja oblaka) — skida brigu o održavanju upravljačkog dijela klastera.

**Obranjiv stav (ovdje je naglasak na objašnjenju, a ne na izboru strane):** Kubernetes se povezuje s oblakom preko zamjenjivih sučelja (za mrežu, pohranu i upravljanje) i preko autoskalera koji **pretvaraju „Podovi nemaju mjesta" u „dodaj još računala" automatski**. To je smisao rada u oblaku: opisno skaliranje i na razini Poda i na razini računala, bez ručnog dodavanja.

**Ispitna kuka:** Kubernetes se s oblakom veže preko upravitelja za oblak (raspodjela prometa, pohrana, identiteti), autoskalera klastera (Podovi bez mjesta dovode do novih računala) i autoskalera Podova (mjerenja dovode do više Podova). Rezultat je elastičan, opisno upravljan kapacitet.

---

## 8. Sigurnost i učvršćivanje: OpenShift u odnosu na obični Kubernetes — zašto regulirane industrije biraju OpenShift

**Kontekst:** **Učvršćivanje (hardening)** je smanjivanje mogućnosti za napad strožim postavkama. **Permisivno** znači da je po zadanim postavkama dopušteno mnogo (manje sigurno dok se ručno ne pooštri). **Regulirane industrije** (na primjer banke, zdravstvo) imaju zakonske zahtjeve za sigurnost.

**Zadane postavke OpenShifta:** **SCC** (Security Context Constraints, ograničenja sigurnosnog konteksta) ograničavaju što Pod smije raditi; kontejneri se **ne pokreću kao administrator (root)** po zadanim postavkama, nego kao običan korisnik s nasumičnim identifikatorom (UID, korisnički identifikator); ugrađena prijava i ovlasti (OAuth i RBAC); ugrađeno pretraživanje slika i registar; učvršćena osnova; uz to dolazi **podrška proizvođača i certifikati**.

**Zadane postavke običnog Kubernetesa:** popustljive (Pod se može pokrenuti kao administrator osim ako se ne doda mehanizam **PodSecurity** ili druga pravila); učvršćivanje se slaže ručno (PodSecurity razine, NetworkPolicy za odvajanje prometa, dodatni alati za pravila, pretraživanje slika).

**Zašto regulirane industrije biraju OpenShift:** učvršćeno po zadanim postavkama, uz certifikate i podršku proizvođača, smanjuje teret dokazivanja da je sustav siguran; lakše je zadovoljiti zakonske zahtjeve.

**Obranjiv stav:** OpenShift je **učvršćen po zadanim postavkama** (stroga sigurnosna pravila, pokretanje bez administratorskih ovlasti, ugrađena prijava), dok je obični Kubernetes **moguće osigurati, ali je popustljiv** dok se ručno ne pooštri. Regulirane industrije biraju OpenShift jer im **zadano učvršćivanje, podrška proizvođača i certifikati smanjuju rizik usklađivanja sa zakonom** — kupuju dokazanu sigurnosnu razinu umjesto da je sami grade.

**Ispitna kuka:** OpenShift dolazi s učvršćenim postavkama (stroga pravila, bez administratorskih ovlasti, ugrađena prijava) i podrškom te certifikatima proizvođača, što je pogodno za usklađivanje sa zakonom. Obični Kubernetes je popustljiv po zadanim postavkama i učvršćivanje se slaže ručno. Regulirane industrije biraju OpenShift da smanje rizik usklađivanja.

---

## 9. Krivulja učenja i potrebne vještine: OpenShift u odnosu na obični Kubernetes — pomaže li ili odmaže timu novom u kontejnerima

**Kontekst:** **Krivulja učenja** znači koliko je teško i dugo nešto naučiti. **Apstrakcija** je skrivanje složenosti iza jednostavnijeg sučelja.

**Kako OpenShift pomaže:** ima grafičko sučelje, izgradnju iz koda (Source-to-Image), i učvršćene zadane postavke, pa je manje koraka i manje prilike za pogrešku; brži je početak. Timu koji je nov u kontejnerima to skriva složenost i ubrzava prve rezultate.

**Kako OpenShift odmaže:** skrivena složenost znači da se ne nauči što se ispod događa, pa kad nešto pukne ili pri selidbi, nedostaje razumijevanje; vezanost za platformu; OpenShift ima i svoje pojmove (Route, SCC, DeploymentConfig) koje također treba naučiti; troši više resursa.

**Argumenti za obični Kubernetes:** na početku je teži, ali se uče temelji koji vrijede svugdje, pa je znanje prenosivo.

**Obranjiv stav:** Timu koji je nov u kontejnerima i želi brzo isporučiti, OpenShiftova apstrakcija na početku **pomaže** (manje koraka, učvršćeno, grafičko sučelje). Dugoročno može **odmoći** jer skriva temelje, pa pri kvaru ili selidbi nedostaje razumijevanje. Ovisi je li potreban brz početak (OpenShift) ili dublje, prenosivo znanje (obični Kubernetes). Treba reći i da OpenShift ima vlastite pojmove, pa nije da s njim nema što učiti.

**Ispitna kuka:** OpenShift skrivanjem složenosti ubrzava početak timu novom u kontejnerima, ali dugoročno može odmoći jer se ne nauče temelji. Obični Kubernetes je teži na početku, ali daje prenosivo znanje. Izbor ovisi o tome treba li brzina ili dubina.

---

## 10. Resursni otisak punog Kubernetesa u odnosu na k3s za malu lokalnu postavku — kada je puni Kubernetes pretjeran

**Kontekst:** **k3s** je lagana inačica Kubernetesa (jedna mala datoteka, troši znatno manje resursa). **Resursni otisak** znači koliko procesora i memorije nešto troši. **Lokalna postavka (on-premise)** znači rad na vlastitoj opremi, a ne u tuđem oblaku.

**Puni Kubernetes:** ima više dijelova upravljačkog sloja, veće zahtjeve za procesorom i memorijom, i složeniju postavku; zauzvrat nudi pun skup mogućnosti i predstavlja standard.

**k3s:** dolazi kao jedna mala datoteka, troši mnogo manje memorije i procesora, brzo se postavlja; sadrži sve bitno za rad, ali pojednostavljeno; pogodan za rub mreže, male poslužitelje, uređaje interneta stvari i razvoj. Važno: k3s je i dalje **pravi** Kubernetes (ista sučelja), samo lakši.

**Obranjiv stav:** Za malu lokalnu postavku k3s je obično bolji jer troši mnogo manje resursa i lakše se postavlja, a daje gotovo sve što treba. Puni Kubernetes je pretjeran kada je malo računala, malo resursa, jednostavne potrebe i nema velikog tima za održavanje. Puni Kubernetes vrijedi kada stvarno trebaju njegove napredne mogućnosti i velika skala.

**Ispitna kuka:** Puni Kubernetes troši više i složeniji je; k3s je lagan, brz za postaviti i i dalje pravi Kubernetes. Za male lokalne postavke k3s je obično dovoljan, a puni Kubernetes je pretjeran kad su resursi i potrebe mali.

---

## 11. Treba li mali tim koristiti Docker Compose na jednom računalu ili odmah prijeći na Kubernetes (složenost, potrebe za skaliranjem, otpornost)

**Kontekst:** **Docker Compose** je alat koji jednom datotekom opisuje aplikaciju od više kontejnera na jednom računalu i diže ih zajedno. **Otpornost (resilience)** je sposobnost oporavka od kvara. **Jedno računalo (single host)** znači da sve radi na jednom poslužitelju.

**Argumenti za Docker Compose:** vrlo jednostavno, jedna datoteka, brzo dizanje, mali teret održavanja; savršeno za jedno računalo i mali tim.

**Argumenti protiv Compose i za Kubernetes:** jedno računalo je jedna točka kvara (ako ono padne, padne sve); nema automatskog oporavka preko više računala; nema lakog skaliranja preko više računala; ograničeno je.

**Argumenti za Kubernetes:** otpornost (kad računalo padne, Podovi se prerasporede na druga), skaliranje i samoizlječenje; ali za mali tim donosi veliku složenost i teret održavanja.

**Obranjiv stav:** Za mali tim s jednostavnom aplikacijom i bez velikih potreba za skaliranjem, Docker Compose na jednom računalu je razuman izbor jer je jednostavan i jeftin. Skok na Kubernetes ima smisla kada zatrebaju otpornost (preživjeti kvar računala), skaliranje preko više računala ili samoizlječenje. Glavna napetost je jednostavnost (Compose) nasuprot otpornosti i skalabilnosti (Kubernetes). Kubernetes se ne uvodi zbog njega samog, nego kada ga potrebe traže.

**Ispitna kuka:** Docker Compose na jednom računalu je jednostavan i jeftin, ali je to jedna točka kvara bez lakog skaliranja. Kubernetes daje otpornost i skaliranje uz veliku složenost. Mali tim ostaje na Compose dok mu ne zatrebaju otpornost ili skaliranje.

---

## 12. Vezanost za proizvođača: Kubernetes (otvoreni kod, zaklada CNCF) u odnosu na OpenShift (Red Hat) — utjecaj na dugoročnu slobodu

**Kontekst:** **Vezanost za proizvođača (vendor lock-in)** znači da je teško i skupo prijeći s jednog proizvođača na drugog. **Otvoreni kod (open source)** znači da je softver javno dostupan i svatko ga smije koristiti i mijenjati. **CNCF** (Cloud Native Computing Foundation) je neutralna zaklada koja održava Kubernetes i srodne projekte.

**Kubernetes:** otvoreni standard koji održava neutralna zaklada; radi kod svih velikih pružatelja oblaka; nema vezanosti za jednog proizvođača; veća dugoročna sloboda i mogućnost selidbe.

**OpenShift:** proizvod jedne tvrtke (Red Hat); dobiva se podrška i integracija, ali nastaje vezanost za njihove posebnosti (Route, SCC, izgradnju iz koda) i za licencu; teže je otići.

**Obranjiv stav:** Kubernetes kao otvoreni standard daje veću dugoročnu slobodu jer nema vezanosti za jednog proizvođača i selidba je moguća. OpenShift više veže (svojim posebnostima i licencom), ali zauzvrat daje podršku i integrirane alate. Napetost je sloboda i prenosivost (Kubernetes) nasuprot udobnosti i podršci uz vezanost (OpenShift). Treba dodati i da se blaža vezanost može stvoriti i na samom Kubernetesu ako se previše osloni na posebnosti jednog pružatelja oblaka.

**Ispitna kuka:** Kubernetes je otvoreni standard pa daje slobodu i prenosivost; OpenShift je proizvod jedne tvrtke pa veže uz sebe, ali nudi podršku i integraciju. Izbor je između dugoročne slobode i udobnosti uz vezanost.

---

## 13. Obrana preporuke OpenShifta umjesto običnog Kubernetesa za tvrtku koja želi integriranu, podržanu platformu s ugrađenim alatima za razvoj i održavanje

**Kontekst:** **Integrirana platforma** znači da je sve na jednom mjestu i radi zajedno (izgradnja, sučelje, sigurnost, registar slika), umjesto da se dijelovi slažu ručno.

**Argumenti za OpenShift (jer pitanje traži njegovu obranu):** sve na jednom mjestu i usklađeno; **podrška proizvođača** (netko koga se zove kad nešto pukne); ugrađeni alati i za razvoj i za održavanje; **učvršćeno po zadanim postavkama**; brži početak jer se sustav ne mora sam sastavljati i osiguravati. Za tvrtku koja izričito želi gotov, podržan paket, to je velika vrijednost.

**Kratko priznanje cijene:** trošak licence, vezanost za proizvođača i veći resursni zahtjevi.

**Obranjiv stav:** Za tvrtku koja izričito želi integriranu, podržanu platformu s ugrađenim alatima, OpenShift je opravdan izbor jer dobiva gotov, učvršćen i podržan paket, uz brži početak i nekoga koga zove kad zatreba. Cijena su licenca i vezanost, ali tvrtka koja vrednuje podršku i integraciju to svjesno prihvaća. Pošto pitanje traži obranu OpenShifta, naglasak je na njegovim prednostima uz kratko priznanje cijene.

**Ispitna kuka:** Za tvrtku koja želi sve na jednom mjestu, s podrškom i ugrađenim alatima, OpenShift je opravdan jer daje gotov, učvršćen i podržan paket uz brži početak. Cijena su licenca i vezanost, što takva tvrtka svjesno prihvaća.

---

## 14. Dodjela pohrane u Kubernetesu u odnosu na klasične poslove na virtualnim strojevima

**Kontekst:** **Dodjela pohrane (provisioniranje)** je priprema i dodjeljivanje diskova. **Posao na virtualnom stroju** je aplikacija koja radi na klasičnom virtualnom računalu, bez Kubernetesa.

**Kod klasičnih virtualnih strojeva:** pohrana se obično priprema ručno ili unaprijed (administrator dodijeli disk određenom računalu); statična je i vezana za to računalo.

**U Kubernetesu:** aplikacija opisno traži pohranu (zahtjevom PVC), a disk se stvori automatski (dinamičko provisioniranje preko razreda pohrane i sučelja CSI); prenosivo je i na zahtjev, i neovisno o pojedinom računalu, jer Pod može završiti na bilo kojem računalu, a disk ga slijedi.

**Obranjiv stav:** Kod klasičnih virtualnih strojeva pohrana se tipično priprema ručno i veže za određeno računalo. Kubernetes to čini opisno i automatski, pa se pohrana traži, ona se stvori i prati Pod gdje god on bio. Kubernetesov pristup je fleksibilniji i prikladniji za okolinu gdje se aplikacije sele između računala; klasični pristup je jednostavniji i predvidiviji za stalne, fiksne poslužitelje.

**Ispitna kuka:** Kod virtualnih strojeva pohrana se priprema ručno i veže se za računalo. Kubernetes je dodjeljuje opisno i automatski, pa prati Pod gdje god bio. Kubernetes je fleksibilniji za pokretne aplikacije, klasični pristup predvidiviji za fiksne poslužitelje.

---

## 15. Mlada tvrtka s dva inženjera mora brzo i jeftino isporučiti proizvod u kontejnerima — preporuka i obrazloženje po trošku i teretu održavanja

**Kontekst:** **Operativni teret (operational overhead)** je koliko vremena i truda održavanje oduzima. **Mlada tvrtka (startup)** je tvrtka u ranoj fazi s malo ljudi i novca.

**Argumenti za jednostavno rješenje (Docker Compose ili Podman na jednom računalu, ili upravljana platforma):** jeftino, malo održavanja, brzo do proizvoda; dvoje inženjera ne troši vrijeme na složen klaster.

**Argumenti protiv samostalno održavanog Kubernetesa:** za dvoje ljudi je prevelik teret; oduzima vrijeme; skuplji je jer traži i znanje i vrijeme.

**Nijansa:** ako Kubernetes baš zatreba, primjereno je uzeti **upravljani** Kubernetes (gdje pružatelj oblaka održava upravljački dio), da se smanji teret. Ali za prvo, brzo i jeftino, jednostavnije rješenje je bolje.

**Obranjiv stav:** Za mladu tvrtku s dva inženjera koja mora brzo i jeftino isporučiti, preporuka je **jednostavno rješenje**: Docker Compose ili Podman na jednom računalu, ili upravljana platforma, jer je jeftino, traži malo održavanja i brzo vodi do proizvoda. Samostalno održavan Kubernetes bio bi prevelik teret za dvoje ljudi. Ako kasnije zatreba Kubernetes, primjereno je uzeti **upravljanu** inačicu da pružatelj nosi teret. Načelo je uskladiti alat s veličinom tima i potrebama, a ne uvoditi složenost prerano.

**Ispitna kuka:** Dvoje inženjera koji moraju brzo i jeftino isporučiti trebaju jednostavno rješenje (Compose ili Podman na jednom računalu, ili upravljanu platformu), jer samostalni Kubernetes nosi prevelik teret. Ako Kubernetes zatreba, primjereno je da bude upravljani.

---

## 16. Docker u odnosu na Podman: ustroj (s pozadinskom uslugom ili bez nje) i sigurnost bez administratorskih ovlasti — zašto bi tim koji pazi na sigurnost mogao izabrati Podman

**Kontekst:** **Pozadinska usluga (daemon)** je program koji stalno radi u pozadini i kroz koji idu svi zahtjevi. **Bez pozadinske usluge (daemonless)** znači da takvog programa nema. **Bez administratorskih ovlasti (rootless)** znači raditi kao obični korisnik, a ne kao najmoćniji korisnik. **Administrator (root)** je korisnik koji smije sve na računalu.

**Kako radi Docker:** preko **stalne pozadinske usluge** koja tipično radi s administratorskim ovlastima; svi zahtjevi prolaze kroz nju. To je jedna središnja, moćna točka, pa ako je netko zlorabi, ima velike ovlasti.

**Kako radi Podman:** **nema stalne pozadinske usluge**; svaka naredba pokreće proces izravno; lako radi **bez administratorskih ovlasti**, kao obični korisnik.

**Zašto bi tim koji pazi na sigurnost izabrao Podman:** bez moćne stalne usluge nema te jedne opasne središnje točke kao mete; pokretanje bez administratorskih ovlasti znači da, i ako napadač provali u kontejner, ima manje ovlasti na računalu; ukupno je manja površina za napad.

**Obranjiv stav:** Docker radi preko stalne pozadinske usluge koja često ima administratorske ovlasti, dok Podman radi bez takve usluge i lako bez administratorskih ovlasti. Tim koji pazi na sigurnost često bira Podman jer **nema moćne središnje usluge kao mete** i jer **rad bez administratorskih ovlasti smanjuje štetu** ako kontejner bude probijen. Docker je zato dugo bio praktičan i raširen, ali Podmanov ustroj je sigurnosno povoljniji.

**Ispitna kuka:** Docker ima stalnu pozadinsku uslugu s velikim ovlastima (jedna moćna meta); Podman je bez takve usluge i radi bez administratorskih ovlasti. Tim koji pazi na sigurnost bira Podman jer nema te moćne mete i jer probijeni kontejner ima manje ovlasti.

---

## Okvir za odlučivanje — kako odgovoriti na svako pitanje „koje rješenje i zašto"

> Ovaj okvir rješava cijelu skupinu pitanja koja traže preporuku rješenja: 10, 11, 13, 15, 17, 18, 19, 20, 21, 22 i 23. Umjesto učenja svakog zasebno, dovoljno je naučiti **jedan način razmišljanja** i primijeniti ga na bilo koju formulaciju.

### Korak 1 — iz zadatka izvući faktore

Kod svakog scenarija valja razmotriti sljedeće. Odgovori određuju rješenje.

1. **Veličina tima i znanje:** koliko ljudi i koliko iskustva? Mali tim ili malo iskustva znači da treba jednostavnije rješenje.
2. **Skala i rast:** koliko kontejnera i računala, i hoće li broj rasti? Mala i stabilna postavka znači jednostavno rješenje; velika ili rastuća znači Kubernetes.
3. **Otpornost:** smije li sve pasti ako jedno računalo padne? Ako treba preživjeti kvar računala, treba Kubernetes (radi preko više računala). Ako ne treba, jedno računalo je dovoljno.
4. **Promet:** je li promet stalan ili nagao i nepredvidiv? Nagao promet znači Kubernetes s automatskim skaliranjem.
5. **Sigurnost i zakon:** je li riječ o reguliranoj industriji sa strogim zahtjevima (na primjer banka, zdravstvo)? Ako da, naginje OpenShiftu (učvršćen po zadanim postavkama, certifikati, podrška).
6. **Trošak i teret održavanja:** koliko vremena i novca smiju otići na održavanje? Malo znači jednostavno ili upravljano rješenje; ako je spremnost na ulaganje veća, dolazi u obzir samostalno održavan Kubernetes.
7. **Podrška proizvođača:** je li poželjan netko koga se zove kad nešto pukne? Ako da, OpenShift ili upravljani Kubernetes; ako ne, dovoljan je otvoreni Kubernetes ili jednostavno rješenje.

### Korak 2 — ljestvica rješenja od najjednostavnijeg do najsloženijeg

Svako sljedeće rješenje daje više mogućnosti, ali traži više znanja i truda. Bira se najniže koje zadovoljava potrebe iz Koraka 1.

- **Jedan kontejner s Podmanom ili Dockerom na jednom računalu** — za vrlo male i jednostavne stvari.
- **Docker Compose na jednom računalu** — više kontejnera koji rade zajedno, jedan tim, jedno računalo. (Pojam: Docker Compose je alat koji jednom datotekom opisuje i diže više kontejnera.)
- **k3s, to jest lagani Kubernetes** — kada trebaju mogućnosti Kubernetesa, ali na maloj ili lokalnoj postavci s malo resursa.
- **Upravljani Kubernetes** — kada treba Kubernetes, ali nema tima za održavanje upravljačkog dijela; pružatelj oblaka ga održava umjesto korisnika. (Pojam: upravljani znači da tuđa tvrtka održava dio sustava.)
- **Samostalno održavan puni Kubernetes** — velika skala, postoji tim i znanje, želi se puna kontrola. (Pojam: samostalno održavan znači da se sve održava samostalno.)
- **OpenShift** — kada se želi integrirana, podržana i učvršćena platforma s ugrađenim alatima; često biraju regulirane industrije i tvrtke koje vrednuju podršku.

### Korak 3 — kako oblikovati odgovor na ispitu

1. Iz scenarija se izvuku faktori (veličina tima, skala, otpornost, sigurnost, trošak).
2. Povežu se s jednim rješenjem s ljestvice.
3. Izbor se obrazloži kroz te faktore uz pošteno navođenje cijene ili kompromisa druge strane (time se pokazuje uvid u obje strane).
4. Odgovor se zaključi jasnom, jednom rečenicom preporuke.

### Brza karta: koji faktor presuđuje u kojem pitanju

- **10 (puni Kubernetes ili k3s):** presuđuju resursi i veličina postavke. Mala lokalna postavka znači k3s.
- **11 (Docker Compose ili Kubernetes):** presuđuju otpornost i skaliranje. Bez tih potreba znači Compose.
- **13 (preporuči OpenShift):** presuđuju želja za integracijom i podrškom. Znači OpenShift.
- **15 (mlada tvrtka, dvoje ljudi):** presuđuju trošak i teret održavanja. Znači jednostavno ili upravljano rješenje.
- **17 (upravljani ili samostalni Kubernetes):** presuđuje koliko ima tima i vremena za održavanje (nadogradnje, skaliranje računala, sigurnosne kopije). Malo znači upravljani.
- **18 (nagao globalni promet):** presuđuje promet. Znači Kubernetes s automatskim skaliranjem.
- **19 (prerastao Docker Swarm ili jedno računalo):** presuđuju otpornost i skala koje su nadišle staro rješenje. Znači selidba na Kubernetes, uz trud selidbe.
- **20 i 21 (Kubernetes kao izbor):** presuđuju orkestracija, otpornost, ekosustav i skalabilnost. Za mikroservise i velike sustave, da.
- **22 i 23 (OpenShift kao izbor):** isto kao 20 i 21, ali s naglaskom na integraciju, podršku i učvršćenu sigurnost.

### Četiri stupa koja se stalno spominju (za pitanja 20 do 23)

Pitanja 20 do 23 izričito traže ova četiri svojstva. Evo kratke rečenice po svakom, za pamćenje.

- **Orkestracija i otpornost:** Kubernetes sam raspoređuje kontejnere i oporavlja ih kad računalo padne. OpenShift to isto radi (jer je Kubernetes ispod), uz dodatne alate.
- **Ekosustav i zajednica:** Kubernetes ima golemu zajednicu i mnoštvo alata jer je otvoreni standard. OpenShift se na to oslanja i dodaje podršku proizvođača.
- **Skalabilnost:** oba skaliraju i kontejnere i računala (kroz automatsko skaliranje). To je glavna prednost za velike i promjenjive sustave.
- **Bogatstvo mogućnosti i sigurnost:** Kubernetes ima širok skup mogućnosti koje se slažu samostalno. OpenShift dolazi s više toga unaprijed posloženog i učvršćenog.

**Jedno upozorenje za obranu:** odgovor nikad ne glasi „uvijek Kubernetes" ili „uvijek OpenShift". Snažan odgovor uvijek glasi „ovisi o sljedećim faktorima", pa se navedu koji, i tek onda zauzima jasan stav za taj scenarij. To pokazuje razumijevanje kompromisa, a ne napamet naučeno guranje jednog rješenja.

---

## 17. Teret održavanja (nadogradnje klastera, skaliranje računala, sigurnosne kopije): samostalno održavan Kubernetes u odnosu na upravljanu uslugu

**Kontekst:** **Teret održavanja nakon puštanja u rad (day-2 operacije)** su poslovi koji traju dok sustav radi: nadogradnje klastera, dodavanje i uklanjanje računala, sigurnosne kopije. **Samostalno održavan** znači da se sve to radi samostalno. **Upravljana usluga** znači da pružatelj oblaka održava dio sustava umjesto korisnika.

**Samostalno održavan Kubernetes:** nadogradnje, skaliranje računala i sigurnosne kopije rade se samostalno; postoji pun nadzor, ali to traži velik trud i znanje.

**Upravljani Kubernetes:** pružatelj oblaka održava upravljački dio klastera (nadogradnje i dostupnost), a briga ostaje samo za vlastite aplikacije; manje truda, ali manje kontrole i plaća se usluga.

**Obranjiv stav:** Samostalno održavan Kubernetes daje punu kontrolu, ali nosi velik teret održavanja (nadogradnje, računala i sigurnosne kopije rade ljudi). Upravljana usluga prebacuje taj teret na pružatelja, pa ostaje briga samo za aplikacije, uz cijenu manje kontrole i plaćanja usluge. Za većinu timova bez velikog odjela za održavanje, upravljana usluga smanjuje teret; samostalno održavanje je primjereno kad treba puna kontrola ili kad postoji tim za to.

**Ispitna kuka:** Samostalno održavan Kubernetes znači da se nadogradnje, računala i sigurnosne kopije rade samostalno (puna kontrola, velik trud). Upravljani prebacuje to na pružatelja (manje truda, manje kontrole, plaća se). Tim bez velikog odjela za održavanje bira upravljani.

---

## 18. Tvrtka za emitiranje medija očekuje nagao i nepredvidiv globalni promet — preporuka rješenja u kontejnerima

**Kontekst:** **Emitiranje medija (media-streaming)** je usluga koja preko interneta šalje video ili zvuk. **Nagao promet (spiky)** znači da broj korisnika naglo skače i pada. **Globalan** znači da su korisnici po cijelom svijetu.

**Preporuka:** **Kubernetes**, najbolje **upravljani** u oblaku.

**Obrazloženje:** glavni faktor je promet. Nagao i nepredvidiv promet je upravo slučaj za **automatsko skaliranje** — Kubernetes dodaje Podove (preko vodoravnog autoskalera Podova) i cijela računala (preko autoskalera klastera) kad promet raste, a uklanja ih kad padne. Uz to daje otpornost i može raditi u više regija svijeta, što globalni korisnici trebaju. Jednostavnija rješenja, poput jednog računala ili Docker Composea, ne bi izdržala nagle skokove.

**Obranjiv stav:** Za tvrtku koja emitira medije uz nagao, nepredvidiv i globalan promet, preporuka je Kubernetes, najbolje upravljani u oblaku, jer automatsko skaliranje prati skokove (dodaje i uklanja Podove i računala prema prometu), daje otpornost i može raditi u više regija svijeta. Jednostavnija rješenja ne bi izdržala nagle skokove. Presudni faktor je upravo promet.

**Ispitna kuka:** Nagao i nepredvidiv globalni promet traži automatsko skaliranje, pa je odgovor Kubernetes (najbolje upravljani u oblaku) jer dodaje i uklanja Podove i računala prema prometu, daje otpornost i radi u više regija. Jednostavna rješenja ne bi izdržala skokove.

---

## 19. Treba li tvrtka koja je prerasla Docker Swarm (ili jedno računalo s Composeom) prijeći na Kubernetes — trud selidbe u odnosu na dugoročnu korist

**Kontekst:** **Docker Swarm** je Dockerov vlastiti, jednostavni sustav za orkestraciju preko više računala. **Prerasti (outgrow)** znači da staro rješenje više ne zadovoljava potrebe.

**Argumenti za selidbu:** Kubernetes daje upravo ono što su prerasli — bolju otpornost, skaliranje, bogat ekosustav i status industrijskog standarda (lakše nalaženje stručnjaka, alata i podrške).

**Argumenti protiv (cijena):** selidba je stvaran trud (prepisati opise, naučiti nove pojmove, postaviti klaster); kratkoročno usporava rad.

**Nijansa:** Docker Swarm je uz to na sporednom kolosijeku (sve manje razvoja), pa dugoročno Kubernetes svejedno postaje sigurniji izbor.

**Obranjiv stav:** Ako je tvrtka stvarno prerasla Docker Swarm ili jedno računalo (treba veću otpornost i skalu), selidba na Kubernetes se dugoročno isplati jer dobiva otpornost, skaliranje, ekosustav i standard. Cijena je stvaran trud selidbe i kratkoročno usporavanje. Pošto je Swarm uz to slabije podržan, dugoročna korist nadmašuje jednokratni trošak. Ali ako potrebe nisu stvarno prerasle staro rješenje, selidba bi bila nepotrebna složenost.

**Ispitna kuka:** Ako su potrebe stvarno prerasle Swarm ili jedno računalo (otpornost, skala), selidba na Kubernetes se isplati dugoročno unatoč trudu, tim više što je Swarm slabije podržan. Ako potrebe nisu prerasle staro, selidba je nepotrebna složenost.

---

## 20. Kubernetes kao izbor za arhitekturu mikroservisa — naglasak na orkestraciji, otpornosti i ekosustavu

**Kontekst:** **Arhitektura mikroservisa** znači da je aplikacija razbijena na mnogo malih, neovisnih servisa koji surađuju (umjesto jedne velike cjeline).

**Stav: Da.** Po tri tražena svojstva:
- **Orkestracija:** Kubernetes sam raspoređuje i povezuje mnoštvo servisa, što mikroservisi upravo traže.
- **Otpornost:** kad neki servis ili računalo padne, Kubernetes ga oporavi i prerasporedi.
- **Ekosustav:** golem skup gotovih alata (mreža, praćenje, sigurnost) izgrađen baš za ovaj način rada.

**Kompromis:** složenost i teret održavanja; ako je servisa vrlo malo, može biti pretjeran.

**Obranjiv stav:** Da, za arhitekturu mikroservisa Kubernetes je primjeren jer njegova orkestracija upravlja mnoštvom neovisnih servisa, otpornost ih oporavlja pri kvaru, a ekosustav nudi gotove alate baš za taj način rada. Kompromis je složenost; ako je servisa vrlo malo, jednostavnije rješenje može biti dovoljno. Ali za pravu mikroservisnu arhitekturu prednosti nadmašuju složenost.

**Ispitna kuka:** Da, za mikroservise Kubernetes jer njegova orkestracija upravlja mnoštvom servisa, otpornost ih oporavlja, a ekosustav nudi gotove alate za taj način rada. Kompromis je složenost, opravdan kad servisa ima više.

---

## 21. Kubernetes kao izbor za sustave razine velikih tvrtki — naglasak na skalabilnosti, podršci zajednice i bogatstvu mogućnosti

**Kontekst:** **Razina velikih tvrtki (enterprise-grade)** znači razinu velikih, ozbiljnih poslovnih sustava (visoka pouzdanost, velika skala, stroge potrebe).

**Stav: Da.** Po tri tražena svojstva:
- **Skalabilnost:** Kubernetes skalira i Podove i računala, pa nosi velike sustave.
- **Podrška zajednice:** golema zajednica daje mnogo znanja i alata, lako nalaženje stručnjaka i dugoročnu održivost.
- **Bogatstvo mogućnosti:** širok skup ugrađenih i dodatnih mogućnosti za složene potrebe.

**Kompromis:** traži znanje i tim za održavanje; za male sustave je pretjeran.

**Obranjiv stav:** Da, za sustave razine velikih tvrtki Kubernetes je primjeren jer skalira na veliko, ima golemu zajednicu (pa i dugoročnu održivost i lako nalaženje stručnjaka) i bogat skup mogućnosti za složene potrebe. Kompromis je da traži znanje i tim za održavanje; za male sustave bio bi pretjeran. Za velike, ozbiljne sustave to je standardni i opravdan izbor.

**Ispitna kuka:** Da, za velike poslovne sustave Kubernetes jer skalira na veliko, ima golemu zajednicu (znanje, stručnjaci, održivost) i bogat skup mogućnosti. Kompromis je potreba za znanjem i timom, opravdana na velikoj skali.

---

## 22. OpenShift kao izbor za arhitekturu mikroservisa — naglasak na orkestraciji, otpornosti i ekosustavu

**Kontekst:** Pitanje traži procjenu OpenShifta za mikroservise po orkestraciji, otpornosti i ekosustavu. **Arhitektura mikroservisa** znači aplikaciju razbijenu na mnogo malih, neovisnih servisa.

**Stav: Da, uz uvjet.** OpenShift je Kubernetes ispod, pa ima **istu orkestraciju i otpornost** te pristup **istom ekosustavu**, a uz to dodaje integrirane alate i podršku proizvođača, što ubrzava rad tima. Za mikroservise je dobar izbor, osobito ako tim želi integriranu, podržanu platformu.

**Kompromis:** trošak licence i vezanost za proizvođača.

**Obranjiv stav:** Da, za mikroservise OpenShift je primjeren jer ima istu orkestraciju i otpornost kao Kubernetes (jest Kubernetes ispod), pristup istom ekosustavu, te dodatno integrirane alate i podršku proizvođača, što ubrzava rad tima. Kompromis su trošak licence i vezanost. Primjeren je kad je tvrtki važna integracija i podrška; ako su važnije sloboda i niži trošak, primjereniji je obični Kubernetes.

**Ispitna kuka:** Da, za mikroservise OpenShift ima istu orkestraciju, otpornost i ekosustav kao Kubernetes (jer to i jest), plus integrirane alate i podršku. Kompromis su trošak i vezanost; primjeren je kad su važni integracija i podrška.

---

## 23. OpenShift kao izbor za sustave razine velikih tvrtki — naglasak na skalabilnosti, podršci zajednice i bogatstvu mogućnosti

**Kontekst:** Pitanje traži procjenu OpenShifta za sustave razine velikih tvrtki po skalabilnosti, podršci zajednice i bogatstvu mogućnosti. **Razina velikih tvrtki (enterprise-grade)** znači velike, ozbiljne poslovne sustave sa strogim potrebama.

**Stav: Da, vrlo prikladno.** Po tri tražena svojstva:
- **Skalabilnost:** skalira poput Kubernetesa (jest Kubernetes ispod).
- **Podrška:** naslanja se na veliku zajednicu Kubernetesa i dodaje **podršku proizvođača (Red Hat)**.
- **Bogatstvo mogućnosti:** bogat skup mogućnosti uz **učvršćenu sigurnost** unaprijed posloženu.

Velike i regulirane tvrtke upravo to traže (pouzdanost, sigurnost, podrška).

**Kompromis:** trošak licence i vezanost za proizvođača.

**Obranjiv stav:** Da, za sustave razine velikih tvrtki OpenShift je vrlo prikladan jer skalira poput Kubernetesa, naslanja se na njegovu veliku zajednicu i dodaje podršku proizvođača, te nosi bogat skup mogućnosti s učvršćenom sigurnošću, što velike i regulirane tvrtke cijene. Kompromis su trošak i vezanost, koje takve tvrtke obično svjesno prihvaćaju za podršku i sigurnost.

**Ispitna kuka:** Da, za velike poslovne sustave OpenShift skalira poput Kubernetesa, dodaje podršku proizvođača na veliku zajednicu i nosi bogate mogućnosti s učvršćenom sigurnošću. Kompromis su trošak i vezanost, koje takve tvrtke svjesno prihvaćaju.

---

## Dodatne teorijske teme — pregled Docker Swarma i k3s-a

> Ove dvije teme nisu među dvadeset i tri pitanja iz PDF-a, ali ih profesor izričito navodi kao dio LO6 (LO6 je čisto teorijski i pokriva širi skup tema, uključujući pregled Docker Swarma i k3s-a). Pisano u istom pristupačnom stilu, bez simbola.

### Docker Swarm — pregled

**Pojam:** Docker Swarm je Dockerov vlastiti, ugrađeni sustav za orkestraciju kontejnera preko više računala. Orkestracija znači automatsko raspoređivanje, pokretanje, skaliranje i oporavak kontejnera. Više računala se udruži u takozvani roj (swarm).

**Kako radi:**
- Računala u roju imaju dvije uloge: **upravljačka računala** (manager, donose odluke i drže željeno stanje) i **radna računala** (worker, pokreću kontejnere).
- Aplikacija se opisuje kao **servis** sa željenim brojem primjeraka (replika). Swarm rasporedi te replike po radnim računalima i pazi da ih uvijek ima onoliko koliko je zadano. Ako neka padne, podigne novu (samoizlječenje).
- Koristi iste Docker naredbe i pojmove koji se već poznaju, pa je prag ulaska nizak.
- Ima ugrađeno otkrivanje servisa, raspodjelu prometa i preklopnu mrežu preko više računala.

**Odnos prema Kubernetesu:** Docker Swarm je **jednostavniji za postaviti i naučiti**, ali ima **manje mogućnosti** od Kubernetesa (manje opcija za skaliranje, mreže, pravila i proširenja). Kubernetes je složeniji, ali bogatiji i postao je industrijski standard.

**Zašto je danas uglavnom nadiđen:** Zajednica i razvoj uglavnom su se preselili na Kubernetes. Swarm dobiva malo novih značajki i sve ga manje organizacija bira za nove sustave, pa se često spominje kao naslijeđe iz kojeg se seli na Kubernetes (vidi pitanje 19).

**Kada ga izabrati:** Docker Swarm je razuman za male, jednostavne postavke i timove koji već znaju Docker i žele brz, lak ulaz u orkestraciju preko nekoliko računala. Za veće, složenije ili dugoročne sustave Kubernetes je sigurniji izbor zbog mogućnosti i podrške zajednice.

**Ispitna kuka:** Docker Swarm je Dockerov jednostavni ugrađeni orkestrator (upravljačka i radna računala, servisi s replikama, iste Docker naredbe), lak za naučiti ali siromašniji od Kubernetesa i danas uglavnom nadiđen.

### k3s — pregled

**Pojam:** k3s je **lagana, potpuna inačica Kubernetesa**. To je i dalje pravi, certificirani Kubernetes (ista sučelja i naredbe, isti kubectl), samo zapakiran tako da troši mnogo manje resursa. Razvio ga je Rancher (danas dio tvrtke SUSE).

**Po čemu se razlikuje od punog Kubernetesa:**
- Dolazi kao **jedna mala izvršna datoteka** i brzo se postavlja.
- Troši **znatno manje memorije i procesora**, jer su neki teški dijelovi zamijenjeni lakšima (na primjer jednostavnija ugrađena pohrana stanja umjesto teže zadane), a nepotrebni dodaci su izbačeni.
- Iako je lakši po resursima, **prema aplikacijama se ponaša kao obični Kubernetes** — ono što se napiše za k3s radi i na punom Kubernetesu i obrnuto.

**Tipične namjene:**
- **Rub mreže** i mali, udaljeni uređaji.
- **Internet stvari** (uređaji s malo resursa).
- **Razvoj i učenje** (brz, lagan klaster na vlastitom računalu).
- **Mali lokalni poslužitelji**, gdje bi puni Kubernetes bio pretjeran.

**Odnos prema punom Kubernetesu:** k3s nije drugi, odvojeni sustav, nego **isti Kubernetes u lakšem pakiranju**. Bira se kad trebaju mogućnosti Kubernetesa, ali ne i njegov puni resursni teret.

**Kada ga izabrati:** za male, lokalne ili resursno ograničene postavke k3s daje gotovo sve što i puni Kubernetes uz mnogo manji teret, pa je često bolji izbor. Puni Kubernetes vrijedi kad se stvarno treba njegova puna skala i napredne mogućnosti (vidi pitanje 10).

**Ispitna kuka:** k3s je lagana ali potpuna inačica Kubernetesa (jedna mala datoteka, mnogo manje resursa, isti kubectl), za rub mreže, internet stvari, razvoj i male poslužitelje. Puni Kubernetes je pretjeran tamo gdje k3s zadovoljava.

---

# LO6 — gotovo: sva pitanja (1–23), okvir za odlučivanje, te pregledi Docker Swarma i k3s-a

Temeljni pojmovi (container, image, Podman, Docker, Kubernetes, terminal, Linux) su u zasebnoj datoteci `DevOps-Temeljni-pojmovi.md`.
