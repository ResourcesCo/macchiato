##### `mod.ts`

```ts
import { Client } from "https://deno.land/x/postgres/mod.ts";

const client = new Client({
  user: "postgres",
  database: "pagila",
  hostname: "localhost",
  port: 5432,
})

await client.connect();
const result = await client.queryArray(
  "select film_id, title, description, release_year from film where title like '%WEEKEND%';"
);
console.log(result.rows);
```
