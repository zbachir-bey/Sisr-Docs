# Application : Installation et Configuration de Nginx

Ce document détaille la mise en place complète du serveur web Nginx, la configuration des Virtual Hosts, la sécurisation SSL/TLS et l'authentification HTTP Basic pour l'intranet.

## Sommaire

1. [Installation et mise à jour de Nginx](#1-installation-et-mise-à-jour-de-nginx)
2. [Configuration des Virtual Hosts](#2-configuration-des-virtual-hosts)
3. [Mise en place du SSL/TLS](#3-mise-en-place-du-ssltls)
4. [Authentification HTTP pour l'intranet](#4-authentification-http-pour-lintranet)
5. [Vérification finale et dépannage](#5-vérification-finale-et-dépannage)
6. [Annexe : modèle générique et commandes rapides](#annexe--modèle-générique-et-commandes-rapides)

---

## 1. Installation et mise à jour de Nginx

Pour installer Nginx et vérifier son bon fonctionnement, exécutez les commandes suivantes.

Mise à jour du système :

```bash
apt-get update && apt-get upgrade -y
```

Installation de Nginx :

```bash
apt-get install -y nginx
```

Vérification du service :

```bash
systemctl status nginx
```

Vérification du port d'écoute (80) :

```bash
netstat -natp | grep :80
```

### Structure des répertoires de configuration

Avant de commencer, familiarisez-vous avec l'organisation des fichiers de Nginx :

```text
/etc/nginx/
├── nginx.conf            # fichier de configuration principal
├── sites-available/      # fichiers de conf de chaque site (inactifs)
│   ├── www.m2l.org.conf
│   ├── intranet.m2l.org.conf
│   ├── extranet.m2l.org.conf
│   └── wiki.m2l.org.conf
├── sites-enabled/        # liens symboliques vers sites-available/
│   └── (liens créés manuellement avec ln -s)
├── conf.d/
└── snippets/
```

### Vérification du fichier nginx.conf

Ouvrez le fichier principal et vérifiez qu'il inclut bien le répertoire `sites-enabled` :

```bash
nano /etc/nginx/nginx.conf
```

Dans la section `http { … }`, vérifiez la présence de la ligne suivante :

```nginx
include /etc/nginx/sites-enabled/*;
```

> [!IMPORTANT]
> Si la ligne `include` n'est pas présente, ajoutez-la à la fin du bloc `http { }`. Cette ligne permet à Nginx de charger automatiquement tous les fichiers de configuration présents dans `sites-enabled`.

---

## 2. Configuration des Virtual Hosts

**Objectif :** créer et activer quatre sites web (www, intranet, extranet, wiki) sur le même serveur.

### A. Création des répertoires web

```bash
mkdir -p /home/htdocs/m2l.org/www
mkdir -p /home/htdocs/m2l.org/intranet
mkdir -p /home/htdocs/m2l.org/extranet
mkdir -p /home/htdocs/m2l.org/wiki

# Donner les droits à l'utilisateur Nginx
chown -R www-data:www-data /home/htdocs/
chmod -R 755 /home/htdocs
```

### B. Fichiers de configuration des sites

Créez le premier Virtual Host pour le site principal :

```bash
nano /etc/nginx/sites-available/www.m2l.org.conf
```

#### Correspondance Apache2 / Nginx

Dans Nginx, l'équivalent du `<VirtualHost>` d'Apache est appelé un bloc `server` :

| Directive Apache2 | Équivalent Nginx |
|---|---|
| `<VirtualHost *:80>` | `server { listen 80; }` |
| `ServerName monsite.org` | `server_name monsite.org;` |
| `ServerAlias www.monsite.org` | `server_name monsite.org www.monsite.org;` |
| `DocumentRoot /var/www/html/site/` | `root /var/www/html/site;` |
| `ErrorLog …` | `error_log /var/log/nginx/site-error.log;` |
| `CustomLog … combined` | `access_log /var/log/nginx/site-access.log;` |
| `<Directory …> Require all granted` | `location / { try_files $uri $uri/ =404; }` |

#### Fichier de configuration `www.m2l.org.conf`

```nginx
server {
    listen 80;
    server_name www.m2l.org m2l.org;
    root /home/htdocs/m2l.org/www;
    index index.html index.htm;

    access_log /var/log/nginx/www.m2l.org-access.log;
    error_log /var/log/nginx/www.m2l.org-error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Copiez ce fichier et adaptez-le pour les trois autres sites (intranet, extranet, wiki).

### Activation des Virtual Hosts

Contrairement à Apache, Nginx n'a pas de commande `a2ensite`. L'activation se fait manuellement en créant des liens symboliques :

```bash
# Créer les liens symboliques pour activer les sites
ln -s /etc/nginx/sites-available/www.m2l.org.conf /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/intranet.m2l.org.conf /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/extranet.m2l.org.conf /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/wiki.m2l.org.conf /etc/nginx/sites-enabled/

# Supprimer le site par défaut (évite les conflits)
rm /etc/nginx/sites-enabled/default

# Vérifier la syntaxe AVANT de redémarrer
nginx -t

# Recharger Nginx
systemctl reload nginx
```

> [!TIP]
> Toujours exécuter `nginx -t` avant de recharger Nginx. Cette commande vérifie la syntaxe sans interrompre le service.

### C. Test des Virtual Hosts

Depuis un poste client, modifiez le fichier `hosts` pour inclure les noms de domaines avec la nouvelle IP, puis testez dans un navigateur :

- `http://www.m2l.org`
- `http://intranet.m2l.org`
- `http://extranet.m2l.org`
- `http://wiki.m2l.org`

---

## 3. Mise en place du SSL/TLS

**Objectif :** sécuriser les échanges avec des certificats (wildcard auto-signé pour tous les sous-domaines).

### A. Installation d'OpenSSL et création des certificats

```bash
# Installer OpenSSL
apt-get install -y openssl

# Créer le répertoire de stockage
mkdir -p /etc/ssl/nginx/

# Générer la clé privée et le certificat auto-signé (valable 365 jours)
openssl req -x509 -newkey rsa:4096 -nodes \
  -keyout /etc/ssl/nginx/m2l.org.key \
  -out /etc/ssl/nginx/m2l.org.pem
```

#### Informations du certificat à renseigner

```text
Country Name (2 letter code) [AU]: FR
State or Province Name (full name) []: Votre Region
Locality Name (eg, city) []: Votre Ville
Organization Name (eg, company) []: m2l
Organizational Unit Name (eg, section) []: SIO
Common Name (e.g. server FQDN or YOUR name) []: *.m2l.org
Email Address []: admin@m2l.org
```

> [!TIP]
> Le Common Name `*.m2l.org` est un wildcard : un seul certificat couvre tous les sous-domaines (www, intranet, extranet, wiki).

#### Sécurisation de la clé privée

```bash
# La clé privée ne doit être lisible que par root
chmod 600 /etc/ssl/nginx/m2l.org.key
chown root:root /etc/ssl/nginx/m2l.org.key

# Vérifier
ls -la /etc/ssl/nginx/
```

#### Configuration HTTPS complète

Chaque Virtual Host doit être mis à jour pour écouter sur le port 443. Voici l'exemple pour `www.m2l.org.conf` :

```nginx
# Bloc HTTP – redirige automatiquement vers HTTPS
server {
    listen 80;
    server_name www.m2l.org m2l.org;
    return 301 https://$host$request_uri;
}

# Bloc HTTPS
server {
    listen 443 ssl;
    server_name www.m2l.org m2l.org;
    root /home/htdocs/m2l.org/www;
    index index.html index.htm;

    # Certificat et clé privée
    ssl_certificate     /etc/ssl/nginx/m2l.org.pem;
    ssl_certificate_key /etc/ssl/nginx/m2l.org.key;

    # Paramètres SSL recommandés
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/www.m2l.org-access.log;
    error_log /var/log/nginx/www.m2l.org-error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

> [!NOTE]
> Le premier bloc `server` redirige toutes les requêtes HTTP vers HTTPS avec un code 301 (redirection permanente).

### B. Rechargement et test

```bash
# Vérifier la syntaxe
nginx -t

# Recharger Nginx
systemctl reload nginx

# Vérifier que les ports 80 et 443 sont ouverts
netstat -natp | grep nginx
```

Testez depuis un navigateur : `https://www.m2l.org` (le navigateur affichera un avertissement pour certificat auto-signé).

---

## 4. Authentification HTTP pour l'intranet

**Objectif :** protéger l'accès à l'intranet par un mot de passe.

> [!IMPORTANT]
> Nginx ne supporte pas les fichiers `.htaccess` comme Apache. Toute la configuration se fait directement dans le bloc `location` du Virtual Host.

### A. Installation de apache2-utils (pour htpasswd)

```bash
# Installer les outils Apache
apt-get install -y apache2-utils

# Créer le répertoire sécurisé
mkdir -p /etc/nginx/auth/

# Créer le fichier .htpasswd et le premier utilisateur
htpasswd -c /etc/nginx/auth/.htpasswd sio

# Ajouter d'autres utilisateurs (sans -c)
htpasswd /etc/nginx/auth/.htpasswd paul
htpasswd /etc/nginx/auth/.htpasswd jacques
```

> [!WARNING]
> Ne jamais utiliser `-c` pour ajouter un deuxième utilisateur : cette option recrée le fichier et efface les utilisateurs existants !

### Configuration de l'authentification dans `intranet.m2l.org.conf`

```nginx
server {
    listen 80;
    server_name intranet.m2l.org;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name intranet.m2l.org;
    root /home/htdocs/m2l.org/intranet;
    index index.html index.htm;

    ssl_certificate     /etc/ssl/nginx/m2l.org.pem;
    ssl_certificate_key /etc/ssl/nginx/m2l.org.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    access_log /var/log/nginx/intranet.m2l.org-access.log;
    error_log /var/log/nginx/intranet.m2l.org-error.log;

    location / {
        # Authentification HTTP Basic
        auth_basic "Accès réservé – Intranet m2l";
        auth_basic_user_file /etc/nginx/auth/.htpasswd;

        try_files $uri $uri/ =404;
    }
}
```

Les deux directives clés sont :

- `auth_basic` : le message affiché dans la fenêtre d'authentification
- `auth_basic_user_file` : chemin absolu vers le fichier `.htpasswd`

### Pages d'erreur personnalisées

```nginx
# Dans le bloc server { ... } de n'importe quel Virtual Host
error_page 401 /401.html;
error_page 403 /403.html;
error_page 404 /404.html;
error_page 500 502 503 504 /50x.html;

# Définir l'emplacement des pages d'erreur
location = /401.html {
    root /home/htdocs/m2l.org/erreurs/;
    internal;
}
location = /404.html {
    root /home/htdocs/m2l.org/erreurs/;
    internal;
}
```

Créez le répertoire et les pages :

```bash
mkdir -p /home/htdocs/m2l.org/erreurs
echo '<h1>401 – Accès non autorisé</h1>' > /home/htdocs/m2l.org/erreurs/401.html
echo '<h1>403 – Accès interdit</h1>' > /home/htdocs/m2l.org/erreurs/403.html
echo '<h1>404 – Page introuvable</h1>' > /home/htdocs/m2l.org/erreurs/404.html
echo '<h1>500 – Erreur serveur</h1>' > /home/htdocs/m2l.org/erreurs/50x.html
```

### B. Application et test

```bash
# Vérifier la configuration
nginx -t

# Recharger Nginx
systemctl reload nginx
```

Testez dans un navigateur : `https://intranet.m2l.org`. Une fenêtre de connexion doit apparaître.

---

## 5. Vérification finale et dépannage

### Structure finale attendue

```text
/etc/nginx/
├── nginx.conf
├── auth/
│   └── .htpasswd
├── sites-available/
│   ├── www.m2l.org.conf
│   ├── intranet.m2l.org.conf
│   ├── extranet.m2l.org.conf
│   └── wiki.m2l.org.conf
└── sites-enabled/
    ├── www.m2l.org.conf -> ../sites-available/www.m2l.org.conf
    ├── intranet.m2l.org.conf -> ../sites-available/intranet.m2l.org.conf
    ├── extranet.m2l.org.conf -> ../sites-available/extranet.m2l.org.conf
    └── wiki.m2l.org.conf -> ../sites-available/wiki.m2l.org.conf

/etc/ssl/nginx/
├── m2l.org.pem
└── m2l.org.key
```

### Commandes de vérification finale

```bash
# Lister les sites activés
ls -la /etc/nginx/sites-enabled/

# Vérifier l'ensemble de la configuration
nginx -t

# Vérifier les ports ouverts
ss -tlnp | grep nginx

# Consulter les journaux en temps réel
tail -f /var/log/nginx/*.log
```

### Tableau de dépannage

| Problème | Solution |
|---|---|
| Nginx ne démarre pas | `journalctl -xeu nginx` |
| Erreur de syntaxe | `nginx -t` (indique le fichier et la ligne) |
| Site non accessible | `netstat -natp \| grep :80` ou `:443` |
| Mauvais Virtual Host affiché | Vérifier `server_name` et les liens dans `sites-enabled/` |
| Certificat SSL non trouvé | Vérifier les chemins dans `ssl_certificate` |
| Authentification ne fonctionne pas | Vérifier le chemin `auth_basic_user_file` et les droits |
| Page 403 sur les fichiers | `chown -R www-data /home/htdocs/ && chmod -R 755 /home/htdocs/` |
| Redirection infinie HTTP vers HTTPS | Vérifier qu'il n'y a pas de `return 301` dans le bloc 443 |

---

## Annexe : modèle générique et commandes rapides

### Modèle générique de Virtual Host Nginx avec SSL

```nginx
# /etc/nginx/sites-available/SITE.m2l.org.conf

# Redirection HTTP → HTTPS
server {
    listen 80;
    server_name SITE.m2l.org;
    return 301 https://$host$request_uri;
}

# Serveur HTTPS
server {
    listen 443 ssl;
    server_name SITE.m2l.org;
    root /home/htdocs/m2l.org/SITE;
    index index.html index.htm;

    # SSL
    ssl_certificate     /etc/ssl/nginx/m2l.org.pem;
    ssl_certificate_key /etc/ssl/nginx/m2l.org.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Logs
    access_log /var/log/nginx/SITE.m2l.org-access.log;
    error_log  /var/log/nginx/SITE.m2l.org-error.log;

    location / {
        # Pour l'intranet uniquement, ajouter :
        # auth_basic "Accès réservé";
        # auth_basic_user_file /etc/nginx/auth/.htpasswd;

        try_files $uri $uri/ =404;
    }
}
```

### Commandes de référence rapide

| Action | Commande |
|---|---|
| Vérifier la config | `nginx -t` |
| Recharger la config | `systemctl reload nginx` |
| Redémarrer Nginx | `systemctl restart nginx` |
| Activer un site | `ln -s /etc/nginx/sites-available/site.conf /etc/nginx/sites-enabled/` |
| Désactiver un site | `rm /etc/nginx/sites-enabled/site.conf` |
| Voir les processus Nginx | `ps aux \| grep nginx` |
| Voir les logs en live | `tail -f /var/log/nginx/error.log` |
| Créer un utilisateur htpasswd | `htpasswd /etc/nginx/auth/.htpasswd utilisateur` |
| Voir les ports ouverts | `netstat -natp \| grep nginx` |
| Tester une URL en CLI | `curl -k -I https://www.m2l.org` |
