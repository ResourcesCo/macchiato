# macchiato

## current instructions

There is a tool called `md_unpack_simple` which unpacks files from markdown,
and which itself contains files for the project, which is available at
https://deno.land/x/md_unpack_simple

There is also a work-in-progress server setup guide in server/setup.md.

## draft

This project is to build a self-hosted Markdown code notebook application in
Markdown.

This is structured as a repository of Markdown files that generate packages
for components.

The most basic thing that turns a notebook into a package of runnable code is
[unpack-simple](./docs/unpack-simple.md), which takes fenced codeblocks that
are preceded by a header with an inline code block containing the path to the
each file, and writes the files.

The packages are then used to provide frontend and backend components for
code notebooks.

## TODO

- [x] Instructions/scripts for Docker container with database, WordPress, gitea
- [ ] App that provides sign-in with IndieAuth
- [ ] Ability to publish notebook pages on subdomains from app when logged in
- [ ] Link to published packages
- [ ] Script to check that published packages match Markdown source code
