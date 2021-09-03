# md_unpack_simple

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

- `encoding` - can be `"base64"`, `"base64url"`, `"hex"`, or `utf8`
  (the default).
- `newline` - `true` if there is to be a trailing newline after the file,
  otherwise false. Defaults to true.
- `eol` - the end of line to use. can be `"lf"` or `"crlf"`.

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

### Reading line by line and storing the state

This reads line by line and stores the state so it doesn't need to go back
to a previous line.

There is a `fence` object that keeps track of if it is in a code fence and
what will end the code fence (the number of backquotes).

The current file state is kept in a `file` variable that contains an object
with the following fields or `undefined` if it is not reading a file:

- `preLine` - holds what goes before the next line. This way it can end
  a file without a newline.
- `data` - holds data before writing to the `writer`, in case the data
  contains options. Once it determines that it is file data, it writes this
  data to the `writer`.
- `path` - the path of the file.
- `options` - The options, which can be overriden with a JSON object
  - `eol` - The end of the line. Default comes from the global `eol`
    option. Can be changed to `\r\n`, the Windows line ending. These are
    given as `lf` and `crlf`.
  - `encoding` - The text or binary encoding. Defaults to `utf8`. Can be
    `utf8`, `base64`, `base64url`, or `hex`.
  - `newline` - Whether to add a newline to the end of a text file.
    The default comes from the `newline` option for the whole file.
- `toBinary` - The method to convert to binary
- `file` - If streaming is on, a file, otherwise undefined
- `buffer` - If streaming is off, a buffer, otherwise undefined
- `bufWriter` - If streaming is on, a buffered writer for the file
- `writer` - The `Writer` to the file or buffer
- `readOptions` - Set to `true` when it has read the options
- `writing` - Set to `true` when it has started writing to the file's
  `writer`.

What the program is doing can be determined by the state of `fence`,
whether `file` is defined, `file.readOptions`, and `file.writing`:

- `fence` is undefined and `file` is undefined - It is looking for a
  path or an opening code fence
- `fence` is a string and `file` is undefined - It is just skipping over
  the fenced code, since there is no file. If it finds a closing code
  fence, it sets fence to undefined.
- `fence` is undefined and `file` has an object with `readOptions` set to
  false and `writing` set to false - It is looking for a fence which will
  either begin a code block or an options block
- `fence` is a string and `file` has an object with `readOptions` set to
  false and `writing` set to false - Reads the line into the `data`
  property of `file` and determines if it is not an options block. If it
  finds based on the data that it is not one, it writes `data` to the
  stream and sets `writing` to true. If it finds a closing code fence,
  it tries loading the option and if it does, it sets the options and
  sets `readOptions` to true and `writing` to true, and empties the
  `data`.
- `fence` is undefined and `file` has an object with `readOptions` set to
  `true` - Looks for a code fence that will start the content block.
- `fence` is defined and `writing` is set to true - If it finds a matching
  closing code fence, it saves the content of the file. If streaming is
  on, it flushes the buffered writer and closes the file. If streaming is
  off, it adds the content to a result object, as a string if text or as
  a Uint8Array if binary. If it hasn't found a matching code block, it
  writes the line to the buffer.

### Getting the path

To get a path of a file, a check for an inline code block at the start of
a line is done. If that matches, it reads the inline code blocks.

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

### Getting the options

If it hasn't read any options, it reads the first lines of the file and
checks if it contains options. If it does, it uses them to set the current
file's options, and sets `readOptions` to indicate that it has read the
options so it knows that the following fenced code block has the content.

### Writing the file

The file is written to its buffer. When it is done, if streaming is on,
it flushes to the file and closes. If streaming is off, it adds a string
or a binary array to the result to be returned.

When streaming is off, a separate `write` function is used to write all
the files. This uses Deno's `writeFile` and `writeTextFile` functions.

### Code

The code is in a single TypeScript module, so it can quickly be created
manually. The tests are spread across multiple files.

##### `mod.ts`

```ts
import {
  readLines,
  BufWriter,
  Buffer,
  StringReader
} from "https://deno.land/std@0.102.0/io/mod.ts";
import { ensureDir } from "https://deno.land/std@0.102.0/fs/mod.ts";
import { normalize, dirname } from "https://deno.land/std@0.102.0/path/mod.ts";
import {
  decode as base64Decode
} from "https://deno.land/std@0.102.0/encoding/base64.ts";
import {
  decode as base64urlDecode
} from "https://deno.land/std@0.102.0/encoding/base64url.ts";
import {
  decode as hexDecodeBytes
} from "https://deno.land/std@0.102.0/encoding/hex.ts";
const fileDataRegexp = /^#####\s+\`/;
const codeFenceRegexp = /^(`{3,})[^`]*$/;
const backquotesRegexp = /(?<!\\)`+/;

function hexDecode(s: string): Uint8Array {
  return hexDecodeBytes(new TextEncoder().encode(s.trim()));
}

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

type FilePack = { [path: string]: string | Uint8Array }

interface UnpackOptions {
  stream: boolean
  eol: 'lf' | 'crlf'
  newline: boolean
}

export async function unpack(text: string | undefined = undefined, {
  stream = false,
  eol = 'lf',
  newline = true,
}: Partial<UnpackOptions> = {}): Promise<FilePack | undefined> {
  const input = text === undefined ? Deno.stdin : new StringReader(text);
  const open = Deno.open;
  const result: FilePack = {};
  let file = undefined;
  let fence = undefined;
  let gap = 0;
  const encoder = new TextEncoder();
  const decoder = new TextDecoder();
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
            } catch (_err) {
              // do nothing
            }
            if (Array.isArray(parsedOptions) && parsedOptions[0] === '$options') {
              options = parsedOptions[1];
              if (parsedOptions.length !== 2 || options === null || typeof options !== 'object') {
                throw new Error(`Invalid options for ${JSON.stringify(file.path)}`);
              }
            }
          }
          if (options) {
            if (typeof options.encoding === 'string' && options.encoding in toBinary) {
              file.options.encoding = options.encoding;
              file.toBinary = toBinary[file.options.encoding];
            }
            if (typeof options.newline === 'boolean') {
              file.options.newline = options.newline;
            }
            if (options.eol === 'crlf') {
              file.options.eol = '\r\n';
            } else {
              file.options.eol = '\n';
            }
            file.data = '';
            file.preLine = '';
            file.readOptions = true;
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
              if (file.options.encoding === 'utf8') {
                result[file.path] = decoder.decode(arr);
              } else {
                result[file.path] = arr;
              }
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
          const path = normalize(inlineStrings[0]);
          if (path.startsWith('..') || path.startsWith('/')) {
            throw new Error(`Path ${JSON.stringify(path)} is not inside directory`);
          }
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
            options: {
              eol: eol === 'crlf' ? '\r\n' : '\n',
              encoding: 'utf8',
              newline,
            },
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

export async function write(files: FilePack) {
  for (const path of Object.keys(files)) {
    const data = files[path];
    if (typeof data === 'string') {
      await Deno.writeTextFile(path, data);
    } else {
      await Deno.writeFile(path, data);
    }
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

## Testing

To test, run:

```bash
deno test --allow-read=. --unstable
```

### Inline Strings

##### `test/read_inline_strings_test.ts`

```js
import { assertEquals } from "https://deno.land/std@0.102.0/testing/asserts.ts";
import { readInlineStrings } from "../mod.ts";

Deno.test("read inline string", () => {
  assertEquals(readInlineStrings("`wow`"), ["wow"]);
  assertEquals(
    readInlineStrings("``w`o`w`` `doge: cool`"),
    ["w`o`w", "doge: cool"]
  );
});
```

### Basic Deno

##### `test/deno_basic_test.ts`

```js
import { assertEquals } from "https://deno.land/std@0.102.0/testing/asserts.ts";
import { unpack } from "../mod.ts";

Deno.test("read example", async () => {
  const text = await Deno.readTextFile('./test/deno-basic.md');
  const files = await unpack(text);
  assertEquals(Object.keys(files || {}).length, 3);
  assertEquals((files || {})['hello.txt'], `Hello, world.\n`);
});
```

### Prevent Directory Escape

##### `test/directory-escape.md`

````md
##### `../hello.md`

```js
console.log("test");
```
````

##### `test/prevent_directory_escape_test.ts`

```js
import { assertThrowsAsync } from "https://deno.land/std@0.102.0/testing/asserts.ts";
import { unpack } from "../mod.ts";

Deno.test("prevent directory escape", async () => {
  const text = await Deno.readTextFile('./test/directory-escape.md');
  await assertThrowsAsync(async () => {
    await unpack(text);
  });
});
```

### Binary

##### `test/binary.md`

````md
##### `test1`

```json
["$options", {"encoding": "hex"}]
```

```
00010203FF
```

##### `test2`

```json
["$options", {"encoding": "base64"}]
```

```
AAECA/8=
```

##### `test3`

```json
["$options", {"encoding": "base64url"}]
```

```
AAECA_8=
```
````

##### `test/binary_test.ts`

```js
import { equals } from "https://deno.land/std@0.102.0/bytes/mod.ts";
import { assert } from "https://deno.land/std@0.102.0/testing/asserts.ts";
import { unpack } from "../mod.ts";

Deno.test("binary", async () => {
  const text = await Deno.readTextFile('./test/binary.md');
  const files = await unpack(text);
  for (const path of ['test1', 'test2', 'test3']) {
    const value = (files || {})[path];
    if (value instanceof Uint8Array) {
      assert(equals(value, new Uint8Array([0, 1, 2, 3, 255])));
    } else {
      throw new Error('value is not a Uint8Array');
    }
  }
});
```

## Continuous Integration

TODO: automate running md_unpack_simple when source file is changed

This GitLab CI pipeline uses a Deno Docker image to test it.

##### `.gitlab-ci.yml`

```yaml
stages:
  - test

image: denoland/deno:ubuntu-1.13.2

deno-lint:
  stage: test
  script:
    - deno lint --unstable

deno-test:
  stage: test
  script:
    - deno test --allow-read=. --unstable
```

## Meta

##### `README.md`

`````md
# md_unpack_simple

[![pipeline status](https://gitlab.com/ResourcesCo/macchiato/md_unpack_simple/badges/main/pipeline.svg)](https://gitlab.com/ResourcesCo/macchiato/md_unpack_simple/-/commits/main)

Unpack a Markdown document into multiple files

To run, pass a Markdown file to standard input, and give permission to write
to the current directory:

```bash
cat source.md | deno run --allow-write=. --unstable https://deno.land/x/md_unpack_simple/mod.ts
```

This will take embedded files in the source Markdown document and write them
to their path within the current directory. If directories are missing, they
will be created.

It only requires permissions to read and write to the current directory, as
well as the `--unstable` flag. The read permission is required for checking
if the directory already exists before creating it.

The embedded files can be defined with a level 5 header with an inline code
block, like this:

````md
##### `js/file1.js`

```js
console.log('Hello, world.')
```

##### `css/file2.css`

```css
body { background-color: yellow; }
```
````
`````

##### `LICENSE`

```
MIT License

Copyright (c) 2021 Resources.co

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
