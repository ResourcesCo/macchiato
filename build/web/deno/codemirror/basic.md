# CodeMirror 6 Editor w/ Deno

Here we'll build a CodeMirror 6 editor with Deno using a custom-built
sourcemap, with raw TypeScript files from GitHub via jsdelivr.

## Importing from jsDelivr

CodeMirror 6 is a powerful and very modular code editor. Two commonly
used modules are [@codemirror/view][https://github.com/codemirror/view]
and [@codemirror/state][https://github.com/codemirror/state]. The `view`
module provides the core user interface, and depends on the `state`
module to keep track of the state. The `state` module doesn't depend
on the DOM so we'll start by running it in Deno, which provides a lot
of browser APIs but not the DOM. However, the state library depends on
one library, [@codemirror/text][https://github.com/codemirror/text].
This one has no external dependencies, so we'll start by getting it
to run inside of Deno. Within it are several modules. Here they are
with their dependencies:

| name            | dependencies                   |
| --------------- | ------------------------------ |
| `src/char.ts`   | *None*                         |
| `src/column.ts` | `./char`                       |
| `src/index.ts`  | `./char`, `./column`, `./text` |
| `src/text.ts`   | *None*                         |

We'll start with `char.ts` which has no dependencies.

For this, we're going to import the source directly, like you do when
using a library from [deno.com/x](https://deno.com/x).

While it is possible to import from a GitHub raw URL, we'll use the
[jsDelivr][jsDelivr-gh] CDN which supports GitHub URLs.

##### `e0/main.js`

```js
import { findClusterBreak } from "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char.ts";
console.log(
  findClusterBreak("🍎abc", 0),
  findClusterBreak("🍎abc", 1),
  findClusterBreak("🍎abc", 2),
  findClusterBreak("abc", 0),
  findClusterBreak("abc", 1),
);
```

When we run the above, it prints out the character positions of the
*cluster break*, which is where the cursor can go. The cursor cannot
go in the middle of the apple, which is two characters long, so for
`findClusterBreak("🍎abc", 0)`, the result is `2`.

```
> deno run e0/main.js
2 2 3 1 2
```

## Importing a file with a single dependency

CodeMirror's relative imports omit the extension, and the build
tools support it by automatically finding files that have the
extension. Deno tries to be simpler and doesn't detect the
extension, to avoid the ambiguity of having to check for files
with different extensions, such as `.ts`, `.js`, and no
extension.

Here's an example of a relative import from CodeMirror's text module:

[github.com/codemirror/text/blob/0.19.5/src/column.ts#L1](https://github.com/codemirror/text/blob/0.19.5/src/column.ts#L1)

```js
import {findClusterBreak} from "./char"
```

One of the exports is `countColumn`. If we try to import it directly
in Deno, it fails:

##### `e1/main.js`

```js
import { countColumn } from "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column.ts";
console.log([0, 1, 2, 3].map(i => countColumn("\tTest", 2, i)));
```

When running the above, it shows an error message:

```
> deno run e1/main.js
Download https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char
error: Import 'https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char' failed, not found.
    at https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column.ts:1:32
```

Deno is aware of the URL of `column.ts` and uses that to create a
relative URL for `./char` - but it doesn't add the extension, so it
tries to download `https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char`
which is not found.

Let's use an import map to point Deno from `char` to `char.ts`:

##### `e2/import-map.json`

```js
{
  "imports": {
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column.ts"
  }
}
```

##### `e2/main.js`

```js
import { countColumn } from "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column";
console.log([0, 1, 2, 3].map(i => countColumn("\tTest", 2, i)));
```

To run it, specify the import map as a command-line argument to Deno.
These arguments must appear before the filename, because otherwise
they will be passed to the program. Deno strives for consistency and
this includes the command line.

```bash
> deno run --import-map e2/import-map.json e2/main.js
[ 0, 2, 3, 4 ]
```

`countColumn` counts the column of an offset in a line with tab sizes
taken into account. The second parameter to countColumn is the tab
size and the third parameter is the offset. The first element of the
output is `0` just like the input. The following elements are `1`
greater than the input because the first character, the tab, is two
columns wide.

The above worked because when Deno resolves the import, it:

- maps the relative URL to an absolute URL
- uses the import map to map the absolute URL *without* an extension
  to the absolute URL *with* the extension.

It behaves the same as if the full absolute URL with the extension
were in the source. Thus, Deno still depends on the full filename
being given - it just allows the import map to assist in getting
the full filename, much like it uses the URL module to map relative
URLs to absolute URLs.

## Importing a module

We've learned how to make it so a CodeMirror module can import
another module from the same package inside of Deno.

However, many CodeMirror modules import across packages. For instance,
[@codemirror/state](https://github.com/codemirror/state) imports from
[@codemirror/text](https://github.com/codemirror/text). These are
normally npm modules, but can be provided in import maps, which is
what we'll be doing.

In this example we'll import from `@codemirror/text` inside a
local module that we run with Deno. If this can work in our own
module, it can be used to load CodeMirror modules that depend on
modules from external CodeMirror packages.

##### `e3/import-map.json`

```json
{
  "imports": {
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/index": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/index.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/text": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/text.ts",
    "@codemirror/text": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/index.ts"
  }
}
```

##### `e3/main.js`

```js
import { countColumn } from "@codemirror/text";
console.log([0, 1, 2, 3].map(i => countColumn("\tTest", 2, i)));
```

Run it:

```bash
> deno run --import-map e3/import-map.json e3/main.js
Download https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/index.ts
Download https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/text.ts
Check file:///Users/bat/proyectos/notebook/macchiato/build/web/deno/codemirror/basic/e3/main.js
error: TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {Line, TextIterator, Text} from "./text"
              ~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/index.ts:3:15
```

This is a roadblock. 🚧

[Checking the Deno docs](https://deno.land/manual/typescript/faqs):

> As of Deno 1.5 we defaulted to *isolatedModules* to `true` and in Deno
> 1.6 we removed the options to set it back to `false` via a
> configuration file. The isolatedModules option forces the TypeScript
> compiler to check and emit TypeScript as if each module would stand
> on its own. TypeScript has a few *type directed emits* in the language
> at the moment. While not allowing type directed emits into the
> language was a design goal for TypeScript, it has happened anyways.
> This means that the TypeScript compiler needs to understand the
> erasable types in the code to determine what to emit, which when you
> are trying to make a fully erasable type system on top of JavaScript,
> that becomes a problem.

Alas, it won't work to import CodeMirror from Deno with just an import
map! We can try patching. In this only the `index.ts` file requires
the patch. If that's the case, it might be possible to make a fairly
small repo that runs CodeMirror with Deno and the Deno bundler.

## Importing a module by patching

We'll need to create a patch for `index.ts`. Patches will be needed for
a number of files, so we'll include the name and version of the module
and also its GitHub organization, since syntax highlighting CodeMirror
depends on [Lezer](https://lezer.codemirror.net/) which is under a
different organization and it may need to be patched as well and it's
good to keep them separate.

The first change is to make it use `export type` for `TextIterator`.
Let's see if this works.

##### `e4/patch/codemirror/text@0.19.5/src/index.ts`

```ts
export {findClusterBreak, codePointAt, fromCodePoint, codePointSize} from "./char"
export {countColumn, findColumn} from "./column"
export {Line, Text} from "./text"
export type {TextIterator} from "./text"
```

##### `e4/import-map.json`

```json
{
  "imports": {
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/index": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/index.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/text": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/text.ts",
    "@codemirror/text": "./patch/codemirror/text@0.19.5/src/index.ts"
  }
}
```

##### `e4/main.js`

```js
import { countColumn } from "@codemirror/text";
console.log([0, 1, 2, 3].map(i => countColumn("\tTest", 2, i)));
```

Run it:

```bash
> deno run --import-map e4/import-map.json e4/main.js
Check file:///Users/bat/proyectos/notebook/macchiato/build/web/deno/codemirror/basic/e4/main.js
error: Cannot load module "file:///Users/bat/proyectos/notebook/macchiato/build/web/deno/codemirror/basic/e4/patch/codemirror/text@0.19.5/src/char".
    at file:///Users/bat/proyectos/notebook/macchiato/build/web/deno/codemirror/basic/e4/patch/codemirror/text@0.19.5/src/index.ts:1:75
```

The imports are local to the patched file, so the imports in the
patched file either need to be converted to absolute URLs or added to
the source map.

We'll add them to the import map, so the number of changes in the
patched file is kept to a minimum, and so no changes are needed to
the patched file just to bump the version.

## Importing a module by patching with scoped import map entries

The `char` import needs to be added as
`./patch/codemirror/text@0.19.5/src/char` in the import map. Relative
entries in the import map are relative to the URL of the import map.
This import map addition is only needed for the patched file, so
we can use a scope in the import map so it only applies to it.
This will also help in case we didn't want to put each patched file
into its own directory.

[The import-maps README shows how to use scopes.](https://github.com/WICG/import-maps#scoping-examples)
In the top level of the import map, add a key called "scopes", and
beneath that, add a key for each scope. Within each scope, define a
key/value import map just like in `"imports"`. The scope can be a
path if it ends with `"/"` or just a single module. We'll use a
single module.

##### `e5/patch/codemirror/text@0.19.5/src/index.ts`

```ts
export {findClusterBreak, codePointAt, fromCodePoint, codePointSize} from "./char"
export {countColumn, findColumn} from "./column"
export {Line, Text} from "./text"
export type {TextIterator} from "./text"
```

##### `e5/import-map.json`

```json
{
  "imports": {
    "@codemirror/text": "./patch/codemirror/text@0.19.5/src/index.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char.ts"
  },
  "scopes": {
    "./patch/codemirror/text@0.19.5/src/index.ts": {
      "./patch/codemirror/text@0.19.5/src/char": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char.ts",
    "./patch/codemirror/text@0.19.5/src/column": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column.ts",
    "./patch/codemirror/text@0.19.5/src/text": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/text.ts"
    }
  }
}
```

##### `e5/main.js`

```js
import { countColumn } from "@codemirror/text";
console.log([0, 1, 2, 3].map(i => countColumn("\tTest", 2, i)));
```

Run it:

```bash
> deno run --import-map e5/import-map.json e5/main.js
Check file:///Users/bat/proyectos/notebook/macchiato/build/web/deno/codemirror/basic/e4/main.js
error: Cannot load module "file:///Users/bat/proyectos/notebook/macchiato/build/web/deno/codemirror/basic/e4/patch/codemirror/text@0.19.5/src/char.ts".
    at file:///Users/bat/proyectos/notebook/macchiato/build/web/deno/codemirror/basic/e4/patch/codemirror/text@0.19.5/src/index.ts:1:75
```

It works!

There are several operations that needed to be done for this module:

1. Copy the `index.ts` file to a patch location
2. Replace relative imports with absolute imports in `index.ts`
3. Move exported types from the `export` statement to an `export type`
   statement

## Importing a module that depends on another module



---

By the way, the `--import-map` can also be passed to `deno repl`,
`deno cache`, `deno eval`, among other deno commands.

[jsDelivr-gh]: https://www.jsdelivr.com/?docs=gh