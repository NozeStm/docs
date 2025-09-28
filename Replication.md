Parfait 👌 tu veux donc du **MariaDB en actif/actif** (aussi appelé **Master ↔ Master replication**).
Chaque serveur est à la fois **maître** et **esclave** : ils répliquent les changements entre eux en continu.

⚠️ Attention :

* Ça marche bien si tu fais attention aux **conflits d’écriture** (évite d’écrire sur les deux en même temps dans les mêmes tables).
* C’est utilisé pour **haute disponibilité**, ou pour répartir les lectures/écritures selon les cas.
* Pour une solution plus robuste et automatique, MariaDB propose **Galera Cluster**, mais restons sur du Master ↔ Master classique.

---

# 🔹 Étapes : MariaDB Master ↔ Master sur Debian 12

## 1. Installer MariaDB

Sur les **deux serveurs (`db1` et `db2`)** :

```bash
sudo apt update && sudo apt install mariadb-server mariadb-client -y
```

---

## 2. Configurer `db1`

Édite `/etc/mysql/mariadb.conf.d/50-server.cnf` :

```ini
[mysqld]
bind-address = 0.0.0.0
server-id = 1
log_bin = /var/log/mysql/mariadb-bin
binlog_do_db = mondb   # la base que tu veux répliquer
auto_increment_increment = 2
auto_increment_offset = 1
```

Puis :

```bash
sudo systemctl restart mariadb
```

---

## 3. Configurer `db2`

Édite `/etc/mysql/mariadb.conf.d/50-server.cnf` :

```ini
[mysqld]
bind-address = 0.0.0.0
server-id = 2
log_bin = /var/log/mysql/mariadb-bin
binlog_do_db = mondb
auto_increment_increment = 2
auto_increment_offset = 2
```

Puis :

```bash
sudo systemctl restart mariadb
```

👉 Les options `auto_increment_*` évitent les conflits sur les colonnes `AUTO_INCREMENT` (db1 prendra les IDs impairs, db2 les pairs).

---

## 4. Créer l’utilisateur de réplication sur les deux serveurs

Sur `db1` :

```sql
CREATE USER 'repl'@'%' IDENTIFIED BY 'passwordfort';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```

Sur `db2` (même commande) :

```sql
CREATE USER 'repl'@'%' IDENTIFIED BY 'passwordfort';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```

---

## 5. Initialiser la réplication

Sur `db1` :

```sql
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
```

→ Note `File` et `Position` (ex: `mariadb-bin.000001`, `1234`).

Fais un dump :

```bash
mysqldump -u root -p mondb > dump.sql
scp dump.sql user@db2:/tmp/
```

Sur `db2` :

```bash
mysql -u root -p mondb < /tmp/dump.sql
```

Puis libère le verrou sur `db1` :

```sql
UNLOCK TABLES;
```

---

## 6. Configurer la réplication croisée

Sur `db2`, dis-lui de suivre `db1` :

```sql
CHANGE MASTER TO
  MASTER_HOST='192.168.1.10',   -- IP de db1
  MASTER_USER='repl',
  MASTER_PASSWORD='passwordfort',
  MASTER_LOG_FILE='mariadb-bin.000001',
  MASTER_LOG_POS=1234;

START SLAVE;
```

Sur `db1`, dis-lui de suivre `db2` (fais aussi `SHOW MASTER STATUS;` sur `db2` avant) :

```sql
CHANGE MASTER TO
  MASTER_HOST='192.168.1.11',   -- IP de db2
  MASTER_USER='repl',
  MASTER_PASSWORD='passwordfort',
  MASTER_LOG_FILE='mariadb-bin.000001',
  MASTER_LOG_POS=5678;

START SLAVE;
```

---

## 7. Vérifier le statut

Sur chaque serveur :

```sql
SHOW SLAVE STATUS\G
```

Tu dois voir :

* `Slave_IO_Running: Yes`
* `Slave_SQL_Running: Yes`

---

## ✅ Résultat

* `db1` et `db2` sont **synchronisés en temps réel**.
* Tu peux écrire sur l’un ou l’autre (mais ⚠️ attention aux conflits).
* Les IDs auto\_increment seront distribués sans collision (pairs/impairs).

---

👉 Veux-tu que je te montre aussi comment faire la **même réplication mais avec Galera Cluster** (vraie solution multi-maître sans risque de conflit), ou tu veux rester sur ce setup Master ↔ Master classique ?
