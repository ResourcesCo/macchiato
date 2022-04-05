# deliver_importmaps: jsDelivr Import Map Generator

This is a utility to build import maps of a list of packages on npm, for loading
them from [jsDelivr][jsd]. It can be used with `deno bundle` to create bundles. It was
built with CodeMirror 6 and ProseMirror in mind.

Once bundles are created, they can easily be added to a web app or distributed via
GitHub, [deno.land/x][dlx], jsDelivr through npm (perhaps with [dnt][dnt]), and others.

## Getting versions via the jsDelivr API

jsDelivr has an endpoint for getting a version for an npm package using a version
range like `*` or `^1.9.3`. This makes the API request with the URI encoded and
gets the version from the response.

##### `get_version.ts`

```ts
const baseUrl = 'https://data.jsdelivr.com/v1';
const headers = {
  Accept: 'application/json',
  'User-Agent': 'Macchiato https://github.com/macchiato-dev/deliver_importmaps',
};

function splitPackageName(name: string): string[] {
  const pos = name.indexOf('/');
  if (pos === -1) {
    return [name];
  } else {
    return [name.substring(0, pos), name.substring(pos + 1)];
  }
}

export async function getVersion(name: string, versionRange = '*'): Promise<string> {
  const urlName = splitPackageName(name).map(s => encodeURI(s)).join('/');
  const urlVersionRange = encodeURI(versionRange);
  const url = `${baseUrl}/package/resolve/npm/${urlName}@${urlVersionRange}`;
  const resp = await fetch(url);
  if (resp.ok) {
    const respBody = await resp.json() as { version: string };
    const version = respBody.version;
    return version;
  } else {
    throw new Error(`Fetch error: ${resp.statusText}`);
  }
}
```

Some quick tests:

##### `get_version_test.ts`

```ts
import { getVersion } from './get_version.ts';
import { assert } from "https://deno.land/std@0.133.0/testing/asserts.ts";

Deno.test('with organization', async () => {
  const version = await getVersion('style-mod');
  assert(/^\d+\./.test(version));
});

Deno.test('without organization', async () => {
  const version = await getVersion('@codemirror/view');
  assert(/^\d+\./.test(version));
});

Deno.test('with version', async () => {
  const version = await getVersion('@codemirror/view', '0.19.48');
  assert(version === '0.19.48');
});
```

Run:

```bash
deno test --allow-net=data.jsdelivr.com get_version_test.ts
```

Output:

```
Check file:///Users/bat/proyectos/notebook/macchiato/build/deno/deliver_importmaps/get_version_test.ts
running 3 tests from file:///Users/bat/proyectos/notebook/macchiato/build/deno/deliver_importmaps/get_version_test.ts
test with organization ... ok (685ms)
test without organization ... ok (285ms)
test with version ... ok (205ms)

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out (1s)
```

These versions can be used to get the `package.json` from the jsDelivr CDN.

## Getting package.json

TODO: get types as well and store them in a separate JSON file

## Documentation

##### `README.md`

```md
# deliver_importmaps

This is a tool for generating an import map for loading JavaScript modules straight
from the source inside of npm packages, delivered by the [jsDelivr CDN][jsd].

## Source

This is generated from a markdown document, [deliver_importmaps.md][src], using
[md_unpack_simple][md_unpack_simple].

## License

MIT

## TODO

- Get `package.json` from jsDelivr CDN for packages
- Recursively get dependencies, with `version`, `module` and `types` from `package.json`
- Generate imports for import maps
- Provide a script for running it from the command line
- Provide directions for using it programatically
- Provide a way to import the types

[jsd]: https://jsdelivr.net/
[src]: https://github.com/ResourcesCo/macchiato/blob/main/build/deno/deliver_importmaps.md
[md_unpack_simple]: https://deno.land/x/md_unpack_simple
```

##### `LICENSE`

```
MIT License

Copyright (c) 2022 Resources.co

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

[jsd]: https://jsdelivr.net/
[dlx]: https://deno.land/x
[dnt]: https://github.com/denoland/dnt/
