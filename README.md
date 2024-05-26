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



