# Configuration DNS

## DNS Primaire

Cloner le conteneur template (le stopper avant) :

```
lxc-copy -n template -N dns
```

Configurer l'adresse IP dans `/etc/network/interfaces` :

![Fichier interfaces](images/Capture%20d%27%C3%A9cran%202026-04-08%20202023.png)

Installer bind9 :

```
apt update
apt upgrade
apt install bind9 bind9utils dnsutils
```

Aller dans `/etc/bind`

---

Configurer `named.conf.local` :

```
zone "m2l.org" IN {
 type master;
 file "/etc/bind/db.m2l.org";
 allow-transfer { localhost; 10.31.16.254; };
 notify yes;
};
```

![Fichier named.conf.local](images/Capture%20d%27%C3%A9cran%202026-04-08%20202112.png)

---

Configurer `named.conf.options` :

```
options {
 directory "/var/cache/bind";
 recursion yes;
 forwarders {8.8.8.8; 8.8.4.4;};
 forward only;
 dnssec-validation no;
 listen-on-v6 { any; };
};
```

![Fichier named.conf.options](images/Capture%20d%27%C3%A9cran%202026-04-08%20202144.png)

---

Créer et configurer `db.m2l.org` dans `/etc/bind` :

```
$TTL 604800
@ IN SOA m2l.org. root.m2l.org. (
        2015122601 ; serial
        604800     ; Refresh
        86400      ; Retry
        2419200    ; Expire
        604800 )   ; Negative Cache TTL
; DNS Servers
@ IN A 10.31.16.53
@ IN NS ns1.m2l.org.
@ IN NS ns2.m2l.org.
ns1 IN A 10.31.16.53
ns2 IN A 10.31.16.54
; Machines
www IN A 10.31.16.80
; Aliases
console IN CNAME www
ftp    IN CNAME www
```

![Fichier db.m2l.org](images/Capture%20d%27%C3%A9cran%202026-04-08%20202156.png)

---

Redémarrer et vérifier bind9 :

```
systemctl restart bind9
systemctl status bind9
```

![Status bind9](images/Capture%20d%27%C3%A9cran%202026-04-08%20202228.png)

Aller dans le conteneur web et modifier le nameserver dans `/etc/resolv.conf` :

```
lxc-attach web
nano /etc/resolv.conf
```

![Fichier resolv.conf](images/Capture%20d%27%C3%A9cran%202026-04-08%20202319.png)

Tester avec un ping :

```
ping google.com
ping m2l.org
```

![Ping google.com et m2l.org](images/Capture%20d%27%C3%A9cran%202026-04-08%20202355.png)

---

## DNS Esclave

Cloner le dns primaire :

```
lxc-copy -n dns1 -N dns2
```

Configurer l'IP dans `/etc/network/interfaces` :

![Fichier interfaces dns2](images/Capture%20d%27%C3%A9cran%202026-04-08%20202422.png)

Modifier `/etc/bind/named.conf.local` :

```
nano /etc/bind/named.conf.local
```

![Fichier named.conf.local esclave](images/Capture%20d%27%C3%A9cran%202026-04-08%20202449.png)

Supprimer le fichier de zone (le DNS esclave le récupère automatiquement depuis le primaire) :

```
rm /etc/bind/db.m2l.org
```
