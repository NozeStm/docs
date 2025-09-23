# Mise en oeuvre d'un cluster de serveur web
- Installation de corosync, pacemaker et crmsh
```bash
apt install corosync pacemaker crmsh
```
- Création de la clé authkey pour le chiffrement des échanges
```bash
corosync-keygen
ls -l /etc/corosync/
```
- Création d'un nouveau fichier de configuration de corosync
```bash
mv corosync.conf corosync.conf.sav
nano corosync.conf
```
```bash
totem {
        version: 2
        cluster_name: cluster_web
        crypto_cipher: aes256
        crypto_hash: sha1
        clear_node_high_bit:yes
}
logging {
        fileline: off
        to_logfile: yes
        logfile: /var/log/corosync/corosync.log
        to_syslog: no
        debug: off
        timestamp: on
        logger_subsys {
                subsys: QUORUM
                debug: off
        }
}
quorum {
        provider: corosync_votequorum
        expected_votes: 2
        two_nodes: 1
}
nodelist {
        node {
                name: srv-web1
                nodeid: 1
                ring0_addr: 172.16.0.10
        }
        node {
                name: srv-web2
                nodeid: 2
                ring0_addr: 172.16.0.11
        }
}
service {
        ver: 0
        name: pacemaker
}
```                  
- Vérification de la configuration
```bash
corosync-cfgtool -s
```
- Cloner le serveur web, modifier IP et nom du clone
- Visualiser l'état du cluster
```bash
crm status
crm configure show
```
- Désactiver le stonith (shot the other node in the head)
```bash
crm configure property stonith-enabled=false
```
- Désactiver le quorum
```bash
crm configure property no-quorum-policy="ignore"
```
- Configuration du failover IP (IP virtuelle)
```bash
crm configure primitive IPFailover ocf:heartbeat:IPaddr2 params ip=172.16.0.12 cidr_netmask=24 nic=ens192 iflabel=VIP
```
- Tests de basculement
```bash
crm node online
crm node standby
```
- Arrêter et supprimer une ressource
```bash
crm resource stop nom_ressource
crm configure delete nom_ressource
```
- Editer la configuration : à utiliser avec prudence
```bash
crm configure edit
```
- Création d'une ressource serviceWeb
```bash
crm configure primitive serviceWeb lsb:apache2 op monitor interval=60s op start interval=0 timeout=60s op stop interval=0 timeout=60s
```
- Regroupement des ressources IPFailover et serviceWeb
```bash
crm configure group servweb IPFailover serviceWeb meta migration-threshold="5"
```
- Définir une préférence de noeud primaire pour l'IP virtuelle
```bash
crm resource move IPFailover srv-web1
```
- Effacer les erreurs sur une ressource
```bash
crm resource cleanup IPFailover
``` 
