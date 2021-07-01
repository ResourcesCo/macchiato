# macchiato

This project is to build a self-hosted computing environment using only Markdown
source, with code that can easily be inspected, along with dependencies that
either can easily be inspected or are maintained by a sizable community.

Here is the stack, which runs on top of a smallish Linus VPS:

- [Deno](https://deno.land/) - A JavaScript runtime that is *secure by default*.
  It has a permission system which 
- [Podman](https://podman.io/) - A containerization platform like Docker, that
  isolates components for security, flexibility, and maintainability.
- [Caddy](https://caddyserver.com/) - A web server focused on website owners,
  with powerful SSL certificate management and remote proxying capabilities.
- [PostgreSQL](https://www.postgresql.org/). A versatile database engine. Runs
  as a shared instance inside the VPS. Note that this is used for caching and
  subscriptions instead of an additional database engine like Redis.
- [Gitea](https://gitea.io/en-us/) - A lighweight and powerful git project host
  written in Go.
- Golang, Rust, Node.js, Python - Other languages, and tools written in these
  languages
- Vite and Vue - Used to build user interfaces

# getting started

To start, set up a VM with the most recent Fedora version, using the instructons
in [server](./docs/server.md). Fedora is closely aligned with Podman.
