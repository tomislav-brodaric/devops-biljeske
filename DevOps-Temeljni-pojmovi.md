# DevOps — Temeljni pojmovi (od nule)

> Ovo je „kreni odavde" datoteka: objašnjava osnovne pojmove na kojima počiva cijeli kolegij (Linux, terminal, kontejner, slika, Docker, Podman, Kubernetes). Sve je napisano riječima, bez simbola, i svaki pojam je definiran. Povezuje se s onim što si radio u ishodima LO1 do LO6.

---

## 1. Operacijski sustav i Linux

**Operacijski sustav** je glavni program koji upravlja računalom: pokreće druge programe, dijeli im procesor i memoriju, te razgovara s diskovima, mrežom i ostalim dijelovima. Bez njega aplikacije ne bi imale na čemu raditi. Poznati primjeri su Windows, macOS i Linux.

**Linux** je operacijski sustav koji je besplatan i otvorenog koda (svatko ga smije koristiti i mijenjati). Najrašireniji je na **poslužiteljima** (računalima koja stalno rade i poslužuju usluge preko mreže) i temelj je gotovo svih kontejnera. Zato se cijeli ovaj kolegij vrti oko Linuxa.

**Jezgra (kernel)** je srce operacijskog sustava: dio koji izravno upravlja procesorom, memorijom i uređajima. Zapamti ovaj pojam jer je ključan za razumijevanje kontejnera (vidi točku 3).

---

## 2. Terminal, ljuska i naredbeni redak

**Terminal** je prozor u koji tipkaš naredbe umjesto da klikćeš mišem. To je tekstualni način upravljanja računalom.

**Ljuska (shell)** je program unutar terminala koji prima tvoje natipkane naredbe, izvršava ih i pokazuje rezultat. Najpoznatija ljuska na Linuxu zove se **bash**.

**Naredbeni redak (command line)** je redak u koji upisuješ naredbu. Na primjer, naredba `ls` izlista datoteke u trenutnoj mapi, a `cd` mijenja mapu u kojoj se nalaziš.

Zašto se to koristi umjesto miša: na poslužiteljima često nema grafičkog sučelja, naredbe se mogu spremiti i ponavljati (automatizirati), i preciznije su. Sve što si radio u LO1 do LO5 izvodio si upravo kroz terminal.

---

## 3. Što je kontejner

**Kontejner** je izolirani paket koji sadrži jednu aplikaciju i sve što joj treba za rad (knjižnice, alate, postavke). Kontejner radi kao da ima vlastito malo računalo, ali zapravo **dijeli jezgru (kernel) glavnog računala** s drugim kontejnerima.

Najvažniji način razmišljanja: **kontejner nije zasebno računalo, nego omotač oko jednog procesa.** Kada se taj glavni proces unutra završi, kontejner prestaje raditi. (To si vidio u LO5: čisti `ubuntu` kontejner odmah završi jer u njemu nema dugotrajnog procesa.)

Što kontejner daje:
- **Izolacija:** kontejner ima svoj pogled na datoteke, mrežu i procese, odvojen od drugih.
- **Prenosivost:** isti kontejner radi jednako na tvom računalu i na poslužitelju, jer nosi sve što mu treba.
- **Lakoća:** pokreće se brzo i troši malo jer dijeli jezgru s računalom, a ne nosi cijeli operacijski sustav.

---

## 4. Kontejner u odnosu na virtualni stroj (ključna razlika)

**Virtualni stroj (VM, virtual machine)** je softverski simulirano cijelo računalo unutar pravog računala. Ima **vlastiti potpuni operacijski sustav**, pa je težak i sporiji za pokretanje.

**Kontejner** ne simulira cijelo računalo. Dijeli jezgru glavnog računala i izolira samo aplikaciju. Zato je **mnogo lakši i brži** od virtualnog stroja.

Slika za pamćenje: virtualni stroj je kao zasebna kuća sa svom infrastrukturom; kontejner je kao zasebna soba u istoj kući koja dijeli temelje i instalacije, ali ima svoja vrata. Ova razlika objašnjava zašto su kontejneri postali tako popularni: daju izolaciju aplikacije uz mnogo manju cijenu od virtualnog stroja.

---

## 5. Slika (image) i kako nastaje

**Slika (image)** je **zamrznuti, nepromjenjivi predložak** iz kojeg se pokreće kontejner. Sadrži aplikaciju i sve što joj treba, spremljeno za pokretanje.

Odnos slike i kontejnera (vidio si ga u LO1):
- **Slika** je samo-za-čitanje (ne mijenja se).
- **Kontejner** je živa pokrenuta inačica slike, s dodanim **slojem za pisanje** na vrhu (tu nastaju privremene promjene dok kontejner radi).
- Kada obrišeš kontejner, taj sloj za pisanje nestaje, ali slika ostaje.

**Slojevi (layers):** slika je složena od slojeva, jedan na drugom. Svaki korak u izradi dodaje sloj. To omogućuje **predmemoriranje (caching)**: ako se sloj nije promijenio, ne mora se ponovno graditi. (U LO2 si vidio kako preslagivanje koraka ubrzava izgradnju.)

**Recept za sliku (Containerfile ili Dockerfile):** tekstualna datoteka s uputama kako izgraditi sliku, korak po korak (na primjer: uzmi osnovnu sliku, kopiraj datoteke, postavi naredbu pokretanja). Iz tog recepta alat izgradi sliku.

**Registar (registry):** mjesto na internetu gdje se slike čuvaju i dijele. Poznati primjeri su **Docker Hub** i **Quay.io** (oba si koristio). Sliku gradiš lokalno, pa je **pošalješ (push)** u registar, ili je **povučeš (pull)** iz njega.

---

## 6. Docker

**Docker** je alat koji je popularizirao kontejnere. Njime gradiš slike, pokrećeš kontejnere i upravljaš njima.

Glavna značajka ustroja: Docker radi preko **stalne pozadinske usluge (daemona)** — programa koji stalno radi u pozadini i kroz koji prolaze svi zahtjevi. Ta usluga tipično ima **administratorske ovlasti (root)**, to jest najviše ovlasti na računalu. To je praktično, ali znači da postoji jedna moćna središnja točka.

---

## 7. Podman

**Podman** je alat vrlo sličan Dockeru (gotovo iste naredbe), ali drukčijeg ustroja. Njega si koristio kroz cijeli kolegij.

Dvije ključne razlike u odnosu na Docker:
- **Bez stalne pozadinske usluge (daemonless):** Podman nema program koji stalno radi u pozadini; svaka naredba pokreće proces izravno. Nema te jedne moćne središnje točke.
- **Bez administratorskih ovlasti (rootless):** Podman lako radi kao **obični korisnik**, a ne kao administrator. To je sigurnije, jer čak i ako napadač provali u kontejner, ima manje ovlasti na računalu.

Zbog toga timovi koji paze na sigurnost često biraju Podman. (To je i tema LO6, pitanje 16.) Na ispitnom računalu radiš upravo s Podmanom bez administratorskih ovlasti, pa neke stvari koje traže administratora (na primjer ograničavanje procesora) ne rade.

---

## 8. Kubernetes

Kada imaš **mnogo kontejnera raspoređenih preko više računala**, ručno upravljanje postaje nemoguće. Tada treba **orkestracija**: automatsko raspoređivanje, pokretanje, skaliranje i oporavak kontejnera. **Kubernetes** (kratica k8s) je danas standardni sustav za to.

Što Kubernetes radi umjesto tebe:
- **Raspoređuje** kontejnere na računala koja imaju mjesta.
- **Oporavlja** ih kad padnu (samoizlječenje): ako kontejner ili računalo otkaže, Kubernetes podigne zamjenu.
- **Skalira:** dodaje ili oduzima kontejnere prema opterećenju.
- **Povezuje:** daje stabilne adrese i imena kontejnerima koji se stalno mijenjaju.

To je cijena u složenosti, ali za velike i promjenjive sustave nema dobre zamjene. (Sve si to radio u LO4 i LO5.)

---

## 9. Ključni pojmovi Kubernetesa (kratko, za povezivanje)

- **Klaster (cluster):** skup računala kojima Kubernetes zajedno upravlja.
- **Čvor (node):** jedno računalo u klasteru. (U tvom slučaju, jedno računalo zvano `minikube`.)
- **Pod:** najmanja jedinica u Kubernetesu; omotač oko jednog ili više kontejnera koji dijele istu mrežu. Kontejneri se u Kubernetesu uvijek pokreću unutar Poda.
- **Deployment:** opisuje željeno stanje za skup istih Podova (na primjer „želim tri primjerka ove aplikacije") i brine da to stanje stalno vrijedi; omogućuje i postupno ažuriranje (rolling update).
- **Servis (Service):** daje jednu stalnu adresu i ime za skup Podova koji se stalno gase i dižu, da ih drugi mogu pouzdano pronaći.
- **Prostor imena (namespace):** način da se objekti unutar klastera podijele u odvojene skupine (na primjer po timovima ili okruženjima). Objekti u jednom prostoru imena ne vide automatski objekte u drugom.
- **kubectl:** naredbeni alat kojim upravljaš Kubernetesom (sve naredbe u LO4 i LO5 počinjale su s `kubectl`).
- **minikube:** alat koji na jednom računalu pokrene mali Kubernetes klaster za vježbu i razvoj.

---

## 10. Kako se sve to slaže (velika slika)

Poredaj pojmove odozdo prema gore i sve dobiva smisao:

1. **Linux** je operacijski sustav koji upravlja računalom; njegova **jezgra** izravno upravlja procesorom i memorijom.
2. **Kontejner** je izolirani omotač oko aplikacije koji **dijeli tu Linux jezgru** umjesto da nosi cijeli operacijski sustav, pa je lagan i brz.
3. **Slika** je zamrznuti predložak iz kojeg kontejner nastaje; gradi se po **receptu** i čuva u **registru**.
4. **Docker** i **Podman** su alati za izradu i pokretanje kontejnera; Podman je bez stalne pozadinske usluge i radi bez administratorskih ovlasti, pa je sigurniji.
5. **Kubernetes** dolazi kad kontejnera ima mnogo, preko više računala: on ih automatski raspoređuje, oporavlja, skalira i povezuje.
6. **OpenShift** je Kubernetes s dodanim alatima, podrškom i strožom sigurnošću (tema LO6).

Sve što si učio kroz kolegij je jedan tok: od pokretanja jednog kontejnera (LO1), preko izrade slika (LO2) i povezivanja više kontejnera (LO3), do upravljanja njima u Kubernetesu (LO4), rješavanja kvarova (LO5) i procjene kada koji sustav izabrati (LO6).
