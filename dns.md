# Mise en oeuvre d'un dns avec bind9
- Installation de bind 9
````bash
apt install bind9
````
- Déclaration d'une zone (srv-dns1)
````bash
nano /etc/bind/name.conf.local

//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
zone "sodecaf.fr" {
        type master;
        file "db.sodecaf.fr";
};
````
- Création du fichier de zone (srv-dns1)
````bash
cd /var/cache/bind
nano /var/cache/bind/db.sodecaf.fr

; fichier de zone db.sodecaf.fr
$TTL 86400
@               IN      SOA     srv-dns1.sodecaf.fr. hostmaster.sodecaf.fr. (
                                2025092201      ;serial
                                86400           ;refresh
                                21600           ;retry
                                3600000         ;expire
                                3600 )          ;negative cache TTL
@               IN      NS      srv-dns1.sodecaf.fr.
@               IN      NS      srv-dns2.sodecaf.fr.

srv-dns1        IN      A       172.16.0.3
srv-dns2        IN      A       172.16.0.4
srv-web1        IN      A       172.16.0.10
srv-web2        IN      A       172.16.0.11
www             IN      A       172.16.0.12
web1            IN      CNAME   srv-web1.sodecaf.fr.
web2            IN      CNAME   srv-web2.sodecaf.fr.
````
- Vérification (srv-dns1)
````bash
named-checkconf = si rien ok
name-chekzone sodecaf.fr db.sodecaf.fr = normalement ok

systemctl restart bind9
````
- Vérification (srv-dns2)
````bash
nslookup web1.sodecaf.fr 172.16.0.3
dig web1.sodecaf.fr @172.16.0.3
````
- Modification du fichier de déclaration de zone (srv-dns1)
````bash
nano /etc/bind/name.conf.local

//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
zone "sodecaf.fr" {
        type master;
        file "db.sodecaf.fr";
};

zone "0.16.172.in-addr.arpa" {
        type master;
        file "db.172.16.0.rev";
};
````
- Création du fichier de zone inverse (srv-dns1)
````bash
cd /var/cache/bind
cp db.sodecaf.fr db.172.16.0.rev

nano db.172.0.16.rev


; Fichier de zone inverse db.172.16.0.rev
$TTL 86400
@               IN      SOA     srv-dns1.sodecaf.fr. hostmaster.sodecaf.fr. (
                                2025092201      ;serial
                                86400           ;refresh
                                21600           ;retry
                                3600000         ;expire
                                3600 )          ;negative cache TTL
@               IN      NS      srv-dns1.sodecaf.fr.
@               IN      NS      srv-dns2.sodecaf.fr.

3               IN      PTR     srv-dns1.sodecaf.fr.
4               IN      PTR     srv-dns1.sodecaf.fr.
10              IN      PTR     srv-web1.sodecaf.fr.
11              IN      PTR     srv-web2.sodecaf.fr.
12              IN      PTR     www.sodecaf.fr.
````
- Vérification (srv-dns1)
````bash
named-checkconf = si rien ok 
named-checkzone 0.16.172.in-addr.arpa db.172.16.0.rev = normalement ok
````
- Vérification (srv-dns2)
````bash
nslookup 172.16.0.10 172.16.0.3
dig -x 172.16.0.10 @172.16.0.3
````
- Déclaration d'une zone (srv-dns2)
````bash
nano /etc/bind/name.conf.local

//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
zone "sodecaf.fr" {
        type slave;
        file "slave/db.sodecaf.fr";
        masters {172.16.0.3;};
};

zone "0.16.172.in-addr.arpa" {
        type slave;
        file "slave/db.172.16.0.rev";
        masters {172.16.0.3;};
};
````
- Création du dossier slave (srv-dns2)
````bash
cd /var/cache/bind
mkdir slave
chgrp bind slave/
chmod g+w slave/
````
- Modification du fichier de déclaration de zone (srv-dns1)
````bash
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
zone "sodecaf.fr" {
        type master;
        file "db.sodecaf.fr";
        allow-transfer {172.16.0.4;};
};

zone "0.16.172.in-addr.arpa" {
        type master;
        file "db.172.16.0.rev";
        allow-transfer {172.16.0.4;};
};
````
- Redémarrer bind9 sur les deux serveur
- Vérification de la copie des fichiers (srv-dns2)
````bash
cd /var/cache/bind/slave
ls
````
- Redirection des requetes (srv-dns1 et srv-dns2)
````bash
cd /etc/bind
nano named.conf.options

forwarders {
                8.8.8.8;
        };

allow-query {any;};

systemctl restart bind9
````
