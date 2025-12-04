
-----

# Documentation TP2 : Gestion ITSM des incidents de sécurité (SODECAF)

## 1\. Contexte et Objectifs

Dans le cadre de l'entreprise **SODECAF**, vous agissez en tant qu'analyste N1 au sein du SOC (Security Operations Center). [cite_start]L'objectif est d'améliorer la gestion des incidents de cybersécurité en interconnectant le SIEM **Wazuh** avec l'outil ITSM **GLPI**[cite: 3, 4].

**Objectifs techniques :**

  * Configurer l'API de GLPI.
  * Automatiser la création de tickets incidents dans GLPI à partir des alertes remontées par Wazuh.
  * [cite_start]Assurer la traçabilité et la standardisation des incidents[cite: 15, 26].

## 2\. Infrastructure

[cite_start]L'infrastructure mise en place comprend les éléments suivants [cite: 62-85] :

  * **LAN SIO (172.17.0.0/16)**
  * **DMZ (192.168.0.0/24)** : Contient le serveur web vulnérable (DVWA).
  * **LAN SODECAF (172.16.0.0/24)** :
      * `172.16.0.1` : SRV-WIN1 (AD, DNS, DHCP).
      * `172.16.0.7` : SRV-WAZUH (Ubuntu).
      * `172.16.0.8` : SRV-GLPI (Debian - Nouveau serveur).

-----

## 3\. Configuration de l'API GLPI

Avant toute interaction, l'API de GLPI doit être activée et configurée pour accepter les connexions externes.

### 3.1. Activation de l'API

1.  [cite_start]Connectez-vous à l'interface GLPI (`http://172.16.0.8/`) avec les identifiants par défaut (`glpi`/`glpi`) [cite: 89-92].
2.  Allez dans **Configuration \> Générale \> API**.
3.  [cite_start]Passez les paramètres suivants à **Oui**[cite: 102, 108, 114]:
      * Activer l'API Rest.
      * Activer la connexion avec un jeton externe.

### 3.2. Création du client API

1.  Ajoutez un client API nommé `wazuh_api`.
2.  Indiquez l'adresse IP du serveur Wazuh (ex: `192.168.10.5` ou l'IP LAN selon votre configuration réseau réelle) dans la plage IPv4.
3.  [cite_start]Générez et notez le **Jeton d'application (App-Token)**[cite: 117].

> **Note :** Le jeton d'application utilisé pour cette documentation est : `b05J3XE12uyo6CF7AJ1pjDzE7kC0hGn9E2tZcehb`.

### 3.3. Création de l'utilisateur Wazuh

1.  Allez dans **Administration \> Utilisateurs \> Ajouter un utilisateur**.
2.  Créez l'utilisateur `wazuh` (il sera l'auteur des tickets).
3.  [cite_start]Dans l'onglet **Clefs d'accès distant** de cet utilisateur, générez le **Jeton d'API (User-Token)**[cite: 140, 143].

> **Note :** Le jeton utilisateur utilisé pour cette documentation est : `LvCJPVEakByk87W6UsCxZtrpQBRE1LcgHnCXy1pJ`.

-----

## 4\. Tests manuels de l'API (depuis le serveur Wazuh)

Nous utilisons la commande `curl` depuis le serveur Wazuh pour valider la communication.

### 4.1. Test d'authentification (initSession)

Cette commande permet de récupérer un *Session-Token*.

```bash
curl -X GET \
-H 'Content-Type: application/json' \
-H "Authorization: user_token LvCJPVEakByk87W6UsCxZtrpQBRE1LcgHnCXy1pJ" \
-H "App-Token: b05J3XE12uyo6CF7AJ1pjDzE7kC0hGn9E2tZcehb" \
'http://172.16.0.8/apirest.php/initSession?get_full_session=true'
```

[cite_start]*Si la configuration est correcte, GLPI retourne un JSON contenant le `session_token`.* [cite: 166-174].

### 4.2. Génération d'un ticket de test

Utilisation du *Session-Token* fourni (`j55dgu3jqth79662nsnim79sj5`) pour créer un ticket.

```bash
curl -X POST \
-H 'Content-Type: application/json' \
-H 'Authorization: user_token LvCJPVEakByk87W6UsCxZtrpQBRE1LcgHnCXy1pJ' \
-H 'App-Token: b05J3XE12uyo6CF7AJ1pjDzE7kC0hGn9E2tZcehb' \
-H 'Session-Token: j55dgu3jqth79662nsnim79sj5' \
-d '{
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
}' \
http://172.16.0.8/apirest.php/Ticket
```

[cite_start][cite: 184-201].

### 4.3. Fermeture de session

```bash
curl -X GET \
-H 'Authorization: user_token LvCJPVEakByk87W6UsCxZtrpQBRE1LcgHnCXy1pJ' \
-H 'App-Token: b05J3XE12uyo6CF7AJ1pjDzE7kC0hGn9E2tZcehb' \
-H 'Session-Token: j55dgu3jqth79662nsnim79sj5' \
http://172.16.0.8/apirest.php/killSession
```

[cite_start][cite: 207-211].

-----

## 5\. Automatisation : Script Bash et Configuration Wazuh

### 5.1. Le script `glpi_ticket.sh`

Ce script récupère les données de l'alerte Wazuh (JSON), se connecte à l'API GLPI, détermine la priorité et crée le ticket.

**Emplacement du fichier :** `/var/ossec/active-response/bin/glpi_ticket.sh`
[cite_start]**Droits :** 750 (Propriétaire `root:wazuh`)[cite: 271, 273].

Voici le script complet avec vos tokens intégrés :

```bash
#!/bin/bash
# Script Active Response Wazuh -> GLPI
# Version améliorée avec logs et mapping gravité -> priorité

# CONFIGURATION AVEC VOS TOKENS
GLPI_URL="http://172.16.0.8/apirest.php"
APP_TOKEN="b05J3XE12uyo6CF7AJ1pjDzE7kC0hGn9E2tZcehb"
USER_TOKEN="LvCJPVEakByk87W6UsCxZtrpQBRE1LcgHnCXy1pJ"
LOGFILE="/var/log/glpi_ticket.log"

# Lire tout le JSON d'entrée (multi-lignes possible)
INPUT_JSON=$(cat)
ALERT_DATE=$(date '+%Y-%m-%d %H:%M:%S %z')

echo "$(date) - Script exécuté par Wazuh" >> $LOGFILE
echo "$INPUT_JSON" >> $LOGFILE

# Extraire infos utiles avec jq
RULE_ID=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.rule.id')
RULE_DESC=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.rule.description')
AGENT_NAME=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.agent.name')
SRC_IP=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.data.srcip // empty')
SEVERITY=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.rule.level')
LOG=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.full_log')

# Mapping gravité Wazuh (level) -> priorité GLPI
if [ "$SEVERITY" -le 3 ]; then
    PRIORITY=1 # Faible
elif [ "$SEVERITY" -le 7 ]; then
    PRIORITY=3 # Moyen
else
    PRIORITY=5 # Critique
fi

# Ouvrir une session GLPI
SESSION_TOKEN=$(curl -s -X GET \
    -H "Authorization: user_token $USER_TOKEN" \
    -H "App-Token: $APP_TOKEN" \
    "$GLPI_URL/initSession" | jq -r '.session_token')

# Créer le ticket
RESPONSE=$(curl -s -X POST \
    -H "Content-Type: application/json" \
    -H "App-Token: $APP_TOKEN" \
    -H "Session-Token: $SESSION_TOKEN" \
    -d "{
    \"input\": {
    \"name\": \"Alerte Wazuh - $RULE_DESC\",
    \"content\": \" Alerte Wazuh\n\n- Date alerte: $ALERT_DATE\n- Règle ID: $RULE_ID\n- Description: $RULE_DESC\n- Agent: $AGENT_NAME\n- Source IP: $SRC_IP\n- Gravité: $SEVERITY\n- Log: $LOG\",
    \"priority\": $PRIORITY,
    \"urgency\": 3,
    \"impact\": 3,
    \"status\": 1,
    \"type\": 1,
    \"itilcategories_id\": 2
    }
    }" \
    "$GLPI_URL/Ticket")

# Fermer la session
curl -s -X GET \
    -H "App-Token: $APP_TOKEN" \
    -H "Session-Token: $SESSION_TOKEN" \
    "$GLPI_URL/killSession" > /dev/null

# Log local pour suivi
echo "$(date '+%Y-%m-%d %H:%M:%S') - Ticket créé: Règle=$RULE_ID, Desc=$RULE_DESC, IP=$SRC_IP, Gravité=$SEVERITY, Priorité=$PRIORITY" >> "$LOGFILE"
```

[cite_start][cite: 217-269].

### 5.2. Préparation du système

1.  **Créer le fichier de log :**

    ```bash
    touch /var/log/glpi_ticket.log
    chown root:wazuh /var/log/glpi_ticket.log
    chmod 660 /var/log/glpi_ticket.log
    ```

    [cite_start][cite: 270].

2.  **Configuration Wazuh (`ossec.conf`) :**
    [cite_start]Modifiez le fichier de configuration de Wazuh pour déclarer la commande et la réponse active (déclenchée ici sur la règle SSH Brute Force ID 5763)[cite: 291].

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

    [cite_start][cite: 293-303].

3.  **Redémarrage :**
    Redémarrez le service Wazuh manager pour appliquer les changements.

-----

## 6\. Validation et Gestion

### 6.1. Test manuel du script

Pour vérifier que le script fonctionne sans déclencher une vraie attaque :

```bash
echo '{"parameters":{"alert":{"rule":{"id":"5710","description":"Tentative brute force SSH", "level":10},"agent":{"id":"001","name":"srv-web"},"data":{"srcip":"172.16.0.50"},"full_log":"sshd failed login"}}}' | /var/ossec/active-response/bin/glpi_ticket.sh
```

[cite_start]*Un ticket doit apparaître dans l'interface GLPI.* [cite: 275-276].

### 6.2. Attaque réelle

Depuis la machine **Kali Linux**, effectuez une attaque bruteforce SSH sur le serveur cible. Wazuh détectera l'attaque (Règle 5763) et déclenchera automatiquement le script.

### 6.3. Gestion du ticket (SOC N1)

Connectez-vous à GLPI avec le compte `SOC-N1`. [cite_start]Vous verrez le ticket créé contenant [cite: 308-316] :

  * **Titre :** Alerte Wazuh - sshd: brute force...
  * **Description :** Date, Agent, IP Source, Log complet.
  * **Priorité :** Calculée automatiquement selon la gravité Wazuh.

Vous pouvez désormais gérer le cycle de vie de l'incident (prise en compte, escalade, clôture).

