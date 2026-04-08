# Déploiement de Graylog sur Debian 12 pour centraliser et analyser les logs

## I. Présentation  
Graylog est une solution open-source de type *puits de logs*, permettant la centralisation, le stockage et l'analyse en temps réel des journaux de machines et d’équipements réseau. Dans ce tutoriel, on installe Graylog (version gratuite) sur une machine Debian 12.  
Graylog reçoit les logs (par exemple via syslog), les indexe et offre des capacités de recherche, de corrélation et d’alerte.  
> Remarque : la version gratuite Graylog Open n’est pas un SIEM complet (manquent certaines fonctions de détection d’intrusion).  


---

## II. Prérequis  
- Version de Graylog : **6.1** recommandée.
- Composants nécessaires sur le serveur :  
  - **MongoDB 6** (requis par Graylog) 
  - **OpenSearch** (fork open-source d’Elasticsearch) 
  - **OpenJDK 17** :contentReference[oaicite:4]{index=4}  
- Machine cible : Debian 12 (les instructions sont faites pour cette version, mais l’architecture peut être distribuée sur plusieurs nœuds).
- Ressources recommandées : au moins **8 Go de RAM**, **256 Go d’espace disque**, pour garantir les performances.
- Configuration initiale :  
  - Adresse IP statique sur la machine Graylog.
  - Mise à jour du système (`apt-get update`) et installation des mises à jour. 
  - Configuration du fuseau horaire :  
    ```bash
    sudo timedatectl set-timezone Europe/Paris
    ```  
    :contentReference[oaicite:9]{index=9}  
  - Synchronisation de l’horloge via NTP (installer/configurer un client NTP).

---

## III. Installation pas à pas de Graylog  

### A. Mise à jour et préparation des paquets  
```bash
sudo apt-get update  
sudo apt-get install curl lsb-release ca-certificates gnupg2 pwgen  
````



---

### B. Installation de MongoDB

1. Importer la clé GPG de MongoDB :

   ```bash
   curl -fsSL https://www.mongodb.org/static/pgp/server-6.0.asc \
     | sudo gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg --dearmor
   ```

2. Ajouter le dépôt MongoDB :

   ```bash
   echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg] \
     http://repo.mongodb.org/apt/debian bullseye/mongodb-org/6.0 main" \
     | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
   ```

3. Mettre à jour la liste des paquets et installer MongoDB :

   ```bash
   sudo apt-get update  
   sudo apt-get install -y mongodb-org  
   ```

4. **Dépendance manquante** : Debian 12 ne fournit pas `libssl1.1` par défaut. Il faut le télécharger et l’installer manuellement :

   ```bash
   wget http://archive.ubuntu.com/ubuntu/pool/main/o/openssl/libssl1.1_1.1.1f-1ubuntu2.23_amd64.deb  
   sudo dpkg -i libssl1.1_1.1.1f-1ubuntu2.23_amd64.deb  
   ```

5. Relancer l’installation de MongoDB :

   ```bash
   sudo apt-get install -y mongodb-org  
   ```

6. Activer et démarrer MongoDB :

   ```bash
   sudo systemctl daemon-reload  
   sudo systemctl enable mongod.service  
   sudo systemctl restart mongod.service  
   sudo systemctl --type=service --state=active | grep mongod  
   ```


---

### C. Installation d’OpenSearch

1. Importer la clé GPG OpenSearch :

   ```bash
   curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp \
     | sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring
   ```

2. Ajouter le dépôt OpenSearch :

   ```bash
   echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] \
     https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" \
     | sudo tee /etc/apt/sources.list.d/opensearch-2.x.list
   ```

3. Mettre à jour et installer OpenSearch :

   ```bash
   sudo apt-get update  
   sudo env OPENSEARCH_INITIAL_ADMIN_PASSWORD=IT-Connect2024! apt-get install opensearch  
   ```

   > Remplacez `IT-Connect2024!` par un mot de passe robuste. ([IT-Connect][1])
4. Configurer OpenSearch : éditez `/etc/opensearch/opensearch.yml` pour y mettre :

   ```yaml
   cluster.name: graylog  
   node.name: ${HOSTNAME}  
   path.data: /var/lib/opensearch  
   path.logs: /var/log/opensearch  
   discovery.type: single-node  
   network.host: 127.0.0.1  
   action.auto_create_index: false  
   plugins.security.disabled: true  
   ```

5. Configurer la mémoire JVM d’OpenSearch : éditez `/etc/opensearch/jvm.options` — remplacer :

   ```
   -Xms1g  
   -Xmx1g  
   ```

   par :

   ```
   -Xms4g  
   -Xmx4g  
   ```

   (adapte selon ta RAM). ([IT-Connect][1])
6. Ajuster la valeur `vm.max_map_count` si nécessaire :

   ```bash
   cat /proc/sys/vm/max_map_count  
   sudo sysctl -w vm.max_map_count=262144  
   ```

7. Activer et démarrer OpenSearch :

   ```bash
   sudo systemctl daemon-reload  
   sudo systemctl enable opensearch  
   sudo systemctl restart opensearch  
   ```

---

### D. Installation de Graylog

1. Télécharger et installer le dépôt Graylog :

   ```bash
   wget https://packages.graylog2.org/repo/packages/graylog-6.1-repository_latest.deb  
   sudo dpkg -i graylog-6.1-repository_latest.deb  
   sudo apt-get update  
   sudo apt-get install graylog-server  
   ```

2. Configurer Graylog :

   * Générer une clé `password_secret` (96 caractères) :

     ```bash
     pwgen -N 1 -s 96  
     ```

   * Copier cette clé dans `/etc/graylog/server/server.conf` à l’option `password_secret`. ([IT-Connect][1])
   * Générer un mot de passe admin, puis hash SHA-256 :

     ```bash
     echo -n "TonMotDePasse@" | shasum -a 256  
     ```

   * Copier le hash dans `root_password_sha2` dans le fichier `server.conf`. ([IT-Connect][1])
   * Configurer l’interface web : `http_bind_address = 0.0.0.0:9000` (pour écouter toutes les interfaces sur le port 9000). ([IT-Connect][1])
   * Configurer `elasticsearch_hosts = http://127.0.0.1:9200` (connexion à OpenSearch local). ([IT-Connect][1])
3. Activer et démarrer Graylog :

   ```bash
   sudo systemctl enable --now graylog-server  
   ```

4. Se connecter à l’interface Web : ouvre un navigateur vers `http://<IP‑ou‑hostname>:9000`, connecte-toi avec `admin` + ton mot de passe défini.
   Si tu ne peux pas te connecter, consulte les logs Graylog :

   ```bash
   tail -f /var/log/graylog-server/server.log  
   ```


---

### E. Création d’un compte administrateur personnalisé

Dans l’interface Graylog :

1. Va dans **System → Users and Teams**
2. Clique sur **Create user**, remplis les champs (nom, prénom, email…)
3. Assigne à ce compte le rôle **administrateur** pour avoir les droits élevés
4. Cela garantit une meilleure traçabilité : chaque admin utilise son propre compte.

---

## IV. Conclusion

* Graylog est maintenant installé sur ta Debian 12 : tu peux centraliser, indexer et analyser tes logs depuis une console unique. ([IT-Connect][1])
* Les prochaines étapes possibles :

  * Déclarer des **Inputs** dans Graylog pour recevoir les logs (Linux, Windows, équipements réseau) ([IT-Connect][1])
  * Créer des **Streams** et des **Index** selon tes besoins. ([IT-Connect][1])
  * Sécuriser OpenSearch en activant le module de sécurité, mettre en place des **alertes email** via les notifications Graylog. ([IT-Connect][1])

---
