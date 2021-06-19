# md2files

*Take a Markdown document and create files from it*

- Platform: Deno
- Permissions: `allow-write` in the output directory

## Introduction

This contains a Deno script that takes a Markdown document and creates files
from it.

A filename is in an inline code block that has an empty line before it and
an empty line after it. The contents of a file are in a fenced code block
that comes after a filename. If the filename has no extension, contains a
space, or starts or ends with a double quote, it must be wrapped in a JSON
string. This is to prevent the rule from accidentally being triggered and
allow a line starting with an inline code block to be used for other
purposes.

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

If the opening code block contains `hex` or `base64` after the file type,
matching the regex `\b(hex|base64)\b`, it will create a binary file.
Otherwise it will create a text file. For the binary data, `txt` is
suggested as the file type so it doesn't get syntax highlighted,
regardless of the output file type. Here's an example
([original image](https://en.wikipedia.org/wiki/File:Ruby-Lossless-Tiny.png):

````
`"example text"`

```txt base64
aGVsbG8=
```
````

This will place "hello" - the decoded base64 above - into the file named 
`example text` - that is, with a space and no extension.

## Implementation

`mod.ts`

```ts
import { readLines } from "https://deno.land/std/io/bufio.ts"

export async function md2files(inputReader, outputReader) {
  for await (const line of readLines(inputReader)) {
    
  }
}

if (import.meta.main) {
  
}
```

## Tests


