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

##3. Génération du certificat de l'autorité de certification
- Création de la clé privée de l'autorité de certification
```bash
