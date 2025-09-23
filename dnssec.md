# Mise en oeuvre d'un dns sécurisé avec dnssec
- Modification du named.conf.options
````bash
nano /etc/bind/named.conf.options
dnssec-enable yes;
````
- Création de la pair de clé
````bash
cd /etc/bind
mkdir keys
cd keys

ATTENTION A BIEN NOTER LES CLES
dnssec-keygen -a rsasha1 -b 1024 -n zone sodecaf.fr
dnssec-keygen -a rsasha1 -b 1024 -f KSK -n zone sodecaf.fr
````
- Modification du fichier
````bash
nano /var/cache/bind/db.sodecaf.fr

Clé publique à ajoutée

; Fichier de zone db.sodecaf.fr
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

; KSK
$include "/etc/bind/keys/Ksodecaf.fr.+005+26006.key"

; ZSK
$include "/etc/bind/keys/Ksodecaf.fr.+005+38466.key"
````
- Signé le fichier de zone
````bash
cd /var/cache/bind

CLE PRIVE KSK puis ZSK
dnssec-signzone -o sodecaf.fr -t -k /etc/bind/keys/Ksodecaf.fr.+005+26006 db.sodecaf.fr /etc/bind/keys/Ksodecaf.fr.+005+38466
````
- Modifier le fichier name.conf.local
````bash
nano /etc/bind/named.conf.options


//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
zone "sodecaf.fr" {
        type master;
        file "db.sodecaf.fr.signed"; <-
        allow-transfer {172.16.0.4;};
};

zone "0.16.172.in-addr.arpa" {
        type master;
        file "db.172.16.0.rev";
        allow-transfer {172.16.0.4;};
};

systemctl restart bind9
````
