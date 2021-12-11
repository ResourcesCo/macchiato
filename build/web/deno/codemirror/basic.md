# CodeMirror 6 Editor w/ Deno

Here we'll build a CodeMirror 6 editor with Deno using a custom-built
sourcemap, with raw TypeScript files from GitHub via jsdelivr.

## Importing a single dependency

CodeMirror's relative imports leave out the extension, and Deno
doesn't assume the extension. Let's see if we can use import maps to
get around this.

To start, let's import `@codemirror/text` and do something with it and
`console.log()` the result.

##### `e0/import-map.json`

```js
{
  "imports": {
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/char.ts",
    "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column": "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column.ts"
  }
}
```

##### `e0/app.js`

```js
import { countColumn } from "https://cdn.jsdelivr.net/gh/codemirror/text@0.19.5/src/column";
console.log([0, 1, 2, 3].map(i => countColumn("\tTest", 2, i)));
```

##### `e0/run.sh`

```bash
deno run --import-map e0/import-map.json e0/app.js
```

This will print out `[0, 2, 3, 4]`. `countColumn` counts the column of
a line with tab sizes taken into account. The second parameter to
countColumn is the tab size and the third parameter is the offset. It
prints out the column number. It jumps from 0 to 2 because of the tab.