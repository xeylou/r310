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

modifications des IPs dans `/etc/network/interfaces`

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

<!-- ajouter dans `/etc/hosts` des vms

```bash
192.168.122.10 r303-deb12-host1
192.168.122.10 r303-deb12-host1.rzo.lan
192.168.122.11 r303-deb12-bind1
192.168.122.11 rzo.lan
192.168.122.12 r303-deb12-bind2
192.168.122.12 r303-deb12-bind2.rzo.lan
``` -->

modification de la configuration de ssh sur les vms pour se connecter au compte root  
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

modifier le fichier `~/.ssh/config` de la machine cliente  


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

accès aux vms

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
apt install -y bind9* dnsutils # bind9-docs bind9-utils
```
<!-- apt install -y bind9 bind9-doc dnsutils -->

## configuration

dans le fichier de gestion des zones `/etc/bind/named.conf` on définit notre zone & sa zone inverse

```conf
zone "rzo.lan" IN {
  type master;
  file "/etc/bind/rzo.lan";
};
  
zone "122.168.192.in-addr.arpa" {
  type master;
  file "/etc/bind/rzo.lan.inverse";
};
```
> *type master: cette vm est le dns primaire/maitre de la zone*  
> on fait référence à des fichiers qui seront la configuration des zones dns

faire `named-checkconf` ou `named-checkconf /etc/bind/named.conf`

`nano /etc/bind/rzo.lan`

```conf
$TTL 86400

@ IN SOA r303-deb12-bind1.rzo.lan. admin.rzo.lan. (
2023092301 ; serial
21600 ; refresh
10800 ; retry
43200 ; expire
10800 ) ; minimum

@ IN NS r303-deb12-bind1.
@ IN NS r303-deb12-bind2.
r303-deb12-bind1 IN A 192.168.122.11
r303-deb12-bind2 IN A 192.168.122.12
```

`nano /etc/bind/rzo.lan.inverse`

```conf
$TTL 86400

@ IN SOA r303-deb12-bind1.rzo.lan. admin.rzo.lan. (
2023092301 ; serial
21600 ; refresh
10800 ; retry
43200 ; expire
10800 ) ; minimum

@ IN NS r303-deb12-bind1.
@ IN NS r303-deb12-bind2.
11 IN PTR r303-deb12-bind1
12 IN PTR r303-deb12-bind2
```

`named-checkzone rzo.lan /etc/bind/rzo.lan`
`named-checkzone rzo.lan.inverse /etc/bind/rzo.lan.inverse`

`systemctl restart bind; systemctl restart named`

maintenant sur la machine r303-deb12-host1

modifier `/etc/resolv.conf`

```bash
nameserver 192.168.122.11
```

tester un domaine  
`dig domain.tld`

connaitre les machines gérant le domaine  
`dig NS domain.tld`

résoudre un nom  
`dig sub-domain.domain.tld`

`nslookup 192.168.122.11`
`nslookup 192.168.122.12`





















mkdir /var/log/bind  
touch /var/log/bind/bind.log

pb qu'ils ont eu avec apparmor

`nano /etc/apparmor.d/usr.sbin.named`
```bash
/var/log/bind** rw,
```
puis
`systemctl restart apparmor && systemctl restart bind9`