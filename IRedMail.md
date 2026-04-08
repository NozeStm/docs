Voici une documentation complète et détaillée pour l'installation et la configuration d'un serveur de messagerie avec **iRedMail**, basée sur le support de TP de la **SODECAF** et la documentation officielle.

---

# Documentation : Mise en place d'un Serveur de Mails avec iRedMail

## 1. Présentation de la Solution
Pour répondre au besoin de communication interne de l'entreprise **SODECAF**, la solution **iRedMail** a été retenue. Il s'agit d'une pile logicielle "tout-en-un" open source regroupant les briques suivantes :
* **Postfix** : Agent de transfert de mail (MTA) pour l'envoi, la réception et le routage SMTP.
* **Dovecot** : Serveur IMAP et POP3 gérant le stockage des messages et l'authentification des utilisateurs.
* **Roundcube** : Interface web (Webmail) pour la lecture et l'organisation des messages.
* **Sécurité** : Intégration de ClamAV (antivirus), SpamAssassin (antispam) et Fail2Ban (protection contre les intrusions).
* **Base de données** : MariaDB ou OpenLDAP pour la gestion des comptes et domaines.

---

## 2. Prérequis Système
* **Système d'exploitation** : Une installation fraîche de **Debian** ou **Ubuntu** est impérative.
* **Mémoire vive** : 4 Go de RAM minimum sont recommandés pour un serveur en production avec les filtres antispam activés.
* **Réseau** : Le **port 25** doit être ouvert pour permettre la communication avec les autres serveurs de messagerie.

---

## 3. Préparation du Serveur

### Configuration du Nom d'Hôte (FQDN)
Il est crucial de définir un nom d'hôte pleinement qualifié (ex: `mx.sodecaf.local`).
1.  Éditer `/etc/hostname` pour y mettre le nom court (ex: `mx`).
2.  Éditer `/etc/hosts` en plaçant le FQDN en premier sur la ligne de l'IP locale :
    `127.0.0.1 mx.sodecaf.local mx localhost`

### Installation des dépendances
Mettez à jour vos dépôts officiels et installez les outils nécessaires à l'installeur :
```bash
sudo apt-get update
sudo apt-get install -y gzip dialog
```

---

## 4. Procédure d'Installation

### Téléchargement et Lancement
1.  Téléchargez la dernière version stable d'iRedMail sur le site officiel.
2.  Décompressez l'archive et lancez le script :
    ```bash
    tar zxf iRedMail-x.y.z.tar.gz
    cd iRedMail-x.y.z/
    bash iRedMail.sh
    ```

### Étapes de l'Assistant Graphique
L'installeur posera plusieurs questions clés :
* **Stockage des mails** : Par défaut dans `/var/vmail/`.
* **Backend** : Choisissez le moteur de base de données (MariaDB est recommandé pour sa simplicité de gestion).
* **Premier Domaine** : Saisissez votre domaine de messagerie (ex: `sodecaf.local`).
* **Administrateur** : Définissez le mot de passe du compte `postmaster` (cet identifiant sert aussi de compte mail complet).
* **Composants optionnels** : Sélectionnez **Roundcube** pour un webmail léger ou **SOGo** si vous avez besoin de synchronisation calendrier/contacts (CalDAV/CardDAV).

---

## 5. Configuration Réseau (Ports et Flux)
Pour que le service soit fonctionnel, les ports suivants doivent être autorisés :

| Service | Port | Description |
| :--- | :--- | :--- |
| **SMTP** | 25 | Réception et envoi entre serveurs |
| **HTTP/HTTPS** | 80 / 443 | Accès au Webmail et au panneau d'administration |
| **IMAPS** | 993 | Accès sécurisé aux messages (recommandé) |
| **POP3S** | 995 | Récupération sécurisée des messages |
| **Submission** | 587 | Envoi de mails depuis un client lourd (Thunderbird, Outlook) via TLS |

---

## 6. Post-Installation et Administration

### Informations Importantes
Consultez le fichier `/root/iRedMail-x.y.z/iRedMail.tips` généré après l'installation. Il contient :
* Tous les identifiants et mots de passe par défaut.
* L'emplacement des fichiers de configuration des différents services.

### Accès aux interfaces
Remplacez `votre_serveur` par l'IP ou le nom de votre machine :
* **Webmail (Roundcube)** : `https://votre_serveur/mail/`
* **Panneau d'administration (iRedAdmin)** : `https://votre_serveur/iredadmin/`

---
*Documentation réalisée pour le contexte SODECAF - Support iRedMail Officiel - 2026*
