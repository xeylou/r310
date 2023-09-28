```conf
$TTL 1d

@ IN SOA r303-deb12-bind1.rzo.lan. admin.rzo.lan. (
1 ; serial
6H ; refresh
3H ; retry
12H ; expire
3H ) ; minimum

@ IN NS r303-deb12-bind1.rzo.lan.
@ IN A 192.168.122.11
rzo.lan. IN A 192.168.122.11
rzo.lan. IN MX 10 192.168.122.11
www IN CNAME rzo.lan.
bind2 IN A 192.168.122.12
host1 IN A 192.168.122.10
```