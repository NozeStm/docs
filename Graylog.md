# Installation de Graylog sur Debian 12
## 1. Mise à jour des paquets
```bash
apt update && sudo apt upgrade -y
```
## 2. Configurer le fuseau horaire de la Debian 12
Configuration du fuseau horaire
```bash
timedatectl set-timezone Europe/Paris
```
Configuration du serevur NTP
```bash
vim /etc/timesync.conf (à vérifier)
```
## 3. Installation d'outils
```bash
apt-get install curl lsb-release a-certificates gnupg2 pwgen
```
## 4. Installation de MongoDB
```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-6.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg --dearmor
```
