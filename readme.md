# macchiato

This project is to build a self-hosted Markdown code notebook application in
Markdown.

This is structured as a repository of Markdown files that generate packages
for components.

The most basic thing that turns a notebook into a package of runnable code is
[unpack-simple](./docs/unpack_simple.md), which takes fenced codeblocks that
are preceded by an inline code block containing the filename, and writes the
data to files.

The packages are then used to provide frontend and backend components for
code notebooks.

There is a [walkthrough](./docs/walkthrough.md) where you can build this
yourself using only notebook pages and a few open source tools that are
either popular or are easy to inspect.

You can also install it from the published packages.
