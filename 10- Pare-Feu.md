# Configuration du Pare-Feu

Le pare-feu s'installe sur le routeur. Se connecter en SSH :

```
ssh groupe16@172.31.16.254
```

Une fois connecté, créer un script de configuration permanent. La meilleure option est un fichier `.sh`.

---

## Fichier `firewall.sh`

```
#!/bin/bash

# Vider les règles existantes
iptables -F

# Bloquer tout par défaut
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP

# Stateful — autoriser les retours
iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT

# SSH vers le routeur
iptables -A INPUT -p tcp -s 10.187.20.0/24 --dport 22 -j ACCEPT

# SSH vers le serveur .1
iptables -A FORWARD -p tcp -s 10.187.20.0/24 -d 10.31.16.1 --dport 22 -j ACCEPT

# Accès au site web port 80 depuis Beaupeyrat
iptables -A FORWARD -p tcp -s 10.187.20.0/24 -d 10.31.16.80 --dport 80 -j ACCEPT

# Serveur FTP accès réseau
iptables -A FORWARD -p tcp -s 10.31.16.21 -d 10.31.16.0 --dport 21 -j ACCEPT
iptables -A FORWARD -p tcp -s 10.187.20.0 -d 10.31.16.0 --dport 21 -j ACCEPT

# DNS — accès à 8.8.8.8 et internet
iptables -A FORWARD -p udp -s 10.31.16.53 -d 8.8.8.8 --dport 53 -j ACCEPT
iptables -A FORWARD -p tcp -s 10.31.16.53 -m multiport --dports 80,443 -j ACCEPT

# SSH depuis Beaupeyrat vers les conteneurs et le serveur
iptables -A INPUT -p tcp -s 10.187.20.0/24 -d 172.31.16.254 --dport 22 -j ACCEPT
iptables -A FORWARD -p tcp -s 10.187.20.0/24 -d 10.31.16.0/20 --dport 22 -j ACCEPT

# Accès internet pour toutes les machines
iptables -A FORWARD -p tcp -s 10.187.20.0/24 -d 10.31.16.0/20 -m multiport --dports 80,443,53,21,3303 -j ACCEPT

# Pings
iptables -A FORWARD -p icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
iptables -A OUTPUT -p icmp --icmp-type echo-request -j ACCEPT
```

![firewall.sh](images/Capture%20d%27%C3%A9cran%202026-04-08%20215742.png)

---

## Fichier `rc.local`

Ce fichier permet d'exécuter `firewall.sh` automatiquement à chaque démarrage du routeur :

```
#!/bin/sh -e
#
# rc.local

# Lancement du script de pare-feu au démarrage
/etc/firewall.sh

exit 0
```

![rc.local](images/Capture%20d%27%C3%A9cran%202026-04-08%20215913.png)
