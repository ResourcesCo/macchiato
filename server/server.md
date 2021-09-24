# Server

This contains a configuration for a server with the following, set up with
docker-compose (see [setup](./setup.md) for how to create the server with
Docker Compose):

- mariadb
- ubuntu shell
- wordpress with IndieAuth
- caddy
- gitea
- deno-based server for API & creating sites on subdomains
- caddy w/ with wildcard SSL for unlimited subdomains

These files can be extracted with `md_unpack_simple`:

```
mkdir -p ~/macchiato/server/server
cd ~/macchiato/server/server
cat ~/macchiato/macchiato/server/server.md | md_unpack_simple
```

## Step 1: Create, connect, and restore database

This has templates that can be copied and edited:

```bash
cp step1/.env .env
cp step1/entrypoint.sh entrypoint.sh
cp step1/Dockerfile.shell Dockerfile.shell
cp step1/docker-compose.yml docker-compose.yml
```

To begin, generate a random root password for MariaDB, as well as three
passwords for the `WordPress`, `gitea`, and `app` users.

One way to generate them is with Deno:

##### `step1/generate-random-passwords.ts`

```ts
import { cryptoRandomString } from 'https://deno.land/x/crypto_random_string@1.1.0/mod.ts'
for (let i=0; i < 4; i++) {
  console.log(cryptoRandomString({length: 30, type: 'alphanumeric'}))
}
```

To run it, generating four 30-character long alphanumeric passwords:

```bash
$ deno run step1/generate-random-passwords.ts
```

Before using a code module to generate random passwords, make sure it uses a
cryptographic random generator. [crypto.getRandomValues](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/getRandomValues),
which [crypto_random_string uses](https://deno.land/x/crypto_random_string@1.1.0/cryptoRandomString.ts#L38)
is a cryptographic random generator; [Math.random](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random) is not.

Now, add these to `.env`:

##### `step1/.env`

```
MARIADB_ROOT_PASSWORD=
MARIADB_WORDPRESS_PASSWORD=
MARIADB_GITEA_PASSWORD=
MARIADB_APP_PASSWORD=
```

Now, let's get the mariadb and shell containers, and use the shell
container to add the three databases and users.

##### `step1/entrypoint.sh`

```bash
cp -n /setup/zshrc /root/.zshrc
cp -n -r /setup/oh-my-zsh /root/.oh-my-zsh

exec "$@"
```

##### `step1/Dockerfile.shell`

```docker
FROM ubuntu:rolling

ARG OHMYZSH_URL=https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh

RUN apt-get update
RUN apt-get upgrade -y
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y -qq mysql-client git neovim curl build-essential zsh unzip
RUN curl -fsSL https://deno.land/x/install/install.sh | DENO_INSTALL=/ sh
RUN sh -c "$(curl -fsSL $OHMYZSH_URL)" "" --unattended
RUN curl https://rclone.org/install.sh | bash

RUN mkdir -p /setup
ADD entrypoint.sh /setup

RUN cp ~/.zshrc /setup/zshrc
RUN cp -r ~/.oh-my-zsh /setup/oh-my-zsh
RUN chmod +x /setup/entrypoint.sh
WORKDIR /root
ENTRYPOINT ["/setup/entrypoint.sh"]
RUN ["/bin/zsh"]
```

The mariadb container has a docker volume for `/var/lib/mysql` which
contains all the data. The shell container has docker volumes for
`/home` and `/etc`. These persist between rebuilds and restarts of
the services.

##### `step1/docker-compose.yml`

```yaml
version: '3.9'
services:
  mariadb:
    image: mariadb:latest
    volumes:
      - mariadb_data:/var/lib/mysql
    command:
      - '--character-set-server=utf8mb4'
      - '--collation-server=utf8mb4_unicode_ci'
    environment:
      MARIADB_ROOT_PASSWORD: "${MARIADB_ROOT_PASSWORD}"
    restart: always
  shell:
    build:
      context: ./
      dockerfile: Dockerfile.shell
    volumes:
      - shell_home:/root
      - shell_etc:/etc
    restart: 'no'
    init: true
volumes:
  mariadb_data: {}
  shell_home: {}
  shell_etc: {}
```

To start the mariadb server and build the shell container, run:

```bash
sudo docker-compose up --detach
```

The shell container will exit right after it starts. This is desired, and
restart is set to `no` so it won't keep trying to start it. This is because
it's for running one-off containers. By default, these won't have the API
keys or access to volumes of other key services, but in the one off commands
with `-e` for environment variables and `-v` for volumes, it can be given
access to credentials and data as needed.

The `mariadb` container, on the other hand, has `restart` set to `always`,
because it is a service that we'll want to keep running and to automatically
be started when restarting the host. The same will apply for caddy and
wordpress because those will power your website.

Now, we'll start a shell container to use the root mariadb account to create
the three mariadb databases. We'll need the environment variables for the
mariadb accounts. The entrypoint for the [mariadb docker image](https://github.com/MariaDB/mariadb-docker/blob/master/docker-entrypoint.sh#L317)
provides an example. Each user will only have access to its database.

First, run a the `shell` service with the four environment variables. This
will bring you into a shell:

```bash
sudo bash -c 'source .env && docker-compose run \
-e MARIADB_ROOT_PASSWORD=$MARIADB_ROOT_PASSWORD \
-e MARIADB_WORDPRESS_PASSWORD=$MARIADB_WORDPRESS_PASSWORD \
-e MARIADB_GITEA_PASSWORD=$MARIADB_GITEA_PASSWORD \
-e MARIADB_APP_PASSWORD=$MARIADB_APP_PASSWORD \
shell /bin/zsh'
```

In this shell, you can now log into MySQL as root:

```bash
MYSQL_PWD=$MARIADB_ROOT_PASSWORD mysql -u root -h mariadb
```

Now, create the WordPress user and database and grant the WordPress user access
to the WordPress database:

```bash
MYSQL_PWD=$MARIADB_ROOT_PASSWORD mysql -u root -h mariadb <<< "CREATE DATABASE wordpress"
MYSQL_PWD=$MARIADB_ROOT_PASSWORD mysql -u root -h mariadb <<< "CREATE USER 'wordpress'@'%' IDENTIFIED BY '$MARIADB_WORDPRESS_PASSWORD';"
MYSQL_PWD=$MARIADB_ROOT_PASSWORD mysql -u root -h mariadb <<< "GRANT ALL ON wordpress.* TO 'wordpress'@'%';"
```

Try logging in as the WordPress user:

```bash
MYSQL_PWD=$MARIADB_WORDPRESS_PASSWORD mysql -u wordpress -h mariadb wordpress
```

Create the gitea and app users:

```bash
MYSQL_PWD=$MARIADB_ROOT_PASSWORD mysql -u root -h mariadb <<< "CREATE DATABASE gitea"
MYSQL_PWD=$MARIADB_ROOT_PASSWORD mysql -u root -h mariadb <<< "CREATE USER 'gitea'@'%' IDENTIFIED BY '$MARIADB_GITEA_PASSWORD';"
MYSQL_PWD=$MARIADB_ROOT_PASSWORD mysql -u root -h mariadb <<< "GRANT ALL ON gitea.* TO 'gitea'@'%';"
MYSQL_PWD=$MARIADB_ROOT_PASSWORD mysql -u root -h mariadb <<< "CREATE DATABASE app"
MYSQL_PWD=$MARIADB_ROOT_PASSWORD mysql -u root -h mariadb <<< "CREATE USER 'app'@'%' IDENTIFIED BY '$MARIADB_APP_PASSWORD';"
MYSQL_PWD=$MARIADB_ROOT_PASSWORD mysql -u root -h mariadb <<< "GRANT ALL ON app.* TO 'app'@'%';"
```

If you have data to restore, you can download it using rclone, which is in the
shell container, and restore it using the mysql command.

To log into rclone, use `rclone config`, and choose environment variables for the
secrets, so they aren't left lying around.

These can be added to your `.env`, which if you followed [setup.md]('./setup.md'),
are only accessible by root.

Once it's downloaded, you can use `gunzip`, `bunzip2`, `unzip`, and `tar` which are
also included in the shell to get a `.sql` file, and restore it with (replacing
`wordpress.sql` with the name of your database):

```bash
MYSQL_PWD=$MARIADB_WORDPRESS_PASSWORD mysql -u wordpress -h mariadb wordpress < wordpress.sql
```

When you're done, type `exit` and enter to exit the shell container.

## Step 2: Set up WordPress and Caddy

To set up WordPress and Caddy, we'll add two containers. For each, we'll add
a configuration file. There are also three volumes to add.

##### `step2/docker-compose.yml`

```yaml{21-44,49-51}
version: '3.9'
services:
  mariadb:
    image: mariadb:latest
    volumes:
      - mariadb_data:/var/lib/mysql
    command:
      - '--character-set-server=utf8mb4'
      - '--collation-server=utf8mb4_unicode_ci'
    environment:
      MARIADB_ROOT_PASSWORD: "${MARIADB_ROOT_PASSWORD}"
    restart: always
  shell:
    build:
      context: ./
      dockerfile: Dockerfile.shell
    volumes:
      - shell_home:/root
      - shell_etc:/etc
    restart: 'no'
    init: true
  wordpress:
    image: wordpress:latest
    volumes:
      - wordpress_site:/var/www/html
      - ./php.conf.ini:/usr/local/etc/php/conf.d/conf.ini
    depends_on:
      - mariadb
    environment:
      WORDPRESS_DB_HOST: mariadb
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: "${MARIADB_WORDPRESS_PASSWORD}"
      WORDPRESS_DB_NAME: wordpress
    restart: always
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
volumes:
  mariadb_data: {}
  shell_home: {}
  shell_etc: {}
  wordpress_site: {}
  caddy_data: {}
  caddy_config: {}
```

##### `step2/Caddyfile`

```
example.com {
  reverse_proxy wordpress:80
}
www.example.com {
  redir https://example.com{uri}
}
```

##### `step2/php.conf.ini`

[From github.com/nezhar/wordpress-docker-compose](https://github.com/nezhar/wordpress-docker-compose/blob/master/docker-compose.yml)

```ini
file_uploads = On
memory_limit = 500M
upload_max_filesize = 30M
post_max_size = 30M
max_execution_time = 600
```

Now, copy the files into your directory, overwriting `docker-compose.yml`. If you
changed `docker-compose.yml`, just copy the relevant lines from
`step2/docker-compose.yml` and add them to `docker-compose.yml`.

```bash
cp step2/docker-compose.yml docker-compose.yml
cp step2/php.conf.ini php.conf.ini
cp step2/Caddyfile Caddyfile
```

Edit `Caddyfile` to have a domain name for your blog, and edit the DNS records
for your domain name to point to your VPS.

Now, bring it up:

```bash
sudo docker-compose up -d
```

### Restoring an existing WordPress site

If you are restoring an existing WordPress site, follow these steps. If not,
skip to the end of this section.

Log into shell with the WP WordPress password:

```bash
sudo bash -c 'source .env && docker-compose run \
-e MARIADB_WORDPRESS_PASSWORD=$MARIADB_WORDPRESS_PASSWORD \
-v server_wordpress_site:/server_wordpress_site \
shell /bin/zsh
```

...and in another terminal session, copy the files into the shell's `/root` directory
(the shell must be running in the background, or another container that has
`shell_home` mounted, to copy with `docker cp`):

```bash
sudo docker ps
# get the ID of the shell container, and replace $CONTAINER_ID with it
sudo docker cp blog.sql $CONTAINER_ID:/root
sudo docker cp blog-site.tar.bz2 $CONTAINER_ID:/root
```

Now, back in your shell, in the original terminal session, restore your database
and site:

```bash
mysql --host=mariadb -u wordpress --password=$MARIADB_WORDPRESS_PASSWORD wordpress < blog.sql
cd /server_wordpress_site
tar xjf ~/blog-site.tar.bz2
# delete wp-config.php since WordPress for docker will configure the database
rm wp-config.php
```

### Setting up WordPress

Now, try going to the domain you're using for the site. If it isn't working, you might
try checking the Docker logs, and making sure your DNS settings have taken effect.

If you didn't restore an existing WordPress site, it should show the WordPress setup
screen. Follow the instructions.

### Installing the WordPress IndieAuth and MicroPub Plugins

Install and activate the WordPress IndieAuth Plugin by going to Plugins > Add Plugin
and searching for them. Then activate it and try posting with
[Quill](https://quill.p3k.io/).

## Step 3: Set up Gitea

##### `step3/docker-compose.yml`

```yaml
version: '3.9'
services:
  mariadb:
    image: mariadb:latest
    volumes:
      - mariadb_data:/var/lib/mysql
    command:
      - '--character-set-server=utf8mb4'
      - '--collation-server=utf8mb4_unicode_ci'
    environment:
      MARIADB_ROOT_PASSWORD: "${MARIADB_ROOT_PASSWORD}"
    restart: always
  shell:
    build:
      context: ./
      dockerfile: Dockerfile.shell
    volumes:
      - shell_home:/root
      - shell_etc:/etc
    restart: 'no'
    init: true
  wordpress:
    image: wordpress:latest
    volumes:
      - wordpress_site:/var/www/html
      - ./php.conf.ini:/usr/local/etc/php/conf.d/conf.ini
    depends_on:
      - mariadb
    environment:
      WORDPRESS_DB_HOST: mariadb
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: "${MARIADB_WORDPRESS_PASSWORD}"
      WORDPRESS_DB_NAME: wordpress
    restart: always
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
  gitea:
    image: gitea/gitea:latest
    environment:
      USER_UID: 1000
      USER_GID: 1000
      GITEA__database__DB_TYPE: mysql
      GITEA__database__HOST: mariadb:3306
      GITEA__database__NAME: gitea
      GITEA__database__USER: gitea
      GITEA__database__PASSWD: "${MARIADB_GITEA_PASSWORD}"
      DISABLE_REGISTRATION: 'true'
      INSTALL_LOCK: 'true'
      DISABLE_SSH: 'true'
      ROOT_URL: https://gitea.example.com
    restart: always
    volumes:
      - gitea_data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
volumes:
  mariadb_data: {}
  shell_home: {}
  shell_etc: {}
  wordpress_site: {}
  caddy_data: {}
  caddy_config: {}
  gitea_data: {}
```

##### `step3/Caddyfile`

```
gitea.example.com {
  reverse_proxy wordpress:80
}
```

Copy the new Docker Compose file and append the `Caddyfile`:

```bash
cp step3/docker-compose.yml docker-compose.yml
cat step3/Caddyfile >> Caddyfile
```

Add a DNS entry for the gitea host, and put the host in your `Caddyfile` and `docker-compose.yml`.

Now start it:

```bash
sudo docker-compose up -d
```

Go to the URL, and it should show the login page.

Now, to create an admin user, run a command with the `gitea` service:

```bash
sudo docker-compose run gitea gitea admin user create --admin --email yourname@example.com --username yourusername --random-password
```

The password will be printed to the screen and you can log in as that user.

## Step 4: A Deno app that creates sites on subdomains

Add a wildcard DNS entry, perhaps at `*.sandbox.yourdomain.com`, and point it
to the server, as well as a regular DNS entry to the app that will create the
sandboxes (`sandbox.yourdomain.com`), also pointing to the server.

This time, manually add the service to `docker-compose.yml`. If you're using
vim, you can type `:r step4/service.yml` to insert it where the cursor is.

##### `step4/service.yml`

```yml
app:
  build:
    context: ./
    dockerfile: Dockerfile.app
  environment:
    APP_DB_HOST: mariadb
    APP_DB_USER: wordpress
    APP_DB_PASSWORD: "${MARIADB_WORDPRESS_PASSWORD}"
    APP_DB_NAME: wordpress
  depends_on:
    - mariadb
  restart: always
```

Copy the Dockerfile:

```bash
cp step4/Dockerfile.app Dockerfile.app
```

##### `step4/Dockerfile.app`

```docker
FROM ubuntu:rolling

RUN apt-get update
RUN apt-get upgrade -y
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y -qq curl unzip
RUN curl -fsSL https://deno.land/x/install/install.sh | DENO_INSTALL=/ sh

ADD ./app/mod.ts /app/mod.ts
WORKDIR /app
RUN deno cache /app/mod.ts

EXPOSE 8080
CMD ["/usr/bin/deno", "run", "--allow-net=:8080", "/app/mod.ts"]
```

Point the subdomains to by appending rules to your `Caddyfile` and editing it:

```bash
cat step4/Caddyfile >> Caddyfile
```

##### `step4/Caddyfile`

```bash
sandbox.example.com {
  reverse_proxy app:8080
}
page1.sandbox.example.com {
  reverse_proxy app:8080
}
page2.sandbox.example.com {
  reverse_proxy app:8080
}
page3.sandbox.example.com {
  reverse_proxy app:8080
}
```

Note that this doesn't have the wildcard. This is to prevent it from using
up your SSL certificate quota if someone goes to dozens of subdomains in the
DNS wildcard. We'll fix it in step 5 by getting a wildcard SSL certificate.

Now, copy the app directory out of `step4`:

```bash
cp -r step4/app app
```

##### `step4/app/mod.ts`

```ts
const server = Deno.listen({ port: 8080 })

async function handleRequest(request: Deno.RequestEvent) {
  request.respondWith(
    new Response(`🦕`, {
      status: 200,
    }),
  );
}

async function serve(conn: Deno.Conn) {
  const httpConn = Deno.serveHttp(conn)

  for await (const request of httpConn) {
    handleRequest(request)
  }
}

for await (const conn of server) {
  serve(conn)
}
```

Bring it up:

```bash
sudo docker-compose up --detach
```