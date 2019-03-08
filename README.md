# linuxmuster-mail für linuxmuster.net 6.2

Skripte und Konfigurationsdateien für linuxmuster.net zur Einrichtung eines voll ausgestatteten Docker-Mailservers mit Anbindung an den LDAP des linuxmuster.net 6.2 Servers.

- Verwendet das [Dockerimage](https://hub.docker.com/r/tvial/docker-mailserver/) von [Thomas Vial](https://hub.docker.com/r/tvial).
- Pre-Release Paket: https://github.com/ironiemix/linuxmuster-mail/releases/tag/0.3.4

## Voraussetzungen 

- Eine Maildomain für die LDAP Benutzer, passende Einträge im DNS für Host und MX.
- Ein ubuntu/debian Server mit docker und docker-compose. Getestet mit ubuntu 18.04, docker ce 19.03 und docker-compose 1.24.0-rc1
- Auf dem Docker Server sollte ein letsecrypt Zertifikat für den Hostnamen des Mailservers vorhanden sein, z.B. indem man einen Apache Webserver und dehydrated installiert. Das Zertifikat sollte nach jeder Erneuerung in das Verzeichnis ``/srv/docker/linuxmuster-mail/ssl/live/<hostname>/*`` kopiert werden, bei dehydrated geht das elegant mit einem hook.
- Auf dem linuxmuster.net Server (6.2) muss ein Benutzer eingerichtet werden, der es dem dovecot des Mailserver-Containers ermöglicht, auf die Passwörter des LDAP zuzugreifen.
  - Am besten legt man einfach einen Lehrer an, z.B. ``ldaplehrer`` mit einem Passwort ``sehrgeheim``
  - In der Datei /etc/ldap/slapd.conf sucht man nun die Zeile mit dem Eintrag  ``userPassword`` unterhalb derer sich bereits einige Zugriffsregeln finden. Dort trägt man eine Zeile zur Leseberechtigung (read) des eben angelegten Benutzers ein und startet dann den LDAP neu (``/etc/init.d/slapd restart``). Die BaseDN muss natürlich passend angepasst werden, die Benutzerdn ist letztlich immer ``uid=benutzer,ou=accounts,BASEDN``.
```
# Limits Access:
access to attrs=sambaLMPassword,sambaNTPassword,sambaPwdLastSet,sambaPwdMustChange,sambaAcctFlags,userPassword
      by dn="uid=ldaplehrer,ou=accounts,dc=linuxmuster,dc=lokal" read
      [... weitere Einträge ...]
```
  
## Installation auf denm Docker Host

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
- Nun kann man den Container mit dem Befehl ``docker-compose up -d`` starten und testen.

## Weitere Hinweise

Benutzername für alle Dienste auf dem Mailserver ist die vollständige Mailadresse wie iim LDAP hinterlegt, allerdings muss die im LDAP hinterlegte Maildomain dieselbe sein wie in der Docker-Mailserver Konfiguration. Das Passwort wird gegen den LDAP abgeglichen. bei mir stimmen die Maildomain und die interne Domain überhaupt nicht überein, außerdem fliegen die Mailadressen ja eh ständig aus dem LDAP, also pfeife ich das mit eine Skript nach den sophomorix-Operationen wieder rein.

```
!/bin/bash

MAILDOMAIN="qgmail.de"

# Groups to set mail for
# Es bekommen nur Gruppen die =1 sind
# die Mailadressen gesetzt.
declare -A mailgroups=( 
 [5a]=0   [5b]=0   [5c]=0  [5d]=0
 [6a]=1   [6b]=1   [6c]=0  [6d]=1
 [7a]=1   [7b]=1   [7c]=1  [7d]=1
 [8a]=1   [8b]=1   [8c]=1  [8d]=1
 [9a]=1   [9b]=1   [9c]=1  [9d]=1
 [10a]=1  [10b]=1  [10c]=1 [10d]=1
 [ks1]=1  [ks2]=1 
 [mailtest]=1
)

# Alle uids ohne die Maschinenaccounts
ALLUIDS=$(ldapsearch -x -H "ldaps://august.qg-moessingen.de:636" -b "dc=qg-moessingen,dc=de"  | grep uid: | awk '{print $2}' | grep -v \$$)

for auid in $ALLUIDS; do
    # Primaere Gruppe
    key=$(id -gn $auid)
    # Soll die Mails bekommen?
    if [ "${mailgroups[$key]}" == "1" ]; then
        echo "Setting mail from user $auid in group $key to ${auid}@$MAILDOMAIN"
        mailaddress=${auid}@${MAILDOMAIN}
        smbldap-usermod -M $mailaddress $auid
    fi
done

```
das kann man z.B. einfach dahingehend erweitern, dass man für SuS und Lehrer verschiende Domains verwendet (was ich benötigte).
