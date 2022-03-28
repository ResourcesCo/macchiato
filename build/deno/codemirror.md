# Bundling CodeMirror 6

The goal here is to make a CodeMirror 6-based code editing component from
npm with Deno. This will use import maps so npm modules can find each
other without modifying the source.

A previous attempt was to bundle straight from npm. It built, but doesn't
work yet:

- [Bundling CodeMirror 6 from GitHub with Deno](./codemirror-from-github.md)

## Getting dependencies and generating an import map

Here is code for loading an editor, from the [Bundling with Rollup][bwr]
example from CodeMirror:

##### `e1/example.ts`

```ts
import {EditorState, EditorView, basicSetup} from "@codemirror/basic-setup"
import {javascript} from "@codemirror/lang-javascript"

let editor = new EditorView({
  state: EditorState.create({
    extensions: [basicSetup, javascript()]
  }),
  parent: document.body
})
```

For `deno bundle` to be able to bundle this, we need an import map. The
import map needs to map `npm` packages to 

We'll do this without relying on the `npm` program, but instead fetching
the files straight from [jsDelivr][jsdelivr], starting with `package.json`
files.

[bwr]: https://codemirror.net/6/examples/bundle/
[jsdelivr]: https://www.jsdelivr.com/