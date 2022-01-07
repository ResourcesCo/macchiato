# CodeMirror 6 Editor w/ Deno

Here we'll build a CodeMirror 6 editor with Deno using a custom-built
sourcemap, with raw TypeScript files from GitHub via jsdelivr.

## Loading modules directly from the source

### Importing from jsDelivr

CodeMirror 6 is a powerful and very modular code editor. Two commonly
used modules are [@codemirror/view](https://github.com/codemirror/view)
and [@codemirror/state](https://github.com/codemirror/state). The `view`
module provides the core user interface, and depends on the `state`
module to keep track of the state. The `state` module doesn't depend
on the DOM so we'll start by running it in Deno, which provides a lot
of browser APIs but not the DOM. However, the state library depends on
one library, [@codemirror/text](https://github.com/codemirror/text).
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
[jsDelivr](https://www.jsdelivr.com/?docs=gh) CDN which supports GitHub
URLs.

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

### Importing a file with a single dependency

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

### Importing a module

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

### Importing a module by patching with scoped import map entries

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
[ 0, 2, 3, 4 ]
```

It works!

There are several operations that needed to be done for this module:

1. Copy the `index.ts` file to a patch location
2. Replace relative imports with absolute imports in `index.ts`
3. Move exported types from the `export` statement to an `export type`
   statement

### Importing a module that depends on another module

Let's get [@codemirror/state](https://github.com/codemirror/state),
which depends on [@codemirror/text](https://github.com/codemirror/text),
working. Soon we'll need to start automating the build, as there are
dozens of packages needed to build a full-featured CodeMirror 6 editor, but
let's do this manually first.

##### `e6/patch/codemirror/text@0.19.5/src/index.ts`

```ts
export {findClusterBreak, codePointAt, fromCodePoint, codePointSize} from "./char"
export {countColumn, findColumn} from "./column"
export {Line, Text} from "./text"
export type {TextIterator} from "./text"
```

##### `e6/patch/codemirror/state@0.19.6/src/index.ts`

```ts
export {EditorState} from "./state"
export type {EditorStateConfig} from "./state"
export type {StateCommand} from "./extension"
export {Facet, StateField, Prec, Compartment} from "./facet"
export type {Extension} from "./facet"
export {EditorSelection, SelectionRange} from "./selection"
export {Transaction, Annotation, AnnotationType, StateEffect, StateEffectType} from "./transaction"
export type {TransactionSpec} from "./transaction"
export {Text} from "@codemirror/text"
export {combineConfig} from "./config"
export {ChangeSet, ChangeDesc, MapMode} from "./change"
export type {ChangeSpec} from "./change"
export {CharCategory} from "./charcategory"
```

##### `e6/import-map.json`

```json
{
  "imports": {
    "@codemirror/text": "./patch/codemirror/text@0.19.5/src/index.ts",
    "@codemirror/state": "./patch/codemirror/state@0.19.6/src/index.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/state": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/state.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/extension": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/extension.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/facet": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/facet.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/selection": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/selection.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/transaction": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/transaction.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/config": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/config.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/change": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/change.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/charcategory": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/charcategory.ts"
  },
  "scopes": {
    "./patch/codemirror/text@0.19.5/src/index.ts": {
      "./patch/codemirror/text@0.19.5/src/char": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char.ts",
    "./patch/codemirror/text@0.19.5/src/column": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column.ts",
    "./patch/codemirror/text@0.19.5/src/text": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/text.ts"
    },
    "./patch/codemirror/state@0.19.6/src/index.ts": {
      "./patch/codemirror/state@0.19.6/src/state": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/state.ts",
      "./patch/codemirror/state@0.19.6/src/extension": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/extension.ts",
      "./patch/codemirror/state@0.19.6/src/facet": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/facet.ts",
      "./patch/codemirror/state@0.19.6/src/selection": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/selection.ts",
      "./patch/codemirror/state@0.19.6/src/transaction": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/transaction.ts",
      "./patch/codemirror/state@0.19.6/src/config": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/config.ts",
      "./patch/codemirror/state@0.19.6/src/change": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/change.ts",
      "./patch/codemirror/state@0.19.6/src/charcategory": "https://cdn.jsdelivr.net/gh/codemirror/state@0.19.6/src/charcategory.ts"
    }
  }
}
```

##### `e6/main.js`

```js
// based on *minimum viable editor* from https://codemirror.net/6/docs/guide/
import { EditorState } from "@codemirror/state";
const startState = EditorState.create({
  doc: "Hello World",
  extensions: []
});
console.log(`The document's length is ${startState.doc.length}`);
```

Run it:

```bash
> deno run --import-map e6/import-map.json e6/main.js
The document's length is 11
```

By the way, the `--import-map` can also be passed to `deno repl`,
`deno cache`, `deno eval`, among other deno commands.

### On to building a working editor

This looks like it's going to work! The patching of the `index.ts`
files is a layer of indirection, but at least the files are short
and only one per module so far. It's nice that the bulk of the code
is loaded unmodified from the official CodeMirror projects.

Modules are now being loaded directly from the source. It is getting
tedious, though, so we'll write a script that uses the GitHub API to
build the import maps and start the patches automatically. We'll also
provide a way to do a diff of the patches, so it's easy to check that
it's patched properly. To check that the import map is correct, users
of the library can run the script and compare the output.

We'll be making API requests to GitHub's REST API with fetch, using
an API key without any spacial permissions. The repos are public and
GitHub's API allows some anonymous requests, but the rate limits are
less with an API key.

First, generate an API key and set it as an environment variable.
We'll use the minimum permissions that are needed. When you generate
the API key, don't select any permissions. The `public_repo`
permission is for writing. It is not needed to read from a public
repo. Follow the instructions here:

[Creating a personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)

Save that token somewhere. A password manager is a good place. When
you run it, you can simply set it in your shell or put it at the
start of a command. If you don't have it set, the program below will
tell you.

Let's get the contents of `package.json`, the current version from
the `package.json`, the JavaScript filenames in `src` for the tag
from the current version, and the contents of `src/index.js` so we
can patch it. Later on we'll use the `package.json` to find
dependencies. We'll start with a list I manually constructed of
packages for `@codemirror/view` needed to make the basic editor.
The GitHub repositories are by organization, and where there is no
organization, it's specified with a list. This is similar to the
babel plugin syntax.

##### `e7/deps.json`

```json
[
  ["style-mod", "marijnh/style-mod"],
  "@codemirror/rangeset",
  "@codemirror/state",
  "@codemirror/text",
  "@codemirror/state",
  ["w3c-keyname", "marijnh/w3c-keyname"]
]
```

We'll make a function to get the text of a file from a GitHub
repository, which we'll use both to read the `package.json`
and to download the `index.ts` for customization. This will
use the [Repositories: Get repository content](https://docs.github.com/en/rest/reference/repos#get-repository-content)
endpoint.

We'll use that function to read `package.json` and with that we'll
get a list of the files for that version. For this we'll use
the [Git database: Get a tree](https://docs.github.com/en/rest/reference/git#get-a-tree) endpoint.

##### `e7/build.js`

```js
import { decode } from "https://deno.land/std@0.118.0/encoding/base64.ts";

const token = Deno.env.get('GITHUB_API_TOKEN');
if (!(typeof token === 'string' && token.length > 10)) {
  console.error('GITHUB_API_TOKEN environment variable is required')
}

const headers = {
  Accept: "application/vnd.github.v3+json",
  Authorization: `token ${token}`,
}

const apiBase = 'https://api.github.com';

function repoPath({owner, repo, path}) {
  const e = {
    owner: encodeURIComponent(owner),
    repo: encodeURIComponent(repo),
    repo: encodeURI(path),
  }
  return `/repos/${owner}/${repo}/contents${path}`;
}

async function getTextFile({owner, repo, path}) {
  const url = `${apiBase}${repoPath({owner, repo, path})}`;
  const resp = await fetch(url);
  const { content } = await resp.json();
  return decode(new TextEncoder().encode(content));
}
```
