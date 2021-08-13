##### `mod.ts`

```ts
const text = await Deno.readTextFile('data.json');
const requestData = JSON.parse(text);
const resp = await fetch('https://httpbin.org/post', {
  method: 'post',
  body: JSON.stringify(requestData),
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
});
const data = await resp.json();
console.log(data);
```

##### `data.json`

```ts
{
  "hello": "world"
}
```