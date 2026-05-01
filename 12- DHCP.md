# Configuration du Service DHCP

## 1. Installation du service

Mise à jour des dépôts puis installation :

```
apt update
apt install isc-dhcp-server
```

---

## 2. Configuration du serveur

Ajouter cette configuration à la fin du fichier `/etc/dhcp/dhcpd.conf` :

<details>
<summary>Contenu du fichier dhcpd.conf (cliquer pour voir)</summary>

```
# Définition d'un sous-réseau à gérer
subnet 10.31.16.0 netmask 255.255.255.0 {
        # Plage d'adresses dynamiquement allouées
        range 10.31.16.2 10.31.16.99;
        # Passerelle par défaut
        option routers 10.31.31.254;
        # Adresse de broadcast
        option broadcast-address 10.31.16.255;
        # Serveurs de noms
        option domain-name-servers 8.8.8.8;
        # Nom de domaine
        option domain-name "m2l.org";
        # Durée du bail par défaut en secondes
        default-lease-time 172800;
        # Durée maximale du bail accordé à un client en secondes
        max-lease-time 604400;
        # Réservation d'adresses (Static Leases)
        group {
                use-host-decl-names true;
                host backup {
                        hardware ethernet 10:66:6a:86:8f:a5;
                        fixed-address 10.31.16.98;
                }
                host dns2 {
                        hardware ethernet 10:66:6a:ae:6b:59;
                        fixed-address 10.31.16.54;
                }
                host web2 {
                        hardware ethernet 10:66:6a:40:18:b8;
                        fixed-address 10.31.16.81;
                }
        }
}
```

</details>

Redémarrer le service :

```
systemctl restart isc-dhcp-server
```

---

## 3. Tests et vérification

Sur un conteneur du même réseau, tester la connectivité.

Libérer l'IP actuelle :

```
dhclient -r
```

Demander une nouvelle IP en mode verbeux :

```
dhclient -v
```

---

## 4. Gestion avancée des logs

Il est recommandé d'isoler les logs DHCP pour faciliter le debug.

**Dans `/etc/dhcp/dhcpd.conf`**, ajouter :

```
log-facility local7;
```

**Créer le fichier de log et définir les droits :**

```
touch /var/log/isc-dhcpd.log
chown root:adm /var/log/isc-dhcpd.log
chmod 0640 /var/log/isc-dhcpd.log
```

**Dans `/etc/rsyslog.conf`**, ajouter :

```
local7.* /var/log/isc-dhcpd.log
```

Puis modifier la ligne de gestion du syslog général :

- Ancien : `*.*;auth,authpriv.none -/var/log/syslog`
- Nouveau : `*.*;auth,authpriv.none;local7.none -/var/log/syslog`

---

## 5. Finalisation

Appliquer les changements en redémarrant les démons :

```
systemctl restart rsyslog
systemctl restart isc-dhcp-server
```
