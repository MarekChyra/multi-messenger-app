## Úvod
Tento projekt som začal robiť, lebo som mal ľudí na rôznych sociálnych sieťach prevažne na Instagrame a na Discorde ale môžete to aplikovať aj na aplikácii Slack, Facebook Messenger, Telegram, Whatsapp, Signal, SMS, RCS, google chat, X/Twitter a iMessage (iMessage funguje tak, že raz funguje a raz nie, ďakujem Apple).

## Inštrukcie (Instagram)
### Linux
1. Nainštalujte si virtualizačnú platformu [Docker](https://www.docker.com/products/docker-desktop/)
2. Stiahnite si konfiguračný súbor [docker-compose.yml](/docker-compose.yml). V tom súbore si zmeňte `$domena` v `SYNAPSE_SERVER_NAME=$domena` na vašu doménu, čo budete používať (nemusíte ju vlastniť, bude to len lokálna identifikácia) a všade, kde sa ocitne `$domena` to prepíšete na vašu doménu. A ešte si zmeňte `$db_user` a `$db_pass` v `POSTGRES_USER=$db_user` a `POSTGRES_PASSWORD=$db_pass`.
3. Spustite `docker-compose run --rm -e SYNAPSE_SERVER_NAME=$domena -e SYNAPSE_REPORT_STATS=yes synapse generate` (nezabudnite prepísať `$domena`)
4. Vytvoria sa vám priečinky `schemas`, `files` a `mautrix-instagram`. 
5. Otvorte priečinok `mautrix-instagram` a stiahnite tam [config.yaml](/config.yaml), otvorte `config.yaml` a zmeňte `$domena`, `$db_user` a `$db_pass`
```yaml
homeserver:
  ...
  domain: $domena
  ...
appservice:
  ...
  database: postgres://$db_user:$db_pass@db:5432/test_db
  ...
```
6. Do `files` si stiahnite [double-puppeting-registration.yaml](/double-puppeting-registration.yaml), [homeserver.yaml](/homeserver.yaml) a **skopírujte** z mautrix-instagram `registration.yaml` a premenujte to na `mautrix-instagram-registration.yaml`
7. V double-puppeting-registration.yaml zmeňte `$domena`:
```yaml
...
namespaces:
  users:
  - regex: '@.*:$domena'
...
```
8. V priečinku `files` otvorte `homeserver.yaml` a zmeňte `$domena`, `$db_user` a `$db_pass`:
```yaml
...
server_name: "$domena"
...
database: 
  ...
  args: 
    user: $db_user
    password: $db_password
    ...
```
#### SSL
Je potrebný pre klienty, ktoré sa nenachádzajú priamo na serveri.

9. Nainštalujte si openssl (malo by to byť v základe na linuxe)
##### Generovanie CA
10. Generovanie RSA `openssl genrsa -aes256 -out ca-key.pem 4096`
11. Generovanie verejného certifikátu CA `openssl req -new -x509 -sha256 -days 3650 -key ca-key.pem -out ca.pem`
##### Generovanie certifikátu
12. Vytvorenie kľúča RSA `openssl genrsa -out cert-key.pem 4096`
13. Vytvorenie žiadosti o podpísanie certifikátu (CSR) `openssl req -new -sha256 -subj "/CN=$domena" -key cert-key.pem -out cert.csr`. Zmeňte `$domena` na vašu doménu.
14. Vytvorenie súboru `extfile` so všetkými alternatívnymi názvami `echo "subjectAltName=DNS:$domena,IP:$ipPC" >> extfile.cnf`.  Zmeňte `$domena` na vašu doménu a `$ipPC` na VPN ip vašho pc.
	(voliteľné) `echo extendedKeyUsage = serverAuth >> extfile.cnf`
15. Vytvorenie certifikátu `openssl x509 -req -sha256 -days 3650 -in cert.csr -CA ca.pem -CAkey ca-key.pem -out cert.pem -extfile extfile.cnf -CAcreateserial`
16. Tvorba `fullchain.pem`: `cat cert.pem ca.pem > fullchain.pem`
##### Aplikovanie certifikátu
17. Skopírujeme si `fullchain.pem` a `cert-key.pem` do `files`
#### Inštalácia CA certifikátu ako dôveryhodnej koreňovej CA

##### On Debian & Derivatives
- Presun certifikátu CA (`ca.pem`) do súboru `/usr/local/share/ca-certificates/ca.crt`.
- Aktualizácia úložiska Cert Store pomocou:
```bash
sudo update-ca-certificates
```

Pozrite si dokumentáciu [tu](https://wiki.debian.org/Self-Signed_Certificate) a [tu](https://manpages.debian.org/buster/ca-certificates/update-ca-certificates.8.en.html).

##### On Fedora
- Presun certifikátu CA (`ca.pem`) do súboru `/etc/pki/ca-trust/source/anchors/ca.pem` alebo `/usr/share/pki/ca-trust-source/anchors/ca.pem`
- Teraz spustite (v prípade potreby so sudo):
```bash
update-ca-trust
```

Pozrite si dokumentáciu [tu](https://docs.fedoraproject.org/en-US/quick-docs/using-shared-system-certificates/).
##### On Arch
Celosystémové - Arch(p11-kit) (Z arch wiki)
- Spustiť (ako root)
```bash
trust anchor --store myCA.crt
```
- Certifikát sa zapíše do adresára /etc/ca-certificates/trust-source/myCA.p11-kit a adresáre "legacy" sa automaticky aktualizujú.
- Ak sa zobrazí hlásenie "no configured writable location" alebo podobná chyba, importujte CA manuálne:
- Skopírujte certifikát do adresára /etc/ca-certificates/trust-source/anchors.
- a potom
```bash 
update-ca-trust
```
wiki stránka [tu](https://wiki.archlinux.org/title/User:Grawity/Adding_a_trusted_CA_certificate)

##### On Windows

Predpokladajte, že cesta k vášmu vygenerovanému certifikátu CA je `C:\ca.pem`, spustite:
```powershell
Import-Certificate -FilePath "C:\ca.pem" -CertStoreLocation Cert:\LocalMachine\Root
```
- Nastavte `-CertStoreLocation` na `Cert:\CurrentUser\Root` v prípade, že chcete dôverovať certifikátom len pre prihláseného používateľa.

ALEBO

V príkazovom riadku spustite:
```sh
certutil.exe -addstore root C:\ca.pem
```

- `certutil.exe` je vstavaný nástroj (klasický `System32`) a pridáva celosystémovú kotvu dôvery.

##### On Android

Presný postup sa líši od zariadenia k zariadeniu, ale tu je všeobecný návod:
1. Otvorte Nastavenia telefónu
2. Vyhľadajte časť `Šifrovanie a poverenia`. Všeobecne sa nachádza v časti `Nastavenia > Zabezpečenie > Šifrovanie a poverenia`.
3. Vyberte možnosť `Inštalovať certifikát`
4. Vyberte `CA certifikát`
5. Vyhľadajte súbor certifikátu `ca.pem` na karte SD/internom úložisku pomocou správcu súborov.
6. Vyberte, že ju chcete načítať.
7. Hotovo!
