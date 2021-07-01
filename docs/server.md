# server

*Initial server setup*

## Step 1: Create a VPS server and ssh into it as root

Go to a VPS provider like DigitalOcean or Vultr, and create a server with
at least 2GB of RAM and Fedora 34. Be sure to choose the option to log in
with an SSH key and upload your public key, because you'll be copying
your `~/.ssh/authorized_keys` so you don't need to log in as root.

## Step 2: Install Deno

SSH into the server, following the instructions from the provider.

You're logged in as root. The command to install Deno will be the first
command to run. I want you to be able to understand each instruction and
determine whether it will lead to building a system that you trust. For
the previous step, you can consider whether to trust Fedora and your VPS
provider. There is also ssh and your system, and whether you've protected
your SSH private key (`~/.ssh/id_rsa`).

For this command, you can first decide whether you want to work with Deno.
Deno is new, but I trust it because it was built by the author of Node.js,
which I also like, and because it has a strong design.

After that, you can either inspect the command, or check that it's correct,
or both. You can go to the README and verify that it's correct, and since
you trust Deno, run it. Or, you can see that it's downloading from
`deno.land` and if you know that `deno.land` is owned by Deno, you can
consider whether they'd have the wrong thing available at `https://deno.land/x/install/install.sh`. Since `/x` is for community
contributions, I don't think it would necessarily be paranoid to go to the
README link below and copy it from there instead.

Run this command from the
[Deno README](https://github.com/denoland/deno#install):

```bash
curl -fsSL https://deno.land/x/install/install.sh | sh
```

Alas, installing this gets an error:

> Error: unzip is required to install Deno (see: https://github.com/denoland/deno_install#unzip-is-required).

Unzip isn't installed. Let's install that.

[dnf](https://docs.fedoraproject.org/en-US/quick-docs/dnf/) is the package
manager for Fedora. Like with other package managers, you can search for
things and install them.

```bash
dnf search unzip
```

This will show that the package is, indeed, called `unzip`.

Run this to install it:

```bash
dnf install unzip
```

This will ask to confirm. You can skip it by passing `-y`.

Now try running the command to install Deno from the
[Deno README](https://github.com/denoland/deno#install) again:

```bash
curl -fsSL https://deno.land/x/install/install.sh | sh
```

It installs, but has a step it asks you to run manually.
We won't do it manually, but will do it by creating a script
and running deno directly from `/root/.deno/bin/deno`. Here
is the script. To save it to a file, you can:

- type `cat > deno-shell-setup.ts`
- press ENTER
- copy the contents of the script to your clipboard
- paste into your terminal window
- press ENTER to add a newline to the end of the file
- press CTRL-D (you can skip the previous step by [pressing 
  CTRL-D twice](https://stackoverflow.com/a/21261742) if you
  prefer not to have a newline at the end of the file - they
  are not required for JavaScript but are added by default
  by most editors)

...or you can use `vi`, or you can copy it from your local computer
with [scp].

Whichever way you choose, place the following into
`deno-shell-setup.ts`:

```ts
const codeToAppend = `
export DENO_INSTALL="/root/.deno"
export PATH="$DENO_INSTALL/bin:$PATH"
`;

async function setup() {
  const path = Deno.env.get('HOME') + '/.bashrc';
  await Deno.writeTextFile(path, codeToAppend, {append: true});
  console.log(`Added setup to end of ${JSON.stringify(path)}`);
}

if (import.meta.main) {
  setup();
}
```

Now run this (it will fail without both of the `--allow` flags - try it out!):

```bash
~/.deno/bin/deno run --allow-write=~/.bashrc --allow-env=HOME deno-shell-setup.ts
```

This change will apply when you log out and log back in. To make it apply now,
just paste the contents of the `codeToAppend` string into your terminal, log
out and log back in, or run `source ~/.bashrc`. Another thing I like to do is
run `tail -n 2 ~/.bashrc` and look at the output, and if it's what I want (it is
in this case, otherwise I adjust the number), run `source <(tail -n 2 ~/.bashrc)`.
The source command evaluates code in the shell. Run `man source` to learn more.
To learn about `<(command_here)`, run `man bash` and search for
`Process Substitution` by typing `/`, then `Process Substitution`, and ENTER.

Once you've done that, you can now run `deno` and it will give you a repl.

## Step 3: Set up a user account

With another Deno script, we'll create a user account named `macchiato`, copy
the SSH key, install its own Deno, and then print out what you need to add to
your `~/.ssh/config` on your local computer to be able to log in with just
`ssh macchiato`.

Insert this into `setup-ssh-sudo-user.ts` (`cat >setup-user-account.ts`, copy,
paste, `ENTER`, `CTRL-D`):

```ts
import { parse } from "https://deno.land/std@0.100.0/flags/mod.ts";

class RunError extends Error {
  constructor(m: string) {
    super(m);
    Object.setPrototypeOf(this, RunError.prototype);
  }
}

class SudoError extends RunError {
  constructor(m: string) {
    super(m);
    Object.setPrototypeOf(this, SudoError.prototype);
  }
}

async function run(cmd: string[]) {
  console.log(`- Running ${JSON.stringify(cmd)}...`)
  const process = await Deno.run({cmd});
  const status = await process.status();
  if (!status.success) {
    throw new RunError(`Error running ${JSON.stringify(cmd)}`);
  }
}

async function sudo(username: string, cmd: string[]) {
  try {
    await run(['sudo', '-u', username, ...cmd]);
  } catch (e) {
    if (e instanceof RunError) {
      throw new SudoError(e.message);
    } else {
      console.error('Unexpected error in sudo()', e.message);
      throw e;
    }
  }
}

async function setupSshSudoUser(username: string) {
  await run(['adduser', username]);
  await sudo(username, ['mkdir', '-p', `/home/${username}/.ssh`]);
  await run([
    'cp',
    '/root/.ssh/authorized_keys',
    `/home/${username}/.ssh/authorized_keys`,
  ]);
  await run([
    'chown',
    `${username}:${username}`,
    `/home/${username}/.ssh/authorized_keys`,
  ]);
  await run(`groupmod -aG ${username} wheel`);
}

if (import.meta.main) {
  const flags = parse(Deno.args);
  if (!(flags._.length === 1 && typeof flags._[0] === 'string')) {
    console.error('One positional argument is required: the username');
    Deno.exit(1);
  }
  const [username] = flags._;
  setupSshSudoUser(username);
}
```

```bash
deno run --allow-run setup-ssh-sudo-user.ts macchiato
```