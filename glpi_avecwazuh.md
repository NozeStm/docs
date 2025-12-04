Absolument. Je vais refaire la documentation complète en intégrant la version détaillée du script Bash que vous avez fournie, en mettant à jour les jetons correspondants.

-----

# 📚 Documentation du TP : Gestion ITSM des Incidents de Sécurité (GLPI & Wazuh)

Ce document retrace les étapes de mise en place d'un système de gestion des tickets d'incidents de cybersécurité, intégrant l'outil ITSM **GLPI** avec la solution de détection **Wazuh** via son API, dans le contexte du SOC de l'entreprise SODECAF.

## 🎯 Objectifs du TP

  * Comprendre le rôle d'un outil ITSM.
  * Déployer et configurer **GLPI**.
  * Activer et tester l'**API REST de GLPI**.
  * Automatiser la création de tickets incidents dans GLPI à partir des alertes générées par **Wazuh** via un script d'**Active Response**.
  * Gérer les tickets incidents générés dans GLPI (processus SOC N1).

-----

## 1\. Introduction au Contexte

### 1.1 Qu'est-ce qu'un outil ITSM ?

Un outil **ITSM** (**Information Technology Service Management**) est une solution logicielle conçue pour gérer l'ensemble des services informatiques d'une organisation.

En **cybersécurité**, il permet :

  * La **réduction des délais de réponse** par l'automatisation.
  * L'**amélioration de la coordination** entre les équipes.
  * La **traçabilité** complète des incidents.
  * La **standardisation** des processus.

### 1.2 Présentation de GLPI

**GLPI** (**Gestionnaire Libre de Parc Informatique**) est la solution open-source choisie. Son module de gestion des tickets est central pour enregistrer, prioriser et suivre le cycle de vie des incidents.

-----

## 2\. Infrastructure et Préparation

### 2.1 Schéma de l'Infrastructure

L'intégration est réalisée sur le réseau SODECAF.

  * **SRV-GLPI** (Serveur ITSM) : `172.16.0.8`
  * **SRV-WAZUH** (Serveur SIEM/IDS) : `172.16.0.7`
  * Accès GLPI : `http://172.16.0.8/`

-----

## 3\. Configuration de l'API GLPI

L'API de GLPI est configurée pour permettre à Wazuh de s'authentifier et de créer des tickets en utilisant les jetons (`app_token` et `user_token`).

### 3.1 Activation de l'API

1.  Dans GLPI, naviguer vers **Configuration \> Générale \> API**.
2.  Activer l'**API Rest** et la connexion avec un **jeton externe**.

### 3.2 Jetons d'Application et Utilisateur

Les jetons configurés dans GLPI pour l'utilisateur API `wazuh` sont :

| Jeton | Description | Valeur à utiliser (issue de la configuration GLPI) |
| :--- | :--- | :--- |
| **`app_token`** | Jeton de l'application cliente (Wazuh API Client) | `IbQdjd32WUg5wAJrpQb7ZwnWdyVHLfITNriLrQHy` |
| **`user_token`** | Jeton de l'utilisateur `wazuh` | `wPNzhNzcwZNdabEJFBVzT4xkBg2BnWyZwQhlDWhv` |

-----

## 4\. Test de l'API (Depuis la console de Wazuh)

### 4.1 Test d'Authentification (`initSession`)

Cette commande permet d'ouvrir une session API et de recevoir un `session_token` temporaire.

```bash
curl -X GET \
-H 'Content-Type: application/json' \
-H "Authorization: user_token wPNzhNzcwZNdabEJFBVzT4xkBg2BnWyZwQhlDWhv" \
-H "App-Token: IbQdjd32WUg5wAJrpQb7ZwnWdyVHLfITNriLrQHy" \
'http://172.16.0.8/apirest.php/initSession?get_full_session=true'
```

### 4.2 Génération Manuelle d'un Ticket de Test

Utilisation du `session_token` reçu pour créer un ticket test. (Remplacer `VOTRE_SESSION_TOKEN` par le jeton obtenu à l'étape 4.1).

```bash
curl -X POST \
-H 'Content-Type: application/json' \
-H 'Authorization: user_token wPNzhNzcwZNdabEJFBVzT4xkBg2BnWyZwQhlDWhv' \
-H 'App-Token: IbQdjd32WUg5wAJrpQb7ZwnWdyVHLfITNriLrQHy' \
-H 'Session-Token: VOTRE_SESSION_TOKEN' \
-d'{
"input": {
"name": "Alerte Wazuh - Test API",
"content": "Ticket de test manuel créé depuis Wazuh.",
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
-H 'Authorization: user_token wPNzhNzcwZNdabEJFBVzT4xkBg2BnWyZwQhlDWhv' \
-H 'App-Token: IbQdjd32WUg5wAJrpQb7ZwnWdyVHLfITNriLrQHy' \
-H 'Session-Token: VOTRE_SESSION_TOKEN' \
http://172.16.0.8/apirest.php/killSession
```

-----

## 5\. Automatisation avec un Script Bash (Active Response)

### 5.1 Script `glpi_ticket.sh`

Ce script est le cœur de l'automatisation. Il est placé dans le répertoire `/var/ossec/active-response/bin/` sur le serveur Wazuh. Il se charge de lire les données JSON de l'alerte, d'établir la connexion API et de créer le ticket.

**Chemin :** `/var/ossec/active-response/bin/glpi_ticket.sh`

```bash
#!/bin/bash
# Script Active Response Wazuh → GLPI
# Version améliorée avec logs et mapping gravité → priorité

GLPI_URL="http://172.16.0.8/apirest.php"
APP_TOKEN="IbQdjd32WUg5wAJrpQb7ZwnWdyVHLfITNriLrQHy"
USER_TOKEN="wPNzhNzcwZNdabEJFBVzT4xkBg2BnWyZwQhlDWhv"

LOGFILE="/var/log/glpi_ticket.log"

# Lire tout le JSON d’entrée (multi-lignes possible)
INPUT_JSON=$(cat)

ALERT_DATE=$(date '+%Y-%m-%d %H:%M:%S %z')

echo "$(date) - Script exécuté par Wazuh" >> /var/log/glpi_ticket.log
echo "$INPUT_JSON" >> /var/log/glpi_ticket.log

# Extraire infos utiles avec jq
RULE_ID=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.rule.id')
RULE_DESC=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.rule.description')
AGENT_NAME=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.agent.name')
SRC_IP=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.data.srcip // empty')
SEVERITY=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.rule.level')
LOG=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.full_log')

# Mapping gravité Wazuh (level) → priorité GLPI
if [ "$SEVERITY" -le 3 ]; then
  PRIORITY=1   # Faible
elif [ "$SEVERITY" -le 7 ]; then
  PRIORITY=3   # Moyen
else
  PRIORITY=5   # Critique
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
      \"content\": \"🚨 Alerte Wazuh\n\n- Date alerte : $ALERT_DATE\n- Règle ID : $RULE_ID\n- Description : $RULE_DESC\n- Agent : $AGENT_NAME\n- Source IP : $SRC_IP\n- Gravité : $SEVERITY\n- Log : $LOG\",
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
echo "$(date '+%Y-%m-%d %H:%M:%S') - Ticket créé : Règle=$RULE_ID, Desc=$RULE_DESC, IP=$SRC_IP, Gravité=$SEVERITY, Priorité=$PRIORITY" >> "$LOGFILE"
```

### 5.2 Préparation et Permissions

1.  Créer le fichier de log : `touch /var/log/glpi_ticket.log`.
2.  Définir les permissions du script : `chown root:wazuh /var/ossec/active-response/bin/glpi_ticket.sh` et `chmod 750 /var/ossec/active-response/bin/glpi_ticket.sh`.

### 5.3 Configuration de la Réponse Active Wazuh

Ajouter la configuration suivante dans le fichier de configuration de Wazuh pour déclencher le script lors d'une attaque SSH (Règle ID `5763`) :

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

Redémarrer le service `wazuh-manager` pour appliquer les changements.

### 5.4 Test Réel de la Création de Ticket

Effectuer une attaque sur le service SSH du serveur cible (DVWA) à l'aide de Kali Linux pour vérifier que Wazuh détecte l'événement, déclenche le script et crée automatiquement un ticket incident dans GLPI.

-----

## 7\. Gestion des Tickets Incidents 🛠️

La phase de gestion vise à valider le cycle de vie de l'incident (ITIL) au sein de GLPI.

### 7.1 Connexion et Prise en Charge

1.  Connectez-vous à l'interface GLPI, idéalement avec le compte de l'analyste **SOC-N1**.
2.  Accédez à la liste des tickets et identifiez les tickets générés par Wazuh (statut "Nouveau").
3.  Prenez en charge l'incident pour signifier que le traitement a commencé.

### 7.2 Processus de Résolution

Pour chaque incident :

  * **Analyse** : L'analyste examine le contenu du ticket (`Règle ID`, `Source IP`, `Log`) pour comprendre la nature de l'attaque.
  * **Actions Correctives** : L'analyste prend les mesures nécessaires (ex: bloquer l'IP source au niveau du pare-feu, notifier l'utilisateur concerné).
  * **Suivi** : Toutes les étapes et actions sont documentées dans l'onglet **Suivi** du ticket GLPI pour assurer la traçabilité.

### 7.3 Escalade et Clôture

  * **Escalade** : Si l'incident nécessite une expertise supérieure ou une intervention sur une autre infrastructure, le ticket est **réaffecté** à un autre groupe (N2/N3) ou à un technicien spécialisé.
  * **Clôture** : Une fois la cause racine traitée et l'incident résolu, l'analyste documente la solution finale et **clôture** le ticket, le faisant passer au statut "Résolu".
