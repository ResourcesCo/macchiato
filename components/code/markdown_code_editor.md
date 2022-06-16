# Markdown Code Editor

These are the dependencies, built with `@codemirror/`:

##### `dependencies.json`

```json
{  
  "@codemirror/view": "*",
  "@codemirror/state": "*",
  "@codemirror/history": "*",
  "@codemirror/fold": "*",
  "@codemirror/language": "*",
  "@codemirror/commands": "*",
  "@codemirror/matchbrackets": "*",
  "@codemirror/closebrackets": "*",
  "@codemirror/search": "*",
  "@codemirror/autocomplete": "*",
  "@codemirror/comment": "*",
  "@codemirror/rectangular-selection": "*",
  "@codemirror/lint": "*",
  "@codemirror/lang-markdown": "*",
  "@codemirror/lang-javascript": "*",
  "@codemirror/lang-css": "*",
  "@codemirror/lang-json": "*",
  "@codemirror/lang-html": "*",
  "@codemirror/highlight": "*"
}
```

These are manual additions to the import map in order to get it to build:

##### `import_map_scopes.json`

```json
{
  "scopes": {
    "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/": {
      "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/tree": "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/tree.d.ts",
      "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/parse": "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/parse.d.ts",
      "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/mix": "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/mix.d.ts"
    },
    "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/": {
      "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/parse": "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/parse.d.ts",
      "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/token": "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/token.d.ts",
      "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/stack": "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/stack.d.ts"
    },
    "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/": {
      "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/markdown": "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/markdown.d.ts",
      "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/nest": "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/nest.d.ts",
      "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/extension": "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/extension.d.ts"
    }
  }
}
```

To generate the import map and typed imports, pipe this file into
`deliver_importmaps` and `md_unpack_simple`. This will write
`import_map.json` and `typed_imports.ts` to the current directory.
Then add the scopes from above. These have been included below.

```
cat markdown_code_editor.md | deliver_importmaps | md_unpack_simple
```

##### `import_map.json`

```json
{
  "imports": {
    "@codemirror/view": "https://cdn.jsdelivr.net/npm/@codemirror/view@6.0.0/dist/index.js",
    "@codemirror/state": "https://cdn.jsdelivr.net/npm/@codemirror/state@6.0.0/dist/index.js",
    "style-mod": "https://cdn.jsdelivr.net/npm/style-mod@4.0.0/src/style-mod.js",
    "w3c-keyname": "https://cdn.jsdelivr.net/npm/w3c-keyname@2.2.4/index.es.js",
    "@codemirror/history": "https://cdn.jsdelivr.net/npm/@codemirror/history@0.19.2/dist/index.js",
    "@codemirror/fold": "https://cdn.jsdelivr.net/npm/@codemirror/fold@0.19.4/dist/index.js",
    "@codemirror/gutter": "https://cdn.jsdelivr.net/npm/@codemirror/gutter@0.19.9/dist/index.js",
    "@codemirror/rangeset": "https://cdn.jsdelivr.net/npm/@codemirror/rangeset@0.19.9/dist/index.js",
    "@codemirror/language": "https://cdn.jsdelivr.net/npm/@codemirror/language@6.0.0/dist/index.js",
    "@codemirror/text": "https://cdn.jsdelivr.net/npm/@codemirror/text@0.19.6/dist/index.js",
    "@lezer/common": "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/index.js",
    "@lezer/lr": "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/index.js",
    "@lezer/highlight": "https://cdn.jsdelivr.net/npm/@lezer/highlight@1.0.0/dist/index.js",
    "@codemirror/commands": "https://cdn.jsdelivr.net/npm/@codemirror/commands@6.0.0/dist/index.js",
    "@codemirror/matchbrackets": "https://cdn.jsdelivr.net/npm/@codemirror/matchbrackets@0.19.4/dist/index.js",
    "@codemirror/closebrackets": "https://cdn.jsdelivr.net/npm/@codemirror/closebrackets@0.19.2/dist/index.js",
    "@codemirror/search": "https://cdn.jsdelivr.net/npm/@codemirror/search@6.0.0/dist/index.js",
    "crelt": "https://cdn.jsdelivr.net/npm/crelt@1.0.5/index.es.js",
    "@codemirror/autocomplete": "https://cdn.jsdelivr.net/npm/@codemirror/autocomplete@6.0.2/dist/index.js",
    "@codemirror/comment": "https://cdn.jsdelivr.net/npm/@codemirror/comment@0.19.1/dist/index.js",
    "@codemirror/rectangular-selection": "https://cdn.jsdelivr.net/npm/@codemirror/rectangular-selection@0.19.2/dist/index.js",
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
    "@codemirror/highlight": "https://cdn.jsdelivr.net/npm/@codemirror/highlight@0.19.8/dist/index.js"
  },
  "scopes": {
    "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/": {
      "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/tree": "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/tree.d.ts",
      "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/parse": "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/parse.d.ts",
      "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/mix": "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/mix.d.ts"
    },
    "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/": {
      "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/parse": "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/parse.d.ts",
      "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/token": "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/token.d.ts",
      "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/stack": "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/stack.d.ts"
    },
    "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/": {
      "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/markdown": "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/markdown.d.ts",
      "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/nest": "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/nest.d.ts",
      "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/extension": "https://cdn.jsdelivr.net/npm/@lezer/markdown@0.16.1/dist/extension.d.ts"
    }
  }
}
```

##### `typed_imports.ts`

```ts
// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/view@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/view@6.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/state@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/state@6.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/style-mod@4.0.0/src/style-mod.d.ts"
import "https://cdn.jsdelivr.net/npm/style-mod@4.0.0/src/style-mod.js";

// @deno-types="https://cdn.jsdelivr.net/npm/w3c-keyname@2.2.4/index.d.ts"
import "https://cdn.jsdelivr.net/npm/w3c-keyname@2.2.4/index.es.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/history@0.19.2/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/history@0.19.2/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/fold@0.19.4/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/fold@0.19.4/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/gutter@0.19.9/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/gutter@0.19.9/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/rangeset@0.19.9/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/rangeset@0.19.9/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/language@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/language@6.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/text@0.19.6/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/text@0.19.6/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@lezer/common@0.15.12/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@lezer/lr@0.15.8/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@lezer/highlight@1.0.0/dist/highlight.d.ts"
import "https://cdn.jsdelivr.net/npm/@lezer/highlight@1.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/commands@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/commands@6.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/matchbrackets@0.19.4/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/matchbrackets@0.19.4/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/closebrackets@0.19.2/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/closebrackets@0.19.2/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/search@6.0.0/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/search@6.0.0/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/crelt@1.0.5/index.d.ts"
import "https://cdn.jsdelivr.net/npm/crelt@1.0.5/index.es.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/autocomplete@6.0.2/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/autocomplete@6.0.2/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/comment@0.19.1/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/comment@0.19.1/dist/index.js";

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/rectangular-selection@0.19.2/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/rectangular-selection@0.19.2/dist/index.js";

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

// @deno-types="https://cdn.jsdelivr.net/npm/@codemirror/highlight@0.19.8/dist/index.d.ts"
import "https://cdn.jsdelivr.net/npm/@codemirror/highlight@0.19.8/dist/index.js";
```

##### `deno.json`

```json
{
  "compilerOptions": {
    "target": "esnext",
    "lib": ["dom", "dom.iterable", "dom.asynciterable", "deno.ns"]
  }
}
```

These are some light and dark styles, that can be dynamically swapped.

##### `styles.ts`

```ts
import "./typed_imports.ts";

// Based on https://github.com/codemirror/theme-one-dark
// Copyright (C) 2018-2021 by Marijn Haverbeke <marijnh@gmail.com> and others
// MIT License: https://github.com/codemirror/theme-one-dark/blob/main/LICENSE

import { HighlightStyle, tags as t } from "@codemirror/highlight";
import { EditorView } from "@codemirror/view";

// Based on https://github.com/codemirror/highlight
// Copyright (C) 2018-2021 by Marijn Haverbeke <marijnh@gmail.com> and others
// MIT License: https://github.com/codemirror/highlight/blob/main/LICENSE

export const lightHighlightStyle = HighlightStyle.define([
  { tag: t.link, textDecoration: "underline" },
  { tag: t.heading, textDecoration: "underline", fontWeight: "bold" },
  { tag: t.emphasis, fontStyle: "italic" },
  { tag: t.strong, fontWeight: "bold" },
  { tag: t.keyword, color: "#708" },
  {
    tag: [t.atom, t.bool, t.url, t.contentSeparator, t.labelName],
    color: "#219",
  },
  { tag: [t.literal, t.inserted], color: "#164" },
  { tag: [t.string, t.deleted], color: "#a11" },
  { tag: [t.regexp, t.escape, t.special(t.string)], color: "#e40" },
  { tag: t.definition(t.variableName), color: "#00f" },
  { tag: t.local(t.variableName), color: "#30a" },
  { tag: [t.typeName, t.namespace], color: "#085" },
  { tag: t.className, color: "#167" },
  {
    tag: [t.special(t.variableName), t.macroName, t.local(t.variableName)],
    color: "#256",
  },
  { tag: t.definition(t.propertyName), color: "#00c" },
  { tag: t.comment, color: "#940" },
  { tag: t.meta, color: "#7a757a" },
  { tag: t.invalid, color: "#f00" },
]);

const chalky = "#e5c07b",
  coral = "#e06c75",
  cyan = "#56b6c2",
  invalid = "#ffffff",
  ivory = "#abb2bf",
  stone = "#5c6370",
  malibu = "#61afef",
  sage = "#98c379",
  whiskey = "#d19a66",
  violet = "#c678dd";

/// The highlighting style for code in the One Dark theme.
export const darkHighlightStyle = HighlightStyle.define([
  { tag: t.keyword, color: violet },
  {
    tag: [t.name, t.deleted, t.character, t.propertyName, t.macroName],
    color: coral,
  },
  { tag: [t.processingInstruction, t.string, t.inserted], color: sage },
  { tag: [t.function(t.variableName), t.labelName], color: malibu },
  { tag: [t.color, t.constant(t.name), t.standard(t.name)], color: whiskey },
  { tag: [t.definition(t.name), t.separator], color: ivory },
  {
    tag: [
      t.typeName,
      t.className,
      t.number,
      t.changed,
      t.annotation,
      t.modifier,
      t.self,
      t.namespace,
    ],
    color: chalky,
  },
  {
    tag: [
      t.operator,
      t.operatorKeyword,
      t.url,
      t.escape,
      t.regexp,
      t.link,
      t.special(t.string),
    ],
    color: cyan,
  },
  { tag: [t.meta, t.comment], color: stone },
  { tag: t.strong, fontWeight: "bold" },
  { tag: t.emphasis, fontStyle: "italic" },
  { tag: t.link, color: stone, textDecoration: "underline" },
  { tag: t.heading, fontWeight: "bold", color: coral },
  { tag: [t.atom, t.bool, t.special(t.variableName)], color: whiskey },
  { tag: t.invalid, color: invalid },
]);

// Based on https://github.com/codemirror/theme-one-dark
// Copyright (C) 2018-2021 by Marijn Haverbeke <marijnh@gmail.com> and others
// MIT License: https://github.com/codemirror/theme-one-dark/blob/main/LICENSE

// Using https://github.com/one-dark/vscode-one-dark-theme/ as reference for the colors

const lightColors = {
  foreground: "rgb(55, 65, 81)",
  lightBackground: "#fff",
  highlightBackground: "#2c313a",
  background: "#fff",
  selection: "#339cec33",
  cursor: "#528bff",
};

/// The editor theme styles for One Dark, customized to light.
export const lightTheme = EditorView.theme(
  {
    "&": {
      color: lightColors.foreground,
      backgroundColor: lightColors.background,
      caretColor: lightColors.cursor,
    },

    "&.cm-editor": {
      "&.cm-focused": { outline: 'none' },
      height: '100%',
    },

    "&.cm-wrap": {
      outline: "none",
    },

    "&.cm-wrap .cm-scroller": {
      outline: "none",
    },

    "&.cm-wrap .cm-content": {
      outline: "none",
    },

    "&.cm-focused .cm-cursor": { borderLeftColor: lightColors.cursor },
    "&.cm-focused .cm-selectionBackground": {
      backgroundColor: lightColors.selection,
    },

    ".cm-panels": {
      backgroundColor: lightColors.lightBackground,
      color: lightColors.foreground,
    },
    ".cm-panels.cm-panels-top": { borderBottom: "2px solid #d9d9d9" },
    ".cm-panels.cm-panels-bottom": { borderTop: "2px solid #d9d9d9" },

    ".cm-searchMatch": {
      backgroundColor: "#339cec33",
      outline: "1px solid #d9d9d9",
    },
    ".cm-searchMatch.cm-searchMatch-selected": {
      backgroundColor: "#339cec33",
    },

    ".cm-activeLine": { backgroundColor: lightColors.background },
    ".cm-selectionMatch": { backgroundColor: "#aafe661a" },

    ".cm-matchingBracket, .cm-nonmatchingBracket": {
      backgroundColor: "#bad0f847",
      outline: "1px solid #515a6b",
    },

    ".cm-gutters": {
      backgroundColor: lightColors.background,
      color: "#545868",
      border: "none",
    },
    ".cm-lineNumbers .cm-gutterElement": { color: "inherit" },

    ".cm-foldPlaceholder": {
      backgroundColor: "transparent",
      border: "none",
      color: "#ddd",
    },

    ".cm-tooltip": {
      border: "1px solid #181a1f",
      backgroundColor: lightColors.lightBackground,
    },
    ".cm-tooltip-autocomplete": {
      "& > ul > li[aria-selected]": {
        backgroundColor: lightColors.highlightBackground,
        color: lightColors.foreground,
      },
    },
  },
  { dark: false }
);

// Based on https://github.com/codemirror/theme-one-dark
// Copyright (C) 2018-2021 by Marijn Haverbeke <marijnh@gmail.com> and others
// MIT License: https://github.com/codemirror/theme-one-dark/blob/main/LICENSE

// Using https://github.com/one-dark/vscode-one-dark-theme/ as reference for the colors

// duplicate colors from highlight theme omitted
const darkColors = {
  darkBackground: "#21252b",
  highlightBackground: "#2c313a",
  background: "#121212",
  selection: "#3E4451",
  cursor: "#528bff",
};

/// The editor theme styles for One Dark.
export const darkTheme = EditorView.theme(
  {
    "&": {
      color: "rgb(229, 231, 235)",
      backgroundColor: darkColors.background,
      "& ::selection": { backgroundColor: darkColors.selection },
      caretColor: darkColors.cursor,
    },

    "&.cm-editor": {
      height: '100%',
      "&.cm-focused": { outline: 'none' },
    },

    "&.cm-wrap": {
      outline: "none",
    },

    "&.cm-wrap .cm-scroller": {
      outline: "none",
    },

    "&.cm-wrap .cm-content": {
      outline: "none",
    },

    "&.cm-focused .cm-cursor": { borderLeftColor: darkColors.cursor },
    "&.cm-focused .cm-selectionBackground, .cm-selectionBackground": {
      backgroundColor: darkColors.selection,
    },

    ".cm-panels": { backgroundColor: darkColors.darkBackground, color: ivory },
    ".cm-panels.cm-panels-top": { borderBottom: "2px solid black" },
    ".cm-panels.cm-panels-bottom": { borderTop: "2px solid black" },

    ".cm-searchMatch": {
      backgroundColor: "#72a1ff59",
      outline: "1px solid #457dff",
    },
    ".cm-searchMatch.cm-searchMatch-selected": {
      backgroundColor: "#6199ff2f",
    },

    ".cm-activeLine": { backgroundColor: darkColors.background },
    ".cm-selectionMatch": { backgroundColor: "#aafe661a" },

    ".cm-matchingBracket, .cm-nonmatchingBracket": {
      backgroundColor: "#bad0f847",
      outline: "1px solid #515a6b",
    },

    ".cm-gutters": {
      backgroundColor: darkColors.background,
      color: stone,
      border: "none",
    },
    ".cm-lineNumbers .cm-gutterElement": { color: "inherit" },

    ".cm-foldPlaceholder": {
      backgroundColor: "transparent",
      border: "none",
      color: "#ddd",
    },

    ".cm-tooltip": {
      border: "1px solid #181a1f",
      backgroundColor: darkColors.darkBackground,
    },
    ".cm-tooltip-autocomplete": {
      "& > ul > li[aria-selected]": {
        backgroundColor: darkColors.highlightBackground,
        color: ivory,
      },
    },
  },
  { dark: true }
);
```

##### `language.ts`

```ts
import "./typed_imports.ts";
import {
  LanguageSupport,
  LanguageDescription,
} from "@codemirror/language";
import { markdown } from "@codemirror/lang-markdown";
import { jsxLanguage, tsxLanguage } from "@codemirror/lang-javascript";
import { cssLanguage } from "@codemirror/lang-css";
import { jsonLanguage } from "@codemirror/lang-json";
import { htmlLanguage } from "@codemirror/lang-html";

export function language() {
  const codeLanguages = [
    LanguageDescription.of({
      name: "javascript",
      alias: ["js", "jsx"],
      async load() {
        return new LanguageSupport(jsxLanguage);
      },
    }),
    LanguageDescription.of({
      name: "typescript",
      alias: ["ts", "tsx"],
      async load() {
        return new LanguageSupport(tsxLanguage);
      },
    }),
    LanguageDescription.of({
      name: "css",
      async load() {
        return new LanguageSupport(cssLanguage);
      },
    }),
    LanguageDescription.of({
      name: "json",
      async load() {
        return new LanguageSupport(jsonLanguage);
      },
    }),
    LanguageDescription.of({
      name: "html",
      alias: ["htm"],
      async load() {
        const javascript = new LanguageSupport(jsxLanguage);
        const css = new LanguageSupport(cssLanguage);
        return new LanguageSupport(htmlLanguage, [css, javascript]);
      },
    }),
  ];
  
  return markdown({
    codeLanguages: [
      ...codeLanguages,
      LanguageDescription.of({
        name: "markdown",
        alias: ["md", "mkd"],
        async load() {
          return markdown({
            codeLanguages: [
              ...codeLanguages,
              LanguageDescription.of({
                name: "markdown",
                alias: ["md", "mkd"],
                async load() {
                  return markdown({
                    codeLanguages: [
                      ...codeLanguages,
                      LanguageDescription.of({
                        name: "markdown",
                        alias: ["md", "mkd"],
                        async load() {
                          return markdown({
                            codeLanguages,
                          });
                        },
                      }),
                    ],
                  });
                },
              }),
            ],
          });
        },
      }),
    ],
  });
}
```

##### `mod.ts`

```ts
import "./typed_imports.ts";
import {
  EditorView,
  highlightSpecialChars,
  drawSelection,
  highlightActiveLine,
  keymap,
  placeholder,
} from "@codemirror/view";
import { EditorState, Compartment } from "@codemirror/state";
import { history, historyKeymap } from "@codemirror/history";
import { foldGutter, foldKeymap } from "@codemirror/fold";
import {
  indentOnInput,
  LanguageSupport,
  LanguageDescription,
} from "@codemirror/language";
import { defaultKeymap } from "@codemirror/commands";
import { bracketMatching } from "@codemirror/matchbrackets";
import { closeBrackets, closeBracketsKeymap } from "@codemirror/closebrackets";
import { searchKeymap, highlightSelectionMatches } from "@codemirror/search";
import { autocompletion, completionKeymap } from "@codemirror/autocomplete";
import { commentKeymap } from "@codemirror/comment";
import { rectangularSelection } from "@codemirror/rectangular-selection";
import { lintKeymap } from "@codemirror/lint";
import { language } from "./language.ts";
import {
  lightHighlightStyle,
  darkHighlightStyle,
  lightTheme,
  darkTheme,
} from "./styles.ts";

class CodeEditor extends HTMLElement {
  constructor() {
    super();
  }

  connectedCallback() {
    const styleCompartment = new Compartment();

    const darkStyleExtension = [darkHighlightStyle, darkTheme];
    const lightStyleExtension = [lightHighlightStyle, lightTheme];
    const getStyleExtension = () => {
      return isDark.value ? darkStyleExtension : lightStyleExtension;
    };
    let styleExtension = getStyleExtension();

    const extensions = [
      highlightSpecialChars(),
      history(),
      foldGutter(),
      drawSelection(),
      EditorState.allowMultipleSelections.of(true),
      indentOnInput(),
      bracketMatching(),
      closeBrackets(),
      autocompletion({activateOnTyping: false}),
      rectangularSelection(),
      highlightActiveLine(),
      highlightSelectionMatches(),
      keymap.of([
        ...closeBracketsKeymap,
        ...defaultKeymap,
        ...searchKeymap,
        ...historyKeymap,
        ...foldKeymap,
        ...commentKeymap,
        ...completionKeymap,
        ...lintKeymap,
      ]),
      markdownLanguage,
      styleCompartment.of(styleExtension),
      EditorView.lineWrapping,
      placeholder("# Enter some markdown here..."),
      EditorView.updateListener.of((v) => {
        if (v.docChanged) {
          ctx.emit("change", editor.state.doc);
        }
      }),
    ];

    this.editor = new EditorView({
      state: EditorState.create({
        doc: props.page.body,
        extensions,
      }),
      parent: root.value,
    });
  }
}
window.customElements.define('code-editor', CodeEditor);
```