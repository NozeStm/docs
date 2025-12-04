-----

# 📚 Documentation du TP : Gestion ITSM des Incidents de Sécurité (GLPI & Wazuh)

Ce document retrace les étapes de mise en place d'un système de gestion des tickets d'incidents de cybersécurité, intégrant l'outil ITSM **GLPI** avec la solution de détection **Wazuh** via son API, dans le contexte du SOC de l'entreprise SODECAF.

## 🎯 Objectifs du TP

  * Comprendre le rôle d'un outil ITSM dans la gestion des services informatiques et des incidents de cybersécurité.
  * Déployer et configurer l'outil open-source ITSM **GLPI**.
  * Activer et tester l'**API REST de GLPI**.
  * Automatiser la création de tickets incidents dans GLPI à partir des alertes générées par **Wazuh** (SIEM/IDS) via un script d'**Active Response**.
  * Simuler et vérifier le processus de création automatique de tickets lors d'une attaque.

## 1\. Introduction au Contexte

### 1.1 Qu'est-ce qu'un outil ITSM ?

[cite_start]Un outil ITSM (**Information Technology Service Management**) est une solution logicielle conçue pour aider les entreprises à gérer l'ensemble de leurs services informatiques (planification, conception, déploiement, exploitation, optimisation)[cite: 8, 9].

Il facilite notamment :

  * [cite_start]La gestion du parc informatique (serveurs, postes clients, licences, etc.)[cite: 10].
  * [cite_start]La gestion des incidents (système de ticketing)[cite: 10].
  * [cite_start]Le service de base documentaire pour les utilisateurs[cite: 11].

### 1.2 Rôle de l'ITSM en Cybersécurité

[cite_start]L'utilisation d'un outil ITSM en cybersécurité offre plusieurs avantages[cite: 12]:

  * [cite_start]**Réduction des délais de réponse** : Automatisation de la détection, priorisation et escalade des incidents[cite: 13].
  * [cite_start]**Amélioration de la coordination** : Centralisation des informations et collaboration facilitée entre les équipes (ex: admin-sys et sécurité)[cite: 14].
  * [cite_start]**Traçabilité** : Suivi détaillé de chaque étape pour les audits de conformité et les analyses post-incident[cite: 15, 25].
  * [cite_start]**Standardisation des processus** : Cohérence dans la gestion des incidents grâce à des modèles prédéfinis[cite: 26].
  * [cite_start]**Automatisation des actions correctives** : Possibilité de déclencher des réponses automatiques[cite: 27].

### 1.3 Présentation de GLPI

[cite_start]**GLPI** (**Gestionnaire Libre de Parc Informatique**) est un logiciel open-source qui combine la gestion des services informatiques (ITSM) et la gestion des actifs[cite: 32].

[cite_start]Son module de gestion des tickets est central pour enregistrer, prioriser, assigner et suivre les incidents et demandes[cite: 33, 34].

## 2\. Infrastructure et Préparation

### 2.1 Schéma de l'Infrastructure

L'infrastructure utilisée pour le TP est celle de **SODECAF** (domaine : `sodecaf.local`). [cite_start]Le nouveau serveur **GLPI** (`SRV-GLPI`) est déployé sur le réseau **LAN** (`172.16.0.0/24`)[cite: 62, 73, 77, 78, 86].

| Machine | Rôle | Adresse IP | Réseau |
| :--- | :--- | :--- | :--- |
| **SRV-GLPI** | Serveur ITSM (Debian) | `172.16.0.8` | LAN SODECAF |
| **SRV-WAZUH** | Serveur SIEM/IDS (Ubuntu) | `172.16.0.7` | LAN SODECAF |
| **SRV-WIN1** | Windows Server (AD, DNS, DHCP) | `172.16.0.1` | LAN SODECAF |
| **SRV-WEB (DVWA)** | Serveur web vulnérable | `192.168.0.x` | DMZ SODECAF |

### 2.2 Connexion à GLPI

[cite_start]L'accès à l'interface web de GLPI se fait via l'adresse `http://172.16.0.8/`[cite: 89].

  * [cite_start]**Identifiant Administrateur :** `glpi` [cite: 91]
  * [cite_start]**Mot de passe :** `glpi` [cite: 92]

## 3\. Configuration de l'API GLPI pour l'Interconnexion avec Wazuh

[cite_start]L'objectif est d'utiliser l'**API REST de GLPI** pour permettre à Wazuh de créer des tickets incidents de manière automatisée[cite: 93, 94]. [cite_start]L'API utilise des requêtes HTTP et un système d'authentification par jetons (`app_token` et `user_token`)[cite: 95, 96].

### 3.1 Activation de l'API

1.  [cite_start]Dans l'interface web de GLPI, naviguer vers **Configuration \> Générale \> API**[cite: 102].
2.  [cite_start]Activer l'**API Rest**[cite: 108, 109].
3.  [cite_start]Activer la connexion avec un **jeton externe**[cite: 114, 115].

### 3.2 Création du Client API (`app_token`)

1.  [cite_start]Dans la partie **Client de l'API**, ajouter un nouveau client nommé `wazuh_api`[cite: 117, 122].
2.  [cite_start]Dans la section **FILTRER L'ACCÈS**, spécifier l'adresse IPv4 du serveur Wazuh (`192.168.10.5` dans l'exemple du document, à adapter si nécessaire) dans les champs *Début de plage d'adresse IPv4* et *Fin de plage d'adr*[cite: 117, 130, 131, 134, 135].
3.  [cite_start]**Régénérer le Jeton d'application (`app_token`)** et le noter[cite: 117, 136].

### 3.3 Création de l'Utilisateur API (`user_token`)

1.  [cite_start]Aller dans **Administration \> Utilisateurs \> Ajouter un utilisateur**[cite: 141].
2.  Créer l'utilisateur :
      * [cite_start]**Identifiant :** `wazuh` [cite: 149]
      * [cite_start]**Mot de passe :** Configurer un mot de passe[cite: 148, 150].
      * [cite_start]**Fuseau horaire :** Configurer `Europe/Paris` (ou approprié)[cite: 142, 156].
3.  [cite_start]Après avoir créé et sélectionné l'utilisateur `wazuh`, aller dans la partie **Clefs d'accès distant**[cite: 143, 144].
4.  [cite_start]**Régénérer le Jeton d'API (`user_token`)** et le noter[cite: 143, 151, 152].

## 4\. Test de l'API (Depuis la console de Wazuh)

### 4.1 Test d'Authentification

[cite_start]Une session est ouverte pour valider l'accès à l'API[cite: 164, 165].

```bash
curl -X GET \
-H 'Content-Type: application/json' \
-H "Authorization: user_token VOTRE_USER_TOKEN" \
-H "App-Token: VOTRE_APP_TOKEN" \
'http://172.16.0.8/apirest.php/initSession?get_full_session=true'
```

[cite_start]Une réponse incluant un `"session_token"` confirme le succès[cite: 171, 174].

### 4.2 Génération Manuelle d'un Ticket de Test

[cite_start]Utilisation de la commande `curl` en méthode `POST` pour créer un nouveau ticket (`/Ticket`)[cite: 182, 183, 184].

```bash
curl -X POST \
-H 'Content-Type: application/json' \
-H 'Authorization: user_token VOTRE_USER_TOKEN' \
-H 'App-Token: VOTRE_APP_TOKEN' \
-H 'Session-Token: VOTRE_SESSION_TOKEN' \
-d'{
"input": {
"name": "Alerte Wazuh - Test API",
"content": "Ceci est un ticket créé automatiquement depuis Wazuh via API",
"priority": 4,
"urgency": 3,
"impact": 3,
"status": 1,
"type": 1,
"itilcategories_id": 2
}
}'\
http://172.16.0.8/apirest.php/Ticket
```

[cite_start]Le ticket doit être vérifié dans GLPI[cite: 202].

### 4.3 Fermeture de Session

[cite_start]Pour fermer la session API ouverte précédemment[cite: 206].

```bash
curl -X GET \
-H 'Authorization: user_token VOTRE_USER_TOKEN' \
-H 'App-Token: VOTRE_APP_TOKEN' \
-H 'Session-Token: VOTRE_SESSION_TOKEN' \
http://172.16.0.8/apirest.php/killSession
```

[cite_start]GLPI doit répondre `killSession true`[cite: 206].

## 5\. Automatisation avec un Script Bash (Active Response)

Pour automatiser la création de tickets lors d'une alerte Wazuh, un script d'Active Response est utilisé.

### 5.1 Préparation du Script

1.  [cite_start]Créer le fichier de log : `/var/log/glpi_ticket.log`[cite: 216, 270].
2.  [cite_start]Copier le script fourni dans le fichier `/var/ossec/active-response/bin/glpi_ticket.sh` sur le serveur Wazuh[cite: 271].
3.  [cite_start]**Adapter les jetons** : Remplacer `APP_TOKEN` et `USER_TOKEN` dans le script par les valeurs obtenues à l'étape 3[cite: 272].
4.  **Permissions** :
      * [cite_start]Modifier le propriétaire : `root:wazuh`[cite: 273].
      * [cite_start]Appliquer les droits : `750`[cite: 273].

[cite_start]Le script effectue les actions suivantes[cite: 215]:

  * Récupère les informations de l'alerte Wazuh (ID, description, agent, IP source, gravité).
  * [cite_start]Mappe la gravité Wazuh sur la priorité GLPI (ex: gravité $\le 3 \rightarrow$ priorité 1 (Faible), $\le 7 \rightarrow$ priorité 3 (Moyen), sinon $\rightarrow$ priorité 5 (Critique))[cite: 230, 235].
  * Ouvre une session API GLPI (`initSession`).
  * Crée un ticket avec les informations extraites.
  * Ferme la session API (`killSession`).
  * [cite_start]Journalise l'envoi du ticket[cite: 216].

### 5.2 Test Manuel du Script

[cite_start]Exécution manuelle du script avec un JSON d'alerte simulée[cite: 274, 275].

```bash
echo '{"parameters":{"alert":{"rule":{"id":"5710","description":"Tentative brute force SSH", "level":10},"agent":{"id":"001","name":"srv-web"},"data":{"srcip":"172.16.0.50"},"full_log":"sshd failed login"}}}' | /var/ossec/active-response/bin/glpi_ticket.sh
```

[cite_start]La création d'un ticket dans GLPI doit être vérifiée[cite: 274].

### 5.3 Configuration de la Réponse Active Wazuh

[cite_start]Ajouter la configuration suivante dans le fichier de configuration de Wazuh pour déclencher le script lors d'une attaque SSH (Règle ID `5763`)[cite: 291, 298, 301]:

```xml
<ossec_config>
  <command>
    <name>glpi_ticket</name>
    <executable>glpi_ticket.sh</executable>
    <expect>none</expect>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <active-response>
    <command>glpi_ticket</command>
    <location>server</location>
    <rules_id>5763</rules_id>
    <timeout>60</timeout>
  </active-response>
</ossec_config>
```

[cite_start]Redémarrer le service `wazuh-manager`[cite: 292].

### 5.4 Test Réel de la Création de Ticket

[cite_start]Effectuer une attaque sur le service SSH du serveur cible (DVWA) à l'aide de Kali Linux[cite: 304].

[cite_start]Wazuh doit détecter l'événement (Règle 5763) et déclencher automatiquement le script, entraînant la création d'un ticket incident dans GLPI[cite: 305].

## 6\. Gestion des Tickets Incidents

[cite_start]Connectez-vous à GLPI avec le compte `SOC-N1` (ou administrateur si droits à revoir) pour tester les fonctions ITSM de gestion des tickets : **réponse, escalade, et clôture** des incidents générés automatiquement[cite: 317, 318, 319].

-----
