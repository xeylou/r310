# création des vms

création des vms

```bash
r303-deb12-host1  
r303-deb12-bind1  
r303-deb12-bind2  
```

maj des hostnames  
```bash
hostnamectl set-hostname r303-deb12-host1
hostnamectl set-hostname r303-deb12-bind1
hostnamectl set-hostname r303-deb12-bind2
```

modifications des IPs sur les vms  
`nano /etc/network/interfaces`

```bash
auto enp1s0
iface enp1s0 inet static
address 192.168.122.10
netmask 255.255.255.0
gateway 192.168.122.1

auto enp1s0
iface enp1s0 inet static
address 192.168.122.11
netmask 255.255.255.0
gateway 192.168.122.1

auto enp1s0
iface enp1s0 inet static
address 192.168.122.12
netmask 255.255.255.0
gateway 192.168.122.1
```

ajouter dans `/etc/hosts` des vms

```bash
192.168.122.10 r303-deb12-host1
192.168.122.10 r303-deb12-host1.rzo.lan
192.168.122.11 r303-deb12-bind1
192.168.122.11 rzo.lan
192.168.122.12 r303-deb12-bind2
192.168.122.12 r303-deb12-bind2.rzo.lan
```

modification de la configuration ssh sur les vms pour se connecter au compte root  
`nano /etc/ssh/sshd_config`

```bash
PermitRootLogin yes
```

redémarrer le daemon ssh pour prendre la modification en compte  
`systemctl restart sshd`

faire une snapshot des vms sur virt-manager
```bash
.10
hostnamectl
ssh using root

.11
hostnamectl
ssh using root

.12
hostnamectl
ssh using root
```

modifier le fichier de configuration ssh de la machine cliente  
`nano ~/.ssh/config`

```bash
host r303-deb12-host1
  Hostname 192.168.122.10
  User root

host r303-deb12-bind1
  Hostname 192.168.122.11
  User root

host r303-deb12-bind2
  Hostname 192.168.122.12
  User root
```

maintenant les vms sont accessibles comme ceci

```bash
ssh r303-deb12-host1
ssh r303-deb12-bind1
ssh r303-deb12-bind2
```

# serveur bind1

domaine qui sera utilisé : `rzo.lan`

## installation

*déjà changé nom d'hôte*  
*ip fixes déjà mises sur les vms*

```bash
apt install -y dbus  # installé normalement
apt install -y bind9* dnsutils
```
<!-- apt install -y bind9 bind9-doc dnsutils -->

## configuration

### ACLs bind

*toute la suite je n'ai pas compris*

définissent quels hôtes ou utilisateurs peuvent effectuer quelles opérations sur le serveur dns

tout se passe dans `/etc/bind/named.conf`

```conf
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.mes-zones"; // fichier pour la gestion des zones
include "/etc/bind/named.conf.logs"; // fichier pour la gestion des logs

acl lan { 192.168.122.0/24; }; // acl "réseau sur lequel vous êtes"
acl ecoute { 192.168.122.11; }; // serveur dns que j'écoute?
acl interne { 127.0.0.0/8; }; // acl "boucle locale"
```

dans le fichier de gestion des logs `/etc/bind/named.conf.logs`  

```conf
logging {
    channel bind_log {
        file "/var/log/bind/bind.log";
        severity info;
        print-category yes;
        print-severity yes;
        print-time yes;
    };
    category default { bind_log; };
    category update { bind_log; };
    category update-security { bind_log; };
    category security { bind_log; };
    category queries { bind_log; };
    category lame-servers { null; };
};
```

> *grand charabia pour dire que les types de logs renseignés vont dans `/var/log/bind/bind.log` selon une certaine syntaxe sauf un exclu*

dans le fichier de gestion des zones `/etc/bind/named.conf.mes-zones`

```conf
zone "rzo.lan" IN {
  type master;
  file "/etc/bind/db.nom";
};
  
zone "122.168.192.in-addr.arpa" {
  type master;
  file "/etc/bind/db.inverse";
};
```
> *type master: cette vm est le dns primaire/maitre de la zone*  
*le fichier de gestion des zones fait une référence au fichier `/etc/bind/db.nom` pour les records (RR - *ressource record*)*

***ajouté zone de résolution inverse***

`nano /etc/bind/db.nom`

***à me pencher dessus***

```conf
$TTL 1d

@ IN SOA localhost. admin.zone. (
1 ; serial
6H ; refresh
3H ; retry
12H ; expire
3H ) ; minimum

@ IN NS localhost.
@ IN A 127.0.0.1
@ IN AAAA ::1

$ORIGIN zone

FQDN IN A 192.168.122.11

bind2 IN A 192.168.122.12
www IN CNAME bind2
```

vérifier la syntaxe des fichiers  
`named-checkconf /etc/bind/named.conf`  
`named-checkzone zone.local /etc/bind/db.nom`


# tester sa configuration dns

*vérifier sa configuration dns dans `/etc/resolv.conf`*

tester un domaine  
`dig domain.tld`

connaitre les machines gérant le domaine  
`dig NS domain.tld`

résoudre un nom  
`dig sub-domain.domain.tld`


mkdir /var/log/bind  
touch /var/log/bind/bind.log

pb qu'ils ont eu avec apparmor

`nano /etc/apparmor.d/usr.sbin.named`
```bash
/var/log/bind** rw,
```
puis
`systemctl restart apparmor && systemctl restart bind9`