# Bundling CodeMirror 6

The goal here is to make a CodeMirror 6-based code editing component from
npm with Deno. This will use import maps so npm modules can find each
other without modifying the source.

A previous attempt was to bundle straight from npm. It built, but doesn't
work yet:

- [Bundling CodeMirror 6 from GitHub with Deno](./codemirror-from-github.md)

## Getting dependencies and generating an import map

Here is code for loading an editor, from the [Bundling with Rollup][bwr]
example from CodeMirror:

##### `e1/example.ts`

```ts
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
import map needs to map `npm` packages to 

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

##### `e1/get_package.ts`

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

##### `e1/get_top_level_packages.ts`

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

[bwr]: https://codemirror.net/6/examples/bundle/
[jsdelivr]: https://www.jsdelivr.com/
[jsdelivr-api]: https://github.com/jsdelivr/data.jsdelivr.com#endpoints