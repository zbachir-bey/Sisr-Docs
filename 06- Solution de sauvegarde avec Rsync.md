# Solution de sauvegarde avec Rsync

Cette documentation détaille la mise en place d'un serveur de sauvegarde centralisé pour les conteneurs LXC.

---

## 1. Préparation de l'environnement

Créer le conteneur de sauvegarde :

```
lxc-copy -n template -N backup
```

Configurer l'adresse IP dans `/etc/network/interfaces` du nouveau conteneur.

Installer Rsync sur toutes les machines (source et destination) :

```
apt update && apt install rsync -y
```

---

## 2. Gestion des utilisateurs et sécurité SSH

**Sur le serveur de sauvegarde (backup)**

Créer l'utilisateur `backup1` puis générer une paire de clés SSH :

```
adduser backup1
su - backup1
ssh-keygen -t rsa
```

La clé publique générée doit être copiée dans le fichier `authorized_keys` des conteneurs cibles. Voir la doc [Configuration SSH](02-%20Configuration%20SSH.md) pour les détails.

**Sur les conteneurs à sauvegarder (sources)**

Créer l'utilisateur `backup1` sur chaque conteneur cible et s'assurer que la clé publique du serveur de sauvegarde est présente dans son dossier `.ssh` :

```
adduser backup1
su - backup1
```

> La sauvegarde fonctionne en mode **PULL** : c'est le serveur de sauvegarde qui tire les données depuis les sources.

---

## 3. Test de la commande de sauvegarde

Exemple de commande manuelle depuis le conteneur backup :

```
rsync -azv -e ssh backup1@10.31.16.80:/etc/apache2 /home/backup1/web1
```

Si le nom de domaine (`www.m2l.org`) ne fonctionne pas, utiliser directement l'adresse IP du conteneur.

---

## 4. Automatisation avec script et Cron

**Création du script**

Créer un dossier `logs` puis le fichier `save.sh` dans le répertoire de `backup1` :

```
#!/bin/bash
DATE=$(date "+%Y-%m-%d")
LOGFILE=/home/backup1/logs/backup1_"$DATE".log

# Sauvegarde Web1
rsync -azv -e ssh --rsync-path="sudo rsync" backup1@10.31.16.80:/etc/apache2 /home/backup1/web1 >>$LOGFILE
rsync -azv -e ssh --rsync-path="sudo rsync" backup1@10.31.16.80:/etc/mysql /home/backup1/web1 >>$LOGFILE
rsync -azv -e ssh --rsync-path="sudo rsync" backup1@10.31.16.80:/etc/network/interfaces /home/backup1/web1 >>$LOGFILE
rsync -azv -e ssh --rsync-path="sudo rsync" backup1@10.31.16.80:/home/htdocs /home/backup1/web1 >>$LOGFILE

# Sauvegarde Dns1
rsync -azv -e ssh --rsync-path="sudo rsync" backup1@10.31.16.53:/etc/bind/named.conf.local /home/backup1/dns1 >>$LOGFILE
rsync -azv -e ssh --rsync-path="sudo rsync" backup1@10.31.16.53:/etc/bind/named.conf.options /home/backup1/dns1 >>$LOGFILE
rsync -azv -e ssh --rsync-path="sudo rsync" backup1@10.31.16.53:/etc/bind/db.m2l.org /home/backup1/dns1 >>$LOGFILE

# Sauvegarde Proftp
rsync -azv -e ssh --rsync-path="sudo rsync" backup1@10.31.16.21:/etc/network/interfaces /home/backup1/proftp >>$LOGFILE
rsync -azv -e ssh --rsync-path="sudo rsync" backup1@10.31.16.21:/etc/proftpd /home/backup1/proftp >>$LOGFILE
```

Rendre le script exécutable et le tester :

```
chmod +x /home/backup1/save.sh
./home/backup1/save.sh
```

**Planification avec Cron**

Ouvrir le crontab en tant que `backup1` :

```
crontab -e
```

Ajouter la ligne suivante pour une exécution à 07h30 du lundi au samedi :

```
30 7 * * 1-6 /home/backup1/save.sh
```

Pour vérifier ou modifier la fréquence : https://crontab.guru
