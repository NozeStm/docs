Voici la version complète et optimisée de votre documentation technique pour le projet **RADIUS / PacketFence**, conçue spécifiquement pour un rendu professionnel sur **GitHub** (Markdown GFM).

---

# 🔐 Sécurisation des accès réseau avec PacketFence (802.1X)

> [!IMPORTANT]
> Cette documentation détaille la mise en œuvre d'une solution de contrôle d'accès réseau (NAC) pour l'infrastructure **SODECAF**. L'objectif est de sécuriser les accès filaires et Wi-Fi en utilisant le standard **IEEE 802.1X** et le serveur **PacketFence**.
> 
> 

---

## 🏗️ Architecture Technique

La solution repose sur un dialogue tripartite entre le client, l'équipement réseau et le serveur d'authentification:

1. 
**Supplicant** : Le client (ordinateur, smartphone) qui souhaite se connecter.


2. 
**Authentificateur (NAS)** : Le commutateur Cisco ou la borne Wi-Fi qui relaye la demande.


3. 
**Serveur RADIUS** : PacketFence, qui accepte ou refuse la demande en interrogeant l'Active Directory.



### 📊 Plan d'adressage et VLANs

| Équipement | Rôle | Adresse IP | VLAN |
| --- | --- | --- | --- |
| **SRV-WIN1** | AD / DNS / DHCP | `172.16.0.1` | 100 (Serveurs) 

 |
| **SRV-RADIUS** | PacketFence (NAC) | `172.16.0.10` | 100 (Serveurs) 

 |
| **Switch-Cisco** | Authentificateur Filaire | `192.168.10.200` | 10 (Administration) 

 |
| **Borne-DLink** | Authentificateur Wi-Fi | `192.168.50.200` | 50 (Wi-Fi) 

 |

---

## 🛠️ Étapes de Configuration

### 1️⃣ Préparation de l'Active Directory

L'annuaire sert de source d'identité unique pour tout le réseau.

* 
**Unités d'Organisation (UO)** : Créer `Utilisateurs_SODECAF` avec les sous-UO `Admin`, `Dev`, `Visiteurs` et `Services`.


* 
**Groupes de sécurité** : Créer `Groupe_Admin`, `Groupe_Dev` et `Groupe_Visiteurs`.


* 
**Compte de service** : Créer l'utilisateur `packetfence` pour la liaison LDAP.


* 
**Permission critique** : Dans les propriétés de chaque compte (onglet **Appel entrant**), cocher obligatoirement **"Autoriser l'accès"**.



### 2️⃣ Configuration du Commutateur Cisco 2960

Le commutateur doit être configuré pour bloquer les ports tant que l'utilisateur n'est pas identifié.

```ios
! Activation globale de l'authentification
aaa new-model
dot1x system-auth-control

! Déclaration du serveur PacketFence
radius server PACKETFENCE
 address ipv4 172.16.0.10 auth-port 1812 acct-port 1813
 key Btssio2017

! Configuration des ports clients (VLAN 10 par exemple)
interface range FastEthernet0/1 - 4
 description 8021X_WIRED_VLAN10
 authentication port-control auto
 dot1x pae authenticator
 dot1x timeout tx-period 10

```

> [!NOTE]
> Le protocole **EAP** (Extensible Authentication Protocol) est le seul autorisé à traverser le port avant l'authentification complète.
> 
> 

### 3️⃣ Paramétrage de PacketFence (NAC)

* 
**Source d'authentification** : Ajouter une source de type "Active Directory" pointant vers `172.16.0.1` avec le compte de service `packetfence`.


* 
**Jointure au domaine** : Indispensable pour supporter la méthode **PEAP-MSCHAPv2** (NTLM).


* 
**Segmentation Dynamique (VLAN Steering)** : Créer des règles basées sur l'attribut `memberOf` pour affecter le bon VLAN selon le groupe AD:


* Membre de `Groupe_Admin` ➡️ Rôle `role_admin` ➡️ **VLAN 10**.


* Membre de `Groupe_Dev` ➡️ Rôle `role_dev` ➡️ **VLAN 20**.


* Membre de `Groupe_Visiteurs` ➡️ Rôle `role_guest` ➡️ **VLAN 50**.





---

## 💻 Configuration du Poste Client (Supplicant)

Pour que Windows initie le dialogue 802.1X sur une prise filaire, une configuration manuelle est nécessaire.

### Activation du service

```powershell
# Démarrer le service de configuration automatique de réseau câblé
net start dot3svc

```

### Paramétrage PEAP

1. Propriétés de la carte Ethernet > Onglet **Authentification**.


2. Cocher **"Activer l'authentification IEEE 802.1X"**.


3. Méthode : **Microsoft: Protected EAP (PEAP)**.


4. Dans **Paramètres supplémentaires**, forcer le mode en **"Authentification utilisateur"**.



---

## 🔍 Validation et Audit

La validation s'effectue en consultant le **Journal d'audit RADIUS** dans PacketFence.

* 
**Statut Accept (Vert)** : L'utilisateur est authentifié.


* 
**Attributs retournés** : Vérifier que la réponse RADIUS contient bien le `Tunnel-Private-Group-Id` correct (ex: `10` pour un administrateur).


* 
**Test d'échec** : Un mot de passe erroné doit générer un statut **Reject** avec une erreur `MS-CHAP-Error`.



> [!TIP]
> En cas de problème, utilisez la commande `show authentication sessions` sur le switch Cisco pour voir l'état en temps réel de chaque port.
> 
> 

---

*Documentation réalisée par Nolann MERLE - BTS SIO SISR - Session 2026.*
