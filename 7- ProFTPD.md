# Installation et Configuration de ProFTPD

Ce document détaille la mise en place d'un serveur FTP, la gestion des accès anonymes, des hôtes virtuels et l'analyse de trafic.

---

## 1. Installation et vérifications

Installation du paquet :

```
apt-get install proftpd
```

Vérification du service :

```
systemctl status proftpd
```

Identification du port d'écoute :

```
netstat -natp | grep proftpd
```

![Commandes initiales](images/Capture%d'écran%2026-04-08%211727.png)

La configuration principale se trouve dans `/etc/proftpd/proftpd.conf`. Voici les directives clés :

| Directive | Description |
| :--- | :--- |
| `Include /etc/proftpd/modules.conf` | Appelle les modules complémentaires |
| `ServerName "Serveur Partage"` | Nom du serveur vu par le client |
| `ServerType standalone` | Mode autonome, meilleures performances |
| `DeferWelcome on` | Message d'accueil après authentification uniquement |
| `DefaultServer on` | Gère les requêtes des machines non référencées |
| `TimeoutNoTransfer 600` | Temps max sans transfert en secondes |
| `DefaultRoot ~` | Emprisonne l'utilisateur dans son home (chroot) |
| `Port 21` | Port de contrôle (transferts sur le port 20) |
| `MaxInstances 30` | Limite les connexions simultanées |
| `AllowOverwrite on` | Autorise l'écrasement des fichiers existants |

Test de connexion avec **FileZilla** sur le compte `std` port 21. Si besoin, réinitialiser le mot de passe :

```
passwd std
```

---

## 2. Configuration de l'accès anonyme

Objectif : accès en lecture seule via le compte `anonymous` ou `ftp` dans un répertoire dédié.

Répertoire racine : `/home/ftpdocs`

Dans `/etc/proftpd/proftpd.conf`, remplacer :

```
<Anonymous ~ftp>
```

par :

```
<Anonymous /home/ftpdocs>
```

![proftpd.conf anonyme 1](images/photo-ftp-2.png)
![proftpd.conf anonyme 2](images/photo-ftp-3.png)
![proftpd.conf anonyme 3](images/photo-ftp-4.png)
![proftpd.conf anonyme 4](images/photo-ftp-5.png)

---

## 3. VirtualHosts — Intranet & Extranet

**A. Création des utilisateurs**

```
useradd intra
useradd extra
passwd intra
passwd extra
```

**B. Configuration réseau — alias IP**

```
ifconfig eth0:0 10.31.16.22/20 up
ifconfig eth0:1 10.31.16.23/20 up
```

**C. Fichier `virtuals.conf`**

Éditer `/etc/proftpd/virtuals.conf` (vérifier qu'il est inclus dans le fichier principal).

![virtuals.conf](images/photo-ftp-6.png)

Deux VirtualHosts à configurer :
- `intra` → IP `10.31.16.22`, port `2100`, répertoire `/srv/ftp/intranet`
- `extra` → IP `10.31.16.23`, port `2200`, répertoire `/srv/ftp/extranet`

```
<VirtualHost 10.31.16.22>
    ServerName "FTP INTRANET"
    Port 2100
    DefaultRoot /srv/ftp/intranet
    <Limit LOGIN>
        AllowGroup intra
        DenyAll
    </Limit>
</VirtualHost>

<VirtualHost 10.31.16.23>
    ServerName "FTP EXTRANET"
    Port 2200
    DefaultRoot /srv/ftp/extranet
    <Limit LOGIN>
        AllowGroup extra
        DenyAll
    </Limit>
</VirtualHost>
```

---

## 4. Analyse réseau avec Wireshark

Le FTP circule en clair — capturer les échanges lors d'une connexion pour observer le processus d'authentification (`USER` / `PASS`).

![Capture Wireshark](images/photo-ftp-7.png)
