# Setting up a server

## Create a server

Create an Ubuntu 21.04 VPS on a VPS provider like DigitalOcean or Vultr.
Choose a name for the server and a username for an admin user. Provide an
SSH key.

For this example, the server name is `denotastic` and the username is
`dinosaurio`.

The admin user is a user who is not root, but who can run sudo. This has
also been called a wheel user. I prefer to require a password for running
sudo, but make sure this password isn't shared with anything else, because
sometimes scripts ask for a sudo password. Usually they can't store it, and
I will exit out and try running as sudo in most cases, but it's always
better to use separate passwords for things. I store this password in my
password manager.

## Add to ssh-config

Add a couple entries to your `~/.ssh/config` on your local machine:

```
Host denotastic
  Hostname x.x.x.x
  User dinosaurio

Host denotastic-root
  Hostname x.x.x.x
  User root
```

Replace:

- `x.x.x.x` with your server's IP address
- `denotastic` with a custom name for your server
- `dinosaurio` with a custom username for your wheel user

Then you can just run `ssh denotastic-root` (with your custom servername)
to log in as root, and once your wheel user is set up, you can log in as
your wheel user with `ssh denotastic`.

## Update Packages and Install Oh My Zsh

Now log in as your root user:

```bash
ssh denotastic-root
```

Run:

```bash
apt update && apt upgrade -y
```

If it asks you to resolve a config file issue, choose either

- Install the package maintainer's version
- Keep the version currently installed

I tend to choose "Keep the version currently installed".

Once that's done, restart your VPS:

```bash
shutdown -r now
```

## Create Admin User

Log back in:

```bash
ssh denotastic-root
```

Now, generate a password, and create your admin user. Run this command.
It will prompt for your password.

```bash
adduser dinosaurio
```

Replace `dinosaurio` with your admin username.

The user needs to be in the admin group in order to be able to use `sudo`
to run commands as root. Run this command to add the user to the admin
group:

```bash
usermod -a -G admin dinosaurio
```

## Enabling direct login as the admin user

After the last command, you are still logged in as root. Now, to make
it possible to directly log in as the admin user, we need to copy the
`authorized_keys` from the root user, that were set when creating the
instance, to the admin user.

Run this, replacing `dinosaurio` with your custom username:

```bash
mkdir /home/dinosaurio/.ssh
cp -r ~/.ssh/authorized_keys /home/dinosaurio/.ssh/
chown -R dinosaurio:dinosaurio /home/dinosaurio/.ssh/
```

Now exit out of your ssh session:

```bash
exit
```

...and ssh as your admin user:

```bash
ssh denotastic
```

## Install unzip for Deno

Deno's installer requires unzip, which isn't installed by default, to be
installed.

```bash
sudo apt install -y unzip
```

## Install Deno

The following installs Deno, adds the configuration to `~/.bashrc`
so it's ready when you start your shell, and does the configuration so
you can use it right away.

Inspect this command below, paste it into the terminal, and hit enter:

```bash
curl -fsSL https://deno.land/x/install/install.sh | sh

cat <<'EOF' >> ~/.bashrc
export DENO_INSTALL="$HOME/.deno"
export PATH="$DENO_INSTALL/bin:$PATH"
EOF

export DENO_INSTALL="$HOME/.deno"
export PATH="$DENO_INSTALL/bin:$PATH"
```

## Install Docker

Install Docker CE and Docker Compose, following the directions on the Docker site:

- [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/) -
  add `-y` to the `apt-get install` commands
- [Install Docker Compose](https://docs.docker.com/compose/install/) -
  Select Linux from the list of tabs

## Clone the repo

Next, clone the repo:

```bash
mkdir ~/macchiato
cd ~/macchiato
git clone https://github.com/ResourcesCo/macchiato.git
```

## Unpack files from this document

This document contains some embedded files, which need to be unpacked.

To unpack them, use a script from this project,
[md_unpack_simple](https://deno.land/x/md_unpack_simple). You can inspect the
code on the link within [deno.land/x](https://deno.land/x). **deno.land** makes
sure that once a version is published, it can't change, so include the version
number when installing or running it!

To see more about **md_unpack_simple**, read the [source Markdown document from
which it was generated](https://github.com/ResourcesCo/macchiato/blob/main/scripts/md_unpack_simple.md).

Also be sure to pay attention to the permissions.

Install it using this command:

```bash
deno install --unstable --allow-read=. --allow-write=. https://deno.land/x/md_unpack_simple@0.0.2/mod.ts
```

This installs **md_unpack_simple** to your path.

The permissions `--allow-read=.` and `--allow-write=.` and the option `--unstable`
are used exactly as they are each time **md_unpack_simple** is run. This means
that reading and writing will be restricted to the current directory.

Now, make a directory and unpack this document, which was cloned in the previous
step, to it:

```bash
mkdir -p ~/macchiato/server/setup
cd ~/macchiato/server/setup
cat ~/macchiato/macchiato/server/setup.md | md_unpack_simple
```

Then run `find .` to get a list of all the files. The files will be everything in
here that is preceded by a line that starts with `` ##### ` `` - that is, five
hash marks and a backquote. It's an inline code block inside of a level 5 header.
This was chosen because it's nested deeply enough that it will rarely interfere
with the outline structure of the document, but is a header so it has deep links.

## Run Caddy with Docker

[Caddy](https://caddyserver.com/) is a web server implemented in
[Go](https://go.dev/). It pioneered [Automatic HTTPS]() - getting a HTTPS
certificate from LetsEncrypt automatically.

We're going to run tiny apps on subdomains, including code notebooks and code
playgrounds/sandboxes, with each app having access to certain resources. Each
subdomain gets its own cookies and LocalStorage, which makes it so resources
only need to be made available within a limited context. If your app doesn't
need any secrets, nor produces output that is used for something important,
you may run it without checking the code. If, on the other hand, it has
write access to private repositories, you'll want to carefully check that you
trust the code (including dependencies).

LetsEncrypt has a quota of up to 50 SSL certificates per week. It also has
wildcard certificates, but those are slightly more involved to set up. Here
we'll start with individual certificates and later move to wildcard
certificates.

First, let's run Caddy with its defaults using docker-compose and access it with
its IP address. Here is the file that defines the Docker Compose Environment. It
simply grabs the
[caddy image](https://hub.docker.com/_/caddy) from
[Docker Hub](https://www.docker.com/products/docker-hub) and runs it with ports
80 and 443 exposed.

##### `caddy-default/docker-compose.yml`

```yaml
version: "3.9"

services:
  caddy:
    image: caddy:latest
    ports:
      - "80:80"
      - "443:443"
    restart: always
```

Change into the directory and start the Docker Compose environment in detached
mode with Docker compose:

```bash
cd ~/macchiato/server/setup/caddy-default
sudo docker-compose up --detach
```

This will run the container in `docker-compose.yml` in the background. If you
leave off `--detach`, it will show it in the foreground, with the logs, and you
can stop it with `Ctrl-C`. To see the logs when it's in detached mode, run:

```
sudo docker-compose logs
```

You can add `--help` to learn to use Docker Compose's log command. It accepts:

- `-f` to stream the logs to your conosle
- `-t` to show timestamps
- `--tail=<n>` to show a certain number of lines (`--tail=50` for 50 lines)

Now, look up your IP address of your server, on your host or in `~/.ssh/config`,
and open the url in your browser:

```
http://x.x.x.x/
```

It shows the introduction page for Caddy.

Next we'll start a simple app on a subdomain.

Now, stop the compose environment so the ports will be open when we start a
new one in the next step:

```
sudo docker-compose down
```

