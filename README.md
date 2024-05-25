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


