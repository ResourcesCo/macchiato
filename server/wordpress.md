# WordPress

This sets up MySQL and WordPress, with a couple of auxillary Docker Compose
services: one for WordPress CLI and one for rclone, which can be used to
download a WordPress backup from a cloud storage service and restore an
existing site from it.

## The Compose Environment

The Compose Environment has three containers which run on `docker-compose up`
and two that only run with `docker-compose start` or when the `setup` profile
is selected. It keeps WordPress behind SSL, and the Caddy admin endpoint
hidden from the public, inside the Docker Compose network.

##### `docker-compose.yml`

```yaml
version: '3.9'

services:
  caddy:
    image: caddy:latest
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    ports:
      - 80:80
      - 443:443
    restart: always
  mysql:
    image: mysql:8
    volumes:
      - mysql_data:/var/lib/mysql
      - backup:/backup
    environment:
      MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASSWORD}"
      MYSQL_DATABASE: wordpress_site
      MYSQL_USER: wordpress_site
      MYSQL_PASSWORD: "${WORDPRESS_DB_PASSWORD}"
    command:
      - '--character-set-server=utf8mb4'
      - '--collation-server=utf8mb4_unicode_ci'
  wordpress:
    image: wordpress:latest
    volumes:
      - wordpress_site:/var/www/html
      - ./php.conf.ini:/usr/local/etc/php/conf.d/conf.ini
      - backup:/backup
    depends_on:
      - mysql
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_USER: wordpress_site
      WORDPRESS_DB_PASSWORD: "${WORDPRESS_DB_PASSWORD}"
      WORDPRESS_DB_NAME: wordpress_site
  wordpress-cli:
    profiles:
      - setup
    image: wordpress:cli
    volumes:
      - wordpress_site:/var/www/html
      - ./php.conf.ini:/usr/local/etc/php/conf.d/conf.ini
      - backup:/backup
    depends_on:
      - mysql
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_USER: wordpress_site
      WORDPRESS_DB_PASSWORD: "${WORDPRESS_DB_PASSWORD}"
      WORDPRESS_DB_NAME: wordpress_site
  rclone:
    profiles:
      - setup
    image: rclone/rclone:latest
    volumes:
      - backup:/backup
      - rclone_config:/config/rclone
volumes:
  mysql_data: {}
  wordpress_site: {}
  caddy_data: {}
  caddy_config: {}
  backup: {}
  rclone_config: {}
```

### Configuration

The .env file is where to add custom passwords.

##### `.env.example`

```
MYSQL_ROOT_PASSWORD=
MYSQL_PASSWORD=
WORDPRESS_DB_PASSWORD=
```

The Caddyfile has the host, which needs to be set to the host of your site.

##### `Caddyfile.example`

```
blog.example.com {
  reverse_proxy wordpress:80
}
```

##### `php.conf.ini`

[From github.com/nezhar/wordpress-docker-compose](https://github.com/nezhar/wordpress-docker-compose/blob/master/docker-compose.yml)

```ini
file_uploads = On
memory_limit = 500M
upload_max_filesize = 30M
post_max_size = 30M
max_execution_time = 600
```

### Deploying

First, point a domain to your server - perhaps a subdomain.

Copy `Caddyfile.example` to `Caddyfile` and set the domain.

Copy `.env.example` to `.env` and make it accessible only by root:

```bash
cp .env.example .env
sudo chown root:root .env
sudo chmod 600 .env
```

Then open it with vi and set the passwords.

Then, bring your site up with:

```bash
sudo docker-compose up
```

### Restoring from backup

First, bring rclone up. This will build the container. By default, it doesn't
build it, because it's in a custom profile (`setup`).

```bash
sudo docker-compose up rclone --no-start
```

Now that it's built, run `rclone config` to configure it:

```bash
sudo docker-compose run rclone config
```

It will ask what to do. Choose `n` to create a new remote, then follow the
instructions.

Once the remote is set up, list files in the remote:

```bash
sudo docker-compose run rclone ls myremote:
```

When you've found the directory you want, sync them to a directory within `/backup`:

```bash
sudo docker-compose run rclone sync myremote:mybucket/backup20210909 /backup/backup20210909
```

The WordPress docker image has `tar` and `bunzip2` and `bash`, so it can be
logged into in order to decompress the sql data for MySQL and restore the `www`
data.

The MySQL docker image has the `mysql` command which can be used to restore the
database.

Another route would be to create a new Docker image and install the commands needed.

To decompress the sql backup (using your filename and the right decompression tool,
such as `gunzip` or `bunzip2`):

```bash
sudo docker-compose run --entrypoint /bin/bash wordpress
cd /backup/backup20210909
bunzip2 blog.sql.bz2
exit
```

To restore the database:

```bash
sudo docker-compose run --entrypoint /bin/bash mysql
cd /backup/backup20210909
mysql --host=mysql -u root -p wordpress_site < blog.sql
# *enter root password from .env*
exit
```

To restore the wordpress site:

```bash
sudo docker-compose run --entrypoint /bin/bash wordpress
cd /backup/backup20210909
tar xjf blog.tar.bz2
cd blog
cp /var/www/html/wp-config.php wp-config.php
rm -r /var/www/html/* /var/www/html/.*
mv * /var/www/html/
mv .* /var/www/html/
chown -R www-data:www-data /var/www/html
```

### Setting up backups

Backups can be set up by making a cron job on the host that does `mysqldump` on
the mysql container and runs `tar` on the wordpress container to copy the database
dump and the site to `/backup` and runs `rclone copy` on the rclone container.
It could also run this inside a docker container which has access to the docker
socket.