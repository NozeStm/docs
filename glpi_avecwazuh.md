-----

# 📚 Documentation du TP : Gestion ITSM des Incidents de Sécurité (GLPI & Wazuh)

Ce document retrace les étapes de mise en place d'un système de gestion des tickets d'incidents de cybersécurité, intégrant l'outil ITSM **GLPI** avec la solution de détection **Wazuh** via son API, dans le contexte du SOC de l'entreprise SODECAF.

## 🎯 Objectifs du TP

  * Comprendre le rôle d'un outil ITSM.
  * Déployer et configurer l'outil open-source ITSM **GLPI**.
  * Activer et tester l'**API REST de GLPI**.
  * Automatiser la création de tickets incidents dans GLPI à partir des alertes générées par **Wazuh** (SIEM/IDS) via un script d'**Active Response**.
  * **Gérer les tickets incidents générés dans GLPI.**

-----

## 1\. Introduction au Contexte

### 1.1 Qu'est-ce qu'un outil ITSM ?

[cite_start]Un outil ITSM (**Information Technology Service Management**) est une solution logicielle qui aide les entreprises à gérer l'ensemble des services informatiques[cite: 8]. [cite_start]Cet outil permet de planifier, concevoir, déployer, exploiter et optimiser les services informatiques[cite: 9].

### 1.2 Rôle de l'ITSM en Cybersécurité

[cite_start]L'utilisation d'un outil ITSM en cybersécurité permet[cite: 12]:

  * [cite_start]**La réduction des délais de réponse**[cite: 13].
  * [cite_start]**L'amélioration de la coordination et communication**[cite: 14].
  * [cite_start]**D'apporter une traçabilité** pour les audits et les analyses post-incident[cite: 15, 25].
  * [cite_start]**La standardisation des processus** via des modèles prédéfinis[cite: 26].

### 1.3 Présentation de GLPI

[cite_start]**GLPI** (**Gestionnaire Libre de Parc Informatique**) est un logiciel open-source de gestion des services informatiques (ITSM) et des actifs[cite: 32]. [cite_start]Son module de gestion des tickets est au cœur de son fonctionnement [cite: 33][cite_start], permettant d'enregistrer, prioriser, assigner et suivre les incidents[cite: 34].

-----

## 2\. Infrastructure et Préparation

### 2.1 Configuration de l'Infrastructure

[cite_start]Le serveur **SRV-GLPI** a l'adresse IP `172.16.0.8` [cite: 89] [cite_start]et le serveur **SRV-WAZUH** a l'adresse IP `172.16.0.7` [cite: 75, 79] [cite_start]dans le réseau LAN SODECAF (`172.16.0.0/24`)[cite: 73]. [cite_start]L'accès à GLPI se fait via `http://172.16.0.8/`[cite: 89].

  * [cite_start]**Identifiant Administrateur GLPI :** `glpi` [cite: 91]
  * [cite_start]**Mot de passe GLPI :** `glpi` [cite: 92]

-----

## 3\. Configuration de l'API GLPI

[cite_start]Wazuh utilise l'API de GLPI [cite: 94] [cite_start]pour créer des tickets, nécessitant un jeton d'application (`app_token`) et un jeton utilisateur (`user_token`) pour l'authentification[cite: 96].

### 3.1 Jeton d'Application et Utilisateur

Les jetons utilisés pour l'automatisation sont les suivants :

| Jeton | Description | Valeur fournie |
| :--- | :--- | :--- |
| **`app_token`** | Jeton de l'application cliente (Wazuh API) | `b05J3XE12uyo6CF7AJ1pjDzE7kC0hGn9E2tZcehb` |
| **`user_token`** | Jeton de l'utilisateur `wazuh` | `LvCJPVEakByk87W6UsCxZtrpQBRE1LcgHnCXy1pJ` |
| **`session_token`** | Jeton temporaire de session (Exemple) | `j55dgu3jqth79662nsnim79sj5` |

-----

## 4\. Test de l'API (Depuis la console de Wazuh)

### 4.1 Test d'Authentification (`initSession`)

Test de connexion à l'API GLPI pour obtenir un jeton de session.

```bash
curl -X GET \
-H 'Content-Type: application/json' \
-H "Authorization: user_token LvCJPVEakByk87W6UsCxZtrpQBRE1LcgHnCXy1pJ" \
-H "App-Token: b05J3XE12uyo6CF7AJ1pjDzE7kC0hGn9E2tZcehb" \
'http://172.16.0.8/apirest.php/initSession?get_full_session=true'
```

### 4.2 Génération Manuelle d'un Ticket de Test

Création d'un ticket test via l'API, en utilisant le jeton de session.

```bash
curl -X POST \
-H 'Content-Type: application/json' \
-H 'Authorization: user_token LvCJPVEakByk87W6UsCxZtrpQBRE1LcgHnCXy1pJ' \
-H 'App-Token: b05J3XE12uyo6CF7AJ1pjDzE7kC0hGn9E2tZcehb' \
-H 'Session-Token: j55dgu3jqth79662nsnim79sj5' \
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

### 4.3 Fermeture de Session (`killSession`)

```bash
curl -X GET \
-H 'Authorization: user_token LvCJPVEakByk87W6UsCxZtrpQBRE1LcgHnCXy1pJ' \
-H 'App-Token: b05J3XE12uyo6CF7AJ1pjDzE7kC0hGn9E2tZcehb' \
-H 'Session-Token: j55dgu3jqth79662nsnim79sj5' \
http://172.16.0.8/apirest.php/killSession
```

-----

## 5\. Automatisation avec un Script Bash (Active Response)

### 5.1 Script `/var/ossec/active-response/bin/glpi_ticket.sh`

[cite_start]Le script Bash est utilisé pour automatiser la génération des tickets incidents depuis Wazuh[cite: 213].

[cite_start]**Extraits du script avec les jetons mis à jour[cite: 221]:**

```bash
GLPI_URL="http://172.16.0.8/apirest.php"
APP_TOKEN="b05J3XE12uyo6CF7AJ1pjDzE7kC0hGn9E2tZcehb"
USER_TOKEN="LvCJPVEakByk87W6UsCxZtrpQBRE1LcgHnCXy1pJ"
LOGFILE="/var/log/glpi_ticket.log"
# ...
```

[cite_start]Les étapes d'installation et de configuration des droits du script (propriétaire `root:wazuh`, droits `750`) et du fichier de log (`/var/log/glpi_ticket.log`) doivent être respectées[cite: 270, 271, 273].

### 5.2 Configuration de la Réponse Active Wazuh

[cite_start]Le fichier de configuration de Wazuh doit être modifié pour déclencher le script `glpi_ticket.sh` lors d'une alerte SSH (rule.id=`5763`)[cite: 291].

```xml
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
```

[cite_start]Le service `wazuh-manager` doit ensuite être redémarré[cite: 292].

### 5.3 Test Réel de la Création de Ticket

[cite_start]Effectuer une attaque sur le service ssh du serveur DVWA avec Kali Linux [cite: 304] [cite_start]et vérifier la création du ticket incident dans GLPI[cite: 305].

-----

## 7\. Gestion des Tickets Incidents 🛠️

[cite_start]La dernière phase consiste à traiter les tickets qui ont été générés automatiquement par Wazuh[cite: 318].

### 7.1 Connexion et Rôles

  * [cite_start]Connectez-vous à l'interface GLPI avec le compte `SOC-N1`[cite: 319].
  * [cite_start]Si nécessaire, vérifiez et ajustez les droits de l'utilisateur `SOC-N1` en vous connectant avec le compte administrateur `glpi`[cite: 319].

### 7.2 Manipulation des Incidents

[cite_start]À partir du tableau de bord de GLPI ou de la liste des tickets, l'analyste N1 du SOC effectue les actions suivantes pour gérer l'incident (par exemple, un ticket **Alerte Wazuh - Tentative de brute force SSH**)[cite: 282]:

1.  **Réponse / Prise en Charge** :

      * [cite_start]L'analyste prend connaissance de l'alerte (date, règle ID, agent, IP Source, gravité)[cite: 284, 285, 287, 288, 289].
      * Il documente les premières analyses et les actions immédiates prises (ex: vérifier le statut du service, isoler temporairement l'agent) en ajoutant des suivis au ticket.

2.  **Escalade** :

      * Si l'incident dépasse les compétences du niveau N1 (ex: nécessite une intervention physique ou une investigation forensique complexe), l'analyste peut **escalader** le ticket.
      * L'escalade consiste généralement à réaffecter le ticket à un groupe de techniciens de niveau supérieur (N2 ou N3) ou à un autre domaine (ex: équipe d'administration système).

3.  **Clôture** :

      * Une fois que l'incident est résolu (par exemple, la cause de la tentative de brute force est identifiée et corrigée, le pare-feu a été mis à jour), l'analyste documente la solution finale.
      * Le ticket est ensuite **clôturé**, marquant la fin du processus de gestion de l'incident.

[cite_start]L'objectif est de vérifier le bon fonctionnement du processus ITSM complet, de la détection (Wazuh) à la résolution et la clôture (GLPI)[cite: 319].

-----
