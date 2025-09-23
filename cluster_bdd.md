# Mise en oeuvre d'un cluster d'une base de donnée Maitre Esclave (UNIDIRECTIONELLE)
- Modification du fichier de conf (srv-web1)
````bash
nano /etc/mysql/mariadb.conf.d/50-server.cnf

#bin-address = 127.0.0.1 (commenter pour écouter toutes ses interfaces)
log_error = /var/log/mysql/error.log
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 10
max_binlog_size = 100M
binlog_do_db = nom base de donnée
````
- Création du dossier log si besoin (srv-web1)
````bash
mkdir -m 2750 /var/log/mysql
chown mysql /var/log/mysql

systemctl restart mariadb.service
````
- Création du compte réplicateur + droit (srv-web1)
````sql
mysql

create user 'replicateur'@'%' identified by 'Btssio2017';
grant replication slave on *.* to 'replicateur'@'%';
````
- Bloqué l'écriture sur les tables de la base de donnée (srv-web1)
````sql
mysql

flush tables with read lock;
unlock tables (Débloque la table si besoin)

show master status; (visualiser le status, si rien = pas bon)
````
- Modification du fichier de conf (srv-web2)
````bash
nano /etc/mysql/mariadb.conf.d/50-server.cnf

log_error = /var/log/mysql/error.log
server-id = 2
expire_logs_days = 10
max_binlog_size = 100M
master-retry-count = 20
replicate-do-db = nom base de donnée

systemctl restart mariadb.service
````
- Création du dossier log si besoin (srv-web2)
````bash
mkdir -m 2750 /var/log/mysql
chown mysql /var/log/mysql

systemctl restart mariadb.service
````
- Stopper l'esclave (srv-web2)
````sql
mysql

stop slave;
````
- Configurer le master (srv-web2)
````sql
mysql

change master to master_host='172.16.0.10', master_user='replicateur', master_password='Btssio2017', master_log_file='mysql-bin.000001', master_log_pos=328;
ATTENTION METTRE LA BONNE POSITION DU MASTER ET LE NOM DU LOG FILE(srv-web1)
````
- Activer l'esclave (srv-web2)
````sql
mysql

start slave;
````
- Vérifier la bonne configuration du master (srv-web2)
````sql
mysql

show slave status \G;
````
- Déverrouiller les tables de la bases (srv-web1)
````sql
mysql

unlock tables;
````
- Vérifier la mise à jour de la base (srv-web1)
````sql
mysql

use gsb_valide
select * from Visiteur;
update gsb_valide.Visiteur set mdp='toto' where login'agest';
````
- Vérifier la mise à jour de la base (srv-web2)
````sql
mysql

use gsb_valide
select * from Visiteur;

Voir si la valeur à bien changer aussi 
````
- Si mise à jour pas faite (srv-web2)
````sql
mysql

stop slave;
change master to master_host='172.16.0.10', master_user='replicateur', master_password='Btssio2017', master_log_file='mysql-bin.000001', master_log_pos=X;
ATTENTION METTRE LA BONNE POSITION DU MASTER ET LE NOM DU LOG FILE(srv-web1)
start slave;
show slave status \G;
````
----------------------------------------------------
# Activer le multi maitre (BIDIRECTIONELLE)
- Modification du fichier de conf (srv-web1)
````bash
nano /etc/mysql/mariadb.conf.d/50-server.cnf

#bin-address = 127.0.0.1 (commenter pour écouter toutes ses interfaces)
log_error = /var/log/mysql/error.log
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 10
max_binlog_size = 100M
binlog_do_db = nom base de donnée
log-slave-updates                 <- Ajouter
master-retry-count = 20           <- Ajouter
replicate-do-db = gsb_valide      <- Ajouter

systemctl restart mariadb.service
````
- Modification du fichier de conf (srv-web2)
````bash
nano /etc/mysql/mariadb.conf.d/50-server.cnf

#bin-address = 127.0.0.1 (commenter pour écouter toutes ses interfaces)   <- Ajouter
log_error = /var/log/mysql/error.log
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log      <- Ajouter
expire_logs_days = 10
max_binlog_size = 100M
binlog_do_db = nom base de donnée  <- Ajouter
log-slave-updates                 <- Ajouter
master-retry-count = 20           <- Ajouter
replicate-do-db = gsb_valide      <- Ajouter

systemctl restart mariadb.service
````
- Création du compte réplicateur + droit (srv-web2)
````sql
mysql

create user 'replicateur'@'%' identified by 'Btssio2017';
grant replication slave on *.* to 'replicateur'@'%';
````
- Stopper l'esclave (srv-web1 et srv-web2)
````sql
mysql

stop slave;
````
- Configurer le master (srv-web1 et srv-web2)
````sql
mysql

change master to master_host='172.16.0.11', master_user='replicateur', master_password='Btssio2017', master_log_file='mysql-bin.000001', master_log_pos=328; <- srv-web1
change master to master_host='172.16.0.10', master_user='replicateur', master_password='Btssio2017', master_log_file='mysql-bin.000002', master_log_pos=342; <- srv-web2
````
- Activer l'esclave (srv-web1 et srv-web2)
````sql
mysql

start slave;
````
- Vérifier la mise à jour des bases (srv-web1 et srv-web2)
````sql
mysql

use gsb_valide
select * from Visiteur;
update gsb_valide.Visiteur set mdp='toto' where login'agest';
select * from Visiteur;
````
-Intégrerla réplication dans le cluster (srv-web1)
````bash
crm status
crm configure primitive serviceMySQL ocf:heartbeat:mysql params socket=/var/run/mysqld/mysql.sock
crm configure clone cServiceMySQL serviceMySQL
crm status
````
- Lié la base de donnée au serveur
````bash
nano /var/www/appliFrais/include/_bdGest
````
