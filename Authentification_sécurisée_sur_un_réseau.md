Voici une documentation structurée en Markdown basée sur le TP de la SODECAF pour ton dépôt GitHub.

---

# TP : Authentification Sécurisée 802.1X avec PacketFence (NAC)

## 1. Présentation du Projet
L'objectif de cette activité est de déployer une solution de **Network Access Control (NAC)** pour l'entreprise **SODECAF** afin de contrôler l'accès aux réseaux filaires et WiFi. La solution retenue est **PacketFence**, un outil open source centralisant les données AAA (Authentification, Autorisation, Accounting) via le protocole RADIUS.

### Architecture Technique
* **Serveur RADIUS / NAC :** PacketFence (sur Debian 12).
* **Annuaire :** Active Directory (Windows Server) pour la base d'utilisateurs.
* **Équipements réseau :** Commutateur Cisco 2960 et Point d'accès WiFi D-Link.
* **Méthode d'authentification :** PEAP-MSCHAPv2 (Tunnel TLS protégeant les identifiants AD).


---

## 2. Infrastructure Réseau (VLANs & IP)

| VLAN | Nom | Réseau | Passerelle | Rôle |
| :--- | :--- | :--- | :--- | :--- |
| **10** | ADMIN | `192.168.10.0/24` | `.254` | [cite_start]Postes administratifs  |
| **20** | DEV | `192.168.20.0/24` | `.254` | [cite_start]Postes développeurs  |
| **50** | WIFI | `192.168.50.0/24` | `.254` | [cite_start]Accès sans-fil / Invités  |
| **-** | SERVEURS | `172.16.0.0/24` | `.254` | [cite_start]Infrastructure (AD, RADIUS)  |

---

## 3. Configuration de l'Active Directory
[cite_start]Pour permettre le mapping dynamique des VLANs, les objets suivants doivent être créés:

* **Unités d'Organisation :** `Admin`, `Dev`, `Visiteurs`, `Services`.
* **Groupes de sécurité :** `Groupe_Admin`, `Groupe_Dev`, `Groupe_Visiteurs`.
* **Compte de service :** `packetfence` (dans l'UO Services) pour la liaison LDAP.
* **Paramètre crucial :** Dans les propriétés des comptes (onglet **Appel entrant / Dial-in**), sélectionner **"Autoriser l'accès"**.

---

## 4. Déploiement de PacketFence

### Installation
Le serveur est déployé sur une VM Debian 12 avec au moins 4 Go de RAM et une interface en mode bridge.
* **IP Fixe :** `172.16.0.10`.
* **DNS :** Doit pointer vers le contrôleur de domaine (`172.16.0.1`).

### Liaison AD et Jointure de Domaine
1.  **Source d'authentification :** Configurer une source de type "Active Directory" en renseignant le Base DN (`DC=sodecaf, DC=local`) et le compte de service `packetfence`.
2.  **Jointure au domaine :** Indispensable pour l'authentification **PEAP-MSCHAPv2** via NTLM/Samba.
    * Menu : `Configuration > Politiques et contrôle d'accès > Domaines AD`.

---

## 5. Politiques de Contrôle d'Accès
Le NAC attribue un rôle PacketFence en fonction du groupe AD de l'utilisateur, ce rôle détermine ensuite le VLAN injecté sur le port.

### Mapping Rôles & VLANs
* **Groupe AD `Admin`** $\rightarrow$ Rôle `role_admin` $\rightarrow$ **VLAN 10**.
* **Groupe AD `Dev`** $\rightarrow$ Rôle `role_dev` $\rightarrow$ **VLAN 20**.
* **Groupe AD `Visiteurs`** $\rightarrow$ Rôle `role_invite` $\rightarrow$ **VLAN 50**.

---

## 6. Configuration des Équipements (Clients RADIUS)

### Point d'Accès WiFi (D-Link)
* **SSID :** `SISR-x`.
* **Sécurité :** WPA2-Enterprise / 802.1X.
* **Serveur RADIUS :** `172.16.0.10`, Port `1812`, Secret `Btssio2017`.

### Commutateur Cisco 2960 (Filaire)
Configuration globale pour activer le 802.1X:
```bash
aaa new-model
radius server PACKETFENCE
 address ipv4 172.16.0.10 auth-port 1812 acct-port 1813
 key Btssio2017
dot1x system-auth-control
```

Configuration d'une interface (ex: ports 1 à 4):
```bash
interface range FastEthernet0/1 - 4
 switchport mode access
 authentication port-control auto
 dot1x pae authenticator
```

---

## 7. Tests et Validation
* **Côté Client :** Sur Windows, le service `dot3svc` (Configuration automatique câblée) doit être démarré pour activer l'onglet "Authentification" dans les propriétés de la carte réseau.
* **Côté Serveur :** Vérifier les accès dans le **Journal d'audit RADIUS** de PacketFence.
    * Un succès affiche le statut **"Accept"** avec le VLAN ID correspondant (ex: `Tunnel-Private-Group-Id = "10"`).
    * Un échec (mauvais mot de passe) affiche **"Reject"** avec une erreur MS-CHAP.

---
*Documentation générée pour le contexte SODECAF - 2026*.
