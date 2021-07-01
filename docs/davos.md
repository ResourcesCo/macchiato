# Davos

**Runs an action with the given data**

- Creates a temporary directory in
  `$XDG_DATA_HOME/davos/tasks/YYYYMMDD/$ACTION-$SHORTID`
- Looks up an action in `$XDG_DATA_HOME/davos/actions/$ACTION`
- Runs Deno, Docker, or another program with access to only the data in the directory
- Places the artifacts in `$XDG_DATA_HOME/davos/artifacts/YYYYMMDD/$ACTION`
- Deletes the task directory according to settings
- Places www data in `$XDG_DATA_HOME/davos/www/$SERVICE/$SUBDOMAIN`
- Places www redirects, FaaS, etc in `$XDG_DATA_HOME/davos/services/$SERVICE/$SUBDOMAIN`
- Updates caddy config and reloads Caddy. Caddy serves it.

The actions are run with Deno (`$WORKDIR` is the task directory):

```bash
cd $WORKDIR && deno run --allow-read=$WORKDIR --allow-write=$WORKDIR \
  --allow-run --cached-only --lock=$XDG_DATA_HOME/actions/request/lock.json \
  $XDG_DATA_HOME/actions/request/mod.ts
```

These need to be checked before installing them, and you should always know
which one you're running. It should be easy to inspect the code.

Later on this can also run in Deno with an API key and allow-net set to the