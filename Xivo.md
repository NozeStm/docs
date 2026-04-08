Voici une documentation technique complète et détaillée pour l'installation et la configuration de **XiVO**, structurée pour ton dépôt GitHub. Elle synthétise les instructions du site officiel et les procédures spécifiques de tes supports de TP.

---

# Documentation Technique : Déploiement de XiVO IPBX

## 1. Installation du Système (Base Debian)
L'installation repose sur une distribution Debian 12 "Bookworm" (64-bit) minimale.

### Prérequis Réseau
Avant de lancer l'installation, les interfaces réseau doivent suivre le nommage traditionnel (`ethX`).
1.  Modifier le GRUB : `nano /etc/default/grub`.
2.  Ajouter `net.ifnames=0` à la ligne `GRUB_CMDLINE_LINUX`.
3.  Mettre à jour : `update-grub` et redémarrer.
4.  Vérifier que l'interface est bien nommée `eth0` dans `/etc/network/interfaces`.

### Script d'Installation
Exécuter les commandes suivantes en tant que root :
```bash
wget http://mirror.xivo.solutions/xivo_install.sh
chmod +x xivo_install.sh
./xivo_install.sh -a 2025.10-latest
```
*Note : XiVO utilise par défaut les sous-réseaux Docker 172.17.0.0/16 et 172.18.1.0/24.*

---

## 2. Configuration Initiale (Wizard)
Une fois l'installation terminée, accédez à `https://<IP_DU_SERVEUR>` pour lancer l'assistant de configuration.

* **Hostname** : `xivo-sodecaf` (sans majuscules).
* **Domaine** : `sodecaf.local`.
* **Contexte interne** : Définit la plage de numéros (ex: 100 à 500) pour les appels internes.
* **Mot de passe Admin** : À définir pour l'accès à l'interface d'administration.

---

## 3. Gestion des Utilisateurs et des Lignes
Dans XiVO, un utilisateur doit toujours être lié à une ligne pour pouvoir passer des appels.

### Création d'une Ligne SIP
1.  Aller dans **Services > IPBX > Gestion de l'IPBX > Lignes**.
2.  Ajouter une nouvelle ligne.
3.  **Protocole** : Choisir `SIP`.
4.  **Numéro** : Assigner un numéro court (ex: 201).
5.  **Contexte** : Sélectionner le contexte interne (ex: `default`).

### Création de l'Utilisateur
1.  Aller dans **Services > IPBX > Gestion de l'IPBX > Utilisateurs**.
2.  Cliquer sur **Ajouter**.
3.  Saisir le **Nom** et le **Prénom**.
4.  Dans l'onglet **Lignes**, associer la ligne SIP précédemment créée.

---

## 4. Ajout et Configuration des Téléphones
XiVO gère les terminaux via un système de "Greffons" (plugins) pour l'auto-provisioning.

### Installation des Greffons (Plugins)
1.  Aller dans **Configuration > Gestionnaire de greffons**.
2.  Installer le greffon correspondant à votre matériel (ex: `xivo-grandstream` ou `xivo-cisco-spa`).

### Enregistrement d'un Terminal
1.  Récupérer l'**adresse MAC** du poste physique (souvent à l'arrière de l'appareil).
2.  Aller dans **Services > IPBX > Gestion de l'IPBX > Terminaux**.
3.  Ajouter un terminal en saisissant son adresse MAC et en choisissant le modèle correspondant.
4.  Associer ce terminal à la **Ligne** de l'utilisateur créé à l'étape précédente.

---

## 5. Configuration Réseau pour la VoIP (DHCP)
Pour que les téléphones IP trouvent automatiquement le serveur XiVO, le serveur DHCP doit envoyer des options spécifiques.

| Option | Valeur | Rôle |
| :--- | :--- | :--- |
| **Option 66** | `http://<IP_XiVO>:80/provd/cfg/` | URL de provisioning pour les postes |
| **Option 42** | `<IP_XiVO>` | Serveur de temps (NTP) |

### Cas des Softphones (Zoiper / Linphone)
Pour un softphone, l'auto-provisioning n'est pas utilisé. Il faut configurer manuellement le compte SIP :
* **Host / Domain** : Adresse IP du serveur XiVO.
* **User ID / Username** : Le numéro de la ligne (ex: 201).
* **Password** : Le mot de passe généré automatiquement par XiVO dans les détails de la ligne.

---

## 6. Vérification du Statut
* **Interface Web** : Le terminal doit apparaître en vert dans la liste des terminaux.
* **Ligne de commande** : Taper `asterisk -r` puis `pjsip show endpoints` pour voir les postes enregistrés en temps réel.

---
*Documentation basée sur les annexes techniques de la SODECAF et la documentation officielle XiVO.*
