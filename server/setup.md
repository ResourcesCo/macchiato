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


