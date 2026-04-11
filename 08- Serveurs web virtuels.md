# Serveurs web virtuels & .htaccess

Hébergement de plusieurs sites sur une même instance Apache :
- `www.m2l.org`
- `intranet.m2l.org`
- `extranet.m2l.org`
- `wiki.m2l.org`

---

## Partie A — Serveurs web virtuels

Avant de procéder, définir les zones DNS sur `dns1` dans `/etc/bind/db.m2l.org`.

![Zones DNS](images/Capture%20d%27%C3%A9cran%202026-04-08%20212706.png)

---

### Configuration Apache

Chaque VirtualHost possède son propre fichier `.conf` dans `/etc/apache2/sites-available/`.

**Fichiers de configuration par site :**

`/etc/apache2/sites-available/www.m2l.org.conf`
![www.m2l.org](images/Capture%20d%27%C3%A9cran%202026-04-08%20213136.png)

`/etc/apache2/sites-available/intranet.m2l.org.conf`
![intranet.m2l.org](images/Capture%20d%27%C3%A9cran%202026-04-08%20213203.png)

`/etc/apache2/sites-available/extranet.m2l.org.conf`
![extranet.m2l.org](images/Capture%20d%27%C3%A9cran%202026-04-08%20213227.png)

`/etc/apache2/sites-available/wiki.m2l.org.conf`
![wiki.m2l.org](images/Capture%20d%27%C3%A9cran%202026-04-08%20213257.png)

---

**Activer / désactiver un site :**

```
a2ensite intranet.m2l.org
a2dissite intranet.m2l.org
```

`a2ensite` crée un lien symbolique dans `/etc/apache2/sites-enabled/`. Après chaque modification, recharger Apache :

```
systemctl reload apache2
```

---

### Création des répertoires et fichiers

Structure racine : `home/htdocs/m2l.org/`

| Site | Répertoire | Action |
| :--- | :--- | :--- |
| www.m2l.org | `/www` | Créer `index.html` |
| intranet.m2l.org | `/intranet` | Créer `index.html` |
| extranet.m2l.org | `/extranet` | Créer `index.html` |
| wiki.m2l.org | `/wiki` | Installation DokuWiki |

**Installation de DokuWiki** dans le répertoire `wiki` :

```
sudo wget https://download.dokuwiki.org/src/dokuwiki/dokuwiki-stable.tgz
sudo tar xvf dokuwiki-stable.tgz
sudo mv dokuwiki-*/ dokuwiki
sudo rm dokuwiki-stable.tgz
mv dokuwiki/* ./
mv dokuwiki/.* ./
```

---

## Partie B — Fichiers .htaccess

### 1. Modification de la configuration Apache

Ajouter `AllowOverride All` dans chaque fichier de configuration VirtualHost.

![AllowOverride All](images/Capture%20d%27%C3%A9cran%202026-04-08%20213327.png)

### 2. Création du fichier .htaccess

Placer le fichier à la racine de chaque site (ex: `home/htdocs/m2l.org/www/`) :

```
AuthType Basic
AuthUserFile /home/htdocs/m2l.org/.htpasswd
ErrorDocument 401 /error/401.html
ErrorDocument 403 /error/403.html
ErrorDocument 404 /error/404.html
ErrorDocument 500 /error/500.html
AuthName "Reserved Access"
Require valid-user
```

![Fichier .htaccess](images/Capture%20d%27%C3%A9cran%202026-04-08%20213427.png)

### 3. Gestion des utilisateurs (.htpasswd)

Créer le fichier avec le premier utilisateur :

```
htpasswd -c /home/htdocs/m2l.org/www/.htpasswd sio
```

Ajouter d'autres utilisateurs :

```
htpasswd /home/htdocs/m2l.org/www/.htpasswd paul
htpasswd /home/htdocs/m2l.org/www/.htpasswd jacques
```

En cas d'erreur de connexion, consulter les logs : `/var/log/apache2/error.log`

### 4. Pages d'erreurs personnalisées

Créer un dossier `error` dans chaque site et y placer les fichiers (`404.html`, etc.).

![Répertoire error](images/Capture%20d%27%C3%A9cran%202026-04-08%20213529.png)
![Vérification .htaccess](images/Capture%20d%27%C3%A9cran%202026-04-08%20213601.png)

---

## Partie C — Répertoires personnels (USER_DIR)

La directive `USER_DIR` permet aux utilisateurs de publier du contenu via leur répertoire `public_html`.

Accès : `http://10.31.16.80/~sio/`

Activer le module :

```
a2enmod userdir
```

Créer `index.html` et `index.php` dans `/home/sio/public_html/` :

```
<?php
phpinfo();
?>
```

Activer PHP pour UserDir en éditant `/etc/apache2/mods-enabled/php8.2.conf` — dans la section `<IfModule mod_userdir.c>`, changer `off` en `on`.
