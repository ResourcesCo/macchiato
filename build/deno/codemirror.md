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

This will also contain some changes that need to be made for the
patch files, that were manually constructed above. The patches will
just be a search and replace of entire lines. Lines can be in a
list to make the JSON easier to read.

##### `e7/deps.json`

```json
{
  "deps": [
    {
      "npm": "style-mod",
      "github": "marijnh/style-mod"
    },
    {
      "npm": "@codemirror/rangeset",
      "github": "codemirror/rangeset"
    },
    {
      "npm": "@codemirror/state",
      "github": "codemirror/state",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ],
          [
            "export {EditorStateConfig, EditorState} from \"./state\"",
            [
              "export {EditorState} from \"./state\"",
              "export type {EditorStateConfig} from \"./state\""
            ]
          ],
          [
            "export {StateCommand} from \"./extension\"",
            "export type {StateCommand} from \"./extension\""
          ],
          [
            "export {Facet, StateField, Extension, Prec, Compartment} from \"./facet\"",
            [
              "export {Facet, StateField, Prec, Compartment} from \"./facet\"",
              "export type {Extension} from \"./facet\""
            ]
          ],
          [
            "export {Transaction, TransactionSpec, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
            [
              "export {Transaction, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
              "export type {TransactionSpec} from \"./transaction\""
            ]
          ],
          [
            "export {ChangeSpec, ChangeSet, ChangeDesc, MapMode} from \"./change\"",
            [
              "export {ChangeSet, ChangeDesc, MapMode} from \"./change\"",
              "export type {ChangeSpec} from \"./change\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/text",
      "github": "codemirror/text",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "w3c-keyname",
      "github": "marijnh/w3c-keyname"
    }
  ]
}
```

We'll make a function to get the text of a file from a GitHub
repository, which we'll use both to read the `package.json`
and to download the `index.ts` for patching. This will
use the [Repositories: Get repository content](https://docs.github.com/en/rest/reference/repos#get-repository-content)
endpoint.

We'll use that function to read `package.json` and with that we'll
get a list of the files for that version. For this we'll use
the [Git database: Get a tree](https://docs.github.com/en/rest/reference/git#get-a-tree) endpoint.

The typescript entry points will be patched by searching and
replacing as specified in `patches` of `deps.json`.

##### `e7/build.js`

```js
import { decode } from "https://deno.land/std@0.118.0/encoding/base64.ts";
import { dirname } from "https://deno.land/std@0.125.0/path/mod.ts";
import { ensureDir } from "https://deno.land/std@0.125.0/fs/mod.ts";

if (Deno.args.length !== 1) {
  throw new Error("Missing argument: path to dependency json file");
}

const deps = JSON.parse(await Deno.readTextFile(Deno.args[0]));

const token = Deno.env.get('GITHUB_API_TOKEN');
if (!(typeof token === 'string' && token.length > 10)) {
  console.error('GITHUB_API_TOKEN environment variable is required')
}

const headers = {
  Accept: "application/vnd.github.v3+json",
  Authorization: `token ${token}`,
}

const apiBase = 'https://api.github.com';

async function getTextFile({github, path}) {
  const url = new URL(path, `${apiBase}/repos/${github}/contents/`).toString();
  const resp = await fetch(url, {headers});
  const json = await resp.json();
  if (resp.ok) {
    return new TextDecoder().decode(decode(json.content));
  } else {
    throw new Error(`Error loading ${path} at ${url}: ${JSON.stringify(json)}`);
  }
}

async function getTree({github, version}) {
  const repoUrl = `${apiBase}/repos/${github}`;
  const url = `${repoUrl}/git/trees/${encodeURIComponent(version)}`;
  const resp = await fetch(`${url}?recursive=true`, {headers});
  return (await resp.json()).tree.map(({path}) => path);
}

function linesToStr(input) {
  const arr = Array.isArray(input) ? input : [input];
  return arr.map(s => s + "\n").join("");
}

function patchText(text, patches) {
  let result = text;
  if (patches) {
    for (const [search, replace] of patches) {
      result = result.replace(linesToStr(search), linesToStr(replace));
    }
  }
  return result;
}

function buildUrl({patch, github, version, file, ext}) {
  const base = patch ? './patch' : 'https://cdn.jsdelivr.net/gh';
  const f = ext ? file : file.replace(/\.[jt]s$/, '');
  return `${base}/${github}@${version}/${f}`
}

async function buildPackage({npm, github, patch, entry: configEntry}) {
  const [owner, repo] = github.split('/');
  const pkg = JSON.parse(await getTextFile({github, path: "package.json"}));
  const entry = configEntry || ((pkg.scripts?.prepare || '').startsWith('cm-buildhelper ') ?
                 (pkg.scripts?.prepare || '').replace('cm-buildhelper ', '') :
                 pkg.module || pkg.main);
  const files = (await getTree({github, version: pkg.version})).filter(f =>
    ((f.endsWith('.ts') || f.endsWith('.js')) && !f.startsWith('test/'))
  );
  const urlParams = {github, version: pkg.version};
  const patchMap = patch || {};
  const patchedFiles = Object.keys(patchMap);
  const relativeMapEntries = files.map(f => ([
    buildUrl({...urlParams, file: f, patch: false, ext: false}),
    buildUrl({...urlParams, file: f, patch: f in patchMap, ext: true}),
  ]));
  let patchMapEntries = [];
  if (patchedFiles.length > 0) {
    patchMapEntries = files.map(f => ([
      buildUrl({...urlParams, file: f, patch: true, ext: false}),
      buildUrl({...urlParams, file: f, patch: f in patchMap, ext: true}),
    ]));
  }
  for (const [f, patches] of Object.entries(patchMap)) {
    const origUrl = buildUrl({...urlParams, file: f, patch: false, ext: true});
    const patchFile = buildUrl({...urlParams, file: f, patch: true, ext: true});
    const origResp = await fetch(origUrl);
    const origText = await origResp.text();
    if (origResp.ok) {
      const patchedText = patchText(origText, patches);
      await ensureDir(dirname(patchFile));
      await Deno.writeTextFile(patchFile, patchedText);
    } else {
      throw new Error(`Error getting source at ${origUrl}: ${origResp.statusCode} ${origText}`);
    }
  }
  return {
    ...Object.fromEntries(relativeMapEntries),
    ...Object.fromEntries(patchMapEntries),
    [npm]: buildUrl({...urlParams, file: entry, patch: entry in patchMap, ext: true}),
  };
}

let outputMap = {};
for (const dep of deps.deps) {
  const packageMap = await buildPackage(dep);
  outputMap = { ...outputMap, ...packageMap };
}
const importMapData = {
  imports: outputMap,
};
Deno.writeTextFile('./import-map.json', JSON.stringify(importMapData, null, 2));
```

To run it:

```bash
deno run --allow-env=GITHUB_API_TOKEN \
--allow-net=api.github.com,cdn.jsdelivr.net \
--allow-read=deps.json,patch \
--allow-write=import-map.json,patch \
build.js \
deps.json
```

Starting by importing some dependencies:

##### `e7/example1.ts`

```ts
import { StyleModule } from 'style-mod';
import { Range } from '@codemirror/rangeset';
console.log({StyleModule, Range});
```

To run it:

```bash
deno run --import-map=import-map.json example1.ts
```

Now we'll get the view bundling. This is based on the "minimum viable editor"
in the [System Guide](https://codemirror.net/6/docs/guide/).

##### `e7/example2.ts`

```ts
import {EditorState} from "@codemirror/state"
import {EditorView, keymap} from "@codemirror/view"
import {defaultKeymap} from "@codemirror/commands"

let startState = EditorState.create({
  doc: "Hello World",
  extensions: [keymap.of(defaultKeymap)]
})

let view = new EditorView({
  state: startState,
  parent: document.body
})
```

We'll attempt to bundle it:

```bash
❯ deno bundle --import-map=import-map.json example2.ts
error: Relative import path "@codemirror/commands" not prefixed with / or ./ or ../ and not in import map from "file:///Users/bat/proyectos/notebook/macchiato/build/deno/codemirror/e7/example2.ts"
    at file:///Users/bat/proyectos/notebook/macchiato/build/deno/codemirror/e7/example2.ts:3:29
```

There's another module that needs to be added to `deps.json`.

##### `e7/deps2.json`

```json
{
  "deps": [
    {
      "npm": "style-mod",
      "github": "marijnh/style-mod"
    },
    {
      "npm": "@codemirror/rangeset",
      "github": "codemirror/rangeset"
    },
    {
      "npm": "@codemirror/state",
      "github": "codemirror/state",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ],
          [
            "export {EditorStateConfig, EditorState} from \"./state\"",
            [
              "export {EditorState} from \"./state\"",
              "export type {EditorStateConfig} from \"./state\""
            ]
          ],
          [
            "export {StateCommand} from \"./extension\"",
            "export type {StateCommand} from \"./extension\""
          ],
          [
            "export {Facet, StateField, Extension, Prec, Compartment} from \"./facet\"",
            [
              "export {Facet, StateField, Prec, Compartment} from \"./facet\"",
              "export type {Extension} from \"./facet\""
            ]
          ],
          [
            "export {Transaction, TransactionSpec, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
            [
              "export {Transaction, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
              "export type {TransactionSpec} from \"./transaction\""
            ]
          ],
          [
            "export {ChangeSpec, ChangeSet, ChangeDesc, MapMode} from \"./change\"",
            [
              "export {ChangeSet, ChangeDesc, MapMode} from \"./change\"",
              "export type {ChangeSpec} from \"./change\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/text",
      "github": "codemirror/text",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/commands",
      "github": "codemirror/commands",
      "patch": {}
    },
    {
      "npm": "w3c-keyname",
      "github": "marijnh/w3c-keyname"
    }
  ]
}
```

We'll build with this new deps json file:

```bash
deno run --allow-env=GITHUB_API_TOKEN \
--allow-net=api.github.com,cdn.jsdelivr.net \
--allow-read=deps2.json,patch \
--allow-write=import-map.json,patch \
build.js \
deps2.json
```

Trying to bundle it again:

```
❯ deno bundle --import-map=import-map.json example2.ts
Download https://cdn.jsdelivr.net/gh/codemirror/commands@0.19.8/src/commands.ts
error: Relative import path "@codemirror/language" not prefixed with / or ./ or ../ and not in import map from "https://cdn.jsdelivr.net/gh/codemirror/commands@0.19.8/src/commands.ts"
    at https://cdn.jsdelivr.net/gh/codemirror/commands@0.19.8/src/commands.ts:7:30
```

There's another dependency, `@codemirror/language, that needs to be added.
New `deps3.json`:

##### `e7/deps3.json`

```json
{
  "deps": [
    {
      "npm": "style-mod",
      "github": "marijnh/style-mod"
    },
    {
      "npm": "@codemirror/rangeset",
      "github": "codemirror/rangeset"
    },
    {
      "npm": "@codemirror/state",
      "github": "codemirror/state",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ],
          [
            "export {EditorStateConfig, EditorState} from \"./state\"",
            [
              "export {EditorState} from \"./state\"",
              "export type {EditorStateConfig} from \"./state\""
            ]
          ],
          [
            "export {StateCommand} from \"./extension\"",
            "export type {StateCommand} from \"./extension\""
          ],
          [
            "export {Facet, StateField, Extension, Prec, Compartment} from \"./facet\"",
            [
              "export {Facet, StateField, Prec, Compartment} from \"./facet\"",
              "export type {Extension} from \"./facet\""
            ]
          ],
          [
            "export {Transaction, TransactionSpec, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
            [
              "export {Transaction, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
              "export type {TransactionSpec} from \"./transaction\""
            ]
          ],
          [
            "export {ChangeSpec, ChangeSet, ChangeDesc, MapMode} from \"./change\"",
            [
              "export {ChangeSet, ChangeDesc, MapMode} from \"./change\"",
              "export type {ChangeSpec} from \"./change\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/text",
      "github": "codemirror/text",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/commands",
      "github": "codemirror/commands",
      "patch": {}
    },
    {
      "npm": "@codemirror/language",
      "github": "codemirror/language",
      "patch": {}
    },
    {
      "npm": "w3c-keyname",
      "github": "marijnh/w3c-keyname"
    }
  ]
}
```

We'll build with this new deps json file:

```bash
deno run --allow-env=GITHUB_API_TOKEN \
--allow-net=api.github.com,cdn.jsdelivr.net \
--allow-read=deps3.json,patch \
--allow-write=import-map.json,patch \
build.js \
deps3.json
```

Trying to bundle it with this dependency added:

```
❯ deno bundle --import-map=import-map.json example2.ts
Download https://cdn.jsdelivr.net/gh/codemirror/commands@0.19.8/src/commands.ts
error: Relative import path "@codemirror/language" not prefixed with / or ./ or ../ and not in import map from "https://cdn.jsdelivr.net/gh/codemirror/commands@0.19.8/src/commands.ts"
    at https://cdn.jsdelivr.net/gh/codemirror/commands@0.19.8/src/commands.ts:7:30
```

Now there is `@lezer/common`. New json file `deps4.json`:

##### `e7/deps4.json`

```json
{
  "deps": [
    {
      "npm": "style-mod",
      "github": "marijnh/style-mod"
    },
    {
      "npm": "@codemirror/rangeset",
      "github": "codemirror/rangeset"
    },
    {
      "npm": "@codemirror/state",
      "github": "codemirror/state",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ],
          [
            "export {EditorStateConfig, EditorState} from \"./state\"",
            [
              "export {EditorState} from \"./state\"",
              "export type {EditorStateConfig} from \"./state\""
            ]
          ],
          [
            "export {StateCommand} from \"./extension\"",
            "export type {StateCommand} from \"./extension\""
          ],
          [
            "export {Facet, StateField, Extension, Prec, Compartment} from \"./facet\"",
            [
              "export {Facet, StateField, Prec, Compartment} from \"./facet\"",
              "export type {Extension} from \"./facet\""
            ]
          ],
          [
            "export {Transaction, TransactionSpec, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
            [
              "export {Transaction, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
              "export type {TransactionSpec} from \"./transaction\""
            ]
          ],
          [
            "export {ChangeSpec, ChangeSet, ChangeDesc, MapMode} from \"./change\"",
            [
              "export {ChangeSet, ChangeDesc, MapMode} from \"./change\"",
              "export type {ChangeSpec} from \"./change\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/text",
      "github": "codemirror/text",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/commands",
      "github": "codemirror/commands",
      "patch": {}
    },
    {
      "npm": "@codemirror/language",
      "github": "codemirror/language",
      "patch": {}
    },
    {
      "npm": "@lezer/common",
      "github": "lezer-parser/common",
      "patch": {}
    },
    {
      "npm": "w3c-keyname",
      "github": "marijnh/w3c-keyname"
    }
  ]
}
```

We'll build with `deps4.json` file:

```bash
deno run --allow-env=GITHUB_API_TOKEN \
--allow-net=api.github.com,cdn.jsdelivr.net \
--allow-read=deps4.json,patch \
--allow-write=import-map.json,patch \
build.js \
deps4.json
```

Trying to bundle it with `@lezer/common` added:

```
❯ deno bundle --import-map=import-map.json example2.ts
Download https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/dist/index.js
error: Relative import path "@codemirror/view" not prefixed with / or ./ or ../ and not in import map from "https://cdn.jsdelivr.net/gh/codemirror/language@0.19.8/src/language.ts"
    at https://cdn.jsdelivr.net/gh/codemirror/language@0.19.8/src/language.ts:6:64
```

This time it's `@codemirror/view`. Copying and adding to the json file:

##### `e7/deps5.json`

```json
{
  "deps": [
    {
      "npm": "style-mod",
      "github": "marijnh/style-mod"
    },
    {
      "npm": "@codemirror/rangeset",
      "github": "codemirror/rangeset"
    },
    {
      "npm": "@codemirror/state",
      "github": "codemirror/state",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ],
          [
            "export {EditorStateConfig, EditorState} from \"./state\"",
            [
              "export {EditorState} from \"./state\"",
              "export type {EditorStateConfig} from \"./state\""
            ]
          ],
          [
            "export {StateCommand} from \"./extension\"",
            "export type {StateCommand} from \"./extension\""
          ],
          [
            "export {Facet, StateField, Extension, Prec, Compartment} from \"./facet\"",
            [
              "export {Facet, StateField, Prec, Compartment} from \"./facet\"",
              "export type {Extension} from \"./facet\""
            ]
          ],
          [
            "export {Transaction, TransactionSpec, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
            [
              "export {Transaction, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
              "export type {TransactionSpec} from \"./transaction\""
            ]
          ],
          [
            "export {ChangeSpec, ChangeSet, ChangeDesc, MapMode} from \"./change\"",
            [
              "export {ChangeSet, ChangeDesc, MapMode} from \"./change\"",
              "export type {ChangeSpec} from \"./change\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/text",
      "github": "codemirror/text",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/commands",
      "github": "codemirror/commands",
      "patch": {}
    },
    {
      "npm": "@codemirror/language",
      "github": "codemirror/language",
      "patch": {}
    },
    {
      "npm": "@codemirror/view",
      "github": "codemirror/view",
      "patch": {}
    },
    {
      "npm": "@lezer/common",
      "github": "lezer-parser/common",
      "patch": {}
    },
    {
      "npm": "w3c-keyname",
      "github": "marijnh/w3c-keyname"
    }
  ]
}
```

We'll build with `deps5.json` file:

```bash
deno run --allow-env=GITHUB_API_TOKEN \
--allow-net=api.github.com,cdn.jsdelivr.net \
--allow-read=deps5.json,patch \
--allow-write=import-map.json,patch \
build.js \
deps5.json
```

Trying to bundle it with `@codemirror/view` added:

```
❯ deno bundle --import-map=import-map.json example2.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/index.ts
Download https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/dist/index.js
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/active-line.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/bidi.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/decoration.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/dom.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/draw-selection.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/dropcursor.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/editorview.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/extension.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/heightmap.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/input.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/keymap.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/matchdecorator.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/placeholder.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/scrollpastend.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/special-chars.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/browser.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/attributes.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/inlineview.ts
Download https://cdn.jsdelivr.net/gh/marijnh/w3c-keyname@2.2.4/index.es.js
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/blockview.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/contentview.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/cursor.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/docview.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/domchange.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/domobserver.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/theme.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/viewstate.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/buildview.ts
Download https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/domreader.ts
error: Relative import path "@lezer/lr" not prefixed with / or ./ or ../ and not in import map from "https://cdn.jsdelivr.net/gh/codemirror/language@0.19.8/src/language.ts"
    at https://cdn.jsdelivr.net/gh/codemirror/language@0.19.8/src/language.ts:3:43
```

This one downloaded quite a few files. `@lezer/lr` is needed. New `deps6.json` file:

##### `e7/deps6.json`

```json
{
  "deps": [
    {
      "npm": "style-mod",
      "github": "marijnh/style-mod"
    },
    {
      "npm": "@codemirror/rangeset",
      "github": "codemirror/rangeset"
    },
    {
      "npm": "@codemirror/state",
      "github": "codemirror/state",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ],
          [
            "export {EditorStateConfig, EditorState} from \"./state\"",
            [
              "export {EditorState} from \"./state\"",
              "export type {EditorStateConfig} from \"./state\""
            ]
          ],
          [
            "export {StateCommand} from \"./extension\"",
            "export type {StateCommand} from \"./extension\""
          ],
          [
            "export {Facet, StateField, Extension, Prec, Compartment} from \"./facet\"",
            [
              "export {Facet, StateField, Prec, Compartment} from \"./facet\"",
              "export type {Extension} from \"./facet\""
            ]
          ],
          [
            "export {Transaction, TransactionSpec, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
            [
              "export {Transaction, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
              "export type {TransactionSpec} from \"./transaction\""
            ]
          ],
          [
            "export {ChangeSpec, ChangeSet, ChangeDesc, MapMode} from \"./change\"",
            [
              "export {ChangeSet, ChangeDesc, MapMode} from \"./change\"",
              "export type {ChangeSpec} from \"./change\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/text",
      "github": "codemirror/text",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/commands",
      "github": "codemirror/commands",
      "patch": {}
    },
    {
      "npm": "@codemirror/language",
      "github": "codemirror/language",
      "patch": {}
    },
    {
      "npm": "@codemirror/view",
      "github": "codemirror/view",
      "patch": {}
    },
    {
      "npm": "@lezer/common",
      "github": "lezer-parser/common",
      "patch": {}
    },
    {
      "npm": "@lezer/lr",
      "github": "lezer-parser/lr",
      "patch": {}
    },
    {
      "npm": "w3c-keyname",
      "github": "marijnh/w3c-keyname"
    }
  ]
}
```

We'll build with `deps6.json` file:

```bash
deno run --allow-env=GITHUB_API_TOKEN \
--allow-net=api.github.com,cdn.jsdelivr.net \
--allow-read=deps6.json,patch \
--allow-write=import-map.json,patch \
build.js \
deps6.json
```

Trying to bundle it with `@lezer/lr` added:

```
❯ deno bundle --import-map=import-map.json example2.ts
Download https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/dist/index.js
Download https://cdn.jsdelivr.net/gh/lezer-parser/lr@0.15.8/dist/index.js
error: Relative import path "@codemirror/matchbrackets" not prefixed with / or ./ or ../ and not in import map from "https://cdn.jsdelivr.net/gh/codemirror/commands@0.19.8/src/commands.ts"
    at https://cdn.jsdelivr.net/gh/codemirror/commands@0.19.8/src/commands.ts:5:29
```

Now `@codemirror/matchbrackets` is needed. Adding:

##### `e7/deps7.json`

```json
{
  "deps": [
    {
      "npm": "style-mod",
      "github": "marijnh/style-mod"
    },
    {
      "npm": "@codemirror/rangeset",
      "github": "codemirror/rangeset"
    },
    {
      "npm": "@codemirror/state",
      "github": "codemirror/state",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ],
          [
            "export {EditorStateConfig, EditorState} from \"./state\"",
            [
              "export {EditorState} from \"./state\"",
              "export type {EditorStateConfig} from \"./state\""
            ]
          ],
          [
            "export {StateCommand} from \"./extension\"",
            "export type {StateCommand} from \"./extension\""
          ],
          [
            "export {Facet, StateField, Extension, Prec, Compartment} from \"./facet\"",
            [
              "export {Facet, StateField, Prec, Compartment} from \"./facet\"",
              "export type {Extension} from \"./facet\""
            ]
          ],
          [
            "export {Transaction, TransactionSpec, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
            [
              "export {Transaction, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
              "export type {TransactionSpec} from \"./transaction\""
            ]
          ],
          [
            "export {ChangeSpec, ChangeSet, ChangeDesc, MapMode} from \"./change\"",
            [
              "export {ChangeSet, ChangeDesc, MapMode} from \"./change\"",
              "export type {ChangeSpec} from \"./change\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/text",
      "github": "codemirror/text",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/commands",
      "github": "codemirror/commands",
      "patch": {}
    },
    {
      "npm": "@codemirror/language",
      "github": "codemirror/language",
      "patch": {}
    },
    {
      "npm": "@codemirror/view",
      "github": "codemirror/view",
      "patch": {}
    },
    {
      "npm": "@codemirror/matchbrackets",
      "github": "codemirror/matchbrackets",
      "patch": {}
    },
    {
      "npm": "@lezer/common",
      "github": "lezer-parser/common",
      "patch": {}
    },
    {
      "npm": "@lezer/lr",
      "github": "lezer-parser/lr",
      "patch": {}
    },
    {
      "npm": "w3c-keyname",
      "github": "marijnh/w3c-keyname"
    }
  ]
}
```

We'll build with `deps7.json`:

```bash
deno run --allow-env=GITHUB_API_TOKEN \
--allow-net=api.github.com,cdn.jsdelivr.net \
--allow-read=deps7.json,patch \
--allow-write=import-map.json,patch \
build.js \
deps7.json
```

Trying to bundle it with `@lezer/lr` added:

```
❯ deno bundle --import-map=import-map.json example2.ts
Download https://cdn.jsdelivr.net/gh/codemirror/matchbrackets@0.19.4/src/matchbrackets.ts
Download https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/dist/index.js
Download https://cdn.jsdelivr.net/gh/lezer-parser/lr@0.15.8/dist/index.js
error: Module not found "https://cdn.jsdelivr.net/gh/lezer-parser/lr@0.15.8/dist/index.js".
    at https://cdn.jsdelivr.net/gh/codemirror/language@0.19.8/src/language.ts:3:43
```

Here, `@lezer/lr` has its entry point wrong. To fix that, the entry point will be given
as `src/index.ts`, and the same for `@lezer/common`:

##### `e7/deps8.json`

```json
{
  "deps": [
    {
      "npm": "style-mod",
      "github": "marijnh/style-mod"
    },
    {
      "npm": "@codemirror/rangeset",
      "github": "codemirror/rangeset"
    },
    {
      "npm": "@codemirror/state",
      "github": "codemirror/state",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ],
          [
            "export {EditorStateConfig, EditorState} from \"./state\"",
            [
              "export {EditorState} from \"./state\"",
              "export type {EditorStateConfig} from \"./state\""
            ]
          ],
          [
            "export {StateCommand} from \"./extension\"",
            "export type {StateCommand} from \"./extension\""
          ],
          [
            "export {Facet, StateField, Extension, Prec, Compartment} from \"./facet\"",
            [
              "export {Facet, StateField, Prec, Compartment} from \"./facet\"",
              "export type {Extension} from \"./facet\""
            ]
          ],
          [
            "export {Transaction, TransactionSpec, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
            [
              "export {Transaction, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
              "export type {TransactionSpec} from \"./transaction\""
            ]
          ],
          [
            "export {ChangeSpec, ChangeSet, ChangeDesc, MapMode} from \"./change\"",
            [
              "export {ChangeSet, ChangeDesc, MapMode} from \"./change\"",
              "export type {ChangeSpec} from \"./change\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/text",
      "github": "codemirror/text",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/commands",
      "github": "codemirror/commands",
      "patch": {}
    },
    {
      "npm": "@codemirror/language",
      "github": "codemirror/language",
      "patch": {}
    },
    {
      "npm": "@codemirror/view",
      "github": "codemirror/view",
      "patch": {}
    },
    {
      "npm": "@codemirror/matchbrackets",
      "github": "codemirror/matchbrackets",
      "patch": {}
    },
    {
      "npm": "@lezer/common",
      "github": "lezer-parser/common",
      "entry": "src/index.ts",
      "patch": {}
    },
    {
      "npm": "@lezer/lr",
      "github": "lezer-parser/lr",
      "entry": "src/index.ts",
      "patch": {}
    },
    {
      "npm": "w3c-keyname",
      "github": "marijnh/w3c-keyname"
    }
  ]
}
```

We'll build with `deps8.json`:

```bash
deno run --allow-env=GITHUB_API_TOKEN \
--allow-net=api.github.com,cdn.jsdelivr.net \
--allow-read=deps8.json,patch \
--allow-write=import-map.json,patch \
build.js \
deps8.json
```

Trying to bundle it with fixed entrypoint for `@lezer/lr`:

```
❯ deno bundle --import-map=import-map.json example2.ts
[lines omitted]
TS2584 [ERROR]: Cannot find name 'document'. Do you need to change your target library? Try changing the 'lib' compiler option to include 'dom'.
  parent: document.body
          ~~~~~~~~
    at file:///Users/bat/proyectos/notebook/macchiato/build/deno/codemirror/e7/example2.ts:12:11

Found 346 errors.
```

Finally it found the files, and there are TypeScript errors. Many of them are because
it doesn't have DOM libraries. We'll fix that by adding a confiuration file!

##### `e7/browser-bundle-config.json`

```
{
  "compilerOptions": {
    "lib": ["dom", "dom.iterable", "dom.asynciterable", "deno.ns"]
  }
}
```

Running `deno bundle` with this configuration file:

```
NO_COLOR=1 deno bundle --config=browser-bundle-config.json --import-map=import-map.json example2.ts > output.txt 2>&1
```

Here is the output:

##### `e7/output.txt`

```
Unsupported compiler options in "file:///Users/bat/proyectos/notebook/macchiato/build/deno/codemirror/e7/browser-bundle-config.json".
  The following options were ignored:
    target
Check file:///Users/bat/proyectos/notebook/macchiato/build/deno/codemirror/e7/example2.ts
error: TS2612 [ERROR]: Property 'point' will overwrite the base property in 'RangeValue'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  point!: boolean
  ~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/decoration.ts:185:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: Text | null
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/inlineview.ts:12:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: HTMLElement | null
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/inlineview.ts:67:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: HTMLElement | null
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/inlineview.ts:154:3

TS2612 [ERROR]: Property 'widget' will overwrite the base property in 'WidgetView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  widget!: CompositionWidget
  ~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/inlineview.ts:238:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: HTMLElement | null
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/inlineview.ts:315:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: HTMLElement | null
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/blockview.ts:18:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: HTMLElement | null
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/blockview.ts:155:3

TS2612 [ERROR]: Property 'parent' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  parent!: DocView | null
  ~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/blockview.ts:156:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: HTMLElement
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/docview.ts:41:3

TS2305 [ERROR]: Module '"https://cdn.jsdelivr.net/gh/marijnh/style-mod@4.0.0/src/style-mod.js"' has no exported member 'StyleSpec'.
import {StyleModule, StyleSpec} from "style-mod"
                     ~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/theme.ts:2:22

TS7006 [ERROR]: Parameter 'sel' implicitly has an 'any' type.
    finish(sel) {
           ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/theme.ts:14:12

TS7006 [ERROR]: Parameter 'm' implicitly has an 'any' type.
      return /&/.test(sel) ? sel.replace(/&\w*/, m => {
                                                 ^
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/theme.ts:15:50

TS2305 [ERROR]: Module '"https://cdn.jsdelivr.net/gh/marijnh/style-mod@4.0.0/src/style-mod.js"' has no exported member 'StyleSpec'.
import {StyleModule, StyleSpec} from "style-mod"
                     ~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/editorview.ts:4:22

TS7053 [ERROR]: Element implicitly has an 'any' type because expression of type 'number' can't be used to index type '{ 8: string; 9: string; 10: string; 12: string; 13: string; 16: string; 17: string; 18: string; 20: string; 27: string; 32: string; 33: string; 34: string; 35: string; 36: string; 37: string; 38: string; 39: string; 40: string; ... 33 more ...; 229: string; }'.
  No index signature with a parameter of type 'number' was found on type '{ 8: string; 9: string; 10: string; 12: string; 13: string; 16: string; 17: string; 18: string; 20: string; 27: string; 32: string; 33: string; 34: string; 35: string; 36: string; 37: string; 38: string; 39: string; 40: string; ... 33 more ...; 229: string; }'.
      (baseName = base[event.keyCode]) && baseName != name) {
                  ~~~~~~~~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/keymap.ts:208:19

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {EditorView, DOMEventMap, DOMEventHandlers} from "./editorview"
                    ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/index.ts:1:21

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {EditorView, DOMEventMap, DOMEventHandlers} from "./editorview"
                                 ~~~~~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/index.ts:1:34

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {Command, ViewPlugin, PluginValue, PluginSpec, PluginFieldProvider, PluginField, ViewUpdate, logException} from "./extension"
        ~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/index.ts:2:9

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {Command, ViewPlugin, PluginValue, PluginSpec, PluginFieldProvider, PluginField, ViewUpdate, logException} from "./extension"
                             ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/index.ts:2:30

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {Command, ViewPlugin, PluginValue, PluginSpec, PluginFieldProvider, PluginField, ViewUpdate, logException} from "./extension"
                                          ~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/index.ts:2:43

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {Decoration, DecorationSet, WidgetType, BlockType} from "./decoration"
                    ~~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/index.ts:3:21

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {MouseSelectionStyle} from "./input"
        ~~~~~~~~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/index.ts:5:9

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {KeyBinding, keymap, runScopeHandlers} from "./keymap"
        ~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/index.ts:7:9

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {Rect} from "./dom"
        ~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/index.ts:14:9

TS7022 [ERROR]: 'parent' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:797:7

TS7023 [ERROR]: 'parent' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:797:7

TS7022 [ERROR]: 'nextSibling' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get nextSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:801:7

TS7023 [ERROR]: 'nextSibling' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get nextSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:801:7

TS7022 [ERROR]: 'prevSibling' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get prevSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:804:7

TS7023 [ERROR]: 'prevSibling' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get prevSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:804:7

TS7022 [ERROR]: 'parent' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:889:7

TS7023 [ERROR]: 'parent' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:889:7

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {DefaultBufferLength, NodeProp, MountedTree, NodePropSource, NodeType, NodeSet, Tree,
                                                    ~~~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/index.ts:1:53

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
        TreeBuffer, SyntaxNode, TreeCursor, BufferCursor} from "./tree"
                    ~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/index.ts:2:21

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
        TreeBuffer, SyntaxNode, TreeCursor, BufferCursor} from "./tree"
                                            ~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/index.ts:2:45

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {ChangedRange, TreeFragment, PartialParse, Parser, Input, ParseWrapper} from "./parse"
        ~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/index.ts:3:9

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {ChangedRange, TreeFragment, PartialParse, Parser, Input, ParseWrapper} from "./parse"
                                    ~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/index.ts:3:37

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {ChangedRange, TreeFragment, PartialParse, Parser, Input, ParseWrapper} from "./parse"
                                                          ~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/index.ts:3:59

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {ChangedRange, TreeFragment, PartialParse, Parser, Input, ParseWrapper} from "./parse"
                                                                 ~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/index.ts:3:66

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {NestedParse, parseMixed} from "./mix"
        ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/index.ts:4:9

TS2580 [ERROR]: Cannot find name 'process'. Do you need to install type definitions for node? Try `npm i --save-dev @types/node`.
const verbose = typeof process != "undefined" && /\bparse\b/.test(process.env.LOG!)
                       ~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/lr@0.15.8/src/parse.ts:12:24

TS2580 [ERROR]: Cannot find name 'process'. Do you need to install type definitions for node? Try `npm i --save-dev @types/node`.
const verbose = typeof process != "undefined" && /\bparse\b/.test(process.env.LOG!)
                                                                  ~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/lr@0.15.8/src/parse.ts:12:67

TS1205 [ERROR]: Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'.
export {LRParser, ParserConfig, ContextTracker} from "./parse"
                  ~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/lr@0.15.8/src/index.ts:1:19

Found 43 errors.
```

There are 43 errors. Some are re-exported types, but there are a few others
as well. Let's patch them up. First we'll patch the re-exported types, by
using regular expressions on the output:

- Loop through each occurrence of the `Re-exporting a type error` (index)
- Get the next line, which is the code (index + 1)
- Get the location of the underline on the line after that (index + 2)
- Get the repo (`:owner/:repo`) and the filename (index + 3)
- Group ones that match the same line
- Get the exported modules and detect which are underlined
- Generate a line for `export` with ones that aren't underlined
- Generate a line for `export type` with ones that are underlined
- Add a patch entry to the contents of `deps8.json` and write it to `deps9.json`

##### `e7/add_patches_for_export.js`

```ts
import deps from './deps8.json' assert { type: "json" };

function* getErrors(denoOutput) {
  let remaining = denoOutput;
  while (true) {
    const index = remaining.findIndex(line => line.startsWith('TS1205'));
    if (index !== -1) {
      yield remaining.slice(index + 1, index + 4);
      remaining = remaining.slice(index + 4);
    } else {
      return;
    }
  }
}

const fileRegex = /([^\//@]+\/[^\//@]+)@([^\//]+)\/([^:]+):([^:]+):/;
const denoOutput = (await Deno.readTextFile('./output.txt')).split("\n");
const lineErrors = {};
for (const [codeLine, underline, fileLine] of getErrors(denoOutput)) {
  const [github, version, path, lineStr] = fileRegex.exec(fileLine).slice(1);
  const line = Number(lineStr) - 1;
  lineErrors[github] = lineErrors[github] || {};
  lineErrors[github][path] = lineErrors[github][path] || {types: new Map(), version};
  const types = lineErrors[github][path].types;
  types.set(line, types.get(line) || []);
  const type = codeLine.slice(underline.indexOf('~')).split(/[,}]/, 1)[0].trim();
  types.get(line).push(type);
}
for (const [github, repoErrors] of Object.entries(lineErrors)) {
  for (const [path, {version, types}] of Object.entries(repoErrors)) {
    const url = `https://cdn.jsdelivr.net/gh/${github}@${version}/${path}`;
    const resp = await fetch(url);
    const lines = (await resp.text()).split("\n");
    const patches = new Map();
    for (const line of types.keys()) {
      const start = lines.slice(0, line + 1).findLastIndex(s => s.includes('{'));
      const end = line + lines.slice(line).findIndex(s => s.includes('}'));
      const patch = patches.get(start) || {types: [], before: lines.slice(start, end + 1)};
      patch.types = [...patch.types, ...types.get(line)];
      patches.set(start, patch);
    }
    for (const patch of patches.values()) {
      const before = patch.before.join(" ");
      const items = before.split("{")[1].split("}")[0].split(",").map(s => s.trim());
      patch.values = items.filter(s => !patch.types.includes(s));
      const pre = before.split("{")[0] + "{";
      const post = "}" + before.split("}")[1];
      patch.after = [
        patch.values.length > 0 ? pre + patch.values.join(", ") + post : undefined,
        patch.types.length > 0 ? pre.replace('export', 'export type') + patch.types.join(", ") + post : undefined,
      ].filter(s => s !== undefined);
    }
    const dep = deps.deps.find(dep => dep.github === github);
    dep.patch[path] = [
      ...(dep.patch[path] || []),
      ...[...patches.values()].map(value => [value.before, value.after].map(v => (
        v.length === 1 ? v[0] : v
      )))
    ];
  }
}
await Deno.writeTextFile("./deps9.json", JSON.stringify(deps, null, 2));
```

Running this:

```
deno run --allow-read=output.txt --allow-write=deps9.json --allow-net=cdn.jsdelivr.net add_patches_for_export.js
```

Here is `deps9.json`. It adds quite a few patches.

##### `e7/deps9.json`

```json
{
  "deps": [
    {
      "npm": "style-mod",
      "github": "marijnh/style-mod"
    },
    {
      "npm": "@codemirror/rangeset",
      "github": "codemirror/rangeset"
    },
    {
      "npm": "@codemirror/state",
      "github": "codemirror/state",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ],
          [
            "export {EditorStateConfig, EditorState} from \"./state\"",
            [
              "export {EditorState} from \"./state\"",
              "export type {EditorStateConfig} from \"./state\""
            ]
          ],
          [
            "export {StateCommand} from \"./extension\"",
            "export type {StateCommand} from \"./extension\""
          ],
          [
            "export {Facet, StateField, Extension, Prec, Compartment} from \"./facet\"",
            [
              "export {Facet, StateField, Prec, Compartment} from \"./facet\"",
              "export type {Extension} from \"./facet\""
            ]
          ],
          [
            "export {Transaction, TransactionSpec, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
            [
              "export {Transaction, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
              "export type {TransactionSpec} from \"./transaction\""
            ]
          ],
          [
            "export {ChangeSpec, ChangeSet, ChangeDesc, MapMode} from \"./change\"",
            [
              "export {ChangeSet, ChangeDesc, MapMode} from \"./change\"",
              "export type {ChangeSpec} from \"./change\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/text",
      "github": "codemirror/text",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/commands",
      "github": "codemirror/commands",
      "patch": {}
    },
    {
      "npm": "@codemirror/language",
      "github": "codemirror/language",
      "patch": {}
    },
    {
      "npm": "@codemirror/view",
      "github": "codemirror/view",
      "patch": {
        "src/index.ts": [
          [
            "export {EditorView, DOMEventMap, DOMEventHandlers} from \"./editorview\"",
            [
              "export {EditorView} from \"./editorview\"",
              "export type {DOMEventMap, DOMEventHandlers} from \"./editorview\""
            ]
          ],
          [
            "export {Command, ViewPlugin, PluginValue, PluginSpec, PluginFieldProvider, PluginField, ViewUpdate, logException} from \"./extension\"",
            [
              "export {ViewPlugin, PluginFieldProvider, PluginField, ViewUpdate, logException} from \"./extension\"",
              "export type {Command, PluginValue, PluginSpec} from \"./extension\""
            ]
          ],
          [
            "export {Decoration, DecorationSet, WidgetType, BlockType} from \"./decoration\"",
            [
              "export {Decoration, WidgetType, BlockType} from \"./decoration\"",
              "export type {DecorationSet} from \"./decoration\""
            ]
          ],
          [
            "export {MouseSelectionStyle} from \"./input\"",
            "export type {MouseSelectionStyle} from \"./input\""
          ],
          [
            "export {KeyBinding, keymap, runScopeHandlers} from \"./keymap\"",
            [
              "export {keymap, runScopeHandlers} from \"./keymap\"",
              "export type {KeyBinding} from \"./keymap\""
            ]
          ],
          [
            "export {Rect} from \"./dom\"",
            "export type {Rect} from \"./dom\""
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/matchbrackets",
      "github": "codemirror/matchbrackets",
      "patch": {}
    },
    {
      "npm": "@lezer/common",
      "github": "lezer-parser/common",
      "entry": "src/index.ts",
      "patch": {
        "src/index.ts": [
          [
            [
              "export {DefaultBufferLength, NodeProp, MountedTree, NodePropSource, NodeType, NodeSet, Tree,",
              "        TreeBuffer, SyntaxNode, TreeCursor, BufferCursor} from \"./tree\""
            ],
            [
              "export {DefaultBufferLength, NodeProp, MountedTree, NodeType, NodeSet, Tree, TreeBuffer, TreeCursor} from \"./tree\"",
              "export type {NodePropSource, SyntaxNode, BufferCursor} from \"./tree\""
            ]
          ],
          [
            "export {ChangedRange, TreeFragment, PartialParse, Parser, Input, ParseWrapper} from \"./parse\"",
            [
              "export {TreeFragment, Parser} from \"./parse\"",
              "export type {ChangedRange, PartialParse, Input, ParseWrapper} from \"./parse\""
            ]
          ],
          [
            "export {NestedParse, parseMixed} from \"./mix\"",
            [
              "export {parseMixed} from \"./mix\"",
              "export type {NestedParse} from \"./mix\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@lezer/lr",
      "github": "lezer-parser/lr",
      "entry": "src/index.ts",
      "patch": {
        "src/index.ts": [
          [
            "export {LRParser, ParserConfig, ContextTracker} from \"./parse\"",
            [
              "export {LRParser, ContextTracker} from \"./parse\"",
              "export type {ParserConfig} from \"./parse\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "w3c-keyname",
      "github": "marijnh/w3c-keyname"
    }
  ]
}
```

We'll build with `deps9.json`:

```bash
deno run --allow-env=GITHUB_API_TOKEN \
--allow-net=api.github.com,cdn.jsdelivr.net \
--allow-read=deps9.json,patch \
--allow-write=import-map.json,patch \
build.js \
deps9.json
```

We'll bundle it and inspect the errors again:

```bash
NO_COLOR=1 deno bundle --config=browser-bundle-config.json --import-map=import-map.json example2.ts > output2.txt 2>&1
```

##### `e7/output2.txt`

```
Check file:///Users/bat/proyectos/notebook/macchiato/build/deno/codemirror/e7/example2.ts
error: TS2612 [ERROR]: Property 'point' will overwrite the base property in 'RangeValue'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  point!: boolean
  ~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/decoration.ts:185:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: Text | null
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/inlineview.ts:12:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: HTMLElement | null
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/inlineview.ts:67:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: HTMLElement | null
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/inlineview.ts:154:3

TS2612 [ERROR]: Property 'widget' will overwrite the base property in 'WidgetView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  widget!: CompositionWidget
  ~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/inlineview.ts:238:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: HTMLElement | null
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/inlineview.ts:315:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: HTMLElement | null
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/blockview.ts:18:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: HTMLElement | null
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/blockview.ts:155:3

TS2612 [ERROR]: Property 'parent' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  parent!: DocView | null
  ~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/blockview.ts:156:3

TS2612 [ERROR]: Property 'dom' will overwrite the base property in 'ContentView'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  dom!: HTMLElement
  ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/docview.ts:41:3

TS2305 [ERROR]: Module '"https://cdn.jsdelivr.net/gh/marijnh/style-mod@4.0.0/src/style-mod.js"' has no exported member 'StyleSpec'.
import {StyleModule, StyleSpec} from "style-mod"
                     ~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/theme.ts:2:22

TS7006 [ERROR]: Parameter 'sel' implicitly has an 'any' type.
    finish(sel) {
           ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/theme.ts:14:12

TS7006 [ERROR]: Parameter 'm' implicitly has an 'any' type.
      return /&/.test(sel) ? sel.replace(/&\w*/, m => {
                                                 ^
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/theme.ts:15:50

TS2305 [ERROR]: Module '"https://cdn.jsdelivr.net/gh/marijnh/style-mod@4.0.0/src/style-mod.js"' has no exported member 'StyleSpec'.
import {StyleModule, StyleSpec} from "style-mod"
                     ~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/editorview.ts:4:22

TS7053 [ERROR]: Element implicitly has an 'any' type because expression of type 'number' can't be used to index type '{ 8: string; 9: string; 10: string; 12: string; 13: string; 16: string; 17: string; 18: string; 20: string; 27: string; 32: string; 33: string; 34: string; 35: string; 36: string; 37: string; 38: string; 39: string; 40: string; ... 33 more ...; 229: string; }'.
  No index signature with a parameter of type 'number' was found on type '{ 8: string; 9: string; 10: string; 12: string; 13: string; 16: string; 17: string; 18: string; 20: string; 27: string; 32: string; 33: string; 34: string; 35: string; 36: string; 37: string; 38: string; 39: string; 40: string; ... 33 more ...; 229: string; }'.
      (baseName = base[event.keyCode]) && baseName != name) {
                  ~~~~~~~~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/keymap.ts:208:19

TS7022 [ERROR]: 'parent' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:797:7

TS7023 [ERROR]: 'parent' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:797:7

TS7022 [ERROR]: 'nextSibling' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get nextSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:801:7

TS7023 [ERROR]: 'nextSibling' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get nextSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:801:7

TS7022 [ERROR]: 'prevSibling' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get prevSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:804:7

TS7023 [ERROR]: 'prevSibling' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get prevSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:804:7

TS7022 [ERROR]: 'parent' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:889:7

TS7023 [ERROR]: 'parent' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:889:7

TS2580 [ERROR]: Cannot find name 'process'. Do you need to install type definitions for node? Try `npm i --save-dev @types/node`.
const verbose = typeof process != "undefined" && /\bparse\b/.test(process.env.LOG!)
                       ~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/lr@0.15.8/src/parse.ts:12:24

TS2580 [ERROR]: Cannot find name 'process'. Do you need to install type definitions for node? Try `npm i --save-dev @types/node`.
const verbose = typeof process != "undefined" && /\bparse\b/.test(process.env.LOG!)
                                                                  ~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/lr@0.15.8/src/parse.ts:12:67

Found 25 errors.
```

Down to 25 errors. We'll modify the script to fix them. Because it will still do
what the previous script did, it will start with `deps8.json` and output `deps10.json`.
Also, these will be provided on the command line instead.

##### `e7/add_patches.js`

```ts
function* getErrors(denoOutput, prefix) {
  let remaining = denoOutput;
  while (true) {
    const index = remaining.findIndex(line => line.startsWith(prefix));
    const blankIndex = index + 1 + remaining.slice(index + 1).findIndex(line => line.trim() === '');
    if (index !== -1) {
      yield remaining.slice(index, blankIndex);
      remaining = remaining.slice(blankIndex + 1);
    } else {
      return;
    }
  }
}

const sourceFiles = {};
async function getSourceLines({github, version, path}) {
  const url = `https://cdn.jsdelivr.net/gh/${github}@${version}/${path}`;
  if (typeof sourceFiles[url] === 'string') {
    return sourceFiles[url].split("\n");
  }
  const resp = await fetch(url);
  const text = await resp.text();
  sourceFiles[url] = text;
  return text.split("\n");
}

const [inputFile, outputFile] = Deno.args;

const deps = JSON.parse(await Deno.readTextFile(inputFile));

const fileRegex = /([^\//@]+\/[^\//@]+)@([^\//]+)\/([^:]+):([^:]+):/;
const denoOutput = (await Deno.readTextFile('./output.txt')).split("\n");
const lineErrors = {};
for (const lines of getErrors(denoOutput, 'TS1205 ')) {
  const [codeLine, underline, fileLine] = lines.slice(1);
  const [github, version, path, lineStr] = fileRegex.exec(fileLine).slice(1);
  const line = Number(lineStr) - 1;
  lineErrors[github] = lineErrors[github] || {};
  lineErrors[github][path] = lineErrors[github][path] || {types: new Map(), version};
  const types = lineErrors[github][path].types;
  types.set(line, types.get(line) || []);
  const type = codeLine.slice(underline.indexOf('~')).split(/[,}]/, 1)[0].trim();
  types.get(line).push(type);
}
for (const [github, repoErrors] of Object.entries(lineErrors)) {
  for (const [path, {version, types}] of Object.entries(repoErrors)) {
    const lines = await getSourceLines({github, version, path});
    const patches = new Map();
    for (const line of types.keys()) {
      const start = lines.slice(0, line + 1).findLastIndex(s => s.includes('{'));
      const end = line + lines.slice(line).findIndex(s => s.includes('}'));
      const patch = patches.get(start) || {types: [], before: lines.slice(start, end + 1)};
      patch.types = [...patch.types, ...types.get(line)];
      patches.set(start, patch);
    }
    for (const patch of patches.values()) {
      const before = patch.before.join(" ");
      const items = before.split("{")[1].split("}")[0].split(",").map(s => s.trim());
      patch.values = items.filter(s => !patch.types.includes(s));
      const pre = before.split("{")[0] + "{";
      const post = "}" + before.split("}")[1];
      patch.after = [
        patch.values.length > 0 ? pre + patch.values.join(", ") + post : undefined,
        patch.types.length > 0 ? pre.replace('export', 'export type') + patch.types.join(", ") + post : undefined,
      ].filter(s => s !== undefined);
    }
    const dep = deps.deps.find(dep => dep.github === github);
    dep.patch[path] = [
      ...(dep.patch[path] || []),
      ...[...patches.values()].map(value => [value.before, value.after].map(v => (
        v.length === 1 ? v[0] : v
      )))
    ];
  }
}

for (const errorLines of getErrors(denoOutput, 'TS2612 ')) {
  const fileLine = errorLines[3];
  const [github, version, path, lineStr] = fileRegex.exec(fileLine).slice(1);
  const line = Number(lineStr) - 1;
  const lines = await getSourceLines({github, version, path});
  const before = lines[line];
  const match = /^(\s*)(\S*)!:(.*)$/.exec(before);
  if (!match) {
    throw new Error(`Didn't match regex: ${before}`);
  }
  const after = match[1] + 'declare ' + match[2] + ':' + match[3];
  const dep = deps.deps.find(dep => dep.github === github);
  dep.patch[path] = [
    ...(dep.patch[path] || []),
    [before, after],
  ];
}

await Deno.writeTextFile(outputFile, JSON.stringify(deps, null, 2));
```

Running it:

```
deno run --allow-read=output.txt,deps8.json --allow-write=deps10.json \
--allow-net=cdn.jsdelivr.net \
add_patches.js deps8.json deps10.json
```

Here is the updated file:

##### `deps10.json`

```json
{
  "deps": [
    {
      "npm": "style-mod",
      "github": "marijnh/style-mod"
    },
    {
      "npm": "@codemirror/rangeset",
      "github": "codemirror/rangeset"
    },
    {
      "npm": "@codemirror/state",
      "github": "codemirror/state",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ],
          [
            "export {EditorStateConfig, EditorState} from \"./state\"",
            [
              "export {EditorState} from \"./state\"",
              "export type {EditorStateConfig} from \"./state\""
            ]
          ],
          [
            "export {StateCommand} from \"./extension\"",
            "export type {StateCommand} from \"./extension\""
          ],
          [
            "export {Facet, StateField, Extension, Prec, Compartment} from \"./facet\"",
            [
              "export {Facet, StateField, Prec, Compartment} from \"./facet\"",
              "export type {Extension} from \"./facet\""
            ]
          ],
          [
            "export {Transaction, TransactionSpec, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
            [
              "export {Transaction, Annotation, AnnotationType, StateEffect, StateEffectType} from \"./transaction\"",
              "export type {TransactionSpec} from \"./transaction\""
            ]
          ],
          [
            "export {ChangeSpec, ChangeSet, ChangeDesc, MapMode} from \"./change\"",
            [
              "export {ChangeSet, ChangeDesc, MapMode} from \"./change\"",
              "export type {ChangeSpec} from \"./change\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/text",
      "github": "codemirror/text",
      "patch": {
        "src/index.ts": [
          [
            "export {Line, TextIterator, Text} from \"./text\"",
            [
              "export {Line, Text} from \"./text\"",
              "export type {TextIterator} from \"./text\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/commands",
      "github": "codemirror/commands",
      "patch": {}
    },
    {
      "npm": "@codemirror/language",
      "github": "codemirror/language",
      "patch": {}
    },
    {
      "npm": "@codemirror/view",
      "github": "codemirror/view",
      "patch": {
        "src/index.ts": [
          [
            "export {EditorView, DOMEventMap, DOMEventHandlers} from \"./editorview\"",
            [
              "export {EditorView} from \"./editorview\"",
              "export type {DOMEventMap, DOMEventHandlers} from \"./editorview\""
            ]
          ],
          [
            "export {Command, ViewPlugin, PluginValue, PluginSpec, PluginFieldProvider, PluginField, ViewUpdate, logException} from \"./extension\"",
            [
              "export {ViewPlugin, PluginFieldProvider, PluginField, ViewUpdate, logException} from \"./extension\"",
              "export type {Command, PluginValue, PluginSpec} from \"./extension\""
            ]
          ],
          [
            "export {Decoration, DecorationSet, WidgetType, BlockType} from \"./decoration\"",
            [
              "export {Decoration, WidgetType, BlockType} from \"./decoration\"",
              "export type {DecorationSet} from \"./decoration\""
            ]
          ],
          [
            "export {MouseSelectionStyle} from \"./input\"",
            "export type {MouseSelectionStyle} from \"./input\""
          ],
          [
            "export {KeyBinding, keymap, runScopeHandlers} from \"./keymap\"",
            [
              "export {keymap, runScopeHandlers} from \"./keymap\"",
              "export type {KeyBinding} from \"./keymap\""
            ]
          ],
          [
            "export {Rect} from \"./dom\"",
            "export type {Rect} from \"./dom\""
          ]
        ],
        "src/inlineview.ts": [
          [
            "  dom!: Text | null",
            "  declare dom: Text | null"
          ],
          [
            "  dom!: HTMLElement | null",
            "  declare dom: HTMLElement | null"
          ],
          [
            "  dom!: HTMLElement | null",
            "  declare dom: HTMLElement | null"
          ],
          [
            "  widget!: CompositionWidget",
            "  declare widget: CompositionWidget"
          ],
          [
            "  dom!: HTMLElement | null",
            "  declare dom: HTMLElement | null"
          ]
        ],
        "src/blockview.ts": [
          [
            "  dom!: HTMLElement | null",
            "  declare dom: HTMLElement | null"
          ],
          [
            "  dom!: HTMLElement | null",
            "  declare dom: HTMLElement | null"
          ],
          [
            "  parent!: DocView | null",
            "  declare parent: DocView | null"
          ]
        ],
        "src/docview.ts": [
          [
            "  dom!: HTMLElement",
            "  declare dom: HTMLElement"
          ]
        ]
      }
    },
    {
      "npm": "@codemirror/matchbrackets",
      "github": "codemirror/matchbrackets",
      "patch": {}
    },
    {
      "npm": "@lezer/common",
      "github": "lezer-parser/common",
      "entry": "src/index.ts",
      "patch": {
        "src/index.ts": [
          [
            [
              "export {DefaultBufferLength, NodeProp, MountedTree, NodePropSource, NodeType, NodeSet, Tree,",
              "        TreeBuffer, SyntaxNode, TreeCursor, BufferCursor} from \"./tree\""
            ],
            [
              "export {DefaultBufferLength, NodeProp, MountedTree, NodeType, NodeSet, Tree, TreeBuffer, TreeCursor} from \"./tree\"",
              "export type {NodePropSource, SyntaxNode, BufferCursor} from \"./tree\""
            ]
          ],
          [
            "export {ChangedRange, TreeFragment, PartialParse, Parser, Input, ParseWrapper} from \"./parse\"",
            [
              "export {TreeFragment, Parser} from \"./parse\"",
              "export type {ChangedRange, PartialParse, Input, ParseWrapper} from \"./parse\""
            ]
          ],
          [
            "export {NestedParse, parseMixed} from \"./mix\"",
            [
              "export {parseMixed} from \"./mix\"",
              "export type {NestedParse} from \"./mix\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "@lezer/lr",
      "github": "lezer-parser/lr",
      "entry": "src/index.ts",
      "patch": {
        "src/index.ts": [
          [
            "export {LRParser, ParserConfig, ContextTracker} from \"./parse\"",
            [
              "export {LRParser, ContextTracker} from \"./parse\"",
              "export type {ParserConfig} from \"./parse\""
            ]
          ]
        ]
      }
    },
    {
      "npm": "w3c-keyname",
      "github": "marijnh/w3c-keyname"
    }
  ]
}
```

In the above, in `blockview.ts` there are two lines that have the same thing. They
change to the same thing, so this will work, but the patch format should have the
line number.

Building the import map and the patched files:

```
deno run --allow-env=GITHUB_API_TOKEN \
--allow-net=api.github.com,cdn.jsdelivr.net \
--allow-read=deps10.json,patch \
--allow-write=import-map.json,patch \
build.js \
deps10.json
```

Bundling again:

```bash
NO_COLOR=1 deno bundle --config=browser-bundle-config.json --import-map=import-map.json example2.ts > output3.txt 2>&1
```

##### `output3.txt`

```
Check file:///Users/bat/proyectos/notebook/macchiato/build/deno/codemirror/e7/example2.ts
error: TS2305 [ERROR]: Module '"https://cdn.jsdelivr.net/gh/marijnh/style-mod@4.0.0/src/style-mod.js"' has no exported member 'StyleSpec'.
import {StyleModule, StyleSpec} from "style-mod"
                     ~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/theme.ts:2:22

TS7006 [ERROR]: Parameter 'sel' implicitly has an 'any' type.
    finish(sel) {
           ~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/theme.ts:14:12

TS7006 [ERROR]: Parameter 'm' implicitly has an 'any' type.
      return /&/.test(sel) ? sel.replace(/&\w*/, m => {
                                                 ^
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/theme.ts:15:50

TS2305 [ERROR]: Module '"https://cdn.jsdelivr.net/gh/marijnh/style-mod@4.0.0/src/style-mod.js"' has no exported member 'StyleSpec'.
import {StyleModule, StyleSpec} from "style-mod"
                     ~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/editorview.ts:4:22

TS2612 [ERROR]: Property 'point' will overwrite the base property in 'RangeValue'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  point!: boolean
  ~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/decoration.ts:185:3

TS7022 [ERROR]: 'parent' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:797:7

TS7023 [ERROR]: 'parent' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:797:7

TS7022 [ERROR]: 'nextSibling' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get nextSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:801:7

TS7023 [ERROR]: 'nextSibling' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get nextSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:801:7

TS7022 [ERROR]: 'prevSibling' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get prevSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:804:7

TS7023 [ERROR]: 'prevSibling' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get prevSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:804:7

TS7022 [ERROR]: 'parent' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:889:7

TS7023 [ERROR]: 'parent' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:889:7

TS2580 [ERROR]: Cannot find name 'process'. Do you need to install type definitions for node? Try `npm i --save-dev @types/node`.
const verbose = typeof process != "undefined" && /\bparse\b/.test(process.env.LOG!)
                       ~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/lr@0.15.8/src/parse.ts:12:24

TS2580 [ERROR]: Cannot find name 'process'. Do you need to install type definitions for node? Try `npm i --save-dev @types/node`.
const verbose = typeof process != "undefined" && /\bparse\b/.test(process.env.LOG!)
                                                                  ~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/lr@0.15.8/src/parse.ts:12:67

TS7053 [ERROR]: Element implicitly has an 'any' type because expression of type 'number' can't be used to index type '{ 8: string; 9: string; 10: string; 12: string; 13: string; 16: string; 17: string; 18: string; 20: string; 27: string; 32: string; 33: string; 34: string; 35: string; 36: string; 37: string; 38: string; 39: string; 40: string; ... 33 more ...; 229: string; }'.
  No index signature with a parameter of type 'number' was found on type '{ 8: string; 9: string; 10: string; 12: string; 13: string; 16: string; 17: string; 18: string; 20: string; 27: string; 32: string; 33: string; 34: string; 35: string; 36: string; 37: string; 38: string; 39: string; 40: string; ... 33 more ...; 229: string; }'.
      (baseName = base[event.keyCode]) && baseName != name) {
                  ~~~~~~~~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/keymap.ts:208:19

Found 16 errors.
```

The first error here is due to missing type definitions:

```
Check file:///Users/bat/proyectos/notebook/macchiato/build/deno/codemirror/e7/example2.ts
error: TS2305 [ERROR]: Module '"https://cdn.jsdelivr.net/gh/marijnh/style-mod@4.0.0/src/style-mod.js"' has no exported member 'StyleSpec'.
import {StyleModule, StyleSpec} from "style-mod"
                     ~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/theme.ts:2:22
```

`style-mod.js` doesn't export a type, because it isn't a typescript file, and doesn't
include the TypeScript definitions in a way that Deno can use. To fix this, a
reference comment needs to be added to the `.js` file:

```js
/// <reference types="./style-mod.d.ts" />
```

Let's see if this fixes it, by manually adding a patch to `deps10.js`. Here we'll take
replace the first three lines of `style-mod.js` so it will match.

After this, we'll change the patch format to include the line number so something
can be added to the top without depending on the content of the file.

##### `e7/add_type_reference.js`

```ts
const [inputFile, outputFile] = Deno.args;
const deps = JSON.parse(await Deno.readTextFile(inputFile));
const dep = deps.deps.find(({github}) => github === 'marijnh/style-mod');
const url = 'https://cdn.jsdelivr.net/gh/marijnh/style-mod@4.0.0/src/style-mod.js';
const lines = (await (await fetch(url)).text()).split("\n");
const newLine = '/// <reference types="https://cdn.jsdelivr.net/gh/marijnh/style-mod@4.0.0/src/style-mod.d.ts" />';
dep.patch = {
  ...dep.patch,
  'src/style-mod.js': [
    ...((dep.patch && dep.patch['src/style-mod.js']) || []),
    [lines.slice(0, 3), [newLine, ...lines.slice(0, 3)]]
  ],
};
await Deno.writeTextFile(outputFile, JSON.stringify(deps, null, 2));
```

Running it:

```
deno run --allow-read=deps10.json --allow-write=deps11.json \
--allow-net=cdn.jsdelivr.net \
add_type_reference.js deps10.json deps11.json
```

Building it:

```bash
deno run --allow-env=GITHUB_API_TOKEN \
--allow-net=api.github.com,cdn.jsdelivr.net \
--allow-read=deps11.json,patch \
--allow-write=import-map.json,patch \
build.js \
deps11.json
```

Bundling:

```bash
NO_COLOR=1 deno bundle --config=browser-bundle-config.json --import-map=import-map.json example2.ts > output4.txt 2>&1
```

##### `e7/output4.txt`

```
Download https://cdn.jsdelivr.net/gh/marijnh/style-mod@4.0.0/src/style-mod.d.ts
Check file:///Users/bat/proyectos/notebook/macchiato/build/deno/codemirror/e7/example2.ts
error: TS7022 [ERROR]: 'parent' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:797:7

TS7023 [ERROR]: 'parent' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:797:7

TS7022 [ERROR]: 'nextSibling' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get nextSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:801:7

TS7023 [ERROR]: 'nextSibling' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get nextSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:801:7

TS7022 [ERROR]: 'prevSibling' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get prevSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:804:7

TS7023 [ERROR]: 'prevSibling' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get prevSibling() {
      ~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:804:7

TS7022 [ERROR]: 'parent' implicitly has type 'any' because it does not have a type annotation and is referenced directly or indirectly in its own initializer.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:889:7

TS7023 [ERROR]: 'parent' implicitly has return type 'any' because it does not have a return type annotation and is referenced directly or indirectly in one of its return expressions.
  get parent() {
      ~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/common@0.15.12/src/tree.ts:889:7

TS2612 [ERROR]: Property 'point' will overwrite the base property in 'RangeValue'. If this is intentional, add an initializer. Otherwise, add a 'declare' modifier or remove the redundant declaration.
  point!: boolean
  ~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/decoration.ts:185:3

TS2580 [ERROR]: Cannot find name 'process'. Do you need to install type definitions for node? Try `npm i --save-dev @types/node`.
const verbose = typeof process != "undefined" && /\bparse\b/.test(process.env.LOG!)
                       ~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/lr@0.15.8/src/parse.ts:12:24

TS2580 [ERROR]: Cannot find name 'process'. Do you need to install type definitions for node? Try `npm i --save-dev @types/node`.
const verbose = typeof process != "undefined" && /\bparse\b/.test(process.env.LOG!)
                                                                  ~~~~~~~
    at https://cdn.jsdelivr.net/gh/lezer-parser/lr@0.15.8/src/parse.ts:12:67

TS7053 [ERROR]: Element implicitly has an 'any' type because expression of type 'number' can't be used to index type '{ 8: string; 9: string; 10: string; 12: string; 13: string; 16: string; 17: string; 18: string; 20: string; 27: string; 32: string; 33: string; 34: string; 35: string; 36: string; 37: string; 38: string; 39: string; 40: string; ... 33 more ...; 229: string; }'.
  No index signature with a parameter of type 'number' was found on type '{ 8: string; 9: string; 10: string; 12: string; 13: string; 16: string; 17: string; 18: string; 20: string; 27: string; 32: string; 33: string; 34: string; 35: string; 36: string; 37: string; 38: string; 39: string; 40: string; ... 33 more ...; 229: string; }'.
      (baseName = base[event.keyCode]) && baseName != name) {
                  ~~~~~~~~~~~~~~~~~~~
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/keymap.ts:208:19

Found 12 errors.
```

Down from 16 errors to 12 errors. We'll patch these up one by one. This took several
runs to get it working, and those extra runs are ommitted.

##### `e7/manual_patches.json`

```json
{
  "lezer-parser/common": {
    "src/tree.ts": [
      [
        [
          "  get parent() {",
          "    return this._parent || this.context.parent.nextSignificantParent()"
        ],
        [
          "  get parent(): TreeNode | BufferNode | null {",
          "    return this._parent || this.context.parent.nextSignificantParent()"
        ]
      ],
      [
        [
          "  get parent() {",
          "    return this._parent ? this._parent.nextSignificantParent() : null"
        ],
        [
          "  get parent(): TreeNode | null {",
          "    return this._parent ? this._parent.nextSignificantParent() : null"
        ]
      ],
      [
        "  get nextSibling() {",
        "  get nextSibling(): TreeNode | BufferNode | null {"
      ],
      [
        "  get prevSibling() {",
        "  get prevSibling(): TreeNode | BufferNode | null {"
      ]
    ]
  },
  "codemirror/view": {
    "src/decoration.ts": [
      [
        "point!: boolean",
        "declare point: boolean"
      ]
    ],
    "src/keymap.ts": [
      [
        "      (baseName = base[event.keyCode]) && baseName != name) {",
        "      (baseName = base[event.keyCode as keyof typeof base]) && baseName != name) {"
      ]
    ]
  },
  "lezer-parser/lr": {
    "src/parse.ts": [
      [
        "const verbose = typeof process != \"undefined\" && /\\bparse\\b/.test(process.env.LOG!)",
        "const verbose = false"
      ]
    ]
  }
}
```

##### `e7/add_manual_patches.js`

```ts
const [inputFile, manualPatchFile, outputFile] = Deno.args;
const deps = JSON.parse(await Deno.readTextFile(inputFile));
const manualPatches = JSON.parse(await Deno.readTextFile(manualPatchFile));
for (const [github, repoPatches] of Object.entries(manualPatches)) {
  const dep = deps.deps.find(({github: value}) => value === github);
  for (const [file, patches] of Object.entries(repoPatches)) {
    dep.patch = {
      ...dep.patch,
      [file]: [
        ...((dep.patch && dep.patch[file]) || []),
        ...patches,
      ],
    };
  }
}

await Deno.writeTextFile(outputFile, JSON.stringify(deps, null, 2));
```

Running it:

```
deno run --allow-read=deps11.json,manual_patches.json --allow-write=deps12.json \
--allow-net=cdn.jsdelivr.net \
add_manual_patches.js deps11.json manual_patches.json deps12.json
```

Building it:

```bash
deno run --allow-env=GITHUB_API_TOKEN \
--allow-net=api.github.com,cdn.jsdelivr.net \
--allow-read=deps12.json,patch \
--allow-write=import-map.json,patch \
build.js \
deps12.json
```

Bundling:

```bash
NO_COLOR=1 deno bundle --config=browser-bundle-config.json --import-map=import-map.json example2.ts > bundle.js
```

Here's an HTML file to import the bundle:

##### `e7/index.html`

```html
<!doctype html>
<html>
  <head>
    <title>Example</title>
  </head>
  <body>
    <script src="/bundle.js" type="module"></script>
  </body>
</html>
```

Now testing it:

```
deno run --allow-net --allow-read https://deno.land/std/http/file_server.ts
> HTTP server listening on http://localhost:4507/
```

There is an error that it's missing `isFieldProvider` which is declared as
`declare const isFieldProvider: unique symbol` in the [source](https://github.com/codemirror/view/blob/5818c19651397af1363c8f85f6ec4f16ae911ba4/src/extension.ts#L95). Nothing shows up.

If I go into `bundle.js` and manually add `const isFieldProvider = Symbol();`, that
error goes away, and it shows a text box, but there are a bunch of errors and it loses
text when selecting it.

The same error shows when trying to run directly from Deno:

```
❯ deno run --config=browser-bundle-config.json --import-map=import-map.json example2.ts
Check file:///Users/bat/proyectos/notebook/macchiato/build/deno/codemirror/e7/example2.ts
error: Uncaught ReferenceError: isFieldProvider is not defined
  private [isFieldProvider]!: true
           ^
    at https://cdn.jsdelivr.net/gh/codemirror/view@0.19.47/src/extension.ts:102:12
```

