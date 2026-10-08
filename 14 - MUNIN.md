# Application : Installation et Configuration de Munin

Ce document détaille la mise en place d'un serveur de supervision Munin, la configuration des nœuds clients (conteneurs LXC), l'activation des plugins et l'interface web.

## Sommaire

1. [Installation et vérifications](#1-installation-et-vérifications)
2. [Configuration du serveur Munin](#2-configuration-du-serveur-munin)
3. [Configuration des clients (nœuds)](#3-configuration-des-clients-nœuds)
4. [Gestion des plugins](#4-gestion-des-plugins)
5. [Interface Web et génération des graphiques](#5-interface-web-et-génération-des-graphiques)
6. [Dépannage](#6-dépannage)

---

## 1. Installation et vérifications

Munin est une solution puissante pour créer des graphiques à travers le réseau, tout en maintenant une installation simple.

**Installation sur le serveur Munin :**

```bash
apt-get install munin munin-node munin-plugins-extra
```

**Installation sur les clients (à surveiller) :**

```bash
apt-get install munin-node munin-plugins-extra
```

**Vérification du service sur le serveur :**

```bash
systemctl status munin-node
```

**Identification du port d'écoute (4949) :**

```bash
netstat -nat | grep 4949
```

### Capture d'écran : conteneurs LXC disponibles

Voici la liste des conteneurs présents sur l'infrastructure :

<details>
<summary>Afficher la capture d'écran</summary>

![Conteneurs LXC disponibles](images/munin-lxc-ls.png)

</details>

```text
root@srV-g16:~# lxc-ls
Nginx   Nginx1   backup   backup1   dhcp   dhcp1   dns1   dns2
munin   munin1   profftp  template  test_dhcp_web1  web2
```

### Détail des paramètres de configuration serveur

La configuration principale du serveur se trouve dans `/etc/munin/munin.conf`. Voici les directives clés :

| Directive | Description |
|---|---|
| `dbdir` | Répertoire des bases de données (`/var/lib/munin`) |
| `htmldir` | Répertoire de sortie des pages HTML (`/var/cache/munin/www`) |
| `logdir` | Répertoire des logs (`/var/log/munin`) |
| `rundir` | Répertoire des fichiers PID (`/var/run/munin`) |
| `includedir` | Inclusion de fichiers de configuration supplémentaires |
| `address` | Adresse IP du nœud surveillé |
| `use_node_name` | Utilise le nom du nœud comme nom d'hôte |

---

## 2. Configuration du serveur Munin

**Objectif :** définir les nœuds à superviser dans le fichier `munin.conf`.

- **Fichier de configuration :** `/etc/munin/munin.conf`
- **Nœuds à superviser :** l'infrastructure contient les conteneurs listés ci-dessous.

### Fichier `munin.conf` personnalisé

Voici le fichier de configuration utilisé pour l'infrastructure :

<details>
<summary>Afficher la capture d'écran</summary>

![Fichier munin.conf](images/munin-conf.png)

</details>

```ini
# /etc/munin/munin.conf

timeout fetch one node 180

# Arborescence simple des hôtes

[munin]
    address 127.0.0.1
    use_node_name yes

[serveru]
    address 10.31.16.1
    use_node_name yes

[web1]
    address 10.31.16.80
    use_node_name yes

[web2]
    address 10.31.16.81
    use_node_name yes

[dns1]
    address 10.31.16.53
    use_node_name yes

[dns2]
    address 10.31.16.54
    use_node_name yes

[profftp]
    address 10.31.16.21
    use_node_name yes

[backup1]
    address 10.31.16.98
    use_node_name yes

[backup2]
    address 10.31.16.99
    use_node_name yes

[dhcp]
    address 10.31.16.67
    use_node_name yes

[ngnix]
    address 10.31.16.90
    use_node_name yes

[ngnix1]
    address 10.31.16.91
    use_node_name yes

[template]
    address 10.31.16.2
    use_node_name yes
```

### Lancement automatique (cron)

Vérifier `/etc/cron.d/munin` :

```cron
*/5 * * * * munin if [ -x /usr/bin/munin-cron ]; then /usr/bin/munin-cron; fi
```

---

## 3. Configuration des clients (nœuds)

Chaque conteneur à superviser doit avoir un `munin-node` configuré.

**Fichier client :** `/etc/munin/munin-node.conf`

### Exemple de configuration client pour web1

```text
# /etc/munin/munin-node.conf (sur web1)
host_name web1
allow ^10\.31\.16\.1$   # Autorise le serveur Munin (serveru)
host *
```

Redémarrage du service sur le client :

```bash
systemctl restart munin-node
```

### Liste des adresses IP des conteneurs

| Conteneur | Adresse IP |
|---|---|
| munin (serveur) | 127.0.0.1 / 10.31.16.1 |
| web1 | 10.31.16.80 |
| web2 | 10.31.16.81 |
| dns1 | 10.31.16.53 |
| dns2 | 10.31.16.54 |
| profftp | 10.31.16.21 |
| backup1 | 10.31.16.98 |
| backup2 | 10.31.16.99 |
| dhcp | 10.31.16.67 |
| Ngnix | 10.31.16.90 |
| Ngnix1 | 10.31.16.91 |
| Template | 10.31.16.2 |

---

## 4. Gestion des plugins

Les données sont récoltées par des plugins. Ils se trouvent dans `/usr/share/munin/plugins` et sont activés par un lien symbolique dans `/etc/munin/plugins`.

Voir les plugins disponibles :

```bash
munin-node-configure | grep yes
```

Activer un plugin (exemple : CPU et interface réseau) :

```bash
cd /etc/munin/plugins
ln -s /usr/share/munin/plugins/cpu .
ln -s /usr/share/munin/plugins/if_ if_enp0s3
systemctl restart munin-node
```

### Galerie de plugins

Consultez la galerie officielle pour découvrir tous les plugins disponibles : [Galerie Munin](https://gallery.munin-monitoring.org/)

---

## 5. Interface Web et génération des graphiques

Par défaut, Munin génère des pages HTML toutes les 5 minutes dans `/var/cache/munin/www`.

Génération manuelle pour test :

```bash
su - munin --shell=/bin/bash
/usr/bin/munin-cron
```

**Configuration VirtualHost :** créez un VirtualHost pointant vers `/var/cache/munin/www` pour accéder à l'interface web.

---

## 6. Dépannage

Vérification des logs :

```bash
tail -f /var/log/munin/munin-node.log
```

### Erreur fréquente : connexion refusée

```text
[4418] Denying connection from: 10.0.0.2
```

**Solution :** ajouter l'IP dans la directive `allow` du fichier `munin-node.conf` du client.

**Absence de graphiques :** consultez la FAQ officielle ("no graphs") de Munin.

Redémarrer cron si nécessaire :

```bash
/etc/init.d/cron restart
```
