# unpack-simple

*Unpack data from a Markdown notebook*

- Platform: Deno
- Permissions: `allow-read` on the input file and `allow-write` in the output
  directory

## Introduction

This contains a Deno command-line utility and library that takes data from a
Markdown document and creates files from it.

This is a simple version, that only supports unpacking from fenced code
blocks. Another version will support unpacking from other sorts of Markdown
content, such as lists and tables.

It is recommended to use this version unless you need more features. It is
designed so you can inspect the Markdown and be confident that you know
exactly what code will be generated from it.

Files are created from a *file definition*, an optional fenced code block
containing some options, and a fenced code block containing the data.

A file definition is a level 5 heading block containing an inline code block
with the relative path, either as a plain string or a JSON string (enclosed
in double quotes):

```md
##### `mod.ts`
```

```md
##### `"filename-\nwith-newline.txt"`
```

After that, the next one or two fenced code blocks are used, for the options,
and for the content of the file. If the first fenced code block is a JSON
array with the first string value equal to `"$options"`, it will be used as
the options, and the content will be read from the following fenced code
block. Otherwise, the first fenced code block will be used as the content.

These elements can only be separated by up to 10 lines, with up to a total
of 1024 characters. Each non-empty line must not start with any whitespace.
This way, short notes can be added, but the path to the file and the file
contents will appear close together, to prevent obfuscation.

The file definition can be followed by a fenced code block containing a JSON
configuration object, like this:

```json
["$options", {"encoding": "base64"}]
```

The options available are:

- `encoding` - can be `"base64"`, `"hex"`, or `utf8` (the default).
- `newline` - `true` if there is to be a trailing newline after the file,
  otherwise false. Defaults to true.
- `eol` - the end of line to use. can be `"crlf"` or `"lf"`.

There can also be an empty options value. This way, it is also possible to
store the same data used for an options block as the contents of the file,
by providing two code blocks that contain a JSON array with with the
string `"$options"` as its first element.

A `--stream` option is available. If given, it will stream the output, and if
an error occurs during the middle of processing it, the file changes already
made will remain. If the option is not given, it won't start writing until it
finishes reading, and that way if an error occurs in reading, the directory
contents will be left unchanged.

Here is a simple example. It is given a filename, because it is used later
in the tests.

##### `test/deno-basic.md`

`````md
##### `hello.txt`

```
Hello, world.
```

##### `hello.ts`

```js
const text = await Deno.readTextFile()
console.log(text)
```

##### `README.md`

This runs Deno.

````
To run:

```bash
deno run --allow-read=hello.txt hello.ts
```
````
`````

This will create three files, `hello.txt`, `hello.ts`, and `README.md`.

Note that more than three backquotes can be used to start a fenced code
block as long as the same number is used to end it. The above example uses
this technique.

Inline code blocks may also use a variable number of backquotes, so the
string may contain a backquote. This is supported by
`macchiato-unpack-simple`, even though backquotes in filenames are rare.

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

##### `mod.ts`

```js
import {
  readLines,
  BufWriter,
  Buffer
} from "https://deno.land/std@0.100.0/io/mod.ts";
import { ensureDir } from "https://deno.land/std@0.100.0/fs/mod.ts";
import { dirname } from "https://deno.land/std@0.100.0/path/mod.ts";
import {
  decode as base64Decode
} from "https://deno.land/std@0.100.0/encoding/base64.ts";
import {
  decode as base64urlDecode
} from "https://deno.land/std@0.100.0/encoding/base64url.ts";
import {
  decodeString as hexDecode
} from "https://deno.land/std@0.100.0/encoding/hex.ts";
const fileDataRegexp = /^#####\s+\`/;
const listDataRegexp = /^-\s*`/;
const codeFenceRegexp = /^(`{3,})[^`]*$/;
const backquotesRegexp = /(?<!\\)`+/;

export function nextInlineString(s: string): [string | null, string] {
  const match = backquotesRegexp.exec(s);
  if (match) {
    const startIndex = match.index + match[0].length;
    const length = s.substr(startIndex).indexOf(match[0]);
    if (length !== -1) {
      const data = s.substr(startIndex, length);
      const rest = s.substr(startIndex + length + match[0].length);
      return [data, rest];
    } else {
      return [null, ''];
    }
  } else {
    return [null, ''];
  }
}

export function readInlineStrings(line: string): string[] {
  const result = [];
  let rest = line;
  while (rest.length > 0) {
    const [data, newRest] = nextInlineString(rest);
    rest = newRest;
    if (data !== null) {
      result.push(data);
    } else {
      return result;
    }
  }
  return result;
}

type ToBinaryFunction = (text: string) => Uint8Array

type ToBinaryMap = {
  [key: string]: ToBinaryFunction
}

export async function unpack(stream: boolean = false): Promise<{ [path: string]: Uint8Array } | undefined> {
  const input = Deno.stdin;
  const open = Deno.open;
  const result: { [path: string]: Uint8Array } = {};
  let file = undefined;
  let fence = undefined;
  let gap = 0;
  const encoder = new TextEncoder();
  const toBinary: ToBinaryMap = {
    utf8: line => encoder.encode(line),
    base64: line => base64Decode(line.trim()),
    base64url: line => base64urlDecode(line.trim()),
    hex: line => hexDecode(line.trim()),
  }
  for await (const line of readLines(input)) {
    if (fence) {
      const match = codeFenceRegexp.exec(line);
      if (match && match[1] === fence) {
        if (file) {
          let options;
          if (!file.writing && !file.readOptions) {
            let parsedOptions;
            try {
              parsedOptions = JSON.parse(file.data);
            } catch (err) {
              // do nothing
            }
            if (Array.isArray(parsedOptions && parsedOptions[0] === '$options')) {
              options = parsedOptions[1];
              if (parsedOptions.length !== 2 || options === null || typeof options !== 'object') {
                throw new Error(`Invalid options for ${JSON.stringify(file.path)}`);
              }
            }
          }
          if (options) {
            file.readOptions = true;
            if (typeof options.encoding === 'string' && options.encoding in toBinary) {
              file.options.encoding = options.encoding;
              file.toBinary = toBinary[file.options.encoding];
            }
            if (typeof options.newline === 'boolean') {
              file.options.newline = options.newline;
            }
            if (options.eol === 'crlf') {
              file.options.eol = '\r\n';
            } else if (options.eol === 'lf') {
              file.options.eol = '\n';
            }
            file.preLine = '';
          } else {
            if (!file.writing) {
              await file.writer.write(file.toBinary(file.data));
              file.data = '';
              file.writing = true;
            }
            if (file.options.newline) {
              file.writer.write(file.toBinary(file.options.eol));
            }
            if (file.file && file.bufWriter) {
              await file.bufWriter.flush();
              await file.file.close();
            } else if (file.buffer) {
              const bufLength = file.buffer.length;
              const arr = new Uint8Array(file.buffer.length);
              const bytes = await file.buffer.read(arr);
              if (bytes !== bufLength) {
                throw new Error(`Error getting file contents from buffer: got ${bytes}, expected ${bufLength}`);
              }
              result[file.path] = arr;
            } else {
              throw new Error('No buffer and no file');
            }
            file = undefined;
          }
        }
        fence = undefined;
      } else if (file?.writing) {
        await file.writer.write(file.toBinary(file.preLine + line));
      } else if (file) {
        file.data += file.preLine + line;
        if (file.readOptions || file.data.length >= 1024) {
          await file.writer.write(file.toBinary(file.data));
          file.writing = true;
          file.data = '';
        }
        file.preLine = file.options.eol;
      }
    } else {
      if (fileDataRegexp.test(line)) {
        const inlineStrings = readInlineStrings(line);
        if (inlineStrings.length >= 1) {
          if (file && gap >= 1024) {
            throw new Error(`No file content found for ${JSON.stringify(file.path)}`);
          }
          gap = 0;
          const path = inlineStrings[0];
          await ensureDir(dirname(path));
          let openFile
          let writer
          let bufWriter
          let buffer
          if (stream) {
            openFile = await open(path, {write: true, create: true});
            bufWriter = BufWriter.create(openFile);
            writer = bufWriter;
          } else {
            buffer = new Buffer();
            writer = buffer;
          }
          file = {
            preLine: '',
            data: '',
            path,
            options: { eol: '\n', encoding: 'utf8', newline: false },
            toBinary: toBinary.utf8,
            file: openFile,
            buffer,
            bufWriter,
            writer,
            readOptions: false,
            writing: false,
          };
        }
      }
      const match = codeFenceRegexp.exec(line);
      if (match) {
        fence = match[1];
        if (file && gap >= 1024) {
          throw new Error(`No file content found for ${JSON.stringify(file.path)}`);
        }
      } else if (file) {
        gap += line.length + 1;
      }
    }
  }
  if (file && gap >= 1024) {
    throw new Error(`No file content found for ${JSON.stringify(file.path)}`);
  }
  if (!stream) {
    return result;
  }
}

export async function write(files: { [path: string]: Uint8Array }) {
  for (const key of Object.keys(files)) {
    await Deno.writeFile(key, files[key]);
  }
}

export async function run() {
  const files = await unpack();
  if (files) {
    await write(files);
  } else {
    throw new Error('No files returned');
  }
}

if (import.meta.main) {
  run();
}
```

## Tests

##### `test/read_inline_strings_test.ts`

```js
import { assertEquals } from "https://deno.land/std@0.99.0/testing/asserts.ts";
import { readInlineStrings } from "../mod.ts";

Deno.test("read inline string", () => {
  assertEquals(readInlineStrings("`wow`"), ["wow"]);
  assertEquals(
    readInlineStrings("``w`o`w`` `doge: cool`"),
    ["w`o`w", "doge: cool"]
  );
});
```
