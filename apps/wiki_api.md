# Wiki API

The idea here is to make a simple wiki API that can edit files in the local
directory, and show the revision history, and revert to old revisions.

It will have a main module that provides functions for limited access to
what is needed for the API, and most of the logic in a worker that can only
access through the main module's functions, using postMessage.

##### `mod.ts`

```ts
import { relative, resolve } from "https://deno.land/std@0.126.0/path/mod.ts";

export class InvalidPathError extends Error {
  constructor(message) {
    super(message);
    this.name = "InvalidPathError";
  }
}

class WikiApi {
  root: string

  constructor(root: string) {
    this.root = root;
  }

  resolve(path: string) {
    const result = resolve(this.root, path);
    const relativePath = relative(this.root, result);
    if (relativePath.startsWith("..") {
      throw new InvalidPathError("The path must be inside the root");
    } else if (relativePath.startsWith(".git/") || relativePath.startsWith(".git\\")) {
      throw new InvalidPathError("The path must not be inside the .git directory");
    } else if (relativePath.startsWith(".env")) {
      throw new InvalidPathError("The path must not be inside a .env directory");
    }
  }

  async readFile(path: string) {
    const resolvedPath = this.resolve(path);
    const content = await Deno.readTextFile(resolvedPath);
    return content;
  }

  async writeFile(path: string, content: string) {
    const resolvedPath = this.resolve(path);
    await Deno.writeTextFile(resolvedPath, content);
  }

  async deleteFile(path: string) {
    await Deno.remove(resolvedPath);
  }

  async gitCommit(message: string) {
    const cmd = ["git", "commit", "-a", "-m", message];
    await Deno.run({cmd});
  }

  async gitPush() {
    const cmd = ["git", "push"];
    await Deno.run({cmd});
  }

  async gitLog(skip: number = 0, limit: number = 25) {
    const cmd = ["git", "log", "-n"];
    await Deno.run({cmd});
    // TODO: capture input, parse elsewhere
  }

  async gitShowCommit(revision: string) {
    const cmd = ["git", "show", "--patch", revision];
    await Deno.run({cmd});
    // TODO: capture input, parse elsewhere
  }

  async gitDiff(revision1: string, revision2: string) {
    const cmd = ["git", "diff", revision1, revision2];
    await Deno.run({cmd});
    // TODO: capture input, parse elsewhere
  }

  async gitShowFile(path: string, revision: string) {
    const cmd = ["git", "show", `${revision}:${path}`];
    await Deno.run({cmd});
  }
}
```
