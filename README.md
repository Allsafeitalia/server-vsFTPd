# Introduzione

Tra i vari server FTP disponibili in Debian, VSFTPD è a mio parere quello più snello, sicuro e prestazionale; inoltre, come ulteriore garanzia, è il server FTP scelto da Red Hat e consigliato da IBM.

In questa guida vedremo come installare un server VSFPD con un backend su database MySQL. Questa configurazione si adatta ad esempio alle seguenti esigenze:

- abbiamo bisogno di più utenti FTP che non si intralcino tra loro
- non vogliamo che gli utenti FTP siano anche utenti del server
- vogliamo gestire le home directory degli utenti in modo personalizzato

Questo è ad esempio il caso di un service provider, che vuole fornire accesso FTP ai propri utenti.

## Prerequisiti

Dopo aver installato Apache (_Installare un ambiente LAMP: Linux, Apache2, SSL, MySQL, PHP5 oppure Installare un ambiente LAMP: Linux, Apache2, SSL, MySQL, PHP5 - Stretch_) e configurato i Virtual Host (_Apache e Virtual Hosts: configurare Apache2 per ospitare più siti web_) abbiamo adesso bisogno di permettere ai proprietari dei domini ospitati sui Virtual Host di accedere al loro spazio web via FTP senza causare danni agli altri Virtual Host e senza avere la possibilità di gironzolare per il nostro server.

Supponiamo quindi di avere una situazione in cui vogliamo creare due utenti virtuali, senza accesso alla console, che possano ad esempio accedere via FTP solo alle directory del loro sito web.

## Installazione di VSFTPD

L'installazione di VSFTPD è semplice:
```bash
apt-get install vsftpd libpam-mysql
```
VSFTPD non ha un supporto built-in per MySQL, per cui le libpam-mysql sono fondamentali per permettere a vsftpd di leggere gli utenti presenti su mysql.
Aggiungiamo anche un utente di sistema per VSFTPD, che ci servirà in seguito:
```bash
useradd --home /home/vsftpd --gid nogroup -m --shell /bin/false vsftpd
```
Questo sarà l'utente di sistema con i cui permessi girerà il demone, aumentando la sicurezza del server.

## Creazione di un database per VSFTPD
Il nostro demone FTP è già in funzione, ma non è ancora collegato ad alcun database MySQL. Quindi apriamo la shell di MySQL e creiamo il nostro database vsftpd, con proprietario un utente vsftpd e password ftpdpass:
```bash
mysql -u root -p
```
```bash
CREATE DATABASE vsftpd;
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP ON vsftpd.* TO 'vsftpd'@'localhost' IDENTIFIED BY 'ftpdpass';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP ON vsftpd.* TO 'vsftpd'@'localhost.localdomain' IDENTIFIED BY 'ftpdpass';
FLUSH PRIVILEGES;
```
sostituendo ovviamente la stringa ftpdpass con la password che vogliamo utilizzare.
Ora che abbiamo il database dobbiamo creare la tabella per memorizzare gli utenti virtuali del nostro server FTP. Restando sempre nella shell di MySQL diamo quindi i seguenti comandi:
```bash
USE vsftpd;
```
```bash
CREATE TABLE `accounts` (
`id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY ,
`username` VARCHAR( 30 ) NOT NULL ,
`pass` VARCHAR( 50 ) NOT NULL ,
`homedir` VARCHAR( 900 ) NOT NULL ,
`active` int(11) NOT NULL,
UNIQUE (
`username`
)
) ENGINE = MYISAM ;
```
```bash
quit;
```
Il database è stato creato. Per la normale amministrazione del database e per la creazione degli utenti virtuali potremo d'ora in poi servirci di phpMyAdmin, se lo abbiamo installato; altrimenti dovremo continuare ad utilizzare la shell di MySQL.

## Configurazione di VSFTPD
Facciamo prima di tutto una copia di backup del file originale di configurazione di VSTFPD, quindi creiamone uno personalizzato e editiamolo:
```bash
cp /etc/vsftpd.conf /etc/vsftpd.conf_orig
cat /dev/null > /etc/vsftpd.conf
nano /etc/vsftpd.conf
```
Il nostro file di configurazione avrà come contenuto:

```c++
# Standalone server
listen=YES

# Disabilito FTP anonimo
anonymous_enable=NO
#anon_upload_enable=NO
#anon_mkdir_write_enable=NO
#anon_other_write_enable=NO

# Imposto la visualizzazione di messaggi
dirmessage_enable=YES
#force_dot_files=NO

# Certificato per FTP via SSL
rsa_cert_file=/etc/ssl/certs/vsftpd.pem


################################
# Configurazioni di base
################################

#hide_ids=YES
listen_port=21
connect_from_port_20=YES

# Abilito le porte passive
pasv_enable=YES
pasv_min_port=60000
pasv_max_port=65000
pasv_addr_resolve=YES

# Abilito utenti locali e/o virtuali
local_enable=YES
local_umask=022
max_login_fails=3
max_per_ip=4
# Senza il server sarebbe read-only
write_enable=YES

# Definisco il sistema di autenticazione
pam_service_name=vsftpd

# Imposto la directory per le configurazioni speciali sugli utenti
user_config_dir=/etc/vsftpd/user_conf

# Imposto il banner di benvenuto
ftpd_banner=EasyLAB Doc FTP Server
#force_dot_files=yes

# I file caricati hanno proprietario www-data
chown_uploads=YES
chown_username=www-data


############################
#Configuro la gabbia chroot
############################
chroot_local_user=YES
secure_chroot_dir=/var/run/vsftpd
user_sub_token=$USER
local_root=/home/vsftpd/$USER

# Mappo gli utenti verso un utente locale
virtual_use_local_privs=YES

# Attivo gli utenti virtuali
guest_enable=YES

# Ogni utente viene mappato come vsftpd
guest_username=vsftpd
nopriv_user=vsftpd


##################
#Gestione dei LOG
##################

log_ftp_protocol=YES
vsftpd_log_file=/var/log/vsftpd.log
dual_log_enable=YES
xferlog_enable=YES
xferlog_std_format=YES
```
L'elenco delle direttive di VSFTPD con una spiegazione dettagliata del significato è presente sul sito ufficiale: vsftpd.conf
Una delle direttive specificate in vsftpd.conf è user_config_dir, che abbiamo impostato a /etc/vsftpd/user_conf. Questo parametro dice a VSFTPD dove cercare le impostazioni specifiche per ogni utente ed è il modo più semplice per avere una home diversa per ognuno di loro, ma anche per poter avere più users con la stessa home.
Assicuriamoci quindi di creare la directory che conterrà le configurazioni specifiche:
```bash
mkdir -p /etc/vsftpd/user_conf
```
Il formato dei file è molto semplice; basta creare un file con il nome utente presente nel database MySQL e inserire poche righe di testo:

```bash
dirlist_enable=YES
download_enable=YES
local_root=/var/www/sitoweb
```
Impostiamo i permessi corretti:

```bash
chown -R vsftpd:www-data /var/www/sitoweb
```
## Connessione al database MySQL
Come ultima cosa dobbiamo istruire il nostro VSFTPD affinchè non cerchi gli utenti in /etc/passwd, ma nel database che abbiamo creato. Innanzitutto creiamo una copia di backup del file di configurazione, quindi ne impostiamo uno personalizzato:
```bash
cp /etc/pam.d/vsftpd /etc/pam.d/vsftpd_orig
cat /dev/null > /etc/pam.d/vsftpd
nano /etc/pam.d/vsftpd
```
con contenuto:
```bash
auth required pam_mysql.so user=vsftpd passwd=ftpdpass host=localhost db=vsftpd table=accounts usercolumn=username passwdcolumn=pass crypt=2 verbose=1 debug=1
account required pam_mysql.so user=vsftpd passwd=ftpdpass host=localhost db=vsftpd table=accounts usercolumn=username passwdcolumn=pass crypt=2 verbose=1 debug=1
```
E' possibile cambiare i valori dei parametri crypt, debug e verbose a piacimento, tuttavia almeno all'inizio è preferibile avere dei log prolissi.
A causa di un baco, è necessario installare anche le librerie:
```bash
apt-get install libgcc1 lib32gcc1 libx32gcc1 libpam-ldap
```
Non preoccupatevi della configurazione di `libpam-ldap` e lasciate tutte le impostazioni di default, a meno che non abbiate un server OpenLDAP attivo sulla macchina.
Alla fine di tutto riavviamo il nostro server:
```bash
/etc/init.d/vsftpd restart
```
## Creazione del primo utente virtuale
Per creare il nostro primo utente virtuale possiamo usare la shell di MySQL:
```bash
mysql -u root -p
```
```bash
USE vsftpd;
```
Creeremo un utente chiamato `testuser` con password `secret`:
```bash
INSERT INTO accounts (username, pass, homedir) VALUES('testuser', PASSWORD('secret'), '/var/www/testuser');
quit;
```
Purtroppo VSFTPD non crea la home directory automaticamente; dobbiamo quindi crearla a mano e impostare i permessi corretti:
```bash
mkdir /home/vsftpd/testuser
chown vsftpd:nogroup /home/vsftpd/testuser
```
Creiamo infine il file di configurazione per il nostro utente, in modo che vengano sovrascritte le impostazioni di default di VSFTPD sulle homedir:
```bash
nano /etc/vsftpd/user_conf/testuser
```
con contenuto:
```c++
dirlist_enable=YES
download_enable=YES
local_root=/var/www/testuser
```
Se tutto è andato per il verso giusto, dovreste riuscire a puntare il vostro client FTP sul server appena installato ed effettuare il login con l'utente appena creato.

## Configurazione di TLS
Il protocollo FTP è un protocollo estremamente insicuro, poichè tutti i dati viaggiano in chiaro. Fortunatamente è possibile aumentare la sicurezza del nostro server configurando TLS per crittare tutte le comunicazioni. Iniziamo installando OpenSSL:
```bash
apt-get install openssl
```
Quindi creiamo il certificato da utilizzare per VSFTPD:
```bash
mkdir -p /etc/vsftpd/ssl
```
```bash
chmod 700 /etc/vsftpd/ssl
```
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:1024 -keyout /etc/vsftpd/ssl/vsftpd.pem -out /etc/vsftpd/ssl/vsftpd.pem
```
e inserendo le risposte che fanno al caso nostro:
```c++
Country Name (2 letter code) [AU]: IT
State or Province Name (full name) [Some-State]: Italy
Locality Name (eg, city) []: Lodi
Organization Name (eg, company) [Internet Widgits Pty Ltd]: Ferdy FTP Server
Organizational Unit Name (eg, section) []: Dipartimento IT
Common Name (eg, YOUR name) []: serverferdy.local
Email Address []: ferdy@xxxx.it
```
Quindi modifichiamo il file di configurazione di VSFTPD e aggiungiamo la sezione:
```bash
nano /etc/vsftpd.conf
```
```c++
# Abilito SSL
ssl_enable=YES
allow_anon_ssl=YES

# YES = forzo SSL per tutte le comunicazioni
# NO = permetto anche comunicazioni non crittate
force_local_data_ssl=NO
force_local_logins_ssl=NO

# Permetto TLS v1
ssl_tlsv1=YES

# Permetto SSL v2
ssl_sslv2=YES

# Permetto SSL v3
ssl_sslv3=YES

# Disabilito SSL session reuse (da usare con WinSCP)
require_ssl_reuse=NO

# Imposto il tipo di crittazione
ssl_ciphers=HIGH

# Il percorso del certificato
rsa_cert_file=/etc/vsftpd/ssl/vsftpd.pem
```
Con un riavvio del server:

```bash
/etc/init.d/vsftpd restart
```
rendiamo attive le modifiche.
Da adesso sarà possibile configurare il nostro client FTP per utilizzare sessioni crittate.
Tramite uno script perl[1] è possibile gestire facilmente le utenze. Il software mantiene allineato il database con i relativi file di configurazione necessario per ogni utente.








