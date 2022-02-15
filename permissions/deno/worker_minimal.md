# Deno worker permissions

This has a worker say how to format the input without actually giving it
access to the input.

This is a very minimal demo.

##### `mod.ts`

```ts
const worker = new Worker(
  new URL("./worker.ts", import.meta.url).href,
  {
    type: 'module',
    deno: {
      namespace: false,
      permissions: 'none',
    },
  },
);
const value = Deno.args[0];
worker.postMessage(Number.isNaN(parseInt(value)) ? 'string' : 'number');
worker.onmessage = async (e) => {
  console.log(e.data.replace('{value}', value));
}
```

##### `worker.ts`

```ts
self.onmessage = async (e) => {
  self.postMessage(
    e.data === 'number' ? 'You entered a number: {value}!' : 'You entered text: {value}'
  );
  self.close();
}
```

To run:

```bash
deno run --unstable --allow-read=. mod.ts 9
deno run --unstable --allow-read=. mod.ts hello
```
