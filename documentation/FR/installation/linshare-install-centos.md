# Installation de LinShare sur CentOS

   * [Téléchargement de LinShare](#dlLinshare)
   * [Déploiement de l'archive et des fichiers de configuration](#installFile)
   * [Installation de OpenJDK Java JRE](#instalOpenJDK)
   * [Installation des Bases de Données](#bdd)
     * [Installation de PostgreSQL](#postgre)
     * [Installation de MongoDB](#mongo)
   * [Activation du moteur d'aperçu (optionnel)](#thumbnail)
   * [Installation de Tomcat](#tomcat)
   * [Installation du serveur web](#apache)
     * [Configuration vhost ui-user](#ui-user)
     * [Configuration vhost ui-admin](#ui-admin)
   * [Configuration du pare-feu](#firewalld)
   * [Configuration et lancement de Linshare](#linconf)
   * [Premier accès](#firstAccess)

> Note :<br/>
Ce guide propose l'installation de la version __LinShare__ 2.3 sur CentOS 7 (les versions antérieures de CentOS ne sont pas supportées). Pour l'installation d'une version antérieure de __LinShare__, se référer aux branches GitHub, par exemple [LinShare 1.12.x](../../../../maintenance-1.12.x/documentation/EN/installation/linshare-install-centos.md)

## <a name="dlLinshare">Téléchargement de LinShare</a>

__LinShare__  est en téléchargement libre à l’adresse suivante :

[http://download.linshare.org/versions/](http://download.linshare.org/versions/)

Il existe plusieurs versions de __LinShare__. Choisir la version de __LinShare__ en accord avec le guide d'installation.
Chaque version de __LinShare__ contient tous les composants nécessaires liés à une version spécifique de __LinShare__.
Ne pas installer et utiliser un composant dont la version est différente de celles présentes dans le même dossier : afin d'éviter des problèmes de dépendances.

Pour cette installation, télécharger les fichiers, nommés ci-dessous, sur le serveur :
  * linshare-core-{VERSION}.war
  * linshare-core-{VERSION}-sql.tar.bz2
  * linshare-ui-admin-{VERSION}.tar.bz2
  * linshare-ui-user-{VERSION}.tar.bz2

> Note :<br/>
Dans cette procédure, on considérera que les fichiers sont téléchargés dans le répertoire temporaire `/opt/tmp`. Il est bien sûr possible d'utiliser n'importe quel autre dossier temporaire.

Afin de manipuler les archives, il sera nécessaire d’utiliser les outils `unzip` et `bzip2` :
```bash
yum install -y unzip bzip2
```

## <a name="installFile">Déploiement de l'archive et des fichiers de configuration</a>

Créer le répertoire de configuration de __LinShare__, puis copier les fichiers de configuration par défaut et renommer le fichier d'exemple comme suit :
```bash
mkdir -p /etc/linshare
mv /opt/tmp/linshare-core-{VERSION}.war /etc/linshare/linshare.war
unzip -j -d /etc/linshare/ linshare.war WEB-INF/classes/{linshare,log4j}.*
Archive:  linshare.war
  inflating: /etc/linshare/linshare.properties.sample  
  inflating: /etc/linshare/log4j.properties
mv /etc/linshare/linshare.properties.sample /etc/linshare/linshare.properties
```

Modifier le fichier `/etc/linshare/log4j.properties` afin de remplacer la ligne ci-dessous:
```java
log4j.rootCategory=INFO, CONSOLE
```
par la ligne suivante :
```java
log4j.rootCategory=INFO, LINSHARE
```
Vérifier l'emplacement du fichier de log suivant :
```java
log4j.appender.LINSHARE.File=/var/log/tomcat/linshare.log
```

## <a name="installOpenJDK">Installation de OpenJDK Java JRE</a>

__LinShare__  supporte l'OpenJDK ou Sun/Oracle Java en version 8. Voici son installation et son activation depuis les dépôts :

```bash
yum -y install java-1.8.0-openjdk.x86_64
update-alternatives --config java
```

> Note :<br/>
Les erreurs éventuelles relatives au plugin Java peuvent être ignorées.

## <a name="bdd">Installation des Bases de Données</a>

> Note :<br />
Historiquement LinShare était développé sur PostgreSQL. Les nouvelles fonctionnalités ont été développées sur MongoDB. La stratégie est de tout migrer vers MongoDB. La tâche étant fastidieuse, LinShare utilise donc actuellement les deux SGBD.

### <a name="postgre">Installation de PostgreSQL</a>

__Linshare__ requière l’utilisation d’une base de données (PostgreSQL) pour ses fichiers et sa configuration. Ce guide présente une installation avec PostgreSQL.

> Note :<br/>
La base de données MySQL n'est pas prise en charge dans LinShare v2.

```bash
yum install -y postgresql postgresql-server
```

Configurer et démarrer le service PostgreSQL :
```bash
postgresql-setup initdb
systemctl enable postgresql
systemctl start postgresql
```

Adapter le fichier de gestion des accès de PostgreSQL `/var/lib/pgsql/data/pg_hba.conf` :
```bash
 # TYPE  DATABASE                  USER          CIDR-ADDRESS         METHOD
 local   all               postgres               peer
 local   linshare                  linshare                           md5
 host    linshare                  linshare      127.0.0.1/32         md5
 host    linshare                  linshare      ::1/128              md5
```

> Note :<br/>
Ces lignes se trouvent généralement à la fin du fichier de configuration.
Pour des raisons de sécurité, le service PostgreSQL n’écoute qu’en local.

Redémarrer le service PostgreSQL :
```bash
systemctl restart postgresql
```

Il convient également d'ajouter ces règles dans les premières. En effet, PostgreSQL utilise la première règle valide qui correspond à la demande d'authentification.

Créer l’utilisateur linshare (mot de passe "PASSWORD") :
```bash
su - postgres
[postgres@localhost ~]$ psql
CREATE ROLE linshare
  ENCRYPTED PASSWORD 'PASSWORD'
  NOSUPERUSER NOCREATEDB NOCREATEROLE INHERIT LOGIN;
\q
```

Commandes : pour quitter, taper `\q` pour obtenir de l’aide sous PSQL, taper `\?`.

Créer et importer les schémas de base de données :
```bash
su - postgres
[postgres@localhost ~]$ psql
CREATE DATABASE linshare
  WITH OWNER = linshare
       ENCODING = 'UTF8'
       TABLESPACE = pg_default
       LC_COLLATE = 'en_US.UTF-8'
       LC_CTYPE = 'en_US.UTF-8'
       CONNECTION LIMIT = -1;
GRANT ALL ON DATABASE linshare TO linshare;
\q
```

> Important :<br/>
Si la base de données est installée en langue française, remplacer toutes les occurrences de chaîne `en_US` par `fr_FR`.

> Note :<br/>
Utiliser éventuellement le script nommé `createDatabase.sh` dans `src/main/resources/sql/postgresql/` qui fournit les commandes pour créer la bases de données.

Importer les fichiers SQL `createSchema.sql` et `import-postgresql.sql` :
```bash
cd /opt/tmp
tar xjvf linshare-core-*-sql.tar.bz2
psql -U linshare -W -d linshare -f linshare-core-sql/postgresql/createSchema.sql
Password for user linshare: PASSWORD
psql -U linshare -W -d linshare -f linshare-core-sql/postgresql/import-postgresql.sql
Password for user linshare: PASSWORD
```

Éditer le fichier de configuration de __LinShare__ `/etc/linshare/linshare.properties` :
```java
#******************** DATABASE
### PostgreSQL
linshare.db.username=linshare
linshare.db.password=PASSWORD
linshare.db.driver.class=org.postgresql.Driver
linshare.db.url=jdbc:postgresql://localhost:5432/linshare
linshare.db.dialect=org.hibernate.dialect.PostgreSQLDialect
```

### <a name="mongo">Installation de MongoDB</a>

Pour l'installation de __LinShare__, il est nécessaire d'installer une base de données mongoDB.
Le paquetage mongodb-org n'existe pas dans les référentiels par défaut de CentOS. Toutefois, MongoDB gère un référentiel dédié : il est donc nécessaire d'ajouter le dépôt.

Créer un fichier `/etc/yum.repos.d/mongodb-org.repo`, et ajouter les informations du référentiel de la dernière version stable au fichier :
```bash
[mongodb-org-3.2]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.2/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.2.asc
```

Installer le paquet mongodb-org à partir du référentiel en utilisant l'utilitaire yum :
```bash
yum install -y mongodb-org
```

Par défaut, MongoDB est configuré de la manière suivante dans la configuration de __LinShare__ `/etc/linshare/linshare.properties` :

```java
#### Mongo storage options ####
linshare.mongo.connect.timeout=30000
linshare.mongo.socket.timeout=30000

#### Write concern
# MAJORITY: waits on a majority of servers for the write operation.
# JOURNALED: Write operations wait for the server to group commit to the journal file on disk.
# ACKNOWLEDGED: Write operations that use this write concern will wait for acknowledgement,
#                               using the default write concern configured on the server.
linshare.mongo.write.concern=MAJORITY

# Standard URI connection scheme
linshare.mongo.client.uri=mongodb://127.0.0.1:27017/linshare
```

Avant de démarer le service MongoDB, s'assurer que le fichier que `/etc/mongod.conf` a l'adresse ip du bind: 127.0.0.1
Configurer et démarrer le service MongoDB:
```bash
systemctl enable mongod
systemctl start mongod
```

## <a name="thumbnail">Activation du moteur d'aperçu (optionnel)</a>

> Note :<br/>
Il y a un bug sur la librairie qui est utilisée par __LinShare__ pour communiquer avec LibreOffice (spécifique à CentOS). L'équipe LinShare travaille actuellement sur la résolution de ce problème (08/2019).

__LinShare__ dispose d'un moteur de génération d'aperçu pour plusieurs types de fichiers :
- Formats OpenDocument (ODT, ODP, ODS, ODG)
- Formats de documents Microsoft (DOCX, DOC, PPTX, PPT, XLSX, XLS)
- PDF documents
- Fichiers images (PNG, JPEG, JPG, GIF)
- Fichiers text (TXT, XML, LOG, HTML, ...)

> Note :<br/>
Avant d'activer le module, il est nécessaire d'installer LibreOffice : la version minimale requise pour LibreOffice est : 4.2.8.

Pour installer LibreOffice, exécuter la commande suivante :
```bash
yum -y install libreOffice
```

Par défault le moteur de génération de thumbnail est mis à `FALSE`. Pour l'activer il est nécessaire de modifier le fichier de configuration de __LinShare__ dans `/etc/linshare/linshare.properties` :
```java
#******** LinThumbnail configuration
# key to enable or disable thumbnail generation
linshare.documents.thumbnail.enable=true
linshare.linthumbnail.dropwizard.server=http://0.0.0.0:8090/linthumbnail?mimeType=%1$s
linshare.documents.thumbnail.pdf.enable=true
```
Cela va permettre de générer des aperçus après chaque dépôt de fichiers.

Pour utiliser ce mode, télécharger les fichiers suivants depuis l'adresse [http://download.linshare.org/versions/](http://download.linshare.org/versions/) :
* linshare-thumbnail-server-{VERSION}.jar
* linshare-thumbnail-server-{VERSION}.yml

> Note :<br/>
Dans cette procédure, on considérera que les fichiers sont téléchargés dans le répertoire temporaire `/opt/tmp`. Il est bien sûr possible d'utiliser n'importe quel autre dossier temporaire.

> Note <br>
Par défaut, le serveur est configuré pour écouter sur le port 80 : il est possible de le changer.

Installer le fichier `linshare-thumbnail-server-{VERSION}.yml` dans `/etc/linshare/linshare-thumbnail-server.yml` et installer l'archive java `linshare-thumbnail-server-{VERSION}.jar` dans le répertoire  `/usr/local/sbin/linshare-thumbnail-server.jar` :
```java
mv /opt/tmp/linshare-thumbnail-server-*.yml /etc/linshare/linshare-thumbnail-server.yml
mv /opt/tmp/linshare-thumbnail-server-*.jar /usr/local/sbin/linshare-thumbnail-server.jar
```

Créer un service systemd permet d'automatiser le lancement du serveur thumbnail en arrière plan. Créer le fichier `/etc/systemd/system/linshare-thumbnail-server.service`, et ajouter le contenu suivant :
```bash
[Unit]
Description=LinShare thumbnail server
After=network.target

[Service]
Type=idle
KillMode=process
ExecStart=/usr/bin/java -jar /usr/local/sbin/linshare-thumbnail-server.jar server /etc/linshare/linshare-thumbnail-server.yml

[Install]
WantedBy=multi-user.target
Alias=linshare-thumbnail-server.service
```

Configurer et activer le nouveau service :
```bash
systemctl daemon-reload
systemctl enable linshare-thumbnail-server.service
systemctl start linshare-thumbnail-server.service
```

## <a name="tomcat">Installation de Tomcat</a>

__LinShare__ étant une application Java compilée et empaquetée au format WAR (**W** eb **A** pplication a **R** chive), il lui faut donc un conteneur de servlets Java (Tomcat ou Jetty) pour fonctionner. Ce paragraphe présente l’installation et la configuration du serveur Tomcat.

Installer Tomcat depuis les dépôts :
```bash
yum install -y tomcat
```

Pour spécifier l’emplacement de la configuration de __LinShare__ (fichier `linshare.properties` ) ainsi que les options de démarrage par défaut nécessaire, récupérer les lignes commentées dans l'en-tête dans le fichier `linshare.properties` et copier-coller les dans le fichier Tomcat (`/etc/sysconfig/tomcat`):

L’ensemble des options de démarrage par défaut nécessaires à __Linshare__ sont indiquées dans les en-têtes des fichiers de configuration suivants :
  * `/etc/linshare/linshare.properties`
  * `/etc/linshare/log4j.properties`

Cependant, pour la déclaration de la variable `JAVA_OPT`, il faut concaténer les options, et donc ajouter la ligne suivante dans `/etc/sysconfig/tomcat` :
```bash
JAVA_OPTS="-Djava.awt.headless=true -Xms512m -Xmx2048m -Dlinshare.config.path=file:/etc/linshare/ -Dlog4j.configuration=file:/etc/linshare/log4j.properties -Dspring.profiles.active=default,jcloud,mongo,batches"
```

Dans le fichier `/usr/share/tomcat/conf/catalina.properties` de tomcat, des retours à la ligne matérialisés par le caractère `\` permettent de réduire la largeur des lignes des valeurs pour chaque clé de configuration. Il y a une clé de configuration nommée `tomcat.util.scan.DefaultJarScanner.jarsToSkip`. Ajouter `jclouds-bouncycastle-1.9.2.jar,bcprov-*.jar,\` quelque part dans la section associée à cette clé.
Voici un exemple d'extrait du fichier `/usr/share/tomcat/conf/catalina.properties` avec la ligne ajoutée au milieu :
```java
jetty-*.jar,oro-*.jar,servlet-api-*.jar,tagsoup-*.jar,xmlParserAPIs-*.jar,\
jclouds-bouncycastle-1.9.2.jar,bcprov-*.jar,\
xom-*.jar
```

Déployer l’archive de l’application __LinShare__ dans le serveur Tomcat :
```bash
mv /etc/linshare/linshare.war /usr/share/tomcat/webapps/linshare.war
```

## <a name="apache">Installation du serveur web</a>

L’interface d’administration de __LinShare__ est une application s’appuyant sur les langages web HTML/CSS et JavaScript. Elle nécessite un simple serveur web de type Apache ou Nginx. Ce guide présente l’installation de Apache HTTP Server.

Installer httpd depuis les dépôts :
```bash
yum install -y httpd
```

### <a name="ui-user">Configuration vhost ui-user</a>

Pour déployer l’application __LinShare__, il est nécessaire d’activer le module `mod_proxy` sur httpd.

Créer les répertoires dans le répertoire `/var/www/`, noter que le nom de répertoire sera le nom de domaine de l'application. Attribuer à l'utilisateur les droits d'accéder aux répertoires aussi.

```bash
mv /opt/tmp/linshare-ui-user-<VERSION>.tar.bz2 /var/www/
cd /var/www/
tar xjf /tmp/linshare_data/linshare-ui-user-<VERSION>.tar.bz2
chown -R apache: linshare-ui-user
rm -fr /var/www/linshare-ui-user-<VERSION>.tar.bz2
```

Pour déployer l'application __LinShare__, il est nécessaire de créer le fichier de configuration du vhost. Ajouter le fichier `/etc/httpd/conf.d/linshare-ui-user.conf` avec le contenu suivant :
```xml
<VirtualHost *:80>
...
ServerName linshare-user.local
DocumentRoot /var/www/linshare-ui-user
<Location /linshare>
    ProxyPass http://127.0.0.1:8080/linshare
    ProxyPassReverse http://127.0.0.1:8080/linshare
    ProxyPassReverseCookiePath /linshare /

    # Workaround to remove httpOnly flag (could also be done with Tomcat)
    Header edit Set-Cookie "(JSESSIONID=.*); Path.*" "$1; Path=/"
    # For https, you should add Secure flag.
    # Header edit Set-Cookie "(JSESSIONID=.*); Path.*" "$1; Path=/; Secure"

    #This header is added to avoid the JSON cache issue on IE.
    Header set Cache-Control "max-age=0,no-cache,no-store"
</Location>

ErrorLog /var/log/httpd/linshare-user-error.log
CustomLog /var/log/httpd/linshare-user-access.log combined
...
</Virtualhost>
```

### <a name="ui-admin">Configuration vhost ui-admin</a>

Deployer l'archive de l'application __LinShare__ UI Admin dans le répertoire du serveur httpd :
```bash
mv /opt/tmp/linshare-ui-admin-{VERSION}.tar.bz2 /var/www
cd /var/www/
tar xjf /tmp/linshare_data/linshare-ui-admin-{VERSION}.tar.bz2
chown -R apache: linshare-ui-admin
rm -fr /var/www/linshare-ui-admin-<VERSION>.tar.bz2
```

Pour déployer l’interface d’administration de __LinShare__, il est nécessaire de créer le fichier de configuration du vhost. Ajouter le fichier `/etc/httpd/conf.d/linshare-ui-admin.conf` avec le contenu suivant :
```xml
<VirtualHost *:80>
...
ServerName linshare-admin.local
DocumentRoot /var/www/linshare-ui-admin
<Location /linshare>
    ProxyPass http://127.0.0.1:8080/linshare
    ProxyPassReverse http://127.0.0.1:8080/linshare
    ProxyPassReverseCookiePath /linshare /

    # Workaround to remove httpOnly flag (could also be done with Tomcat)
    Header edit Set-Cookie "(JSESSIONID=.*); Path.*" "$1; Path=/"
    # For https, you should add Secure flag.
    # Header edit Set-Cookie "(JSESSIONID=.*); Path.*" "$1; Path=/; Secure"

    #This header is added to avoid the  JSON cache issue on IE.
    Header set Cache-Control "max-age=0,no-cache,no-store"
</Location>

ErrorLog /var/log/httpd/linshare-admin-error.log
CustomLog /var/log/httpd/linshare-admin-access.log combined
...
</Virtualhost>
```

> Note :<br/>
    Des exemples de vhosts sont disponibles dans le repertoire : [utils/apache2/vhosts-sample/](../../../utils/apache2/vhosts-sample/)

Configurer et démarrer le service httpd :
```bash
systemctl enable httpd
systemctl start httpd
```

## <a name="firewalld">Configuration du pare-feu</a>

Si besoin, ajouter le port 80 et éventuellement tous les autres ports nécessaires dans le fichier `/etc/firewalld/zones/public.xml` :
```xml
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  ...
  <port protocol="tcp" port="80"/>
</zone>
```

Puis redémarrer le service pour que les effets soient pris en compte:
```bash
systemctl restart firewalld
```

## <a name="linconf">Configuration et Lancement de LinShare</a>

Configurer l’emplacement de stockage des fichiers dans le fichier de configuration de __LinShare__ `/etc/linshare/linshare.properties` :
```java
linshare.documents.storage.filesystem.directory=/var/lib/linshare/filesystemstorage
linshare.encipherment.tmp.dir=/var/lib/linshare/tmp
```

Dans cette configuration il est nécessaire de créer le dossier avec les bonnes permissions :
```bash
mkdir -p /var/lib/linshare
chown -R tomcat:tomcat /var/lib/linshare
```

Configurer l’accès à un service SMTP, pour l’envoi des messages de notification dans le fichier de configuration de __LinShare__ `/etc/linshare/linshare.properties` :
```java
mail.smtp.host=<smtp.yourdomain.com>
mail.smtp.port=25
mail.smtp.user=linshare
mail.smtp.password=<SMTP-PASSWORD>
mail.smtp.auth.needed=false
mail.smtp.charset=UTF-8
```

Sur __LinShare__, il existe deux modes d'authentification possibles, le permier est celui par défaut et le second est une authification par SSO. Pour démarrer LinShare il est nécessaire d'activer l'un des modes suivants :
* default : processus d'authentification par défaut.
* sso : permet l'injection d'entête pour le SSO. Ce profil inclue les "..." du profil par défaut.

Le profil par défaut est jcloud pour le filesystem pour les tests.

Il est possible de surcharger ces paramètres en utilisant `-Dspring.profiles.active=xxx`
Ou bien d'utiliser une variable d'environnement : `SPRING_PROFILES_ACTIVE`.

Activer au moins un des profils de système de sockage de fichiers ci-dessous :
* jcloud : Utilisant jcloud comme système de stockage de fichier : Amazon S3, Swift, Ceph, filesystem (que pour les tests).
* gridfs : Using gridfs (mongodb) comme système de stockage de fichier.

> Note :<br/>
Le profil recommandé est jcloud avec swift.

Configurer et démarrer le service tomcat, afin de démarrer l'application __LinShare__ :
```bash
systemctl enable tomcat
systemctl start tomcat
```

Afin de vérifier le fonctionnement de __LinShare__ consulter les fichiers des journaux (logs) :
```bash
tail -f /var/log/tomcat/catalina.out
```

En fin d’un démarrage correct du service, le message suivant devrait apparaître :
```
org.apache.coyote.http11.Http11Protocol start
INFO: Démarrage de Coyote HTTP/1.1 sur http-8080
org.apache.catalina.startup.Catalina start
INFO: Server startup in 23151 ms
```

### <a name="firstAccess">Premier accès</a>

Le service __LinShare__ est désormais accessible aux adresses suivantes :

Pour l’interface utilisateur :
  * http://linshare-user.local/linshare

![linshare-user-000002010000047E01400157A9D6C9G6](../../img/linshare-user-000002010000047E01400157A9D6C9G6.png)

Pour l’interface d’administration :
  * http://linshare-admin.local/

Voici les identificants par défaut du compte administrateur système :
  * Identifiant : root@localhost.localdomain
  * Mot de passe : adminlinshare

Veuiller changer le mot de passe depuis l'interface d'administration.

> Note :<br/>
  Il n'est pas possible d'ajouter d'autres utilisateurs standards LinShare en local sans LDAP. Voir la section dédiée à la configuration du LDAP dans la [paramétrage applicatif](../administration/linshare-admin.md).

![linshare-admin-000002010000047E01400157A9D6C9G6](../../img/linshare-admin-000002010000047E01400157A9D6C9G6.png)
