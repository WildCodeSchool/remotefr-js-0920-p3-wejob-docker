# Dockerizer WeJob

L'idée est de simplifier le setup de l'environnement de dev local.

On commence par le WordPress

## Images Docker

L'idée est de se faire sa propre image PHP/Apache pour être autant que possible en "conditions réelles"

* Image WP : <https://hub.docker.com/_/wordpress>
* Mentionne "you'll need to create your own image FROM [this one](https://raw.githubusercontent.com/docker-library/wordpress/618490d4bdff6c5774b84b717979bfe3d6ba8ad1/apache/Dockerfile)"

## Setup volume

On va stocker toute l'arborescence de WP dans un .VHDX pour ne pas saturer C: (qui n'en peut plus).

* Création d'un VHDX : *Computer Management > Disk Management > Action > Create VHD* puis dans la liste, bouton droit et *Initialize Disk*
* Création d'un volume ext4 dans un .VHDX avec [DiskGenius](https://www.diskgenius.com/) puis formatage puis **Quitter DiskGenius**
* Montage de puis PowerShell en admin : `wsl --mount \\.\PhysicalDrive4 --partition 2`
* Dans WSL2 : `/mnt/wsl/PhysicalDrive4p2$ sudo unzip /mnt/d/__BackupDownloadsC/WP_WeJob/wejob-wordpress.zip`

## Compose file

Déjà le fichier d'exemple](http://bit.do/wp-docker-php5-6) (Dockerfile) est très bien : c'est du PHP 5.6 (avant migration, leur PHP sur OVH est un 5.6.40). Côté WeJob c'est du WP 5.1.8 à la base.

### Erreurs

Pendant le build de l'image (basée sur Stretch) :
* [Package 'libpng12-dev' has no installation candidate](https://askubuntu.com/questions/991706/e-package-libpng12-dev-has-no-installation-candidate)
* `ERROR: for wj-dockerized_wordpress_1  Cannot start service wordpress: OCI runtime create failed: container_linux.go:370: starting container process caused: exec: "/entrypoint.sh": permission denied: unknown` -> **VIRER le entrypoint** ~~[solution](https://github.community/t/permission-denied-exec-entrypoint-sh/16216) : `chmod +x entrypoint.sh`~~

## Compose up !

* `docker-compose up -d`
* injection dump : `docker exec -i wj-dockerized_db_1 mysql -uroot -pwjroot exampledb < /mnt/d/__BackupDownloadsC/WP_WeJob/wejobcom
tjadmin.sql`

## Installation de WP-CLI (à ajouter dans le Dockerfile)

Entrer dans le container :

```
docker exec -it wj-dockerized_wordpress_1 /bin/bash`
```

Download & install :

```
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
php wp-cli.phar --info
chmod +x wp-cli.phar
mv wp-cli.phar /usr/local/bin/wp
```

Installer less qui manque : `apt-get update && apt-get install less`

Run en root pour faire search-replace :

```
wp --allow-root search-replace "http://www.we-job.com" "http://localhost:8080"
```

## Up & Running 😎

Petits réglages : `chown -R 999:root /mnt/wsl/PhysicalDrive4p2/wejob-wordpress` (même owner que le `wejob-wpdb`)

Ne change rien, on a toujours une erreur mineure sur la home page...

> **Warning**: fopen(/var/www/html/wp-content/uploads/wp-file-manager-pro/fm_backup/index.html): failed to open stream: Permission denied in **/var/www/html/wp-content/plugins/wp-file-manager/file_folder_manager.php** on line **67**
>
> **Warning**: fclose() expects parameter 1 to be resource, boolean given in **/var/www/html/wp-content/plugins/wp-file-manager/file_folder_manager.php** on line **68**
>
> **Warning**: chmod(): Operation not permitted in **/var/www/html/wp-content/plugins/wp-file-manager/file_folder_manager.php** on line **69**

**Réglé dans le container** : `chown -R www-data:www-data /var/www/html/wp-content/`

## Le dossier `wejob-wordpress`

On a fait un `git init` dans l'état initial. Puis :

```
root@DESKTOP-TMB3JDP:/mnt/wsl/PhysicalDrive4p2/wejob-wordpress# git status
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        index.php
        license.txt
        readme.html
        wp-activate.php
        wp-admin/
        wp-blog-header.php
        wp-comments-post.php
        wp-config-sample.php
        wp-config.php
        wp-config.php.default
        wp-content/
        wp-cron.php
        wp-includes/
        wp-links-opml.php
        wp-load.php
        wp-login.php
        wp-mail.php
        wp-settings.php
        wp-signup.php
        wp-trackback.php
        xmlrpc.php

nothing added to commit but untracked files present (use "git add" to track)
```

Au passage : <https://github.com/nezhar/wordpress-docker-compose>

## Régler erreurs 404

<http://localhost:8080/solution-recherche-emploi-aquitaine/cadres-jeune-diplome-booster-recherche-emploi/> => Not Found

Probablement une histoire de ré-écriture et/ou droits sur .htaccess.

En effet, dans le `docker-entrypoint.sh` on a le contenu du `.htaccess` à sauver sous `/var/www/html` :

```
# BEGIN WordPress
  <IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteRule ^index\.php$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /index.php [L]
  </IfModule>
# END WordPress
```

Puis `chown www-data:www-data .htaccess`

## Création d'une page avec shortcode pour intégrer l'app React

* Nouvelle page "Base de données candidats" contenant le shortcode `[wejob_candidates_db]`, sous "Prestations RH aux entreprises"
* Ajout de cette page dans le menu, puis accès via :<http://localhost:8080/prestations-aux-entreprises/base-de-donnees-candidats/>

## Todo update WP

* Repartir du thème Ample en version clean
* Faire un child theme
* Faire un layout sans la sidebar

## Todo Dockerize

* Variable d'env pour le chemin du dossier WP ? <https://stackoverflow.com/q/45103843/>
* Ajouter `.htaccess`