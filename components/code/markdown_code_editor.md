# Markdown Code Editor

This is a component for editing Markdown with languages inside code blocks.

## Language

This is a language for highlighting that supports nested Markdown.

##### `language.ts`

```ts
import "./typed_imports.ts";
import { LanguageDescription, LanguageSupport } from "@codemirror/language";
import { markdown } from "@codemirror/lang-markdown";
import { jsxLanguage, tsxLanguage } from "@codemirror/lang-javascript";
import { cssLanguage } from "@codemirror/lang-css";
import { jsonLanguage } from "@codemirror/lang-json";
import { htmlLanguage } from "@codemirror/lang-html";

export function language() {
  const jsxSupport = new LanguageSupport(jsxLanguage);
  const tsSupport = new LanguageSupport(tsxLanguage);
  const cssSupport = new LanguageSupport(cssLanguage);
  const jsonSupport = new LanguageSupport(jsonLanguage);
  const htmlSupport = new LanguageSupport(htmlLanguage);
  const codeLanguages = [
    LanguageDescription.of({
      name: "javascript",
      alias: ["js", "jsx"],
      support: jsxSupport,
    }),
    LanguageDescription.of({
      name: "typescript",
      alias: ["ts", "tsx"],
      support: tsSupport,
    }),
    LanguageDescription.of({
      name: "css",
      support: cssSupport,
    }),
    LanguageDescription.of({
      name: "json",
      support: jsonSupport,
    }),
    LanguageDescription.of({
      name: "html",
      alias: ["htm"],
      support: htmlSupport,
    }),
  ];

  const markdownRoot = markdown({
    codeLanguages,
  });

  const markdownNested = markdown({
    codeLanguages: [
      ...codeLanguages,
      LanguageDescription.of({
        name: "markdown",
        alias: ["md", "mkd"],
        support: markdownRoot,
      }),
    ],
  });

  const markdownDoubleNested = markdown({
    codeLanguages: [
      ...codeLanguages,
      LanguageDescription.of({
        name: "markdown",
        alias: ["md", "mkd"],
        support: markdownNested,
      }),
    ],
  });

  return markdown({
    codeLanguages: [
      ...codeLanguages,
      LanguageDescription.of({
        name: "markdown",
        alias: ["md", "mkd"],
        support: markdownDoubleNested,
      }),
    ],
  });
}
```

## Extensions

##### `extensions.ts`

```ts
import "./typed_imports.ts";

import { EditorState } from "@codemirror/state";
import {
  crosshairCursor,
  drawSelection,
  EditorView,
  highlightActiveLine,
  highlightActiveLineGutter,
  highlightSpecialChars,
  keymap,
  rectangularSelection,
} from "@codemirror/view";
import {
  bracketMatching,
  codeFolding,
  foldGutter,
  foldKeymap,
  indentOnInput,
  LanguageDescription,
  LanguageSupport,
} from "@codemirror/language";
import { defaultKeymap, history, historyKeymap } from "@codemirror/commands";
import { highlightSelectionMatches, searchKeymap } from "@codemirror/search";
import {
  autocompletion,
  closeBrackets,
  closeBracketsKeymap,
  completionKeymap,
} from "@codemirror/autocomplete";
import { lintKeymap } from "@codemirror/lint";
import { oneDark } from "@codemirror/theme-one-dark";

export function baseExtensions() {
  return [
    codeFolding(),
    foldGutter(),
    highlightActiveLineGutter(),
    highlightSpecialChars(),
    history(),
    drawSelection(),
    EditorState.allowMultipleSelections.of(true),
    indentOnInput(),
    bracketMatching(),
    closeBrackets(),
    autocompletion({ activateOnTyping: false }),
    rectangularSelection(),
    crosshairCursor(),
    highlightActiveLine(),
    highlightSelectionMatches(),
    keymap.of([
      ...closeBracketsKeymap,
      ...defaultKeymap,
      ...searchKeymap,
      ...historyKeymap,
      ...completionKeymap,
      ...foldKeymap,
      ...lintKeymap,
    ]),
    EditorView.lineWrapping,
    oneDark,
  ];
}
```

## Code Editor Web Component

This has the plugins from the [CodeMirror Basic](https://cdn.jsdelivr.net/npm/codemirror@6.0.0/dist/index.js)
setup with some customizations to make a code editor for Markdown.

##### `mod.ts`

```ts
import "./typed_imports.ts";

import { EditorState } from "@codemirror/state";
import { EditorView, placeholder } from "@codemirror/view";

import { language } from "./language.ts";
import { baseExtensions } from "./extensions.ts";

export class CodeEditor extends HTMLElement {
  editor?: EditorView;

  constructor() {
    super();
    const shadowRoot = this.attachShadow({ mode: "open" });
    const style = document.createElement('style');
    style.textContent = `
      .cm-editor { height: 100%; width: 100%; }
    `;
    shadowRoot.append(style);
  }

  connectedCallback() {
    const extensions = this.getExtensions();
    this.editor = new EditorView({
      state: EditorState.create({
        doc: "", /*props.page.body*/
        extensions,
      }),
      parent: this.shadowRoot,
    });
  }

  getExtensions() {
    return [
      ...baseExtensions(),
      language(),
      placeholder("# Enter some markdown here..."),
      /*EditorView.updateListener.of((v) => {
        if (v.docChanged) {
          //ctx.emit("change", editor.state.doc);
        }
      }),*/
    ]
  }
}
```

To bundle:

```
deno bundle --import-map=import_map.json mod.ts bundle.js
```

To test it out:

##### `example.js`

```js
import { CodeEditor } from "./bundle.js";

window.addEventListener('DOMContentLoaded', (event) => {
  const editorEl = document.querySelector('code-editor');
  console.log(editorEl);
});

window.customElements.define("code-editor", CodeEditor);
```


##### `example.html`

```html
<!doctype html>
<html>
  <head>
    <title>Markdown Editor</title>
    <style type="text/css">
      html, body {
        background-color: #000;
        height: 1em;
        margin: 0;
        padding: 0;
      }
      .wrap {
        height: 100vh;
        display: flex;
        align-items: stretch;
      }
      code-editor {
        flex-grow: 1;
      }
      .cm-editor {
        height: 100%;
        width: 100%;
      }
    </style>
  </head>
  <body>
    <div class="wrap">
      <code-editor></code-editor>
    </div>
    <script type="module" src="./example.js"></script>
  </body>
</html>
```

Then run `file-server` and open it.

# Build

These are the dependencies, built with `@codemirror/`:

##### `dependencies.json`

```json
{
  "@codemirror/view": "*",
  "@codemirror/state": "*",
  "@codemirror/language": "*",
  "@codemirror/commands": "*",
  "@codemirror/search": "*",
  "@codemirror/autocomplete": "*",
  "@codemirror/lint": "*",
  "@codemirror/lang-markdown": "*",
  "@codemirror/lang-javascript": "*",
  "@codemirror/lang-css": "*",
  "@codemirror/lang-json": "*",
  "@codemirror/lang-html": "*",
  "@codemirror/theme-one-dark": "*"
}
```

These are manual additions to the import map in order to get it to build:

```json
{
  "imports": {
    "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/tree": "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/tree.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/parse": "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/parse.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/mix": "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/mix.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/parse": "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/parse.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/token": "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/token.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/stack": "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/stack.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/markdown": "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/markdown.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/nest": "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/nest.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/extension": "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/extension.d.ts"
  }
}
```

To generate the import map and typed imports, pipe this file into
`deliver_importmaps` and `md_unpack_simple`. This will write
`import_map.json` and `typed_imports.ts` to the current directory.
Then add the entries from above. These have been included below.

```
cat markdown_code_editor.md | deliver_importmaps | md_unpack_simple
```

##### `import_map.json`

```json
{
  "imports": {
    "@codemirror/view": "https://cdn.jsdelivr.net/npm/@codemirror/view@6.0.1/dist/index.js",
    "@codemirror/state": "https://cdn.jsdelivr.net/npm/@codemirror/state@6.0.1/dist/index.js",
    "style-mod": "https://cdn.jsdelivr.net/npm/style-mod@4.0.0/src/style-mod.js",
    "w3c-keyname": "https://cdn.jsdelivr.net/npm/w3c-keyname@2.2.4/index.es.js",
    "@codemirror/language": "https://cdn.jsdelivr.net/npm/@codemirror/language@6.1.0/dist/index.js",
    "@lezer/common": "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/index.js",
    "@lezer/highlight": "https://cdn.jsdelivr.net/npm/@lezer/highlight@1.0.0/dist/index.js",
    "@lezer/lr": "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/index.js",
    "@codemirror/commands": "https://cdn.jsdelivr.net/npm/@codemirror/commands@6.0.0/dist/index.js",
    "@codemirror/search": "https://cdn.jsdelivr.net/npm/@codemirror/search@6.0.0/dist/index.js",
    "crelt": "https://cdn.jsdelivr.net/npm/crelt@1.0.5/index.es.js",
    "@codemirror/autocomplete": "https://cdn.jsdelivr.net/npm/@codemirror/autocomplete@6.0.2/dist/index.js",
    "@codemirror/lint": "https://cdn.jsdelivr.net/npm/@codemirror/lint@6.0.0/dist/index.js",
    "@codemirror/lang-markdown": "https://cdn.jsdelivr.net/npm/@codemirror/lang-markdown@6.0.0/dist/index.js",
    "@codemirror/lang-html": "https://cdn.jsdelivr.net/npm/@codemirror/lang-html@6.0.0/dist/index.js",
    "@codemirror/lang-css": "https://cdn.jsdelivr.net/npm/@codemirror/lang-css@6.0.0/dist/index.js",
    "@lezer/css": "https://cdn.jsdelivr.net/npm/@lezer/css@1.0.0/dist/index.es.js",
    "@codemirror/lang-javascript": "https://cdn.jsdelivr.net/npm/@codemirror/lang-javascript@6.0.0/dist/index.js",
    "@lezer/javascript": "https://cdn.jsdelivr.net/npm/@lezer/javascript@1.0.0/dist/index.es.js",
    "@lezer/html": "https://cdn.jsdelivr.net/npm/@lezer/html@1.0.0/dist/index.es.js",
    "@lezer/markdown": "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/index.js",
    "@codemirror/lang-json": "https://cdn.jsdelivr.net/npm/@codemirror/lang-json@6.0.0/dist/index.js",
    "@lezer/json": "https://cdn.jsdelivr.net/npm/@lezer/json@1.0.0/dist/index.es.js",
    "@codemirror/theme-one-dark": "https://cdn.jsdelivr.net/npm/@codemirror/theme-one-dark@6.0.0/dist/index.js",
    "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/tree": "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/tree.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/parse": "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/parse.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/mix": "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/mix.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/parse": "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/parse.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/token": "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/token.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/stack": "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/stack.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/markdown": "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/markdown.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/nest": "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/nest.d.ts",
    "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/extension": "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/extension.d.ts"
  }
}
```

##### `typed_imports.ts`

```ts
// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/view@6.0.1/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/view@6.0.1/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/state@6.0.1/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/state@6.0.1/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/style-mod@4.0.0/src/style-mod.d.ts"
import "https://cdn.jsdelivr.net/npm/style-mod@4.0.0/src/style-mod.js";

// @deno-types="https://cdn.jsdelivr.net/npm/w3c-keyname@2.2.4/index.d.ts"
import "https://cdn.jsdelivr.net/npm/w3c-keyname@2.2.4/index.es.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/language@6.1.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/language@6.1.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@lezer/common@1.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@lezer/highlight@1.0.0/dist/highlight.d.ts"
import "https://cdn.jsdelivr.net/npm/@lezer/highlight@1.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@lezer/lr@1.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/commands@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/commands@6.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/search@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/search@6.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/crelt@1.0.5/index.d.ts"
import "https://cdn.jsdelivr.net/npm/crelt@1.0.5/index.es.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/autocomplete@6.0.2/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/autocomplete@6.0.2/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/lint@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/lint@6.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/lang-markdown@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/lang-markdown@6.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/lang-html@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/lang-html@6.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/lang-css@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/lang-css@6.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@lezer/css@1.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@lezer/css@1.0.0/dist/index.es.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/lang-javascript@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/lang-javascript@6.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@lezer/javascript@1.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@lezer/javascript@1.0.0/dist/index.es.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@lezer/html@1.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@lezer/html@1.0.0/dist/index.es.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@lezer/markdown@1.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/lang-json@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/lang-json@6.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@lezer/json@1.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@lezer/json@1.0.0/dist/index.es.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/theme-one-dark@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/theme-one-dark@6.0.0/dist/index.js";
```

##### `deno.json`

```json
{
  "compilerOptions": {
    "lib": ["dom", "dom.iterable", "dom.asynciterable", "deno.ns"]
  }
}
```

##### `.gitignore`

```
bundle.js
```
