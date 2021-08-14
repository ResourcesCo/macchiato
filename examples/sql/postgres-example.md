# SQL Query

This makes an SQL query using the Pagila example database. Before running this, create a
database named pagila and load the pagila sample data into it:

https://github.com/devrimgunduz/pagila

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
