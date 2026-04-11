# Chiffrement des communications (HTTPS & FTPS)

Ce document détaille la mise en place du protocole SSL/TLS sur Apache et ProFTPD, la génération de certificats auto-signés et la configuration complète des services sécurisés.

---

## Partie A — HTTPS

### 1. Installation et génération des certificats

Installer OpenSSL et préparer le répertoire :

```
apt-get update && apt-get install openssl
mkdir /etc/ssl/localcerts
```

Générer le certificat auto-signé valable 365 jours :

```
DIR=/etc/ssl/localcerts
openssl req -x509 -newkey rsa:4096 -nodes -keyout $DIR/m2l.org.key -out $DIR/m2l.org.pem -days 365
```

Détail des paramètres :

| Paramètre | Description |
| :--- | :--- |
| `-x509` | Génère un certificat auto-signé |
| `-newkey rsa:4096` | Crée une clé RSA de 4096 bits |
| `-nodes` | Pas de phrase de passe (redémarrage auto du service) |
| `-keyout` | Chemin de la clé privée `.key` |
| `-out` | Chemin du certificat public `.pem` |
| `-days 365` | Durée de validité |

### 2. Configuration du VirtualHost SSL

Répertoire : `/etc/apache2/sites-available/`
Fichier : `default-ssl.conf`

Les deux lignes essentielles à renseigner :

```
SSLCertificateFile      /etc/ssl/localcerts/m2l.org.pem
SSLCertificateKeyFile   /etc/ssl/localcerts/m2l.org.key
```

### 3. Activation et vérification

Activer le module SSL et le site :

```
a2enmod ssl
a2ensite default-ssl
systemctl restart apache2
```

Vérifier que le service écoute sur le port 443 :

```
netstat -natp | grep 443
```

![Port 443](images/Capture%20d%27%C3%A9cran%202026-04-08%20215118.png)

> Ne pas oublier d'ouvrir le port 443 dans le script IPTables si un pare-feu est actif.

Modifier la première ligne des fichiers VirtualHost existants : remplacer le port `80` par `443`.

![Certificat](images/Capture%20d%27%C3%A9cran%202026-04-08%20215149.png)

---

## Partie B — FTPS

### 1. Génération des certificats

Préparer le répertoire et générer le certificat pour ProFTPD :

```
mkdir /etc/proftpd/ssl/
DIR=/etc/proftpd/ssl/
openssl req -x509 -newkey rsa:4096 -nodes -keyout $DIR/proftpd.key -out $DIR/proftpd.pem -days 365
```

### 2. Configuration de ProFTPD

Activer TLS dans `/etc/proftpd/proftpd.conf` en décommentant :

```
Include /etc/proftpd/tls.conf
```

Éditer `/etc/proftpd/tls.conf` avec les directives suivantes :

| Directive | Description |
| :--- | :--- |
| `TLSEngine on` | Active le moteur TLS |
| `TLSLog /var/log/proftpd/tls.log` | Log dédié aux connexions chiffrées |
| `TLSRSACertificateFile` | Chemin vers le certificat `.pem` |
| `TLSRSACertificateKeyFile` | Chemin vers la clé `.key` |
| `TLSOptions` | Options de sécurité |

![Logs TLS](images/Capture%20d%27%C3%A9cran%202026-04-08%20215233.png)

### 3. Tests et résultats

Redémarrer le service :

```
systemctl restart proftpd
```

Test de connexion avec FileZilla :

![FileZilla FTPS 1](images/Capture%20d%27%C3%A9cran%202026-04-08%20215316.png)
![FileZilla FTPS 2](images/Capture%20d%27%C3%A9cran%202026-04-08%20215349.png)
