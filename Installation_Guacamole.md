# Documentation : Mise en place d'un Bastion Apache Guacamole sur Debian

## 1. Présentation de la Solution
Apache Guacamole est une passerelle de bureau à distance "clientless" (sans client). Elle permet d'accéder à des machines distantes via les protocoles standards (**RDP, SSH, VNC**) directement depuis un navigateur web grâce au HTML5. 

### Rôle de Bastion d'Administration
Utiliser Guacamole comme bastion permet de :
* **Centraliser les accès** : Un seul portail web pour toute l'infrastructure.
* **Isoler le réseau** : Seul le serveur Guacamole est exposé (souvent via HTTPS), les serveurs cibles restent sur le réseau privé.
* **Auditer et tracer** : Enregistrement vidéo des sessions et journaux de connexions.
* **Sécuriser** : Support natif du MFA (authentification multi-facteurs).

---

## 2. Architecture et Composants
Le fonctionnement repose sur deux composants majeurs :
1.  **Guacamole Server (guacd)** : Le démon natif qui gère les protocoles de bureau à distance et communique avec les machines cibles.
2.  **Guacamole Client** : Une application web Java qui sert l'interface aux utilisateurs.
3.  **Base de données** (MySQL/MariaDB ou PostgreSQL) : Pour stocker les utilisateurs, les groupes et les configurations de connexion.

---

## 3. Prérequis
* Un serveur sous **Debian 11 ou 12** à jour.
* Accès privilège (**sudo** ou root).
* Installation de Docker et Docker Compose (méthode recommandée pour la simplicité de maintenance).

---

## 4. Installation via Docker Compose
L'utilisation de Docker permet de déployer l'intégralité de la stack rapidement.

### Préparation du dossier
```bash
# Création du répertoire de travail
mkdir ~/guacamole-bastion && cd ~/guacamole-bastion
```

### Configuration du fichier `docker-compose.yml`
Ce fichier définit les services nécessaires : `guacd`, `guacamole` (l'interface) et `mysql` (la base de données).

```yaml
version: '3'
services:
  # Démon de communication avec les protocoles distants
  guacd:
    image: guacamole/guacd
    container_name: guacd
    restart: always

  # Base de données pour les paramètres et utilisateurs
  mysql:
    image: mysql:8.0
    container_name: guacamole-db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 'votre_mot_de_passe_root'
      MYSQL_DATABASE: 'guacamole_db'
      MYSQL_USER: 'guacamole_user'
      MYSQL_PASSWORD: 'votre_mot_de_passe_utilisateur'

  # Interface web de Guacamole
  guacamole:
    image: guacamole/guacamole
    container_name: guacamole-client
    restart: always
    ports:
      - "8080:8080"
    environment:
      GUACD_HOSTNAME: guacd
      MYSQL_HOSTNAME: mysql
      MYSQL_DATABASE: guacamole_db
      MYSQL_USER: guacamole_user
      MYSQL_PASSWORD: votre_mot_de_passe_utilisateur
    depends_on:
      - guacd
      - mysql
```

---

## 5. Configuration Initiale
1.  **Lancement** : Exécuter `docker-compose up -d`.
2.  **Premier accès** : Naviguez vers `http://<IP_SERVEUR>:8080/guacamole/`.
3.  **Identifiants par défaut** : Utilisez `guacadmin` / `guacadmin`.
4.  **Sécurisation** : Créez immédiatement un nouvel utilisateur administrateur, puis désactivez ou supprimez le compte par défaut `guacadmin`.

---

## 6. Ajout de Connexions

### Configuration RDP (Windows)
* **Nom** : Serveur-Compta-01
* **Protocole** : RDP
* **Paramètres réseau** : Indiquez l'adresse IP et le port (3389).
* **Authentification** : Renseignez les identifiants Windows ou laissez vide pour une demande à la connexion.
* **Option importante** : Cochez "Ignore server certificate" si vous utilisez des certificats auto-signés sur Windows.

### Configuration SSH (Linux)
* **Nom** : Serveur-Web-Prod
* **Protocole** : SSH
* **Paramètres réseau** : Adresse IP et port (22).
* **Authentification** : Supporte le mot de passe ou l'import d'une **clé privée SSH**.

---

## 7. Fonctionnalités de Bastion Avancées

### Enregistrement de session
Dans les paramètres d'une connexion, section "Screen Recording" :
* Indiquez un chemin sur le serveur (ex: `/var/lib/guacamole/recordings`).
* Le fichier généré est un format `.guac` qui peut être converti en vidéo pour des besoins d'audit juridique ou technique.

### Partage de session
Guacamole permet de générer un lien de partage temporaire (en lecture seule ou avec contrôle) pour collaborer à distance sur une même session sans transmettre les identifiants.

---
*Documentation réalisée pour un usage personnel - Référence : Tutoriel IT-Connect - 2026*
