# DevOps — Često zamijenjeni pojmovi

> Parovi pojmova koje ispitivači rado provjeravaju. Za svaki par dana je kratka, jasna razlika. Sve riječima, bez simbola.

---

## Port, ciljani port i port na čvoru (kod Servisa)

- **Port (Servisa):** vrata na kojima Servis sluša unutar klastera.
- **Ciljani port (targetPort):** vrata na samom Podu, kamo Servis prosljeđuje promet. Mora odgovarati portu na kojem aplikacija stvarno sluša. (Ako ne odgovara, javlja se odbijanje veze, kao u LO5 zadatku 41.)
- **Port na čvoru (nodePort):** vrata otvorena na svakom računalu klastera, za pristup izvana (samo kod tipa NodePort).

## Zahtjevi i granice (requests i limits)

- **Zahtjev (request):** koliko Pod zajamčeno dobije. Po tome ga raspoređivač smješta na računalo.
- **Granica (limit):** gornja granica koju Pod ne smije prijeći. Probije li memorijsku granicu, biva ugašen (izlazni kod 137).

## ConfigMap i tajna (ConfigMap i Secret)

- **ConfigMap:** za obične postavke koje nisu tajne.
- **Tajna (Secret):** za osjetljive podatke (lozinke, ključevi). Spremljena je kodirana (ne šifrirana) i s nešto strožim postupanjem.

## Deployment, StatefulSet i DaemonSet

- **Deployment:** za obične aplikacije bez stanja. Podovi su zamjenjivi i nemaju stalni identitet.
- **StatefulSet:** za aplikacije sa stanjem. Svaki Pod ima stalno ime i svoju trajnu pohranu (na primjer baza podataka).
- **DaemonSet:** po jedan Pod na svakom računalu (na primjer agent za nadzor).

## ENTRYPOINT i CMD (u izradi slike)

- **ENTRYPOINT:** glavna naredba koja se uvijek pokreće pri startu kontejnera.
- **CMD:** zadani argumenti ili zadana naredba, koju je lako pregaziti pri pokretanju.

## ADD i COPY (u izradi slike)

- **COPY:** samo kopira datoteke s računala u sliku. Jednostavno i predvidivo; preporučeno.
- **ADD:** kao COPY, ali dodatno može raspakirati arhive i dohvatiti s mreže. Koristi samo kad to baš treba.

## Slika i kontejner

- **Slika (image):** zamrznuti predložak koji se ne mijenja.
- **Kontejner:** pokrenuta inačica slike, sa slojem za pisanje na vrhu. Kad ga obrišeš, taj sloj nestaje, a slika ostaje.

## Izlazni kodovi kod ponavljanog pada (CrashLoopBackOff)

- **Kod 0:** proces je uredno završio, ali nije bio dugotrajan (fali mu trajna naredba). Vidi LO5 zadatak 29.
- **Kod 1 (ili drugi različit od nule):** proces je pao s greškom. Vidi LO5 zadatak 22.
- **Kod 137:** proces je ugašen jer je probio memorijsku granicu (OOMKilled). Vidi LO5 zadatak 23.

## Tri uzroka stanja Pending

- **Nedovoljno resursa:** zahtjev za memorijom ili procesorom veći je od slobodnog. Vidi LO5 zadatak 19.
- **Premašen kapacitet računala:** zahtjev je veći od onoga što ijedno računalo uopće ima. Vidi LO5 zadatak 27.
- **Volumen:** Pod čeka trajni svezak koji ne postoji. Vidi LO5 zadatak 28.

## Tipovi Servisa (ClusterIP, NodePort, LoadBalancer)

- **ClusterIP:** dostupan samo unutar klastera. Ovo je zadana vrsta.
- **NodePort:** dostupan izvana, preko porta otvorenog na svakom računalu.
- **LoadBalancer:** dostupan izvana, preko vanjskog uređaja za raspodjelu prometa. U oblaku radi; na minikubeu ostaje na čekanju.

## Sonde: živost, spremnost i pokretanje (probes)

- **Sonda živosti (liveness):** provjerava je li aplikacija živa. Ako nije, Pod se ponovno pokreće.
- **Sonda spremnosti (readiness):** provjerava je li aplikacija spremna primati promet. Ako nije, miče se iz Servisa dok ne bude spremna.
- **Sonda pokretanja (startup):** daje sporoj aplikaciji vremena da se digne prije nego druge sonde počnu provjeravati.

## Politika ponovnog pokretanja: Always, OnFailure, Never (restartPolicy)

- **Always (zadano):** uvijek ponovno pokreni kontejner kad završi. Za servise koji moraju stalno raditi.
- **OnFailure:** ponovno pokreni samo ako je kontejner završio s greškom.
- **Never:** nikad ne pokreći ponovno. Za jednokratne poslove.

## Ingress i Route (izlaganje aplikacije prema van)

- **Ingress:** standardni Kubernetes objekt za izlaganje aplikacije prema van. Treba poseban upravljač (controller) da bi išta radio.
- **Route:** OpenShiftov objekt za isto. Upravljač je ugrađen, ne treba ga posebno postavljati.

## emptyDir i trajni svezak (PVC)

- **emptyDir:** privremena pohrana koja živi koliko i Pod. Nestaje kad Pod nestane.
- **Trajni svezak (PVC, PersistentVolumeClaim):** trajna pohrana koja preživi i kad Pod nestane.

## Provjera prije primjene: na računalu i na poslužitelju (dry-run)

- **Provjera na računalu (client):** brza, lokalna provjera. Može propustiti grešku u polju.
- **Provjera na poslužitelju (server):** stroža, kao da stvaraš objekt (uključuje dodatne provjere), ali bez snimanja. Hvata što lokalna propusti. Vidi LO5 zadatak 40.
