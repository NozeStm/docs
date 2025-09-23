# Memo SQL
- Créer une base de donnée
```sql
create database gsb_valide;
```
- Créer un nouvel utilisateur
```sql
create user 'userGsb'@'localhost' identified by 'secret';
```
- Importer un fichier SQL
```bash
mysql gsb_valide < nom_fichier.sql
```
- Donner les droits d'un utilisateur à une base
```sql
grant all privileges on gsb_valide.* to 'userGsb'@'localhost';
flush privileges;
```
- Afficher le contenu d'une table
```sql
use gsb_valide;
select * from Visiteur;
```
