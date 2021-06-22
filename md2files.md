# md2files

*Take a Markdown document and create files from it*

- Platform: Deno
- Permissions: `allow-write` in the output directory

## Introduction

This contains a Deno script that takes a Markdown document and creates files
from it.

A filename is in an inline code block that has an empty line before it and
an empty line after it, and can contain other code blocks after it. The
contents of a file are in a fenced code block that comes after a filename.
If the filename has no extension, contains a space, or starts or ends with
a double quote, it must be wrapped in a JSON string. This is to prevent the
rule from accidentally being triggered and allow a line starting with an
inline code block to be used for other purposes.

There can be small notes between the filename and the contents of the file.
There is a maximum of 20 lines and 1024 characters, not counting newlines,
between the line containing the filename and the opening code fence.

Data is streamed. If there is an EOF before the closing code block, the
file will still be created.

Here is an example:

`````
`hello.txt`

```
Hello, world.
```

`hello.js`

```js
const text = await Deno.readTextFile()
console.log(text)
```

`README.md`

This runs Deno.

````
To run:

```bash
deno --allow-read=hello.txt
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

## Implementation

`md2files.js`

```js
import { readLines } from "https://deno.land/std/io/bufio.ts"

const codeFence = /^(`{3,})[^`]*$/

export async function md2files(input) {
  let codeBlock = null;
  for await (const line of readLines(input)) {
    const match = codeFence.exec(line)
    if (match) {
      if (!codeBlock) {
        codeBlock = { fence: match[1] }
      } else if (codeBlock.fence === match[1]) {
        codeBlock = null
      } else {
        console.log(line)
      }
    } else if (codeBlock) {
      console.log(line)
    }
  }
}

if (import.meta.main) {
  md2files(Deno.stdin)
}
```

## Tests


