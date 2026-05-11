# Déploiement d'une authentification 802.1X sécurisée via PacketFence

## 1. Introduction et Objectifs

Cette documentation détaille la mise en œuvre d'une solution de contrôle d'accès réseau (NAC) pour l'entreprise **SODECAF**. L'objectif principal est de sécuriser les accès filaires et Wi-Fi en utilisant le standard **IEEE 802.1X** et le serveur **PacketFence**.

## 2. Concepts de base

* 
**RADIUS (Remote Authentication Dial-In User Service)** : Protocole client-serveur centralisant les données AAA (Authentification, Autorisation et Accounting).


* 
**802.1X** : Standard définissant le contrôle d'accès basé sur les ports. Il repose sur trois composants :


* 
**Supplicant** : Le client (ordinateur, smartphone) souhaitant se connecter.


* 
**Authentificateur / NAS** : Équipement réseau (switch, borne Wi-Fi) relayant la demande.


* 
**Serveur d'authentification** : Serveur RADIUS (ici PacketFence) acceptant ou refusant l'accès.




* 
**Protocoles de sécurité** : Utilisation du cadre **EAP** encapsulé dans un tunnel **PEAP** (TLS) pour sécuriser l'échange des identifiants via **MS-CHAPv2**.



## 3. Architecture Technique

Contexte Réseau (Domaine : `sodecaf.local`) 

* 
**VLAN ADMIN (10)** : `192.168.10.0/24`.


* 
**VLAN DEV (20)** : `192.168.20.0/24`.


* 
**VLAN WIFI (50)** : `192.168.50.0/24`.


* 
**VLAN SERVEURS** : `172.16.0.0/24`.



### Équipements

* 
**SRV-WIN1** (Windows Server) : AD, DNS, DHCP (IP : `172.16.0.1`).


* 
**SRV-RADIUS** (PacketFence sur Debian 12) : IP `172.16.0.10`.


* 
**NAS** : Commutateur Cisco 2960 (IP : `192.168.10.200`) et Point d'accès D-Link DAP 1665 (IP : `192.168.50.200`).



## 4. Étapes de Mise en Œuvre

### 4.1. Configuration de l'Active Directory

1. 
**Unités d'Organisation (UO)** : Créer `Utilisateurs_SODECAF` avec les sous-UO `Admin`, `Dev`, `Visiteurs` et `Services`.


2. 
**Groupes et Utilisateurs** : Créer les groupes `Groupe_Admin`, `Groupe_Dev` et `Groupe_Visiteurs`, puis y affecter les utilisateurs respectifs (ex: `uadmin1`, `udev1`, `uguest1`).


3. 
**Compte de service** : Créer l'utilisateur `packetfence` dans l'UO `Services` pour la liaison LDAP.


4. 
**Autorisation réseau** : Dans l'onglet "Appel entrant" des propriétés utilisateurs, sélectionner "Autoriser l'accès".



### 4.2. Configuration de PacketFence

1. 
**Liaison LDAP** : Ajouter la source Active Directory avec le Bind DN du compte `packetfence` et tester la connexion.


2. 
**Jointure au domaine** : Indispensable pour l'authentification PEAP-MSCHAPv2 via NTLM. Redémarrer les services `ntlm-auth-pi` et `radiusd-auth` après jointure.


3. 
**Mapping des rôles** : Créer des règles d'accès basées sur l'attribut LDAP `memberOf`.


* 
`memberOf` égal à `CN=Groupe_Admin...` $\rightarrow$ rôle `role_admin`.


* 
`memberOf` égal à `CN=Groupe_Dev...` $\rightarrow$ rôle `role_dev`.





### 4.3. Configuration des Équipements Réseau (NAS)

#### Commutateur Cisco (Exemple partiel)

```bash
[cite_start]aaa new-model [cite: 6537]
radius server PACKETFENCE
 [cite_start]address ipv4 172.16.0.10 auth-port 1812 acct-port 1813 [cite: 6540]
 [cite_start]key Btssio2017 [cite: 6541]
[cite_start]dot1x system-auth-control [cite: 6550]

interface range FastEthernet0/1 - 4
 [cite_start]description 8021X_WIRED_VLAN10 [cite: 6551]
 [cite_start]authentication port-control auto [cite: 6555]
 [cite_start]dot1x pae authenticator [cite: 6556]

```

#### Point d'accès Wi-Fi

* Déclarer le SSID en mode **WPA2-Enterprise / 802.1X**.


* Pointer vers l'IP de PacketFence (`172.16.0.10`) avec la clé secrète `Btssio2017`.



### 4.4. Configuration du Poste Client (Windows)

1. Activer le service **dot3svc** (Configuration automatique de réseau câblé) via PowerShell : `net start dot3svc`.


2. Dans les propriétés de la carte Ethernet (onglet Authentification), activer l'authentification IEEE 802.1X et choisir **Microsoft: Protected EAP (PEAP)**.


3. Dans "Paramètres supplémentaires", forcer le mode en "Authentification utilisateur".



## 5. Validation et Audit

Les tests permettent de confirmer l'affectation dynamique des VLANs :

* Un utilisateur du `Groupe_Admin` doit obtenir une IP dans le VLAN 10.


* Un utilisateur du `Groupe_Dev` doit obtenir une IP dans le VLAN 20.


* Les logs RADIUS dans PacketFence (menu Audit) doivent afficher un statut **Accept** avec le `Tunnel-Private-Group-Id` correspondant au VLAN cible.
