Voici une documentation structurée en Markdown basée sur le TP de la SODECAF pour ton dépôt GitHub.

---

# TP : Authentification Sécurisée 802.1X avec PacketFence (NAC)

## 1. Présentation du Projet
[cite_start]L'objectif de cette activité est de déployer une solution de **Network Access Control (NAC)** pour l'entreprise **SODECAF** afin de contrôler l'accès aux réseaux filaires et WiFi[cite: 3]. [cite_start]La solution retenue est **PacketFence**, un outil open source centralisant les données AAA (Authentification, Autorisation, Accounting) via le protocole RADIUS[cite: 5, 24].

### Architecture Technique
* [cite_start]**Serveur RADIUS / NAC :** PacketFence (sur Debian 12)[cite: 141, 221].
* [cite_start]**Annuaire :** Active Directory (Windows Server) pour la base d'utilisateurs[cite: 145].
* [cite_start]**Équipements réseau :** Commutateur Cisco 2960 et Point d'accès WiFi D-Link[cite: 153, 162].
* [cite_start]**Méthode d'authentification :** PEAP-MSCHAPv2 (Tunnel TLS protégeant les identifiants AD)[cite: 46, 55].


---

## 2. Infrastructure Réseau (VLANs & IP)

| VLAN | Nom | Réseau | Passerelle | Rôle |
| :--- | :--- | :--- | :--- | :--- |
| **10** | ADMIN | `192.168.10.0/24` | `.254` | [cite_start]Postes administratifs [cite: 161, 184] |
| **20** | DEV | `192.168.20.0/24` | `.254` | [cite_start]Postes développeurs [cite: 167, 184] |
| **50** | WIFI | `192.168.50.0/24` | `.254` | [cite_start]Accès sans-fil / Invités [cite: 149, 184] |
| **-** | SERVEURS | `172.16.0.0/24` | `.254` | [cite_start]Infrastructure (AD, RADIUS) [cite: 165] |

---

## 3. Configuration de l'Active Directory
[cite_start]Pour permettre le mapping dynamique des VLANs, les objets suivants doivent être créés[cite: 187, 192, 196]:

* **Unités d'Organisation :** `Admin`, `Dev`, `Visiteurs`, `Services`.
* **Groupes de sécurité :** `Groupe_Admin`, `Groupe_Dev`, `Groupe_Visiteurs`.
* [cite_start]**Compte de service :** `packetfence` (dans l'UO Services) pour la liaison LDAP[cite: 200].
* [cite_start]**Paramètre crucial :** Dans les propriétés des comptes (onglet **Appel entrant / Dial-in**), sélectionner **"Autoriser l'accès"**[cite: 207, 212].

---

## 4. Déploiement de PacketFence

### Installation
[cite_start]Le serveur est déployé sur une VM Debian 12 avec au moins 4 Go de RAM et une interface en mode bridge[cite: 221, 222, 225].
* [cite_start]**IP Fixe :** `172.16.0.10`[cite: 227].
* [cite_start]**DNS :** Doit pointer vers le contrôleur de domaine (`172.16.0.1`)[cite: 232].

### Liaison AD et Jointure de Domaine
1.  [cite_start]**Source d'authentification :** Configurer une source de type "Active Directory" en renseignant le Base DN (`DC=sodecaf, DC=local`) et le compte de service `packetfence`[cite: 253, 257, 260].
2.  [cite_start]**Jointure au domaine :** Indispensable pour l'authentification **PEAP-MSCHAPv2** via NTLM/Samba[cite: 268, 269].
    * [cite_start]Menu : `Configuration > Politiques et contrôle d'accès > Domaines AD`[cite: 278, 279].

---

## 5. Politiques de Contrôle d'Accès
[cite_start]Le NAC attribue un rôle PacketFence en fonction du groupe AD de l'utilisateur, ce rôle détermine ensuite le VLAN injecté sur le port[cite: 304, 312].

### Mapping Rôles & VLANs
* [cite_start]**Groupe AD `Admin`** $\rightarrow$ Rôle `role_admin` $\rightarrow$ **VLAN 10**[cite: 316, 468].
* [cite_start]**Groupe AD `Dev`** $\rightarrow$ Rôle `role_dev` $\rightarrow$ **VLAN 20**[cite: 343, 468].
* [cite_start]**Groupe AD `Visiteurs`** $\rightarrow$ Rôle `role_invite` $\rightarrow$ **VLAN 50**[cite: 349, 468].

---

## 6. Configuration des Équipements (Clients RADIUS)

### Point d'Accès WiFi (D-Link)
* [cite_start]**SSID :** `SISR-x`[cite: 395].
* [cite_start]**Sécurité :** WPA2-Enterprise / 802.1X[cite: 396].
* [cite_start]**Serveur RADIUS :** `172.16.0.10`, Port `1812`, Secret `Btssio2017`[cite: 399, 400].

### Commutateur Cisco 2960 (Filaire)
[cite_start]Configuration globale pour activer le 802.1X[cite: 478, 481, 491]:
```bash
aaa new-model
radius server PACKETFENCE
 address ipv4 172.16.0.10 auth-port 1812 acct-port 1813
 key Btssio2017
dot1x system-auth-control
```

[cite_start]Configuration d'une interface (ex: ports 1 à 4)[cite: 492, 496, 497]:
```bash
interface range FastEthernet0/1 - 4
 switchport mode access
 authentication port-control auto
 dot1x pae authenticator
```

---

## 7. Tests et Validation
* [cite_start]**Côté Client :** Sur Windows, le service `dot3svc` (Configuration automatique câblée) doit être démarré pour activer l'onglet "Authentification" dans les propriétés de la carte réseau[cite: 528, 530].
* [cite_start]**Côté Serveur :** Vérifier les accès dans le **Journal d'audit RADIUS** de PacketFence[cite: 413, 416].
    * [cite_start]Un succès affiche le statut **"Accept"** avec le VLAN ID correspondant (ex: `Tunnel-Private-Group-Id = "10"`)[cite: 454, 582, 591].
    * [cite_start]Un échec (mauvais mot de passe) affiche **"Reject"** avec une erreur MS-CHAP[cite: 597, 605].

---
[cite_start]*Documentation générée pour le contexte SODECAF - 2026*[cite: 1, 147].
