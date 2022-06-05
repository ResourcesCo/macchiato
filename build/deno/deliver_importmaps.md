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
const baseUrl = "https://data.jsdelivr.com/v1";
const headers = {
  Accept: "application/json",
  "User-Agent": "Macchiato https://github.com/macchiato-dev/deliver_importmaps",
};

function splitPackageName(name: string): string[] {
  const pos = name.indexOf("/");
  if (pos === -1) {
    return [name];
  } else {
    return [name.substring(0, pos), name.substring(pos + 1)];
  }
}

export async function getVersion(
  name: string,
  versionRange = "*",
): Promise<string> {
  const urlName = splitPackageName(name).map((s) => encodeURI(s)).join("/");
  const urlVersionRange = encodeURI(versionRange);
  const url = `${baseUrl}/package/resolve/npm/${urlName}@${urlVersionRange}`;
  const resp = await fetch(url, {headers});
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
import { getVersion } from "./get_version.ts";
import { assert } from "https://deno.land/std@0.133.0/testing/asserts.ts";

Deno.test("with organization", async () => {
  const version = await getVersion("style-mod");
  assert(/^\d+\./.test(version));
});

Deno.test("without organization", async () => {
  const version = await getVersion("@codemirror/view");
  assert(/^\d+\./.test(version));
});

Deno.test("with version", async () => {
  const version = await getVersion("@codemirror/view", "0.19.48");
  assert(version === "0.19.48");
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

## Getting info from package.json

This gets the information from `package.json` and checks that the needed information
is there.

##### `get_package_info.ts`

```ts
export interface PackageInfo {
  name: string;
  version: string;
  module: string;
  types?: string;
  dependencies: { [key: string]: string };
}

export async function getPackageInfo(
  name: string,
  version: string,
): Promise<PackageInfo> {
  const url = `https://cdn.jsdelivr.net/npm/${name}@${version}/package.json`;
  const resp = await fetch(url);
  if (!resp.ok) {
    throw new Error(`Error getting package.json for ${name}@${version}`);
  }
  const pkg = await resp.json();
  if (typeof pkg.name !== "string" || typeof pkg.version !== "string") {
    throw new Error(
      `Missing name or version in package.json for ${name}@${version}`,
    );
  }
  const module = (
    typeof pkg.module === "string" ? pkg.module : (
      pkg.type === "module"
        ? (typeof pkg.main === "string" ? pkg.main : "index.js")
        : undefined
    )
  );
  if (typeof module !== "string") {
    throw new Error(`Cannot resolve module for ${name}@${version}`);
  }
  if (typeof pkg.types !== "string" && pkg.types !== undefined) {
    throw new Error(`Invalid types in package.json for ${name}@${version}`);
  }
  const dependencies = (pkg.dependencies || {}) as { [key: string]: string };
  return {
    name: pkg.name,
    version: pkg.version,
    module: module,
    ...(pkg.types ? { types: pkg.types } : undefined),
    dependencies,
  };
}
```

Here are some basic tests.

##### `get_package_info_test.ts`

```ts
import { getPackageInfo } from "./get_package_info.ts";
import { assertEquals } from "https://deno.land/std@0.133.0/testing/asserts.ts";

Deno.test("with dependencies", async () => {
  const info = await getPackageInfo("@codemirror/state", "0.19.9");
  assertEquals(Object.keys(info.dependencies).length, 1);
});

Deno.test("without dependencies", async () => {
  const info = await getPackageInfo("style-mod", "4.0.0");
  assertEquals(info.dependencies, {});
});
```

Next, the information can be used to get all dependencies and construct an
import map.

## Getting dependencies

##### `get_dependencies.ts`

This will recursively get dependencies for a module.

```ts
import { getVersion } from "./get_version.ts";
import { getPackageInfo, PackageInfo } from "./get_package_info.ts";

async function visitDependency(
  packages: { [key: string]: PackageInfo },
  name: string,
  versionRange: string,
): Promise<void> {
  const version = await getVersion(name, versionRange);
  const packageInfo = await getPackageInfo(name, version);
  packages[name] = packageInfo;
  for (
    const [depName, depVersion] of Object.entries(packageInfo.dependencies)
  ) {
    if (!packages[depName]) {
      await visitDependency(packages, depName, depVersion);
    }
  }
}

export async function getDependencies(
  dependencies: { [key: string]: string },
): Promise<{ [key: string]: PackageInfo }> {
  const packages = {};
  for (const [name, version] of Object.entries(dependencies)) {
    await visitDependency(packages, name, version);
  }
  return packages;
}
```

##### `get_dependencies_test.ts`

We'll test by getting the dependencies for a couple of packages.

```ts
import { getDependencies } from "./get_dependencies.ts";
import { assertEquals } from "https://deno.land/std@0.133.0/testing/asserts.ts";

Deno.test("with dependencies", async () => {
  const packages = await getDependencies({ "@codemirror/state": "0.19.9" });
  assertEquals(Object.keys(packages).length, 2);
});

Deno.test("without dependencies", async () => {
  const packages = await getDependencies({ "style-mod": "4.0.0" });
  assertEquals(Object.keys(packages).length, 1);
});
```

Run:

```bash
deno test --allow-net=data.jsdelivr.com,cdn.jsdelivr.net get_dependencies_test.ts
```

## Generate import maps and imports with types

The import maps simply map a package's short name on npm to its `.js` file
on jsDelivr. The imports with types will import each file and add a type
reference comment above the import.

##### `generate.ts`

```ts
import type { PackageInfo } from "./get_package_info.ts";

interface SimpleImportMap {
  imports: { [key: string]: string };
}

export function generateImportMap(
  packages: { [key: string]: PackageInfo },
): SimpleImportMap {
  const imports = Object.fromEntries(
    Object.entries(packages).map(
      ([
        name,
        { version, module },
      ]) => [name, `https://cdn.jsdelivr.net/npm/${name}@${version}/${module}`],
    ),
  );
  return { imports };
}

export function generateTypedImports(
  packages: { [key: string]: PackageInfo },
): string {
  return Object.entries(packages).map(
    ([
      name,
      { version, module, types },
    ]) =>
      [
        `// @deno-types="https://cdn.jsdelivr.net/npm/${name}@${version}/${types}"`,
        `import "https://cdn.jsdelivr.net/npm/${name}@${version}/${module}";`,
      ].join("\n"),
  ).join("\n\n") + "\n";
}
```

Testing:

##### `generate_test.ts`

```ts
import { generateImportMap, generateTypedImports } from "./generate.ts";
import { assert, assertEquals } from "https://deno.land/std@0.133.0/testing/asserts.ts";

const packages = {
  "@codemirror/state": {
    name: "@codemirror/state",
    version: "0.19.9",
    module: "dist/index.js",
    types: "dist/index.d.ts",
    dependencies: { "@codemirror/text": "^0.19.0" },
  },
  "@codemirror/text": {
    name: "@codemirror/text",
    version: "0.19.6",
    module: "dist/index.js",
    types: "dist/index.d.ts",
    dependencies: {},
  },
};

Deno.test("generate import map", () => {
  const packageInfo = packages["@codemirror/text"];
  const result = generateImportMap({ "@codemirror/text": packageInfo });
  const url =
    "https://cdn.jsdelivr.net/npm/@codemirror/text@0.19.6/dist/index.js";
  assertEquals({ imports: { "@codemirror/text": url } }, result);
});

Deno.test("generate typed endpoints", () => {
  const text = generateTypedImports(packages);
  const match1 = /deno-types.*text.*"/.exec(text);
  assert(match1 && match1[0].endsWith("dist/index.d.ts\""));
  const match2 = /deno-types.*state.*"/.exec(text);
  assert(match2 && match2[0].endsWith("dist/index.d.ts\""));
});
```

Run:

```bash
deno test generate_test.ts
```

## Main module

The main module provides a method that runs the whole thing and exports
functions and variables.

##### `mod.ts`

```ts
export type { PackageInfo } from "./get_package_info.ts";
export { getDependencies } from "./get_dependencies.ts";
export { generateImportMap, generateTypedImports } from "./generate.ts";
```

## Command

The command takes some dependencies and outputs an import map and a
file containing import statements with types. It can operate on files
in the filesystem or on Markdown stdin/stdout with the files embedded
in code blocks. The input can also come from positional command-line
arguments. If no arguments are given it acts as a simple filter
taking Markdown from stdin and stdout and using default filenames.

##### `main.ts`

```ts
import { getDependencies, generateImportMap, generateTypedImports } from "./mod.ts";
import { parse, Args } from "https://deno.land/std@0.140.0/flags/mod.ts";
import { unpack } from "https://deno.land/x/md_unpack_simple@0.0.3/mod.ts";
import { readAll } from "https://deno.land/std@0.140.0/streams/conversion.ts";
import * as JSONC from "https://deno.land/std@0.140.0/encoding/jsonc.ts";

const MISSING_DATA_MESSAGE = (
  'Expected valid JSON input or valid JSON code block named' +
  ' "dependencies.json"'
)

function stringToDep(s: string) {
  const i = s.lastIndexOf('@');
  return i === -1 ? [s, '*'] : [s.substring(0, i), s.substring(i + 1)];
}

async function getInputDependencies(flags: Args): Promise<{ [key: string]: string }> {
  if (flags._.length > 0) {
    return Object.fromEntries(flags._.map(s => stringToDep(s.toString())));
  } else {
    let text
    if ('input' in flags) {
      text = await Deno.readTextFile(flags.input.toString());
    } else {
      text = new TextDecoder().decode(await readAll(Deno.stdin));
    }
    let input
    try {
      input = JSONC.parse(text);
    } catch (inputErr) {
      const unpackedData = await unpack(text);
      if (unpackedData && Object.keys(unpackedData).length > 0) {
        const codeBlockData = unpackedData['dependencies.json'];
        if (typeof codeBlockData === 'string') {
          input = JSONC.parse(codeBlockData);
        } else {
          throw new Error(MISSING_DATA_MESSAGE);
        }
      } else {
        if (/\s+[/{]/.test(text)) {
          throw inputErr;
        } else {
          throw new Error(MISSING_DATA_MESSAGE);
        }
      }
    }
    if (
      input !== null && typeof input === 'object' && !Array.isArray(input) &&
      Object.values(input).every(v => typeof v === 'string')
    ) {
      return input as { [key: string]: string }
    } else {
      throw new Error('Dependencies must a be map of strings')
    }
  }
}

export async function run(args: string[]) {
  const flags = parse(args);
  const inputDependencies = await getInputDependencies(flags);
  const dependencies = await getDependencies(inputDependencies);
  const importMap = generateImportMap(dependencies);
  const typedImports = generateTypedImports(dependencies);
  console.log(
`##### \`import_map.json\`

\`\`\`json
${JSON.stringify(importMap, null, 2)}
\`\`\`

##### \`typed_imports.ts\`

\`\`\`ts
${typedImports.trim()}
\`\`\`
`
  )
}

if (import.meta.main) {
  run(Deno.args);
}
```

## Documentation

##### `README.md`

````md
# deliver_importmaps

[![pipeline status](https://gitlab.com/ResourcesCo/macchiato/deliver_importmaps/badges/main/pipeline.svg)](https://gitlab.com/ResourcesCo/macchiato/deliver_importmaps/-/commits/main)

This is a tool for generating an import map for loading JavaScript modules
straight from the source inside of npm packages, delivered by the
[jsDelivr CDN][jsd].

## Usage

```ts
import {
  generateImportMap,
  generateTypedImports,
  getDependencies,
} from "https://deno.land/x/deliver_importmaps@0.0.2/mod.ts";
const dependencies = await getDependencies({ "style-mod": "*" });
const importMap = generateImportMap(dependencies);
const typedImports = generateTypedImports(dependencies);
console.log({ importMap, typedImports });
```

This permission is needed:

```
--allow-net=data.jsdelivr.com,cdn.jsdelivr.net
```

## Command

To install, run:

```
deno install --allow-net=data.jsdelivr.com,cdn.jsdelivr.net \
  https://deno.land/x/deliver_importmaps@0.0.2/main.ts
```

This will install it as `deliver_importmaps`.

To run, create a Markdown file with a code block in it that has a list
of packages and version numbers in a code block called `dependencies.json`
and pipe it into the command:

```
cat input.md | deliver_importmaps
```

The output will contain a code block that has an import map and another
with the types for creating a file. They can be put into files using
[md_unpack_simple][md_unpack_simple]:

```
cat input.md | deliver_importmaps | md_unpack_simple
```

## Source

This is generated from a markdown document, [deliver_importmaps.md][src], using
[md_unpack_simple][md_unpack_simple].

## License

MIT

[jsd]: https://jsdelivr.net/
[src]: https://github.com/ResourcesCo/macchiato/blob/main/build/deno/deliver_importmaps.md
[md_unpack_simple]: https://deno.land/x/md_unpack_simple
````

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

## Continuous Integration

##### `.gitlab-ci.yml`

```yaml
stages:
  - test

image: denoland/deno:ubuntu-1.22.2

deno-lint:
  stage: test
  script:
    - deno lint --unstable

deno-test:
  stage: test
  script:
    - deno test --allow-net=data.jsdelivr.com,cdn.jsdelivr.net --unstable
```

[jsd]: https://jsdelivr.net/
[dlx]: https://deno.land/x
[dnt]: https://github.com/denoland/dnt/
