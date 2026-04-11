# Installation LAMP

## Partie 1 — Serveur HTTP Apache2

Installer apache2 (vérifier d'abord avec `dpkg -l | grep apache`) :

```
apt update
apt install apache2
```

Le port écouté par apache2 est le **port 80**.

![Port 80](images/Capture%20d%27%C3%A9cran%202026-04-08%20195849.png)

On peut le modifier dans le fichier `ports.conf` dans le répertoire `apache2`.

![Fichier ports.conf](images/Capture%20d%27%C3%A9cran%202026-04-08%20200007.png)

![Fichier ports.conf 2](images/Capture%20d%27%C3%A9cran%202026-04-08%20200027.png)

Le groupe et l'utilisateur utilisés par apache2 est `www-data`. Pour vérifier :

```
ps aux | grep apache2
```

![Groupe et utilisateur apache2](images/Capture%20d%27%C3%A9cran%202026-04-08%20200355.png)

Le fichier de log d'apache2 est accessible via : `/var/log/apache2`

![Fichier de log apache2](images/Capture%20d%27%C3%A9cran%202026-04-08%20200439.png)

---

## Partie 2 — SQLite3

Installer SQLite3 :

```
apt update
apt install sqlite3 php-sqlite3 libapache2-mod-php
```

Créer un script `info.php` dans `/var/www/html` :

```
nano info.php
```

```
<?php
phpinfo();
?>
```

Vérifier la librairie PDO pour SQLite3 en ouvrant : `http://10.31.16.80/info.php`

Créer la base de données `myCDS.db` dans `/var/www/html` :

```
sqlite3 myCDS.db
```

Créer les deux tables :

```
create table artist (art_id INTEGER PRIMARY KEY, art_name TEXT);

create table cd (cd_id INTEGER PRIMARY KEY, art_id INTEGER NOT NULL, cd_title TEXT NOT NULL, cd_date TEXT);
```

Insérer les données :

```
insert into artist (art_id,art_name) values (NULL,'Peter Gabriel');
insert into artist (art_id,art_name) values (NULL,'Bruce Hornsby');
insert into artist (art_id,art_name) values (NULL,'Lyle Lovett');
insert into artist (art_id,art_name) values (NULL,'Beach Boys');
insert into cd (cd_id,art_id,cd_title,cd_date) values (NULL,1,'Us','1992');
insert into cd (cd_id,art_id,cd_title,cd_date) values (NULL,2,'The Way It Is','1986');
insert into cd (cd_id,art_id,cd_title,cd_date) values (NULL,2,'Scenes from the Southside','1990');
insert into cd (cd_id,art_id,cd_title,cd_date) values (NULL,1,'Security','1990');
insert into cd (cd_id,art_id,cd_title,cd_date) values (NULL,3,'Joshua Judges Ruth','1992');
insert into cd (cd_id,art_id,cd_title,cd_date) values (NULL,4,'Pet Sounds','1966');
```

Créer le script `mycds.php` dans `/var/www/html` :

```
<?php
try {
    $db = new PDO('sqlite:myCDS.db');
    $db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

    echo "<b>Les artistes</b><br />";
    $artists = $db->query('SELECT * FROM artist');
    foreach ($artists as $row) {
        echo $row['art_id'] . ". " . $row['art_name'] . "<br />";
    }

    echo "<br /><b>Tous les Albums (avec leur artiste)</b><br />";

    $sqlcd = 'SELECT cd.cd_title, cd.cd_date, artist.art_name
              FROM cd
              INNER JOIN artist ON artist.art_id = cd.art_id';

    $results = $db->query($sqlcd);
    foreach ($results as $rows) {
        echo "- " . $rows['art_name'] . " (" . $rows['cd_date'] . ") " . $rows['cd_title'] . "<br />";
    }

    $db = null;
} catch(PDOException $e) {
    echo 'Exception : ' . $e->getMessage();
}
?>
```

Vérifier en ouvrant : `http://10.31.16.80/mycds.php`

![Test mycds.php](images/Capture%20d%27%C3%A9cran%202026-04-08%20200750.png)

---

## Partie 3 — MariaDB

À retenir avant de commencer :
- Tout service modifié doit être redémarré
- En cas de problème : vérifier `ps`, `netstat`, puis `/var/log/`

Installer MariaDB :

```
apt update
apt install mariadb-server php-mysql
```

Se connecter à MariaDB :

```
mysql -u root -p
```

Créer le compte `dba` avec tous les droits :

```
create user 'dba'@'localhost' identified by 'drowssap';
grant all privileges on *.* to 'dba'@'localhost' with grant option;
flush privileges;
```

Tester la connexion avec le nouveau compte et afficher les bases de données disponibles.

Par défaut MariaDB écoute sur localhost. Pour autoriser l'accès distant, modifier `/etc/mysql/mariadb.conf.d/50-server.cnf` :

```
nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Changer la ligne `bind-address` de `127.0.0.1` vers `0.0.0.0`.

![Configuration MariaDB](images/Capture%20d%27%C3%A9cran%202026-04-08%20200842.png)

---

## Partie 4 — phpMyAdmin

Installer les modules PHP nécessaires :

```
apt install php-json php-mbstring php-zip php-gd php-xml php-curl
```

Redémarrer apache2 :

```
systemctl restart apache2
```

Télécharger phpMyAdmin :

```
wget https://files.phpmyadmin.net/phpMyAdmin/5.2.3/phpMyAdmin-5.2.3-all-languages.zip
```

Dézipper, renommer, nettoyer et ajuster les droits :

```
unzip phpMyAdmin-5.2.3-all-languages.zip
mv phpMyAdmin-5.2.3-all-languages phpmyadmin
rm phpMyAdmin-5.2.3-all-languages.zip
chown -R www-data:www-data phpmyadmin/
systemctl restart apache2
```

Accéder à phpMyAdmin via : `http://10.31.16.80/phpmyadmin/`

Créer la table `users` :

```
CREATE TABLE `users` (
  `id` int(11) NOT NULL auto_increment,
  `name` varchar(100) NOT NULL,
  `age` int(3) NOT NULL,
  `email` varchar(100) NOT NULL,
  PRIMARY KEY (`id`)
);
```

Pour afficher les logs : `/var/log/apache2`
