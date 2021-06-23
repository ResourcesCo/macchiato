# md2files

*Take a Markdown document and create files from it*

- Platform: Deno
- Permissions: `allow-write` in the output directory

## Introduction

This contains a Deno script that takes a Markdown document and creates files
from it.

A filename is in a line starting with an inline code block, that has at least
one empty line before it (whitespace containing two or more newlines), and
at least one empty line after it.

The contents of a file are in a fenced code block that comes after the block
containing the filename, or in an unordered list containing key-value pairs.
Binary files can be encoded as base64 or hex.

The filename block can also contain keyword arguments, using a colon to
separate the keyword and the value. The `key: value` can be in a single code
block or split between two, with the colon in the first code block. The key
or value may be wrapped in double quotes and contain a JSON string. Newlines
in inline code blocks are allowed by CommonMark, but not supported by
md2files. The newlines in inline code blocks in CommonMark are interpreted
as spaces and are used to support starting or ending an inline code block
with spaces (otherwise spaces are eliminated). To include spaces at the start
or end of arguments in md2files, wrap it in double quotes and provide a JSON
string.

```
`encoding: base64`
```

If the filename meets any of these conditions:

- has no extension and does not end with `file`
- contains a space
- starts or ends with a double quote

...it must be wrapped in a JSON string. This is to prevent the rule from
accidentally being triggered and allow a line starting with an inline code
block to be used for other purposes.

There can be small notes between the filename and the contents of the file.
There is a maximum of 20 lines and 1024 characters, not counting newlines,
between the line containing the filename and the opening code fence.

Data is streamed by line. If there is an EOF before the closing code block,
the file will still be created, however, a warning will be shown. It can
handle long files, but very long lines should be avoided (add newlines to
base64 and hex data).

Here is an example:

`````
`hello.txt`

```
Hello, world.
```

`hello.ts`

```js
const text = await Deno.readTextFile()
console.log(text)
```

`README.md`

This runs Deno.

````
To run:

```bash
deno run --allow-read=hello.txt hello.ts
```
````
`````

Note that more than three backquotes can be used to start a fenced code
block as long as the same number is used to end it. The above example uses
this technique.

If after the filename there is another inline code block containing
`encoding: hex` or `encoding: base64`, it will create a binary file.
Otherwise it will create a text file. For the binary data, `txt` is
suggested as the file type so it doesn't get syntax highlighted,
regardless of the output file type. Here's an example
([original image](https://en.wikipedia.org/wiki/File:Ruby-Lossless-Tiny.png):

````
`"example text"` `encoding: base64`

```txt
aGVsbG8=
```
````

This will place "hello" - the decoded base64 above - into the file named 
`example text` - that is, with a space and no extension.

The contents can also be an unordered list for JSON. If the list starts with
an inline code block, it will be treated as a content block for the preceding
file definition.

The key and value are inline code blocks, either separate or combined. If
combined, the key will be what comes before the first colon, and the value
will be what comes after. If separate, the colon goes at the end of the first
inline code block. If the value is an array or object, it must be represented
as `{}` for an object or `[]` for an array, with the contents in a nested
list, or as the inline JSON value. For example, these three are equivalent:

```md
`package.json`

- `name:` `hello`
- `keywords`: `[]`
  - `utility`
  - `cli`
- `dependencies:` `{}`
  - `chalk`: `*`
  - `yargs`: `*`

`package-combined.json`

- `name: hello`
- `keywords: []`
  - `utility`
  - `cli`
- `dependencies: {}`
  - `chalk: *`
  - `yargs: *`

`package-inline.json`

- `name:` `hello`
- `keywords:` `["utility", "cli"]`
- `dependencies:` `{"chalk": "*", "yargs": "*"}`
```

If the first line doesn't contain a colon, or it is inside of a JSON string,
the root will be an array. For example:

Another supported format is *lines*. These are like a JSON array, but with
the option `format: text` in the file definition line (the filename line).
For an empty line, use an inline code block containing whitespace (such as
a space). Non-JSON-quoted leading and trailing whitespace will be stripped.

For example these files will be the same:

````md
`Dockerfile`

- `FROM node`
- ` `
- `COPY hello.js`
- `RUN node hello.js`

`"Dockerfile-fenced"`

Note that the filename is wrapped in double quotes because it doesn't end
in `file`. Note also that since this note is less than 1024 characters,
the following content is still linked with the filename above.

```
FROM node

COPY hello.js
RUN node hello.js
```
````

For JSON data, anything parsable as JSON will have its JSON value. It must
be quoted if it is needed as a string. This does not apply to a list for
file content that has `format: text`. So if you want the boolean value
`true`, include `true`, but if you want the string `"true"`, specify
`"true"`. Whitespace will be stripped from the beginning and end of
unquoted values inside inline code blocks, so to include the leading or
trailing whitespace in the output file, use a quoted JSON string.

This doesn't include CSV support at present but a JSON to CSV tool could
be used to create a CSV file. CSV output for lists as well as Markdown
table support may be added in the future.

Anything outside of the code blocks on a filename line or a content list
line is ignored, except that one inline code block must be at the start
of the line for it to match. The rest can contain comments.

Here's an example of some comments:

```md
`Dockerfile` *This is the Dockerfile*

- Hello
- Test
```

## Implementation

To get a filename, a check for an inline code block at the start of a
line is done. If that matches, it reads the inline code blocks.

The inline code blocks are surrounded by a matching number of quotes.
If one quote is found at the start, one quote must be found at the end,
and for two quotes at the start, two quotes at the end, and so on. To
read this, a regex is used to find the starting quotes, and indexOf is
used to find an exact match for the end. If no match is found, it is
not added to the list of strings.

The whitespace is then stripped from the beginning and end of the
inside of each of the strings, and if it can be parsed as JSON, the
parsed value is used, otherwise the string is used.

The current file is then set and the file is opened for writing.

`md2files.js`

```js
import { readLines, BufWriter } from "https://deno.land/std/io/mod.ts";

const fileDataRegexp = /^[ ]{0,3}`/;
const listDataRegexp = /^-\s*`/;
const codeFenceRegexp = /^(`{3,})[^`]*$/;
const backquotesRegexp = /(?<!\\)`+/;

export function nextInlineString(s) {
  const match = backquotesRegexp.exec(s);
  if (match) {
    const startIndex = match.index + match[0].length;
    const length = s.substr(startIndex).indexOf(match[0]);
    if (length !== -1) {
      const data = s.substr(startIndex, length);
      const rest = s.substr(startIndex + length + match[0].length);
      return [data, rest];
    } else {
      return [undefined, ''];
    }
  } else {
    return [undefined, ''];
  }
}

export function readInlineStrings(line) {
  const result = [];
  let rest = line;
  while (rest.length > 0) {
    const [data, newRest] = nextInlineString(rest);
    rest = newRest;
    if (data !== undefined) {
      result.push(data);
    } else {
      return result;
    }
  }
  return result;
}

export async function md2files() {
  let file = undefined;
  let nextFile = undefined;
  let fence = undefined;
  let lastLineEmpty = false;
  const encoder = new TextEncoder('utf8');
  for await (const line of readLines(Deno.stdin)) {
    if (nextFile) {
      if (line.trim() === '') {
        file = nextFile;
      }
      nextFile = undefined;
    }
    if (fence) {
      const match = codeFenceRegexp.exec(line);
      if (match && match[1] === fence) {
        if (file?.file) {
          await file.buffer.flush();
          await file.file.close();
        }
        file = undefined;
        fence = undefined;
      } else if (file?.file) {
        await file.buffer.write(encoder.encode(line + "\n"));
      }
    } else {
      if (lastLineEmpty && fileDataRegexp.test(line)) {
        const inlineStrings = readInlineStrings(line);
        if (inlineStrings.length >= 1) {
          const filename = inlineStrings[0]
          nextFile = {
            filename,
          };
        }
      }
      if (codeFenceRegexp.test(line)) {
        const match = codeFenceRegexp.exec(line);
        fence = match[1];
        if (file) {
          file.file = await Deno.open(file.filename, {write: true, create: true});
          file.buffer = BufWriter.create(file.file);
        }
      }
      lastLineEmpty = line.trim() === '';
    }
  }
}

if (import.meta.main) {
  md2files();
}
```

## Tests

`md2files_readInlineStrings_test.js`

```js
import { assertEquals } from "https://deno.land/std@0.99.0/testing/asserts.ts";
import { readInlineStrings } from "./md2files.js";

Deno.test("read inline string", () => {
  assertEquals(readInlineStrings("`wow`"), ["wow"]);
  assertEquals(
    readInlineStrings("``w`o`w`` `doge: cool`"),
    ["w`o`w", "doge: cool"]
  );
});
```
