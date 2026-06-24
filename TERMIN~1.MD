# DevOps — Terminal i Podman (brza referenca)

> Referenca za snalaženje: lijevo komanda, desno opis što radi. Ne treba se čitati od početka do kraja — služi kao rječnik.
> Oznaka ⭐ = korišteno u LO1. Sve ostalo su osnove koje vrijedi znati.
> Skoro svaka komanda ima `--help` (npr. `podman run --help`, `grep --help`) i `man <komanda>` za puni popis.

## Rječnik kratica

- **PID** — Process ID, identifikator procesa.
- **UID** — User ID, identifikator korisnika.
- **GID** — Group ID, identifikator grupe.
- **SIGTERM** — signal za uljudan prekid procesa (broj 15); proces ga može uhvatiti i počistiti za sobom.
- **SIGKILL** — signal za tvrdo ubijanje procesa (broj 9); ne može se uhvatiti ni ignorirati.
- **FS** — datotečni sustav (file system).
- **DNS** — Domain Name System, razrješavanje imena u IP adrese.
- **SSL** — Secure Sockets Layer, šifrirana mrežna veza (danas TLS).
- **HTTP** — HyperText Transfer Protocol, protokol za web zahtjeve.
- **stdin / stdout / stderr** — standardni ulaz (kanal 0), standardni izlaz (kanal 1), standardni izlaz za greške (kanal 2).
- **NAT** — Network Address Translation, prevođenje mrežnih adresa.
- **IP** — Internet Protocol; IP adresa identificira sučelje na mreži.
- **RAM** — radna memorija (Random Access Memory).
- **CPU** — procesor (Central Processing Unit).
- **OS** — operacijski sustav.
- **YAML** — format za zapis konfiguracije (manifesti).
- **JSON** — format za zapis strukturiranih podataka.
- **VM** — virtualni stroj (virtual machine).

---

# DIO 1 — LINUX / SHELL OSNOVE

## Kretanje po mapama
| Komanda | Što radi |
|---|---|
| `pwd` | ispisuje trenutnu putanju (puna putanja do trenutne mape) |
| `cd ~/devops` ⭐ | ulazi u mapu `devops` u home-u |
| `cd ~` ili samo `cd` | vraća u home (`/home/student`) |
| `cd ..` | prelazi jednu mapu gore (roditelj) |
| `cd -` | vraća u **prethodnu** mapu |
| `cd /` | prelazi u vrh sustava (root mapa — sadržaj se ne dira) |
| `ls` | ispisuje popis datoteka u trenutnoj mapi |
| `ls -l` | ispisuje dugi format (dozvole, vlasnik, veličina, datum) |
| `ls -a` | prikazuje i skrivene datoteke (one koje počinju s `.`) |
| `ls -la` | dugi format + skrivene |
| `ls -lh` | prikazuje veličine čitljive ljudima (K, M, G) |
| `ls -lt` | poredaja po vremenu (najnovije gore) |
| `ls -ltr` | poredaja po vremenu, obrnuto (najnovije dolje) |
| `ls -R` | ispisuje rekurzivno (i sve podmape) |
| `tree` | prikazuje stablo mapa (ako je instaliran) |

## Rad s datotekama i mapama
| Komanda | Što radi |
|---|---|
| `mkdir mapa` | stvara mapu |
| `mkdir -p a/b/c` ⭐ | stvara mapu i sve roditelje; ne javlja grešku ako postoji |
| `rmdir mapa` | briše **praznu** mapu |
| `rm datoteka` | briše datoteku |
| `rm -r mapa` | briše mapu **sa sadržajem** (rekurzivno) |
| `rm -f datoteka` ⭐ | briše bez pitanja (force) |
| `rm -rf mapa` ⭐ | briše mapu sa sadržajem, bez pitanja (OPREZNO!) |
| `rm -i datoteka` | pita prije svakog brisanja |
| `cp izvor odrediste` | kopira datoteku |
| `cp -r mapa1 mapa2` | kopira mapu sa sadržajem |
| `mv izvor odrediste` | premješta **ili** preimenuje |
| `touch datoteka` | stvara praznu datoteku (ili osvježava vrijeme izmjene) |
| `ln -s meta link` | stvara simbolički link (prečac) |

## Pregled sadržaja datoteka
| Komanda | Što radi |
|---|---|
| `cat datoteka` ⭐ | ispisuje cijeli sadržaj |
| `cat a b > c` | spaja a i b u c |
| `less datoteka` | prikazuje sadržaj stranicu po stranicu (q za izlaz, / za traženje) |
| `more datoteka` | sličan less-u, jednostavniji |
| `head datoteka` | ispisuje prvih 10 redaka |
| `head -n 40 datoteka` ⭐ | ispisuje prvih 40 redaka |
| `tail datoteka` | ispisuje zadnjih 10 redaka |
| `tail -n 20 datoteka` | ispisuje zadnjih 20 redaka |
| `tail -f datoteka` | prati uživo (nove retke kako stižu); Ctrl+C za izlaz |
| `wc -l datoteka` | broji retke |
| `wc -w datoteka` | broji riječi |
| `nano datoteka` | otvara jednostavan editor (prečaci su navedeni dolje) |
| `vi` / `vim datoteka` | otvara moćan editor (prečaci su navedeni dolje) |

## Preusmjeravanje i pipe (vrlo važno)
| Znak | Što radi |
|---|---|
| `>` | preusmjerava izlaz u datoteku (**prepisuje** je) — `echo bok > f.txt` |
| `>>` | preusmjerava izlaz u datoteku (**dopisuje** na kraj) |
| `<` | uzima ulaz iz datoteke |
| `\|` ⭐ | pipe: izlaz lijeve komande → ulaz desne (`ls \| grep txt`) |
| `2>` ⭐ | preusmjerava **greške** (stderr, kanal 2) |
| `2>/dev/null` ⭐ | baca greške u "crnu rupu" (skriva ih) |
| `2>&1` | spaja greške u normalni izlaz |
| `&>` ili `> f 2>&1` | preusmjerava **i** izlaz **i** greške |
| `/dev/null` ⭐ | "crna rupa" — sve poslano ovamo nestaje |
| `command \| tee f` | prikazuje na ekran **i** sprema u datoteku |

> Kanali: `0` = ulaz (stdin), `1` = normalni izlaz (stdout), `2` = greške (stderr).

## Ispis teksta
| Komanda | Što radi |
|---|---|
| `echo tekst` ⭐ | ispisuje tekst |
| `echo -n tekst` | ispisuje bez novog reda na kraju |
| `echo -e "a\nb"` | tumači `\n` (novi red), `\t` (tab) itd. |
| `printf 'a\nb\n'` ⭐ | precizniji ispis; uvijek tumači `\n`, `\t` |

## Pretraživanje
| Komanda | Što radi |
|---|---|
| `grep uzorak datoteka` ⭐ | ispisuje retke koji sadrže uzorak |
| `grep -i` | ignorira velika/mala slova |
| `grep -v` | **obrće** — ispisuje retke koji NE sadrže uzorak |
| `grep -n` | dodaje broj retka |
| `grep -c` | samo broji pogotke |
| `grep -o` | ispisuje samo poklopljeni dio, ne cijeli redak |
| `grep -w` | traži samo cijelu riječ |
| `grep -r uzorak mapa` | pretražuje rekurzivno kroz sve datoteke |
| `grep -l` | ispisuje samo imena datoteka s pogotkom |
| `grep -E 'a\|b'` ⭐ | prošireni regex; `\|` znači ILI (a ili b) |
| `grep -A 3` / `-B 3` / `-C 3` | prikazuje 3 retka poslije / prije / oko pogotka |
| `find . -name "*.txt"` | nalazi datoteke po imenu |
| `find . -type f` / `-type d` | traži samo datoteke / samo mape |
| `find . -mtime -1` | nalazi promijenjene u zadnjih 24 h |
| `find . -size +1M` | nalazi veće od 1 MB |
| `which komanda` ⭐ | ispisuje gdje je izvršna datoteka neke komande |
| `whereis komanda` | ispisuje lokacije binarke + man stranice |

## Dozvole (permissions)
| Komanda | Što radi |
|---|---|
| `chmod +x skripta.sh` | dodaje pravo izvršavanja |
| `chmod 755 datoteka` | postavlja rwx za vlasnika, r-x za ostale |
| `chmod 644 datoteka` | postavlja rw za vlasnika, r za ostale |
| `chmod 600 datoteka` | postavlja rw samo za vlasnika (npr. lozinke) |
| `chmod -R 755 mapa` | primjenjuje rekurzivno na cijelu mapu |
| `chown korisnik datoteka` | mijenja vlasnika |
| `chown korisnik:grupa datoteka` | mijenja vlasnika i grupu |

> Brojevi: 4=čitanje (r), 2=pisanje (w), 1=izvršavanje (x). Vrijednosti se zbrajaju: 7=rwx, 6=rw, 5=r-x. Tri znamenke = vlasnik / grupa / ostali.

## Procesi
| Komanda | Što radi |
|---|---|
| `ps` | ispisuje procese u trenutnom terminalu |
| `ps aux` | ispisuje **sve** procese na sustavu (detaljno) |
| `ps -ef` | ispisuje sve procese (drugi format) |
| `ps aux \| grep httpd` | nalazi proces po imenu |
| `top` | prikazuje živi prikaz procesa i opterećenja (q za izlaz) |
| `htop` | ljepši `top` (ako je instaliran) |
| `kill PID` ⭐ | šalje procesu signal za prekid (SIGTERM, 15) |
| `kill -9 PID` ⭐ | tvrdo ubija proces (SIGKILL); ne može se ignorirati |
| `kill -15 PID` | uljudno zaustavlja proces (default) |
| `killall ime` | ubija sve procese tog imena |
| `pkill ime` | ubija procese po imenu (uzorak) |
| `command &` | pokreće u pozadini |
| `jobs` | ispisuje poslove u pozadini ovog terminala |
| `fg` | vraća pozadinski posao u prvi plan |
| `bg` | nastavlja zaustavljeni posao u pozadini |
| `nohup command &` | pokreće tako da posao preživi zatvaranje terminala |

> Exit kodovi: `0` = uspjeh; `137` = proces ubijen SIGKILL-om (128+9); `143` = prekinut SIGTERM-om (128+15). Vidi `$?`.

## Sistemske informacije
| Komanda | Što radi |
|---|---|
| `whoami` ⭐ | ispisuje ime trenutnog korisnika |
| `id` ⭐ | ispisuje UID, GID, grupe |
| `hostname` | ispisuje ime stroja |
| `hostname -I` ⭐ | ispisuje IP adrese stroja |
| `uname -a` | ispisuje informacije o jezgri i OS-u |
| `uptime` | ispisuje koliko sustav radi + opterećenje |
| `df -h` | prikazuje zauzeće diskova (čitljivo) |
| `du -sh mapa` | prikazuje veličinu mape |
| `free -h` | prikazuje zauzeće RAM-a |
| `date` | ispisuje trenutni datum i vrijeme |
| `env` ⭐ | ispisuje sve environment varijable |
| `export VAR=val` | postavlja varijablu za sesiju |
| `history` | ispisuje povijest komandi |
| `clear` (ili Ctrl+L) | čisti ekran |
| `man komanda` | otvara priručnik za komandu (q za izlaz) |
| `watch -n 2 komanda` | ponavlja komandu svake 2 s |
| `sleep 5` | čeka 5 sekundi |

## Mreža
| Komanda | Što radi |
|---|---|
| `ping host` | provjerava dostupnost (Ctrl+C za stop) |
| `curl URL` ⭐ | dohvaća sadržaj s URL-a / šalje HTTP zahtjev |
| `curl -s` ⭐ | radi tiho (bez progress bara) |
| `curl -o f URL` ⭐ | sprema u datoteku f |
| `curl -O URL` | sprema pod originalnim imenom |
| `curl -I URL` | dohvaća samo HTTP zaglavlja |
| `curl -L URL` | slijedi preusmjeravanja |
| `curl -X POST URL` | odabire HTTP metodu |
| `curl -H "K: V" URL` | dodaje zaglavlje |
| `curl -d 'data' URL` | šalje podatke (POST body) |
| `curl -w '%{http_code}\n' URL` ⭐ | ispisuje samo status kod |
| `curl --max-time 3 URL` ⭐ | odustaje nakon 3 s |
| `curl --noproxy '*' URL` ⭐ | zaobilazi proxy |
| `curl -k URL` | preskače provjeru SSL certifikata |
| `wget URL` | preuzima datoteku |
| `ip addr` / `ip a` ⭐ | ispisuje mrežna sučelja i IP-ove |
| `ss -tulpn` | ispisuje otvorene portove / slušatelje |
| `getent hosts ime` ⭐ | razrješava ime u IP (preko sustava) |
| `nslookup ime` / `dig ime` | šalje DNS upit |

## Arhive i kompresija
| Komanda | Što radi |
|---|---|
| `tar -czf arh.tar.gz mapa` | stvara gzip arhivu (**c**reate **z**ip **f**ile) |
| `tar -xzf arh.tar.gz` | raspakirava (e**x**tract) |
| `tar -tzf arh.tar.gz` | prikazuje sadržaj bez raspakiravanja |
| `tar -xzf arh.tar.gz -C mapa` | raspakirava u zadanu mapu |
| `gzip datoteka` / `gunzip f.gz` | kompresira / dekompresira |
| `zip -r arh.zip mapa` / `unzip arh.zip` | pakira u zip / raspakirava zip |

## Upravljanje paketima (Debian/Ubuntu — httpd slika je Debian)
| Komanda | Što radi |
|---|---|
| `apt-get update` ⭐ | osvježava popis dostupnih paketa |
| `apt-get install -y paket` ⭐ | instalira paket (`-y` = automatski potvrđuje) |
| `apt-get remove paket` | uklanja paket |
| `apt-get purge paket` | uklanja paket + konfiguraciju |
| `apt-get upgrade` | nadograđuje sve pakete |
| `apt-get autoremove` | uklanja nepotrebne ovisnosti |
| `apt list --installed` | ispisuje popis instaliranih paketa |
| `dnf` / `yum install` | isto, ali za Red Hat / Fedora |

## Varijable i supstitucija
| Zapis | Što radi |
|---|---|
| `VAR=vrijednost` | postavlja varijablu (bez razmaka oko `=`!) |
| `$VAR` ili `${VAR}` | koristi vrijednost varijable |
| `export VAR=val` | čini varijablu dostupnom i podprocesima |
| `$?` | exit kod zadnje komande (0 = ok) |
| `$(komanda)` | ubacuje izlaz komande (`echo "danas: $(date)"`) |
| `` `komanda` `` | starija sintaksa za isto |
| `$0 $1 $2` | argumenti skripte (ime, prvi, drugi…) |
| `$#` | broj argumenata |

## Sudo i korisnici
| Komanda | Što radi |
|---|---|
| `sudo komanda` | pokreće komandu kao root (treba dozvolu) |
| `su - korisnik` | prebacuje na drugog korisnika |
| `exit` ili Ctrl+D ⭐ | izlazi iz shella / odjavljuje / izlazi iz kontejnera |

## Tipkovnički prečaci (terminal)
| Prečac | Što radi |
|---|---|
| `Tab` | automatski dopunjava imena |
| `↑` / `↓` | lista povijest komandi |
| `Ctrl+C` ⭐ | prekida trenutnu komandu |
| `Ctrl+D` | označava kraj ulaza / odjavu / izlaz iz kontejnera |
| `Ctrl+Z` | zaustavlja (suspend) proces u pozadinu |
| `Ctrl+L` | čisti ekran (kao `clear`) |
| `Ctrl+R` | pretražuje povijest komandi |
| `Ctrl+A` / `Ctrl+E` | pomiče na početak / kraj retka |
| `Ctrl+U` / `Ctrl+K` | briše do početka / do kraja retka |
| `Ctrl+W` | briše riječ unatrag |

## Editor: nano (jednostavan)
| Prečac | Što radi |
|---|---|
| `Ctrl+O` pa Enter | sprema (WriteOut) |
| `Ctrl+X` | izlazi (pita za spremanje) |
| `Ctrl+W` | traži |
| `Ctrl+K` / `Ctrl+U` | izrezuje redak / lijepi |
| `Ctrl+G` | otvara pomoć |

## Editor: vi / vim (moćan, ali drukčiji)
| Tipka | Što radi |
|---|---|
| `i` | ulazi u način pisanja (insert) |
| `Esc` | vraća u naredbeni način |
| `:w` | sprema |
| `:q` | izlazi |
| `:wq` ili `ZZ` | sprema i izlazi |
| `:q!` | izlazi bez spremanja (force) |
| `dd` | briše redak |
| `/tekst` | traži |

---

# DIO 2 — PODMAN

## Kako pročitati ime slike
```
docker.io/library/httpd:latest
└─registry─┘└─repo──┘ └─tag┘
```
- registar: `docker.io` (Docker Hub), `quay.io`, `localhost` (lokalno napravljene slike)
- ako oznaka (tag) nije navedena → podrazumijeva se `:latest`

## Životni ciklus kontejnera
| Komanda | Što radi |
|---|---|
| `podman run IMAGE` ⭐ | stvara **i** pokreće novi kontejner |
| `podman create IMAGE` | stvara kontejner, ali ga ne pokreće |
| `podman start <c>` ⭐ | pokreće postojeći (zaustavljeni) kontejner |
| `podman stop <c>` ⭐ | uljudno zaustavlja (SIGTERM, pa SIGKILL nakon ~10 s) |
| `podman restart <c>` | radi stop + start |
| `podman kill <c>` ⭐ | tvrdo ubija (SIGKILL po defaultu) |
| `podman pause <c>` ⭐ | zamrzava sve procese (freezer cgroup) |
| `podman unpause <c>` ⭐ | odmrzava procese |
| `podman rm <c>` ⭐ | briše kontejner |
| `podman rm -f <c>` ⭐ | briše i ako se vrti (force) |
| `podman rm -f $(podman ps -aq)` | briše **sve** kontejnere |
| `podman rename staro novo` ⭐ | preimenuje kontejner (ID ostaje isti) |
| `podman exec <c> komanda` ⭐ | pokreće komandu **unutar** kontejnera |
| `podman exec -it <c> bash` ⭐ | otvara interaktivni shell unutra |
| `podman attach <c>` | spaja se na glavni proces kontejnera |
| `podman wait <c>` | blokira dok kontejner ne izađe (vraća exit kod) |
| `podman cp ...` ⭐ | kopira datoteke host ↔ kontejner |
| `podman commit <c> ime:tag` ⭐ | sprema stanje kontejnera u novu sliku |

## Pregled i informacije
| Komanda | Što radi |
|---|---|
| `podman ps` ⭐ | ispisuje popis **pokrenutih** kontejnera |
| `podman ps -a` ⭐ | ispisuje i zaustavljene (sve) |
| `podman ps -q` | ispisuje samo ID-eve |
| `podman ps -aq` | ispisuje ID-eve svih (zgodno za skripte) |
| `podman ps -l` | ispisuje zadnji stvoreni kontejner |
| `podman ps -s` | dodaje veličinu zapisivog sloja |
| `podman ps --filter name=web` ⭐ | filtrira (name, status, label, id…) |
| `podman ps --format '{{.Names}}'` | prilagođava ispis |
| `podman logs <c>` ⭐ | ispisuje logove kontejnera |
| `podman logs --tail 5 <c>` ⭐ | ispisuje zadnjih 5 redaka |
| `podman logs --since 1m <c>` ⭐ | ispisuje retke iz zadnje minute |
| `podman logs -f <c>` ⭐ | prati uživo (Ctrl+C za izlaz) |
| `podman logs -t <c>` | dodaje vremenske oznake |
| `podman inspect <c>` ⭐ | ispisuje puni JSON o kontejneru/slici |
| `podman inspect --format '{{...}}' <c>` ⭐ | izvlači jedno polje (vidi cheatsheet dolje) |
| `podman top <c>` ⭐ | ispisuje procese koji se vrte unutar kontejnera |
| `podman stats <c>` ⭐ | prikazuje živu potrošnju resursa |
| `podman stats --no-stream <c>` ⭐ | prikazuje jedan snimak pa izlazi |
| `podman stats -a` | prikazuje resurse svih kontejnera |
| `podman port <c>` ⭐ | ispisuje popis objavljenih portova |
| `podman diff <c>` ⭐ | prikazuje promjene u datotečnom sustavu (A/C/D) |
| `podman events` ⭐ | prikazuje životni ciklus svih kontejnera uživo |
| `podman events --since 1m --stream=false` | ispisuje nedavne događaje bez čekanja |

## Slike (images)
| Komanda | Što radi |
|---|---|
| `podman images` | ispisuje popis lokalnih slika |
| `podman images -q` | ispisuje samo ID-eve |
| `podman images --filter reference=httpd` ⭐ | filtrira |
| `podman pull IMAGE` | povlači sliku s registra |
| `podman push IMAGE` | gura sliku na registar |
| `podman build -t ime:tag .` | gradi sliku iz `Dockerfile` u trenutnoj mapi |
| `podman build -f Putanja -t ime .` | gradi s drugačijim imenom Dockerfilea |
| `podman tag stari novi:tag` | dodaje/mijenja oznaku (tag) |
| `podman rmi IMAGE` ⭐ | briše sliku |
| `podman rmi -f IMAGE` | briše i ako sliku koristi kontejner |
| `podman save -o arh.tar IMAGE` | sprema sliku u tar datoteku |
| `podman load -i arh.tar` | učitava sliku iz tar datoteke |
| `podman history IMAGE` | ispisuje slojeve od kojih je slika sastavljena |
| `podman search ime` | traži sliku na registru |
| `podman image prune` | briše "viseće" (neiskorištene) slike |
| `podman image prune -a` | briše sve neiskorištene slike |

## Mreže (networks)
| Komanda | Što radi |
|---|---|
| `podman network create ime` ⭐ | stvara korisnički definiran most (bridge mreža) |
| `podman network create --subnet 10.89.0.0/24 ime` | stvara most sa zadanim subnetom |
| `podman network ls` | ispisuje popis mreža |
| `podman network inspect ime` ⭐ | ispisuje detalje (subnet, gateway, spojeni kontejneri) |
| `podman network rm ime` ⭐ | briše mrežu |
| `podman network connect mreza <c>` | spaja kontejner na mrežu |
| `podman network disconnect mreza <c>` | odspaja kontejner s mreže |
| `podman network prune` | briše neiskorištene mreže |

> Posebne mreže: `--network bridge` (default), `--network host` (dijeli host mrežu), `--network none` (bez mreže) ⭐.

## Volumeni (volumes) — trajna pohrana
| Komanda | Što radi |
|---|---|
| `podman volume create ime` | stvara imenovani volumen |
| `podman volume ls` | ispisuje popis volumena |
| `podman volume inspect ime` | ispisuje detalje (gdje su podaci na hostu) |
| `podman volume rm ime` | briše volumen |
| `podman volume prune` | briše neiskorištene volumene |

> Korištenje: `podman run -v ime-volumena:/putanja IMAGE` (imenovani volumen) vs `-v /host/mapa:/putanja` (bind-mount).

## Sistem / ostalo
| Komanda | Što radi |
|---|---|
| `podman system prune` | čisti zaustavljene kontejnere, neiskorištene mreže/slike |
| `podman system prune -a --volumes` | čisti baš sve neiskorišteno |
| `podman system df` | prikazuje koliko diska troše kontejneri/slike/volumeni |
| `podman info` | ispisuje informacije o Podman okruženju (verzija, cgroups, storage…) |
| `podman version` | ispisuje verziju Podmana |
| `podman login registry` | prijavljuje na registar (quay.io, Docker Hub) |
| `podman logout registry` | odjavljuje s registra |
| `podman generate systemd --name <c>` ⭐ | generira systemd unit za kontejner (zastarjelo → Quadlet) |
| `podman generate kube <c>` | generira Kubernetes YAML iz kontejnera |
| `podman play kube datoteka.yaml` | pokreće iz Kubernetes YAML-a |

> Noviji oblik komandi je `podman kube generate` / `podman kube play`; stariji oblici `podman generate kube` / `podman play kube` i dalje rade (aliasi su, oba su valjana).

## Pods (grupe kontejnera koje dijele mrežu)
| Komanda | Što radi |
|---|---|
| `podman pod create --name ime` | stvara pod |
| `podman pod ps` | ispisuje popis podova |
| `podman pod start/stop/rm ime` | upravlja podom |
| `podman run --pod ime IMAGE` | dodaje kontejner u pod |

---

## ⭐ TABLICA: `podman run` flagovi (najvažnije)
| Flag | Što radi |
|---|---|
| `-d`, `--detach` ⭐ | pokreće u pozadini (terminal slobodan) |
| `-i`, `--interactive` ⭐ | drži stdin otvoren (za tipkanje) |
| `-t`, `--tty` ⭐ | dodjeljuje terminal (lijep prompt) |
| `-it` ⭐ | kombinacija `-i -t` (interaktivni shell) |
| `--name ime` ⭐ | postavlja ime kontejnera |
| `--hostname ime`, `-h` ⭐ | postavlja hostname **unutar** kontejnera |
| `-p HOST:CONT`, `--publish` ⭐ | mapira port (može `-p IP:HOST:CONT`) |
| `-P`, `--publish-all` | objavljuje sve EXPOSE portove na nasumične host portove |
| `--network ime` ⭐ | spaja na mrežu (`bridge`/`host`/`none`/ime) |
| `--restart politika` ⭐ | `no` / `on-failure` / `always` / `unless-stopped` |
| `-m`, `--memory 256m` ⭐ | postavlja gornju granicu RAM-a |
| `--cpus 0.5` ⭐ | ograničava broj jezgri (treba cpu cgroup delegaciju; u rootless načinu zna ne raditi) |
| `--cpu-shares N` | postavlja relativnu težinu CPU-a (kad je gužva) |
| `-e`, `--env K=V` ⭐ | ubacuje jednu environment varijablu |
| `--env-file datoteka` ⭐ | učitava varijable iz datoteke |
| `-v HOST:CONT[:ro]`, `--volume` ⭐ | radi bind-mount/volumen (`:ro` = samo čitanje) |
| `--mount type=...` | eksplicitniji način mountanja |
| `--tmpfs /putanja` | stvara privremeni FS u RAM-u (nestaje s kontejnerom) |
| `--rm` ⭐ | automatski briše kontejner nakon izlaska |
| `-u`, `--user UID[:GID]` ⭐ | pokreće kao taj korisnik (ne root) |
| `-l`, `--label K=V` ⭐ | dodaje labelu (metapodatak) |
| `-w`, `--workdir /put` | postavlja radni direktorij unutar kontejnera |
| `--entrypoint cmd` | mijenja početnu komandu slike |
| `--read-only` | postavlja root datotečni sustav samo za čitanje |
| `--privileged` | daje sve ovlasti (OPREZNO — rijetko treba) |
| `--cap-add` / `--cap-drop` | dodaje / oduzima pojedinu Linux capability |
| `--dns IP` | postavlja DNS server |
| `--add-host ime:IP` | ubacuje unos u `/etc/hosts` kontejnera |
| `--pull politika` | `always` / `missing` / `never` (kad povući sliku) |
| `--replace` | zamjenjuje postojeći kontejner istog imena |
| `--health-cmd "..."` | postavlja komandu za provjeru zdravlja kontejnera |
| `--ulimit` | postavlja resource limite (npr. broj otvorenih datoteka) |
| `--device /dev/...` | prosljeđuje hardverski uređaj u kontejner |

> Puni popis (ima ih 100+): `podman run --help`.

---

## ⭐ CHEATSHEET: `inspect --format` (Go-template)
| Format | Vraća |
|---|---|
| `{{.Id}}` ⭐ | puni ID kontejnera |
| `{{.Name}}` ⭐ | ime |
| `{{.State.Status}}` ⭐ | `running` / `paused` / `exited` |
| `{{.State.Running}}` | `true` / `false` |
| `{{.State.Pid}}` ⭐ | host-PID glavnog procesa |
| `{{.RestartCount}}` ⭐ | koliko je puta restartan |
| `{{.Config.Image}}` ⭐ | iz koje je slike |
| `{{.Config.Hostname}}` ⭐ | hostname unutra |
| `{{.HostConfig.Memory}}` ⭐ | memorijski limit (bajtovi) |
| `{{.HostConfig.NanoCpus}}` ⭐ | CPU limit (nano jedinice) |
| `{{.NetworkSettings.IPAddress}}` | IP (na default mreži) |
| `{{.NetworkSettings.Networks.IME.IPAddress}}` ⭐ | IP na imenovanoj mreži |
| `{{.NetworkSettings.Ports}}` ⭐ | mapiranja portova |
| `{{json .State}}` | cijela `State` grana kao JSON |

> Točka `.` = cijeli objekt; spuštanje kroz razine ide točkama, kao kroz JSON.

---

## Najčešći obrasci čišćenja
```bash
podman rm -f <ime>                 # obriši container (i ako se vrti)
podman rm -f $(podman ps -aq)      # obriši SVE containere
podman rmi <image>                 # obriši image
podman network rm <mreza>          # obriši mrežu
podman volume rm <volumen>         # obriši volumen
podman system prune -a --volumes   # počisti sve neiskorišteno odjednom
```
