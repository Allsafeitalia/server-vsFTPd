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

```bash
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




