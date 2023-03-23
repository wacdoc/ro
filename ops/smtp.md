# Creați-vă propriul server de trimitere a e-mailurilor SMTP

## preambul

SMTP poate achiziționa direct servicii de la furnizorii de cloud, cum ar fi:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali cloud e-mail push](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

De asemenea, vă puteți construi propriul server de e-mail - trimitere nelimitată, cost global scăzut.

Mai jos, demonstrăm pas cu pas cum să ne construim propriul server de e-mail.

## Selectarea serverului

Serverul SMTP auto-găzduit necesită un IP public cu porturile 25, 456 și 587 deschise.

Norurile publice utilizate în mod obișnuit au blocat aceste porturi în mod implicit și ar putea fi posibilă deschiderea lor prin emiterea unei comenzi de lucru, dar este foarte deranjant până la urmă.

Recomand să cumpărați de la o gazdă care are aceste porturi deschise și care acceptă configurarea numelor de domenii inverse.

Aici, recomand [Contabo](https://contabo.com) .

Contabo este un furnizor de hosting cu sediul în Munchen, Germania, fondat în 2003, cu prețuri foarte competitive.

Dacă alegeți euro ca monedă de cumpărare, prețul va fi mai ieftin (un server cu memorie de 8 GB și 4 procesoare costă aproximativ 529 de yuani pe an, iar taxa de instalare inițială este gratuită timp de un an).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Când plasați o comandă, remarcați `prefer AMD` , iar serverul cu procesor AMD va avea performanțe mai bune.

În cele ce urmează, voi lua ca exemplu VPS-ul Contabo pentru a demonstra cum să vă construiți propriul server de e-mail.

## Configurarea sistemului Ubuntu

Sistemul de operare de aici este Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Dacă serverul de pe ssh afișează `Welcome to TinyCore 13!` (după cum se arată în figura de mai jos), înseamnă că sistemul nu a fost încă instalat. Deconectați ssh și așteptați câteva minute pentru a vă conecta din nou.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Când apare `Welcome to Ubuntu 22.04.1 LTS` , inițializarea este completă și puteți continua cu următorii pași.

### [Opțional] Inițializați mediul de dezvoltare

Acest pas este opțional.

Pentru comoditate, am pus instalarea și configurarea sistemului software-ului ubuntu în [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Rulați următoarea comandă pentru a instala cu un singur clic.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Utilizatori chinezi, vă rugăm să utilizați următoarea comandă, iar limba, fusul orar etc. vor fi setate automat.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo activează IPV6

Activați IPV6, astfel încât SMTP să poată trimite și e-mailuri cu adrese IPV6.

editați `/etc/sysctl.conf`

Modificați sau adăugați următoarele rânduri

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Continuați cu [tutorialul contabo: Adăugarea conectivității IPv6 la serverul dvs](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Editați `/etc/netplan/01-netcfg.yaml` , adăugați câteva rânduri, așa cum se arată în figura de mai jos (fișierul de configurare implicită Contabo VPS are deja aceste rânduri, doar anulați-le comentariile).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Apoi `netplan apply` pentru ca configurația modificată să intre în vigoare.

După ce configurarea a reușit, puteți utiliza `curl 6.ipw.cn` pentru a vedea adresa ipv6 a rețelei externe.

## Clonează operațiunile depozitului de configurare

```
git clone https://github.com/wactax/ops.soft.git
```

## Generați un certificat SSL gratuit pentru numele dvs. de domeniu

Trimiterea e-mailurilor necesită un certificat SSL pentru criptare și semnare.

Folosim [acme.sh](https://github.com/acmesh-official/acme.sh) pentru a genera certificate.

acme.sh este un instrument de semnare automată a certificatelor open source,

Introduceți depozitul de configurare ops.soft, rulați `./ssl.sh` și va fi creat un folder `conf` în **directorul de sus** .

Găsiți furnizorul DNS de pe [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , editați `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Apoi rulați `./ssl.sh 123.com` pentru a genera certificate `123.com` și `*.123.com` pentru numele dvs. de domeniu.

Prima rulare va instala automat [acme.sh](https://github.com/acmesh-official/acme.sh) și va adăuga o sarcină programată pentru reînnoirea automată. Puteți vedea `crontab -l` , există o astfel de linie după cum urmează.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Calea pentru certificatul generat este ceva de genul `/mnt/www/.acme.sh/123.com_ecc。`

Reînnoirea certificatului va apela scriptul `conf/reload/123.com.sh` , editați acest script, puteți adăuga comenzi precum `nginx -s reload` pentru a reîmprospăta memoria cache a certificatelor aplicațiilor asociate.

## Construiți un server SMTP cu chasquid

[chasquid](https://github.com/albertito/chasquid) este un server SMTP open source scris în limba Go.

Ca înlocuitor pentru vechile programe de server de e-mail, cum ar fi Postfix și Sendmail, chasquid este mai simplu și mai ușor de utilizat și este, de asemenea, mai ușor pentru dezvoltarea secundară.

Rulați `./chasquid/init.sh 123.com` va fi instalat automat cu un singur clic (înlocuiți 123.com cu numele domeniului de trimitere).

## Configurați semnătura de e-mail DKIM

DKIM este folosit pentru a trimite semnături de e-mail pentru a preveni ca scrisorile să fie tratate ca spam.

După ce comanda rulează cu succes, vi se va solicita să setați înregistrarea DKIM (după cum se arată mai jos).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Doar adăugați o înregistrare TXT la DNS (după cum se arată mai jos).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Vedeți starea serviciului și jurnalele

 `systemctl status chasquid` Vedeți starea serviciului.

Starea de funcționare normală este cea prezentată în figura de mai jos

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` sau `journalctl -xeu chasquid` poate vizualiza jurnalul de erori.

## Configurarea numelui de domeniu invers

Numele de domeniu invers trebuie să permită ca adresa IP să fie rezolvată cu numele de domeniu corespunzător.

Setarea unui nume de domeniu invers poate împiedica identificarea e-mailurilor ca spam.

Când mesajul este primit, serverul de primire va efectua o analiză inversă a numelui de domeniu pe adresa IP a serverului de trimitere pentru a confirma dacă serverul de trimitere are un nume de domeniu invers valid.

Dacă serverul de expediere nu are un nume de domeniu invers sau dacă numele de domeniu invers nu se potrivește cu adresa IP a serverului de trimitere, serverul de primire poate recunoaște e-mailul ca spam sau îl poate respinge.

Vizitați [https://my.contabo.com/rdns](https://my.contabo.com/rdns) și configurați așa cum se arată mai jos

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

După setarea numelui de domeniu invers, nu uitați să configurați rezoluția directă a numelui de domeniu ipv4 și ipv6 către server.

## Editați numele de gazdă al chasquid.conf

Modificați `conf/chasquid/chasquid.conf` la valoarea numelui de domeniu invers.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Apoi rulați `systemctl restart chasquid` pentru a reporni serviciul.

## Backup conf în depozitul git

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

De exemplu, fac o copie de rezervă a folderului conf în propriul meu proces github, după cum urmează

Creați mai întâi un depozit privat

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Intrați în directorul conf și trimiteți la depozit

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Adăugați expeditor

alerga

```
chasquid-util user-add i@wac.tax
```

Se poate adăuga expeditor

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Verificați dacă parola este setată corect

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

După adăugarea utilizatorului, `chasquid/domains/wac.tax/users` va fi actualizat, nu uitați să îl trimiteți la depozit.

## DNS adaugă înregistrare SPF

SPF (Sender Policy Framework) este o tehnologie de verificare a e-mailurilor folosită pentru a preveni frauda prin e-mail.

Acesta verifică identitatea unui expeditor de e-mail, verificând dacă adresa IP a expeditorului se potrivește cu înregistrările DNS ale numelui de domeniu pe care pretinde că este, împiedicând fraudatorii să trimită e-mailuri false.

Adăugarea înregistrărilor SPF poate împiedica pe cât posibil identificarea e-mailurilor ca spam.

Dacă serverul dumneavoastră de nume de domeniu nu acceptă tipul SPF, trebuie doar să adăugați înregistrarea de tip TXT.

De exemplu, SPF-ul `wac.tax` este următorul

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF pentru `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Rețineți că am `include:_spf.google.com` aici, deoarece voi configura `i@wac.tax` ca adresă de trimitere în căsuța poștală Google mai târziu.

## Configurare DNS DMARC

DMARC este abrevierea (Domain-based Message Authentication, Reporting & Conformance).

Este folosit pentru a capta respingerile SPF (poate cauzate de erori de configurare sau altcineva se preface că ești tu pentru a trimite spam).

Adăugați înregistrarea TXT `_dmarc` ,

Conținutul este următorul

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Semnificația fiecărui parametru este după cum urmează

### p (politică)

Indică modul de gestionare a e-mailurilor care eșuează verificarea SPF (Sender Policy Framework) sau DKIM (DomainKeys Identified Mail). Parametrul p poate fi setat la una dintre cele trei valori:

* niciunul: nu se întreprinde nicio acțiune, doar rezultatul verificării este transmis expeditorului prin mecanismul de raportare prin e-mail.
* Carantină: Puneți e-mailul care nu a trecut verificarea în dosarul de spam, dar nu va respinge mesajul direct.
* respinge: respinge direct e-mailurile care nu reușesc verificarea.

### fo (Opțiuni de eșec)

Specifică cantitatea de informații returnate de mecanismul de raportare. Poate fi setat la una dintre următoarele valori:

* 0: Raportați rezultatele validării pentru toate mesajele
* 1: Raportați numai mesajele care nu au eșuat verificarea
* d: Raportați numai eșecurile de verificare a numelui de domeniu
* s: raportați numai eșecurile de verificare SPF
* l: Raportați numai eșecurile de verificare DKIM

### rua & ruf

* rua (URI de raportare pentru rapoarte agregate): adresă de e-mail pentru primirea rapoartelor agregate
* ruf (URI de raportare pentru rapoartele criminalistice): adresa de e-mail pentru a primi rapoarte detaliate

## Adăugați înregistrări MX pentru a redirecționa e-mailurile către Google Mail

Deoarece nu am putut găsi o cutie poștală corporativă gratuită care să accepte adrese universale (Catch-All, poate primi orice e-mail trimis la acest nume de domeniu, fără restricții privind prefixele), am folosit chasquid pentru a redirecționa toate e-mailurile către căsuța mea poștală Gmail.

**Dacă aveți propria dvs. căsuță poștală de afaceri plătită, vă rugăm să nu modificați MX și să omiteți acest pas.**

Editați `conf/chasquid/domains/wac.tax/aliases` , setați căsuța poștală de redirecționare

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` indică toate e-mailurile, `i` este prefixul adresei de e-mail a utilizatorului expeditor creat mai sus. Pentru a redirecționa e-mailurile, fiecare utilizator trebuie să adauge o linie.

Apoi adăugați înregistrarea MX (indic direct la adresa numelui de domeniu invers aici, așa cum se arată în prima linie din figura de mai jos).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

După finalizarea configurației, puteți utiliza alte adrese de e-mail pentru a trimite e-mailuri către `i@wac.tax` și `any123@wac.tax` pentru a vedea dacă puteți primi e-mailuri în Gmail.

Dacă nu, verificați jurnalul chasquid ( `grep chasquid /var/log/syslog` ).

## Trimiteți un e-mail la i@wac.tax cu Google Mail

După ce Google Mail a primit e-mailul, am sperat desigur să răspund cu `i@wac.tax` în loc de i.wac.tax@gmail.com.

Accesați [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) și faceți clic pe „Adăugați o altă adresă de e-mail”.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Apoi, introduceți codul de verificare primit prin e-mailul către care a fost redirecționat.

În cele din urmă, poate fi setată ca adresă implicită a expeditorului (împreună cu opțiunea de a răspunde cu aceeași adresă).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

În acest fel, am finalizat înființarea serverului de e-mail SMTP și, în același timp, folosim Google Mail pentru a trimite și primi e-mailuri.

## Trimiteți un e-mail de test pentru a verifica dacă configurația a reușit

Introduceți `ops/chasquid`

Rulați `direnv allow` să instalați dependențe (direnv a fost instalat în procesul anterior de inițializare cu o singură cheie și a fost adăugat un cârlig la shell)

apoi fugi

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Semnificația parametrilor este următoarea

* utilizator: nume de utilizator SMTP
* trece: parola SMTP
* către: destinatar

Puteți trimite un e-mail de test.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Este recomandat să utilizați Gmail pentru a primi e-mailuri de test pentru a verifica dacă configurările au succes.

### Criptare standard TLS

După cum se arată în figura de mai jos, există această mică blocare, ceea ce înseamnă că certificatul SSL a fost activat cu succes.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Apoi faceți clic pe „Afișați e-mailul original”

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

După cum se arată în figura de mai jos, pagina de e-mail originală Gmail afișează DKIM, ceea ce înseamnă că configurarea DKIM a avut succes.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Verificați primit în antetul e-mailului original și puteți vedea că adresa expeditorului este IPV6, ceea ce înseamnă că și IPV6 este configurat cu succes.
