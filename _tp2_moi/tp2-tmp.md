vms utilisées
> r303-deb12-postfix  
r303-deb12-dovecot  
r303-deb12-bind3

# postfix
`apt install -y postfix mailutils # Internet Site -> rzo.lan`  

`systemctl status postfix`  

`telnet localhost 25`  

`nano /etc/postfix/main.cf`  
*commenter tout ce qui touche au tls + quelques modifs*
```conf
# TLS parameters
# smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
# smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
# smtpd_tls_security_level=may

# smtp_tls_CApath=/etc/ssl/certs
# smtp_tls_security_level=may
# smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

mydomain = rzo.lan
myhostname = r303-deb12-postfix.rzo.lan
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
myorigin = /etc/mailname
mydestination = $mydomain, r303-deb12-postfix.$mydomain, localhost.$mydomain, localhost
default_transport = smtp
# relayhost =
mynetworks = 127.0.0.0/8 192.168.122.0/24
home_mailbox = Maildir/
mailbox_size_limit = 51200000
recipient_delimiter = +
inet_interfaces = all
inet_protocols = ipv4
```
`postfix check`

`systemctl restart postfix`

`adduser user1 && adduser user2`

`su user1`  
`mail user2`  
> Cc: *Entrer*  
Subject: intitule  
contenu du mail  

>*CTRL + D*

`su user2`
> `ls Maildir/new`
# dovecot

`apt install -y dovecot-imapd`  

`nano /etc/dovecot/conf.d/10-auth.conf`  
*variables à modifier*
```conf
disable_plaintext_auth = no
auth_mechanisms = login plain
```
`nano /etc/dovecot/conf.d/10-mail.conf`
```conf
mail_location = maildir:~/Maildir
```
`nano /etc/dovecot/conf.d/10-logging.conf`
```conf
log_path = syslog
syslog_facility = mail
auth_verbose = yes
auth_debug = yes
```
`systemctl restart dovecot`  
`telnet -l user1 192.168.122.20 143`
> `a login user1 user1`  

<!-- > Trying 192.168.122.20...  
telnet: Unable to connect to remote host: Connection refused

`systemctl enable --now nftables`

`iptables -A INPUT -p tcp -s 192.168.122.21 --dport 143 -j ACCEPT` -->

*sur postix*

`mkdir /opt/messagerie`
`groupadd -g 5000 vmail`
`useradd -g vmail -u 5000 vmail -d /opt/messagerie -m`
`chown -R vmail:vmail /opt/messagerie`

`nano /etc/postfix/main.cf`
```conf
mydomain = rzo.lan
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated de>
myhostname = r303-deb12-postfix.rzo.lan
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
myorigin = /etc/mailname
mydestination = $mydomain, r303-deb12-postfix.rzo.lan, localhost.rzo.lan,>
default_transport = smtp
# relayhost = 
mynetworks = 127.0.0.0/8 192.168.122.0/24
home_mailbox = Maildir/
mailbox_size_limit = 51200000
# changement à partir d'ici
virtual_transport = dovecot
mail_spool_directory = /opt/messagerie/
virtual_mailbox_base = /opt/messagerie/
virtual_mailbox_domains = hash:/etc/postfix/vdomain
virtual_mailbox_maps = hash:/etc/postfix/vmail
virtual_alias_maps = hash:/etc/postfix/valias
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000
# fin des changements
recipient_delimiter = +
inet_interfaces = all
inet_protocols = ipv4
```
`postifx check`

`nano /etc/postfix/vdomain`
```conf
rzo.lan #
```
`nano /etc/postfix/vmail`
```conf
loulou@rzo.lan rzo.lan/loulou/
mimi@rzo.lan rzo.lan/mimi/
admin@rzo.lan rzo.lan/admin/
```
`nano /etc/postfix/valias`
```conf
root: admin@rzo.lan
```

`nano /etc/postfix/master.cf`
*à sa fin*
```conf
dovecot unix - n n - - pipe
  flags=DRhu user=vmail:vmail argv=/usr/lib/dovecot/deliver -f ${sender} -d ${recipient}
```

`postmap /etc/postfix/vdomain`  
`postmap /etc/postfix/vmail`  
`postalias /etc/postfix/valias`  
`postfix check`  

`nano /etc/dovecot/conf.d/10-auth.conf`
```conf
disable_plaintext_auth = yes
auth_mechanisms = cram-md5 login plain
!include auth-static.conf.ext
```
`nano /etc/dovecot/conf.d/auth-static.conf.ext`
```conf
passdb {
#  driver = static
#  args = proxy=y host=%1Mu.example.com nopassword=y
driver = passwd-file
args = username_format=%u /etc/dovecot/dovecot.users
}

userdb {
driver = static
args = uid=vmail gid=vmail home=/opt/messagerie/%d/%n/ allow_all_users=yes
}
```
`nano /etc/dovecot/conf.d/auth-static.conf.ext`
```conf
mail_uid = 5000
mail_gid = 5000
# [...]
mail_privileged_group = vmail
```
`nano /etc/dovecot/conf.d/10-master.conf`
```conf
# [...]
service auth {
    # [...]
    unix_listener auth-userdb {
        mode = 0666
        user = postfix
        group = postfix
    }

    unix_listener /var/spool/postfix/private/auth {
        mode = 0666
        user = postfix
        group = postfix
    }
    # [...]
}
# [...]
```
`doveadm pw -s CRAM-MD5`

*le copier ({CRAM-MD5}243d3b6dc0f181054afdab6cca700ef1e3db95c0fd03e795b006016b537ac504)*

`nano /etc/dovecot/dovecot.users`
```conf
loulou@rzo.lan:{CRAM-MD5}243d3b6dc0f181054afdab6cca700ef1e3db95c0fd03e795b006016b537ac504
mimi...
admin...
```
`systemctl restart postfix`  
`systemctl restart dovecot`