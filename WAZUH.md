Voici une documentation technique intégrale et structurée, combinant les deux Travaux Pratiques (TP1 et TP2). Elle est conçue pour servir de guide pas-à-pas pour refaire l'intégralité du laboratoire.

-----

# Documentation Technique : Mise en œuvre SIEM Wazuh et Gestion ITSM GLPI

Cette documentation couvre le déploiement d'un SIEM (Wazuh), la sécurisation d'une infrastructure web, et l'automatisation de la création de tickets d'incidents dans un outil ITSM (GLPI).

## Architecture Réseau et Prérequis

[cite_start]**Plan d'adressage IP[cite: 61, 413]:**

  * **Zone LAN (172.16.0.0/24) :**
      * `SRV-WIN1` (AD, DNS, DHCP) : `172.16.0.1`
      * `SRV-WAZUH` (Ubuntu Server 22.04) : `172.16.0.7`
      * `SRV-GLPI` (Debian) : `172.16.0.8`
      * `Kali Linux` (Attaquant/Client) : DHCP
  * **Zone DMZ (192.168.0.0/24) :**
      * `SRV-WEB` (Debian - DVWA) : `192.168.0.1`
  * **Routeur/Pare-feu :**
      * `OPNsense` : LAN `172.16.0.254` / DMZ `192.168.0.254`

-----

## PARTIE 1 : Mise en œuvre du SIEM WAZUH (TP1)

### A. Installation du Serveur Web (SRV-WEB)

Ce serveur hébergera l'application vulnérable DVWA.

1.  **Configuration NTP :**
    Editez `/etc/systemd/timesyncd.conf` :

    ```ini
    [Time]
    NTP=ntp.univ-rennes2.fr
    ```

    Activez et vérifiez :

    ```bash
    timedatectl set-ntp true
    systemctl restart systemd-timesyncd.service
    timedatectl timesync-status
    ```

    [cite_start][cite: 451, 452, 454, 455]

2.  **Installation de DVWA :**
    [cite_start]Exécutez le script d'installation automatique (ne rentrez pas de mot de passe à l'installation SQL, faites juste "Entrée")[cite: 460, 463]:

    ```bash
    apt install curl
    bash -c "$(curl --fail --show-error --silent --location https://raw.githubusercontent.com/IamCarron/DVWA-Script/main/Install-DVWA.sh)"
    ```

3.  **Initialisation :**

      * Accédez à `http://192.168.0.1/DVWA` depuis un client.
      * [cite_start]Identifiants par défaut : `admin` / `password`[cite: 470].
      * [cite_start]Cliquez sur **"Create / Reset Database"** pour finaliser l'installation[cite: 473].

-----

### B. Installation du Serveur WAZUH (SRV-WAZUH)

1.  **Configuration Réseau (Netplan) :**
    [cite_start]Créez/éditez `/etc/netplan/99_config.yaml` (respectez l'indentation)[cite: 484, 485]:

    ```yaml
    network:
      version: 2
      renderer: networkd
      ethernets:
        eth0:
          addresses:
            - 172.16.0.7/24
          routes:
            - to: default
              via: 172.16.0.254
          nameservers:
            search: [sodecaf.local]
            addresses: [172.16.0.1, 1.1.1.1]
    ```

    [cite_start]Appliquez : `netplan apply`[cite: 501].

2.  **Installation de Wazuh (All-in-one) :**

    ```bash
    curl -sO https://packages.wazuh.com/4.12/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
    ```

    > [cite_start]**Note :** À la fin, notez bien le mot de passe `admin` affiché[cite: 504, 512].

3.  **Déploiement des Agents :**

      * Allez sur l'interface web `https://172.16.0.7`.
      * Cliquez sur **"Deploy new agent"**.
      * [cite_start]Installez l'agent sur : `SRV-WIN1`, `SRV-WEB` (Linux) et `OPNsense`[cite: 529, 531, 534].
      * *Pour OPNsense :* Installez le plugin `os-wazuh-agent` et configurez l'IP du serveur dans Services \> Agent Wazuh.
      * [cite_start]*Pare-feu :* Assurez-vous que les ports 1514 et 1515 sont ouverts[cite: 532].

-----

### C. Configuration des Réponses Actives (Active Response)

#### 1\. Protection SSH (Brute Force)

Objectif : Bannir l'IP attaquante via `iptables` sur le SRV-WEB.

1.  **Sur le Serveur Wazuh (`/var/ossec/etc/ossec.conf`) :**
    [cite_start]Ajoutez le bloc suivant pour déclencher le drop firewall sur la règle 5763 (SSH brute force)[cite: 633]:

    ```xml
    <active-response>
      <command>firewall-drop</command>
      <location>local</location>
      <rules_id>5763</rules_id>
      <timeout>180</timeout>
    </active-response>
    ```

    [cite_start]*Redémarrez le manager :* `systemctl restart wazuh-manager`[cite: 641].

2.  **Test (Attaque depuis Kali) :**

    ```bash
    hydra -t 4 -l etudiant -P /usr/share/wordlists/rockyou.txt.gz 192.168.0.1 ssh
    ```

    [cite_start]Vérifiez sur le dashboard Wazuh (Event 5763) et le blocage de l'IP[cite: 644, 645].

#### 2\. Désactivation de compte utilisateur Linux

Objectif : Désactiver un compte après échecs répétés.

1.  [cite_start]**Configuration :** Suivez la documentation officielle liée dans le TP pour configurer `disable-account`[cite: 695].
2.  **Test :** Tentez 3 connexions SSH échouées avec un utilisateur valide (ex: `user1`). [cite_start]Vérifiez le verrouillage avec `passwd --status user1` (Status `L`)[cite: 700].

-----

### D. Détection d'Attaques Web (Wazuh + Teler)

#### 1\. Installation de Teler (IDS) sur SRV-WEB

1.  **Téléchargement et Installation :**

    ```bash
    wget https://github.com/teler-sh/teler/releases/download/v2.0.0/teler_2.0.0_linux_amd64.tar.gz
    tar -xvzf teler_2.0.0_linux_amd64.tar.gz
    mkdir /var/log/teler
    ```

    [cite_start][cite: 716, 730]

2.  **Configuration (`teler.yaml`) :**
    [cite_start]Créez le fichier de configuration avec le format de log Apache[cite: 720, 722]:

    ```yaml
    log_format: $remote_addr - - [$time_local] "$request_method $request_uri $request_protocol" $status $body_bytes_sent "$http_referer" "$http_user_agent"
    logs:
      file:
        active: true
        json: true
        path: /var/log/teler/output.log
    ```

#### 2\. Configuration de l'Agent Wazuh (SRV-WEB)

[cite_start]Editez `/var/ossec/etc/ossec.conf` sur SRV-WEB pour lire les logs Teler[cite: 733]:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/teler/output.log</location>
</localfile>
```

[cite_start]Redémarrez l'agent : `systemctl restart wazuh-agent`[cite: 738].

#### 3\. Configuration du Serveur Wazuh (Règles et Réponse)

1.  **Règles personnalisées (`/var/ossec/etc/rules/local_rules.xml`) :**
    [cite_start]Ajoutez les règles pour détecter les logs Teler[cite: 760, 773, 782]:

    ```xml
    <group name="teler,">
      <rule id="100012" level="10">
        <decoded_as>json</decoded_as>
        <field name="category" type="pcre2">Common Web Attack(: .*)?|CVE-[0-9]{4}-[0-9]{4,7}</field>
        <field name="request_uri" type="pcre2">\D.+ -</field>
        <field name="remote_addr" type="pcre2">\d+.\d+.\d+.\d+|::1</field>
        <mitre><id>T1210</id></mitre>
        <description>teler detected $(category) against resource $(request_uri) from $(remote_addr)</description>
      </rule>
      </group>
    ```

2.  **Réponse Active (`ossec.conf`) :**
    [cite_start]Bloquer les IP déclenchant ces règles pendant 6 minutes[cite: 750]:

    ```xml
    <active-response>
      <command>firewall-drop</command>
      <location>local</location>
      <rules_id>100012,100013,100014</rules_id>
      <timeout>360</timeout>
    </active-response>
    ```

3.  **Lancement et Test :**

      * Sur SRV-WEB, lancez Teler : `tail -f /var/log/apache2/access.log | [cite_start]./teler -c teler.yaml`[cite: 740].
      * [cite_start]Depuis Kali, attaquez : `nikto -h http://192.168.0.1/DVWA`[cite: 798].

-----

### E. Détection d'Intrusion Réseau (Wazuh + Suricata)

1.  **Installation sur SRV-WEB :**

    ```bash
    apt install suricata
    cd /tmp/ && curl -LO https://rules.emergingthreats.net/open/suricata-6.0.10/emerging.rules.tar.gz
    mkdir -p /etc/suricata/rules/
    tar -xvzf emerging.rules.tar.gz && sudo mv rules/*.rules /etc/suricata/rules/
    chmod 640 /etc/suricata/rules/*.rules
    ```

    [cite_start][cite: 851, 855]

2.  **Configuration Suricata (`/etc/suricata/suricata.yaml`) :**
    [cite_start]Modifiez les variables `HOME_NET` (votre IP) et l'interface réseau[cite: 860, 870].

3.  **Liaison Wazuh :**
    [cite_start]Dans `/var/ossec/etc/ossec.conf` (SRV-WEB), ajoutez la lecture du log Suricata[cite: 874]:

    ```xml
    <localfile>
      <log_format>json</log_format>
      <location>/var/log/suricata/eve.json</location>
    </localfile>
    ```

    Redémarrez agent et Suricata. [cite_start]Testez avec des Pings depuis Kali[cite: 881].

-----

## PARTIE 2 : Gestion ITSM des Incidents (TP2)

L'objectif est d'automatiser la création de tickets dans GLPI lorsqu'une alerte critique est détectée par Wazuh.

### A. Configuration de GLPI et de l'API

1.  **Préparation GLPI :**

      * [cite_start]Connectez-vous à `http://172.16.0.8/` (admin: `glpi` / `glpi`)[cite: 89, 91].
      * Allez dans **Configuration \> Générale \> API**.
      * [cite_start]Activez l'API Rest, activez "Connexion avec identifiants" et "Connexion avec jeton externe"[cite: 102, 114].

2.  **Création du Client API (pour Wazuh) :**

      * Ajoutez un Client API nommé `wazuh_api`.
      * Activez-le.
      * [cite_start]Définissez la plage IP autorisée : `172.16.0.7` (IP Wazuh)[cite: 131].
      * [cite_start]Générez et sauvegardez le **Jeton d'application (`app_token`)**[cite: 136].

3.  **Création de l'Utilisateur API :**

      * Créez un utilisateur `wazuh` (Administration \> Utilisateurs).
      * [cite_start]Dans l'onglet "Clefs d'accès distant", générez un **Jeton d'API personnel (`user_token`)**[cite: 143, 151].

4.  **Test de connexion (depuis le serveur Wazuh) :**

    ```bash
    curl -X GET \
    -H 'Content-Type: application/json' \
    -H "Authorization: user_token VOTRE_USER_TOKEN" \
    -H "App-Token: VOTRE_APP_TOKEN" \
    'http://172.16.0.8/apirest.php/initSession?get_full_session=true'
    ```

    [cite_start]Vous devez recevoir un `session_token`[cite: 166, 174].

-----

### B. Script d'Automatisation (Intégration Wazuh-GLPI)

1.  **Création du Script :**
    [cite_start]Sur le serveur Wazuh, créez `/var/ossec/active-response/bin/glpi_ticket.sh`[cite: 271].
    [cite_start]Copiez le contenu suivant (corrigeant les erreurs OCR du document source)[cite: 217]:

    ```bash
    #!/bin/bash
    # Script Active Response Wazuh -> GLPI

    GLPI_URL="http://172.16.0.8/apirest.php"
    # Remplacez par vos tokens réels
    APP_TOKEN="VOTRE_APP_TOKEN_ICI"
    USER_TOKEN="VOTRE_USER_TOKEN_ICI"
    LOGFILE="/var/log/glpi_ticket.log"

    # Lire l'entrée JSON venant de Wazuh
    INPUT_JSON=$(cat)
    ALERT_DATE=$(date +'%Y-%m-%d %H:%M:%S %z')

    # Logs pour débogage
    echo "$(date) - Script exécuté par Wazuh" >> $LOGFILE

    # Extraction des données via jq
    RULE_ID=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.rule.id')
    RULE_DESC=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.rule.description')
    AGENT_NAME=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.agent.name')
    SRC_IP=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.data.srcip // empty')
    SEVERITY=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.rule.level')
    LOG=$(echo "$INPUT_JSON" | jq -r '.parameters.alert.full_log')

    # Mapping Gravité Wazuh -> Priorité GLPI
    if [ "$SEVERITY" -le 3 ]; then
        PRIORITY=1 # Faible
    elif [ "$SEVERITY" -le 7 ]; then
        PRIORITY=3 # Moyen
    else
        PRIORITY=5 # Critique
    fi

    # 1. Ouvrir Session GLPI
    SESSION_TOKEN=$(curl -s -X GET \
      -H "Authorization: user_token $USER_TOKEN" \
      -H "App-Token: $APP_TOKEN" \
      "$GLPI_URL/initSession" | jq -r '.session_token')

    # 2. Créer le Ticket
    RESPONSE=$(curl -s -X POST \
      -H "Content-Type: application/json" \
      -H "App-Token: $APP_TOKEN" \
      -H "Session-Token: $SESSION_TOKEN" \
      -d "{
        \"input\": {
          \"name\": \"Alerte Wazuh - $RULE_DESC\",
          \"content\": \"Alerte Wazuh\n\n- Date: $ALERT_DATE\n- Règle ID: $RULE_ID\n- Description: $RULE_DESC\n- Agent: $AGENT_NAME\n- Source IP: $SRC_IP\n- Gravité: $SEVERITY\n- Log: $LOG\",
          \"priority\": $PRIORITY,
          \"urgency\": 3,
          \"impact\": 3,
          \"status\": 1,
          \"type\": 1,
          \"itilcategories_id\": 2
        }
      }" \
      "$GLPI_URL/Ticket")

    # 3. Fermer Session
    curl -s -X GET \
      -H "App-Token: $APP_TOKEN" \
      -H "Session-Token: $SESSION_TOKEN" \
      "$GLPI_URL/killSession" > /dev/null

    echo "$(date) - Ticket créé pour règle $RULE_ID" >> $LOGFILE
    ```

2.  **Permissions :**

    ```bash
    touch /var/log/glpi_ticket.log
    chown root:wazuh /var/log/glpi_ticket.log
    chmod 660 /var/log/glpi_ticket.log
    chown root:wazuh /var/ossec/active-response/bin/glpi_ticket.sh
    chmod 750 /var/ossec/active-response/bin/glpi_ticket.sh
    ```

    [cite_start][cite: 270, 273]

3.  **Test Manuel du Script :**
    [cite_start]Injectez un faux JSON pour vérifier la création du ticket[cite: 275]:

    ```bash
    echo '{"parameters":{"alert":{"rule":{"id":"5710","description":"Tentative brute force SSH", "level":10},"agent":{"id":"001","name":"srv-web"},"data":{"srcip":"172.16.0.50"},"full_log":"sshd failed login"}}}' | /var/ossec/active-response/bin/glpi_ticket.sh
    ```

-----

### C. Automatisation sur Alerte SSH

1.  **Configuration Wazuh (`ossec.conf`) :**
    [cite_start]Définissez la commande et l'Active Response pour lier le script à la règle 5763 (attaque SSH)[cite: 293, 298]:

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

    Redémarrez le service : `systemctl restart wazuh-manager`.

2.  **Validation Finale :**
    Lancez une attaque brute force SSH réelle depuis Kali. [cite_start]Vérifiez qu'un nouveau ticket apparaît automatiquement dans l'interface GLPI avec les détails de l'attaque[cite: 304, 305].
