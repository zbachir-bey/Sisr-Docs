# Mise en place de conteneurs LXC

Se connecter au serveur en SSH :

```
ssh root@10.31.16.1
```

Installer et mettre à jour LXC :

```
apt-get update
apt-get upgrade
apt-get install lxc
```

Vérifier la configuration :

```
lxc-checkconfig
```

---

## Création du pont (bridge)

Installer bridge-utils :

```
apt-get install bridge-utils
```

Créer un premier pont :

```
brctl addbr br0
```

Vérifications :

```
brctl show
ifconfig
```

Configurer l'adresse IP en ajoutant les commandes suivantes dans `/etc/rc.local` :

```
#!/bin/bash
brctl addbr br0
ifconfig eno1 0.0.0.0
ifconfig eth0 10.31.16.1/20 up
route add default gw 10.31.31.254
brctl addif br0 eno1
```

![Configuration du fichier rc.local](images/Capture%20d%27%C3%A9cran%202026-04-07%20174726.png)

Vérifier la création du pont :

```
brctl show
```

Vérifier la configuration réseau :

```
ifconfig
```

Configuration par défaut des conteneurs dans `/etc/lxc/default.conf` :

```
lxc.net.0.type = veth
lxc.net.0.link = br0
lxc.net.0.flags = up
lxc.net.0.name = eth0
lxc.apparmor.profile = generated
lxc.apparmor.allow_nesting = 1
```

---

## Création du conteneur template

```
lxc-create -n template -t debian -- -r bookworm
```

### Commandes utiles

| Commande | Description |
| :--- | :--- |
| `lxc-start -n conteneur` | Démarrer un conteneur |
| `lxc-stop -n conteneur` | Arrêter un conteneur |
| `lxc-copy -n source -N destination` | Cloner un conteneur |
| `lxc-destroy -n conteneur` | Supprimer définitivement un conteneur |
| `lxc-checkconfig` | Vérifier la compatibilité du kernel avec LXC |
| `lxc-attach -n conteneur` | Entrer dans un conteneur en cours d'exécution |
| `lxc-info -n conteneur` | Afficher le statut détaillé d'un conteneur |
| `lxc-ls -f` | Lister tous les conteneurs avec leur état et IP |

Afficher la liste des conteneurs :

```
lxc-ls
```

Afficher les infos du conteneur template :

```
lxc-info template
```

Démarrer le conteneur template :

```
lxc-start template
lxc-info template
```

**Démarrage automatique** — ajouter `lxc.start.auto = 1` dans `/var/lib/lxc/template/config` :

```
lxc.net.0.type = veth
lxc.net.0.hwaddr = 00:16:3e:11:85:c7
lxc.net.0.link = br0
lxc.net.0.flags = up
lxc.net.0.name = eth0
lxc.apparmor.profile = generated
lxc.apparmor.allow_nesting = 1
lxc.rootfs.path = dir:/var/lib/lxc/template/rootfs
lxc.include = /usr/share/lxc/config/debian.common.conf
lxc.tty.max = 4
lxc.uts.name = template
lxc.arch = amd64
lxc.pty.max = 1024
lxc.start.auto = 1
```

Accéder au conteneur template :

```
lxc-attach template
```

Configuration IP temporaire du conteneur :

```
ifconfig eth0 10.31.16.2/20 up
route add default gw 10.31.31.254
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Ou directement dans `/etc/network/interfaces` :

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
        address 10.31.16.2/20
        gateway 10.31.31.254
        dns-nameservers 8.8.8.8
```

Mise à jour et installation des outils de base :

```
apt update
apt upgrade
apt install sudo net-tools tcpdump nano iputils-ping dbus
```

Configurer la timezone :

```
ln -fs /usr/share/zoneinfo/Europe/Paris /etc/localtime
dpkg-reconfigure -f noninteractive tzdata
```

Créer un utilisateur sudo :

```
adduser std
usermod -aG sudo std
```

> ⚠️ **Ne pas oublier d'ajouter les clés SSH dans le template** — fichier : `/root/.ssh/authorized_keys`

---

## Clonage du template

Le template sert uniquement de modèle. On l'arrête définitivement après configuration.

Arrêt et clonage :

```
lxc-stop template
lxc-copy -n template -N web
```

Démarrage et accès au conteneur web :

```
lxc-start web
lxc-attach web
```

Modifier l'adresse IP dans `/etc/network/interfaces` :

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
        address 10.31.16.80/20
        gateway 10.31.31.254
        dns-nameservers 8.8.8.8
```

Arrêter et redémarrer le conteneur web, puis installer Apache :

```
apt update
apt install apache2
```
