# Installation och konfiguration av grundläggande Apache webbserver

### Steg 0: Prerequisites

Se till att du har följande innan du börjar:

- En fungerande Ubuntu Server installation på en VM
- En fungerande SSH koppling till din Ubuntu Server
- Grundläggande Linux CLI kunskap

### Steg 1: Installation av Apache

Installera Apache och dess dependencies med följande kommandon:

```bash
sudo apt update
sudo apt install apache2
```

Efter dessa kommandon har kört så kan vi testa att det fungerar genom att söka på våran IP address i våran webbläsare.

_Tips:_ IP addressen får du ut genom att köra `ip a` i terminalen.

### Steg 2: Skapa din egen webbsida

Apache kommer installerat med en egen default hemsida (Den som förhoppningsvis visades när du sökte på din IP i webbläsaren). Vi kan ändra dess innehåll i `/var/www/html` eller dess inställningar i `/etc/apache2/sites-enabled/000-default.conf`

Vi kan även ändra hur Apache hanterar inkommande förfrågningar genom att ändra dess Virtual Hosts fil. I det här steget kommer vi skapa vår egen konfiguration på blog.example.com

Börja med att skapa en ny mapp till våran webbsida i `/var/www/` med följande kommando:

```bash
sudo mkdir /var/www/blog/
```

Notera att ägaren av mappen kan bli root, så för att läsa och ändra i mappen måste du antingen köra `chmod` eller `chown` eller så använder du dig av `sudo` kommandot när du ska göra ändringar.

Navigera sedan in i mappen och skapa en ny html fil.

```bash
cd /var/www/blog/
nano index.html
```

Klistra sedan in följande HTML kod:

```html
<html>
  <head>
    <title>I love Apache!</title>
  </head>
  <body>
    <p>I'm running this website on an Ubuntu Server!</p>
  </body>
</html>
```

Nu måste vi konfigurera så att vår nya hemsida nås när du surfar till blog.example.com

### Steg 3: Konfigurera VirtualHost filen

Börja med att gå in i konfigurationfilens katalog.

```bash
cd /etc/apache2/sites-available/
```

Eftersom apache kommer med en default VirtualHost fil så kan vi använda den som grund

```bash
sudo cp 000-default.conf blog.conf
sudo nano blog.conf
```

Redigera config filen med:

```bash
sudo nano blog.conf
```

Ändra ServerAdmin till din email

```
ServerAdmin yourname@example.com
```

Ändra DocumentRoot så att den pekar på katalogen för vår nya html fil

```
DocumentRoot /var/www/blog/
```

Default filen kommer inte med ett ServerName, så det får vi lägga till

```
ServerName blog.example.com
```

Detta gör att man når rätt sida när man söker på `blog.example.com`. Nu återstår bara att aktivera vår webbsida.

### Steg 4: Aktivera nya VirtualHost filen

Aktivera sidan genom att köra följande kommando i konfiguration filens katalog:

```bash
sudo a2ensite blog.conf
```

Du bör se följande output:

```
Enabling site blog.
To activate the new configuration, you need to run:
  service apache2 reload
root@ubuntu-server:/etc/apache2/sites-available#
```

För att ladda sidan så måste vi starta om Apache. Detta görs genom att köra följande kommando:

```bash
systemctl reload apache2
```

Nu bör du kunna komma åt din hemsida.

### Extra om loggfiler

Två viktiga loggfiler att hålla koll på är

1. error.log: Error loggen innehåller information om alla fel och varningar som din server stöter på. Det kan vara t.ex konfigurationsfel, överbelastningar eller problem med filrättigheter.

2. access.log: Access loggen registrerar information om varje förfrågan som görs till servern. Den innehåller information om vem som gjorde förfrågan, när förfrågan gjordes och vilken resurs som efterfrågades. Access loggen är väsentlig för att analysera användarbeteende och trafikmönster av din applikation.

# Bonus: Installera och konfigurera wordpress på din Apache webbserver

### Steg 1: Installera dependencies

För att köra wordpress så behöver du PHP och MySQL installerat. Kör följande kommando:

```bash
sudo apt update
sudo apt install ghostscript \
                 libapache2-mod-php \
                 mysql-server \
                 php \
                 php-bcmath \
                 php-curl \
                 php-imagick \
                 php-intl \
                 php-json \
                 php-mbstring \
                 php-mysql \
                 php-xml \
                 php-zip
```

### Steg 2: Installera WordPress

Vi kommer installera wordpress från `wordpress.org` istället för att använda apt paketet då detta är den föreslagna metoden från WordPress. Skapa först installations katalogen och installera sedan WordPress.

Kör följande kommando:

```bash
sudo mkdir -p /srv/www
sudo chown www-data: /srv/www
curl https://wordpress.org/latest.tar.gz | sudo -u www-data tar zx -C /srv/www
```

### Steg 3: Konfigurera Apache för WordPress

Börja med att skapa en sida för Wordpress med följande kommando:

```bash
sudo nano /etc/apache2/sites-available/wordpress.conf
```

Fyll sedan conf filen med följande rader:

```
<VirtualHost *:80>
    DocumentRoot /srv/www/wordpress
    <Directory /srv/www/wordpress>
        Options FollowSymLinks
        AllowOverride Limit Options FileInfo
        DirectoryIndex index.php
        Require all granted
    </Directory>
    <Directory /srv/www/wordpress/wp-content>
        Options FollowSymLinks
        Require all granted
    </Directory>
</VirtualHost>
```

Navigera sedan till webbsidans katalog och aktivera sidan

```bash
sudo a2ensite wordpress
```

Aktivera sedan URL rewriting

```bash
sudo a2enmod rewrite
```

Inaktivera sedan eventuella andra sidor

```bash
sudo a2dissite <site-name>
```

Ladda sedan om Apache för att applicera ändringar till konfigurationen

```bash
systemctl reload apache2
```

### Steg 4: Konfigurera databas

För att köra WordPress så måste vi konfigurera en MySQL Databas.

Kör följande kommando och logga in med ditt lösenord för att starta MySQL.

```bash
sudo mysql -u root
```

Kör sedan följande kommandon. _OBS_ byt ut \<your-password> med ett faktiskt lösenord.

```sql
mysql> CREATE DATABASE wordpress;
Query OK, 1 row affected (0,00 sec)

mysql> CREATE USER wordpress@localhost IDENTIFIED BY '<your-password>';
Query OK, 1 row affected (0,00 sec)

mysql> GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,ALTER
    -> ON wordpress.*
    -> TO wordpress@localhost;
Query OK, 1 row affected (0,00 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 1 row affected (0,00 sec)

mysql> quit
Bye
```

Sätt sedan igång MySQL med

```bash
sudo systemctl start mysql
```

Kontrollera sedan status av MySQL

```bash
systemctl status mysql
```

### Steg 5: Koppla WordPress till databasen

Börja med att kopiera config mallen till `wp-config.php`

```bash
sudo -u www-data cp /srv/www/wordpress/wp-config-sample.php /srv/www/wordpress/wp-config.php

```

Sedan måste du sätta databasens inloggningsuppgifter till de som du konfigurerade tidigare.

_Byt inte ut \*\_here, men byt ut \<your-password> mot lösenordet du valt._

```bash
sudo -u www-data sed -i 's/database_name_here/wordpress/' /srv/www/wordpress/wp-config.php
sudo -u www-data sed -i 's/username_here/wordpress/' /srv/www/wordpress/wp-config.php
sudo -u www-data sed -i 's/password_here/<your-password>/' /srv/www/wordpress/wp-config.php
```

Öppna sedan konfigurationsfilen

```bash
sudo -u www-data nano /srv/www/wordpress/wp-config.php
```

Lokalisera sedan följande rader i filen:

```
define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );
```

Ta bort dessa rader och byt ut de med slumpmässigt genererade nycklar från denna länk https://api.wordpress.org/secret-key/1.1/salt/

### Steg 6: Konfigurera WordPress

Surfa in på din servers IP address. Du kommer nu bli efterfrågad om en titel till din nya webbsida, en email, ett användarnamn och ett lösenord.

Notera att dessa uppgifter endast är för inloggning till Wordpress och kommer inte ge access till din databas eller server. Välj nya inloggningsuppgifter som du inte använder till din databas eller server.

Du har nu en fungerande wordpress hemsida! Du kan nu göra inlägg, ta bort inlägg och mycket mer!
