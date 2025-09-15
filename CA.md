# Mis en oeuvre de l'HTTPS sur un serveur web
# Utilisation d'une autrité de certification interne

## 1. Préparation de la machine CA
- Configuration IP : /etc/network/interfaces
```bash
allow-hotplug ens33
iface ens33 inet static
address 172.16.0.20/24
gateway 172.16.0.254
```
- Installation d'openssl
```bash
apt update && apt upgrade -y
apt install openssl
```
#2

##3. Génération du certificat de l'autorité de certification
- Création de la clé privée de l'autorité de certification
```bash
openssl genrsa -ds3
chmod 400 cakey.pem
```
-Création du certificat auto-signé de l'autorité de certification
```bash
cd /etc/ssl/sodecaf
openssl req -news -days 
