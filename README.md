# linuxmuster-mail für linuxmuster.net 6.2

Skripte und Konfigurationsdateien für linuxmuster.net zur Einrichtung eines voll ausgestatteten Docker-Mailservers mit Anbindung an den LDAP des linuxmuster.net 6.2 Servers.

- Verwendet das [Dockerimage](https://hub.docker.com/r/tvial/docker-mailserver/) von [Thomas Vial](https://hub.docker.com/r/tvial).
- Pre-Release Paket: https://github.com/ironiemix/linuxmuster-mail/releases/tag/0.3.4

## Voraussetzungen 

- Ein ubuntu/debian Server mit docker und docker-compose. Getestet mit ubuntu 18.04, docker ce 19.03 und docker-compose 1.24.0-rc1
- Auf dem Docker Server sollte ein letsecrypt Zertifikat für den Hostnamen des Mailservers vorhanden sein, z.B. indem man einen Apache Webserver und dehydrated installiert.

## Installation 

- deb-Paket herunterladen und installieren: https://github.com/ironiemix/linuxmuster-mail/releases/tag/0.3.4
- Eine Beispielhafte ini-Datei findet sich unter /usr/share/linuxmuster-mail/templates/mailserver.ini-example
```
[setup]
# Hostname des docker-mailservers
mailhostname=mbox
# Maildomain der Mailadressen
domainname=mymaildomain.de
# IP Adresse oder Servername des LDAP Servers
# (also des LMN6.2 Servers)
serverip=192.168.111.111
# dieser Benutzer muss auf dem Server angelegt werden und
# in der Konfiguration des slapd 
# mit den entsprechenden Rechten versehen werden
binduser=mailserverldapuser
binduserpw=geheim
# BaseDN wie auf dem Server in der Datei 
# /var/lib/linuxmuster/network.settings
# hinterlegt
basedn=dc=linuxmuster,dc=lokal
# Wenn die Konfiguration einen Smarthost
# zum Mailversand verwenden soll, sonst leer
smtprelay=
smtpuser=
smtppw=
```
- Der Befehl ``linuxmuster-mail.py -c mailserver.ini`` erzeugt die Konfigurationsdateien für den Container

