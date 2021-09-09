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
volumes:
  mysql_data: {}
  wordpress_site: {}
  caddy_data: {}
  caddy_config: {}
  backup: {}
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

##### `Caddyfile`

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

Then, bring your site up with:

```bash
sudo docker-compose up
```

TODO: instructions for downloading and restoring sql and a WordPress site from
a backup