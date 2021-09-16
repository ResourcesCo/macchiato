# Server

This contains a configuration for a server with the following, set up with
docker-compose (see [setup](./setup.md) for how to create the server with
Docker Compose):

- mariadb
- ubuntu shell
- wordpress with IndieAuth
- caddy with wildcard SSL
- gitea
- deno-based server for API & creating sites on wildcard subdomains

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
container to add the three mysql databases and users.

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
    init: true
    restart: 'no'
volumes:
  mariadb_data: {}
  shell_home: {}
  shell_etc: {}
```

To start the mysql server and build the shell container, run:

```bash
sudo docker-compose up --detach
```

Now, run the shell container:

```bash
sudo docker-compose run shell
```

