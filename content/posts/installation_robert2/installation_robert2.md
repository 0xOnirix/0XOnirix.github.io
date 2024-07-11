---
title: "Installation de Loxya(Robert2)"
date: 2024-07-08T16:04:20+02:00
draft: false
toc: true
images:
tags: 
  - Robert2
  - Loxya(Robert2)
  - Loxya
  - Debian
  - Debian 12
categories:
  - Administration système
---

**/!\ Cet installation a été réalisée sur Debian 12 ! /!\\**

## Préambule

Loxya(Robert2) est un serveur de gestion et suivis d'inventaire.

La version gratuite est open-source et comprend les fonctions suivantes:

- Gestion du stock
- Gestion des sorties
- Calendrier
- Gestion des clients et emprunteurs
- Gestion du personnel
- Edition de fiches de sorties, devis et factures

Ce qu'elle ne comprend PAS:

- Authentification LDAP (ou tout autre centralisation d'authentification)
- Demande de réservation en ligne
- Scanner de code barre
- Assistance Loxya

---

## Installation

---

### Installer les dépendances

Robert 2 a besoin d'un serveur Web (ici Apache2), une base de données (ici MariaDB) et de certains modules PHP (version minimum supportée: php 8.0):

```sh
sudo apt install apache2 mariadb-server zip php libapache2-mod-php php-bcmath php-curl php-php-gettext php-intl php-mbstring php-xml php-bcmath php-mysql w3m -y
```

---

### Base de données

Robert2 est compatible uniquement **MYSQL** ou **MariaDB** pour stocker ses données, je vais ici utiliser MariaDB et la configurer.

#### Installation sécurisée de MariaDB

```sh
 sudo mysql_secure_installation

 Enter current password for root (enter for none): MotDePasseRoot
 Switch to unix_socket authentication [Y/n] n
 Remove anonymous users? [Y/n] y
 Disallow root login remotely? [Y/n] y
 Remove test database and access to it? [Y/n] y
 Reload privilege tables now? [Y/n] y
```

#### Création de la base de données

- Se connecter à MariaDB : `sudo mysql -u root -p`
- Créer une base de donnée: `CREATE DATABASE robert2_db;`
- Créer un utilisateur autre que root: `CREATE USER 'robert2'@'localhost' IDENTIFIED BY 'motdepasse';`
- Donner tous les droits à l'utilisateur sur la base de données `GRANT ALL PRIVILEGES ON robert2_db.* TO 'robert2'@'localhost';`
- Recharger les privilèges: `FLUSH PRIVILEGES;`
- Fermer la connexion: `quit;`

Modifier le fichier `/etc/mysql/my.cnf` en ajoutant les lignes suivantes:
```sh
[client-server]
...
default-character-set=utf8mb4

[mysqld]
...
collation-server = utf8mb4_unicode_ci
```

---

### Serveur Web

Robert2 nécessite un serveur Web pour fonctionner, il ne supporte que **NGINX** et **Apache2**, je vais ici utiliser ce dernier.

#### HTTP

**/!\ Attention /!\ il est fortemment déconseillé de déployer le serveur en HTTP en production.**

Créer et modifier le fichier `/etc/apache2/sites-available/robert2.mydomain.local.conf`:
```sh
<VirtualHost *:80>
        ServerAdmin admin@mydomain.local
        ServerName robert2.mydomain.local
        ServerAlias www.robert2.mydomain.local
        DocumentRoot /var/www/Robert2

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        <Directory /var/www/Robert2>
                Options +FollowSymLinks
                AllowOverride All
                Require all granted
        </Directory>
        <Location "/server-status">
                SetHandler server-status
                Require all granted
        </Location>
</VirtualHost>
```
#### HTTPS

- Mettre respectivement le certificat (ex: robert2.mydomain.local.crt) et la clé privée (ex: robert2.mydomain.local.key) du serveur dans `/etc/ssl/certs` et `/etc/ssl/private`
- Activer le module php **ssl**: `sudo a2enmod ssl`
- Modifier `/etc/apache2/sites-available/robert2.mydomain.local.conf`:

```sh
<VirtualHost *:80>
		ServerName robert2.mydomain.local
		ServerAlias www.robert2.mydomain.local
		Redirect permanent / https://robert2.mydomain.local/
</VirtualHost>

<VirtualHost *:443>
ServerAdmin admin@mydomain.local
        ServerName robert2.mydomain.local
        ServerAlias www.robert2.mydomain.local
        DocumentRoot /var/www/Robert2
		
        SSLEngine On
        SSLCertificateFile /etc/ssl/certs/robert2.mydomain.local.crt
        SSLCertificateKeyFile /etc/ssl/private/robert2.mydomain.local.key

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        <Directory /var/www/Robert2>
                Options +FollowSymLinks
                AllowOverride All
                Require all granted
        </Directory>
        <Location "/server-status">
                SetHandler server-status
                Require all granted
        </Location>
</VirtualHost>
```
#### Paramétrage d'Apache2

Afin de paramétrer correctement Apache2, il faut éditer le fichier `/etc/apache2/apache2.conf` et rajouter la ligne `ServerName 127.0.0.1` à la fin.

Sans ça, l'erreur **Apache Configuration Error AH00558: Could not reliably determine the server's fully qualified domain name** va apparaître quadn on va utiliser **apache2ctl configtest**.

On active ensuite le VHost du serveur, désactive le site par défaut et vérifie la configuration:
```sh
sudo a2ensite robert2.mydomain.local
sudo a2dissite 000-default.conf
```

```sh
sudo apache2ctl configtest
AH00112: Warning: DocumentRoot [/var/www/Robert2] does not exist
Syntax OK
```

L'avertissement est normale car on n'a pas encore créé le répertoire.

Enfin, on active le module URLRewriting et on redémarre Apache2:
```sh
sudo a2enmod rewrite
sudo systemctl restart apache2
```

---

### Robert2

Ceci est la dernière étape de l'article, l'installation de Robert2.

#### Téléchargement

- Télécharger le .zip à cette [adresse](https://github.com/Robert-2/Robert2/releases)
- Le mettre dans `/var/www/`
- L'extraire et renommer le dossier **Robert2**
- Donner les droits sur le dossier à l'utilisateur **www-data**: `sudo chown -R www-data:www-data /var/www/Robert2`

#### Finalisation de l'installation

Afin de finaliser l'installation, il faut ouvrir un navigateur et se rendre sur l'IP du serveur, normalement, voila ce qui devrait s'afficher.
<div>
        <img src="/posts/installation_robert2/images/Capture1.png" alt="Capture1">
</div>

On peut lancer l'assistant en cliquant sur **C'est parti !**.

<div>
        <img src="/posts/installation_robert2/images/Capture2.png" alt="Capture2">
</div>

Cette page demande l'**URL d'application** qui est l'URL à laquelle on va se connecter à Robert2, elle est très importante car si on ne rentre pas EXACTEMENT cette dernière (protocole inclut) dans notre navigateur, il sera impossible de s'identifier ou le serveur sera inaccessible.

Après ça, il y a deux pages nous demandant de valider des paramètres et renseigner les informations de notre société avant d'arriver au paramétrage de la BDD.

<div>
        <img src="/posts/installation_robert2/images/Capture5.png" alt="Capture5">
</div>

Ici, on va pouvoir renseigner les informations de la BDD créée précedemment:

- **Serveur MYSQL (host):** localhost
- **Utilisateur MYSQL (user):** Le nom de l'utilisateur (ici robert2)
- **Mot de passe MYSQL (password):** Le mot de passe de l'utilisateur
- **Nom de la base de données:** Ici robert2_db

Une fois cela fait, il n'y a plus qu'a laisser Robert2 créer la structure de la BDD, créer un utilisateur et le tour est joué !

---

## Erreurs
### Page blanche

Donnez l'appartenance du dossier et sous-dossiers `/var/www/Robert2` à **www-data** (chown).

### apachectl status renvoie un problème avec www-browser

Installer le paquet **w3m**.

### Désolé, mais l'API est inaccessible...

Cela signifie que l'URL a laquelle vous vous connectez n'est pas la même que celle inscrite après **baseUrl** dans le fichier `/var/www/Robert2/src/App/Config/settings.json`.
Cela peut être au niveau du protocole (HTTPS/HTTP) ou parce que vous vous y connecter depuis l'adresse IP alors que le nom de domaine y est paramétré.

---

## Sources

- [https://robertmanager.org/wiki/install](https://robertmanager.org/wiki/install)
- [https://www.debian-fr.org/t/certificat-packages-sury-org-expire/86215](https://www.debian-fr.org/t/certificat-packages-sury-org-expire/86215)
- [https://www.digitalocean.com/community/tutorials/apache-configuration-error-ah00558-could-not-reliably-determine-the-server-s-fully-qualified-domain-name](https://www.digitalocean.com/community/tutorials/apache-configuration-error-ah00558-could-not-reliably-determine-the-server-s-fully-qualified-domain-name)
