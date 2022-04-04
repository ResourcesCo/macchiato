# Bundling CodeMirror 6

The goal here is to make a CodeMirror 6-based code editing component from
npm with Deno. This will use import maps so npm modules can find each
other without modifying the source.

## Step 1: Getting dependencies

Here is code for loading an editor, from the [Bundling with Rollup][bwr]
example from CodeMirror:

##### `example.js`

```js
import {EditorState, EditorView, basicSetup} from "@codemirror/basic-setup"
import {javascript} from "@codemirror/lang-javascript"

let editor = new EditorView({
  state: EditorState.create({
    extensions: [basicSetup, javascript()]
  }),
  parent: document.body
})
```

For `deno bundle` to be able to bundle this, we need an import map. The
import map needs to map `npm` packages from their names to the es modules
on `jsDelivr`.

We'll do this without relying on the `npm` program, but instead fetching
the files straight from [jsDelivr][jsdelivr], starting with `package.json`
files.

The URLs for npm packages on jsDelivr are of this form:

```
https://cdn.jsdelivr.net/npm/package@version/file
```

`package` can have an organization name. Here is the `package.json` for
`@codemirror/basic-setup`:

```
https://cdn.jsdelivr.net/npm/@codemirror/basic-setup@0.19.1/package.json
```

These can be found by typing the name of the repository in the search box
on [jsdelivr.com][jsdelivr]. This will pull up the latest version.

Another way to get the latest version is to get the URL without the verison:

```
https://cdn.jsdelivr.net/npm/@codemirror/basic-setup/package.json
```

The `version` and the `module` in `package.json` can be used to get a URL
for the import map, and `dependencies` can be used to find similar URLs
for dependencies. However, the dependencies have their own version
requirements. There is an [official jsDelivr API][jsdelivr-api] with an
[endpoint that resolves versions from semver version ranges][resolve-version].
We'll use that.

They ask to provide a user agent. For that we'll use:

```
User-Agent: Macchiato https://github.com/ResourcesCo/macchiato
```

Let's make the first API requests and get the version numbers for the
`@codemirror/basic-setup` and `@codemirror/lang-javascript` with '*'
as the semver version range, and then gather their dependencies.

##### `get_package.ts`

```ts
const baseUrl = 'https://data.jsdelivr.com/v1';
const headers = {
  Accept: 'application/json',
  'User-Agent': 'Macchiato https://github.com/ResourcesCo/macchiato',
};

function splitPackageName(name: string): string[] {
  const pos = name.indexOf('/');
  if (pos === -1) {
    return [name];
  } else {
    return [name.substring(0, pos), name.substring(pos + 1)];
  }
}

export async function getPackage(name: string, versionRange = '*'): Promise<any> {
  const urlName = splitPackageName(name).map(s => encodeURI(s)).join('/');
  const urlVersionRange = encodeURI(versionRange);
  const url = `${baseUrl}/package/resolve/npm/${urlName}@${urlVersionRange}`;
  const resp = await fetch(url);
  const respBody = await resp.json();
  const version = respBody.version;
  return version;
}
```

##### `get_top_level_packages.ts`

```ts
import { getPackage } from './get_package.ts';

const imports: {[key: string]: string} = {};
for (const name of ['@codemirror/basic-setup', '@codemirror/lang-javascript']) {
  const version = await getPackage(name);
  imports[name] = version;
}
console.log(JSON.stringify({imports}, null, 2));
```

Running this:

```
deno run --allow-net=data.jsdelivr.com get_top_level_packages.ts
```

It outputs the following:

```json
{
  "imports": {
    "@codemirror/basic-setup": "0.19.1",
    "@codemirror/lang-javascript": "0.19.7"
  }
}
```

Now this needs to be run recursively, getting the dependencies, adding them to
the data, and skipping any that have already been retrieved.

##### `get_dependencies.ts`

```ts
import { getPackage } from './get_package.ts';

export type DependencyMap = {[key: string]: string}

interface PackageJson {
  dependencies: DependencyMap | undefined
  version: string
}

export interface Package {
  dependencies ?: DependencyMap
  dependents ?: DependencyMap
  version: string
  packageJson: any
}

export type PackageInfo = {[key: string]: Package}

export async function getPackageJson(
  name: string,
  version: string
): Promise<PackageJson> {
  const url = `https://cdn.jsdelivr.net/npm/${name}@${version}/package.json`;
  const resp = await fetch(url);
  const text = await resp.text();
  const result = JSON.parse(text) as PackageJson;
  return result;
}

export async function visitDependency(
  packages: PackageInfo, name: string, versionRange: string
): Promise<void> {
  const version = await getPackage(name, versionRange);
  if (typeof version !== 'string') {
    throw new Error('expected version to be a string');
  }
  const packageJson = await getPackageJson(name, version);
  packages[name] = {
    version: packageJson.version,
    packageJson,
  };
  const dependencies = packageJson.dependencies;
  if (dependencies) {
    packages[name].dependencies = dependencies
    for (const [depName, depVersion] of Object.entries(dependencies)) {
      let depPackage = packages[depName];
      if (depPackage === undefined) {
        await visitDependency(packages, depName, depVersion);
      }
      depPackage = packages[depName]
      if (depPackage !== undefined) {
        let dependents = depPackage.dependents
        if (dependents === undefined) {
          dependents = {};
          depPackage.dependents = dependents;
        }
        dependents[name] = packageJson.version;
      }
    }
  }
}

export async function getDependencies(
  dependencies: DependencyMap
): Promise<PackageInfo> {
  const packages: PackageInfo = {};
  for (const [depName, depVersion] of Object.entries(dependencies)) {
    await visitDependency(packages, depName, depVersion);
  }
  return packages;
}
```

Here is a script to run it:

##### `get_packages_with_dependencies.ts`

```ts
import { getDependencies } from './get_dependencies.ts';

const dependencies = await getDependencies({
  '@codemirror/basic-setup': '*',
  '@codemirror/lang-javascript': '*',
});
const json = JSON.stringify(dependencies, null, 2);
await Deno.writeTextFile('dependencies.json', json);
```

Running this:

```
deno run --allow-net=data.jsdelivr.com,cdn.jsdelivr.net \
--allow-write=dependencies.json get_packages_with_dependencies.ts
```

## Step 2: Building an import map

The import map will map the package names in npm to module URLs on jsDelivr.

##### `build_import_map.ts`

```ts
if (Deno.args.length !== 2) {
  throw new Error('Must give inputFile and outputFile as arguments');
}
const [inputFile, outputFile] = Deno.args;
interface Package {
  version: string
  packageJson: {
    name: string
    module?: string
  }
}
const inputJson = await Deno.readTextFile(inputFile);
const dependencies = JSON.parse(inputJson) as {[key: string]: Package};
const imports = Object.fromEntries(Object.entries(dependencies).map(
  ([name, {version, packageJson: {module}}]) => {
    if (![name, version, module].every(v => typeof v === 'string')) {
      throw new Error(
        'expected these to be strings: ' + JSON.stringify({name, version, module})
      );
    }
    const url = `https://cdn.jsdelivr.net/npm/${name}@${version}/${module}`
    return [name, url];
  }
))
Deno.writeTextFile(outputFile, JSON.stringify({imports}, null, 2));
```

Running it:

```
deno run --allow-read=dependencies.json --allow-write=import-map.json \
build_import_map.ts dependencies.json import-map.json
```

## Step 3: Bundling with the import map and running it

Bundling it using `deno bundle`:

```
deno bundle --import-map=import-map.json example.js bundle.js
```

##### `index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <title>CodeMirror example</title>
    <meta charset="utf-8">
  </head>
  <body>
    <script type="module" src="/bundle.js"></script>
  </body>
</html>
```

Now, run it using a simple development HTTP server like Deno's `file_server`
and load it in the browser:

```
deno run --allow-net --allow-read https://deno.land/std@0.133.0/http/file_server.ts
```

[bwr]: https://codemirror.net/6/examples/bundle/
[jsdelivr]: https://www.jsdelivr.com/
[jsdelivr-api]: https://github.com/jsdelivr/data.jsdelivr.com#endpoints
