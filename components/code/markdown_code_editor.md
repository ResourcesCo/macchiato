# Markdown Code Editor

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
  "@codemirror/lang-html": "*"
}
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

class CodeEditor extends HTMLElement {
  constructor() {
    super();
  }

  
}
window.customElements.define('code-editor', CodeEditor);
```