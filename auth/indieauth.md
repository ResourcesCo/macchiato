# IndieAuth Client Library for Deno

This is a Markdown code notebook has embedded files that can be unpacked
with [md_unpack_simple][md_unpack_simple].

## Introduction

From [IndieAuth.net][indieauth]:

> IndieAuth is a decentralized identity protocol built on top of OAuth 2.0.
>
> This allows individual websites like someone's WordPress, Mastodon, or
> Gitea server to become its own identity provider, and can be used to sign
> in to other instances. Both users and applications are identified by URLs,
> avoiding the need for getting API keys or making new accounts.

There are several steps to signing in using IndieAuth that this library
will support:

1. A user initiates sign-in by submitting their identity URL.
2. The client application uses their identity URL to get details about how
   it can sign in with the identity URL. It gets the **authorization_endpoint**
   from the URL and uses that to construct a redirect URL. This redirect
   URL contains the URL of the client as well as any permissions being
   requested. It also contains a [PKCE Code Challenge][pkce] and a state
   parameter that are used to check that the user that completes sign-in is
   the same one who initiated sign-in.
4. When the user is redirected, the authorization server gets information
   about the client, so it can use that information when asking if the user
   would like to sign into the client with their identity, and with access
   permissions. It shows a sign-in form to the user. When the user approves
   sign-in, it redirects back to the client, with a temporary
   **authorization code**.
5. The client receives the redirect with the authorization code and uses
   the code to get an access token as well as a profile url. The client
   also checks the state parameter. If it doesn't match, it denies access.
   The client checks that the user's profile URL declares the same
   authorization server.
6. The client redirects the user, and stores the access token for future use,
   if needed. The user is now signed in.

Some parts of this will be in the library and some will be in an example
application.

## Example Sign-in Form

The sign-in form will provide a field for the URL and a hidden
[CSRF token](https://laravel.com/docs/8.x/csrf). The CSRF token will be
hashed and verified using HMAC from WebCrypto, and will expire after 10
minutes. See [Sign & Verify JWT (HMAC SHA256) in Deno][sign-verify-deno].

##### `example/0/templates/sign_in.html`

```html
<!doctype html>
<html charset="en">
  <head>
    <title>IndieAuth sign-in example</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/water.css@2/out/water.css">
  </head>
  <body>
    <h1>IndieAuth sign-in example</h1>
    <main>
      <form action="/auth/indieauth" method="post">
        <label for="url"></label>
        <input type="text" name="url" id="url">
        <input type="submit" value="Sign In">
        <input type="hidden" name="csrf_token" value="{csrfToken}">
      </form>
    </main>
  </body>
</html>
```

##### `example/0/sign_in.ts`

```ts
import { listenAndServe } from "https://deno.land/std@0.108.0/http/server.ts";
import {encode, decode} from "https://deno.land/std/encoding/base64url.ts";

const secretKey = Deno.env.get('SECRET_KEY') as string;
const key = await crypto.subtle.importKey(
  'raw',
  new TextEncoder().encode(secretKey),
  {name: 'HMAC', hash: 'SHA-256'},
  false,
  ['sign', 'verify']
);

async function sign(text: string): Promise<string> {
  const signature = await crypto.subtle.sign(
    {name: "HMAC"},
    key,
    new TextEncoder().encode(text)
  );
  const encodedSignature = encode(new Uint8Array(signature));
  return `${text}|${encodedSignature}`;
}

async function verify(signedText: string): Promise<string> {
  const index = signedText.lastIndexOf('|');
  if (index === -1) {
    throw new Error('Verification failed: signature not found');
  }
  const signature = decode(signedText.substr(index + 1));
  const text = signedText.substr(0, index);
  const verified = await crypto.subtle.verify(
    {name: "HMAC"},
    key,
    signature,
    new TextEncoder().encode(text)
  );
  if (verified) {
    return text
  } else {
    throw new Error('Verification failed: crypto.subtle.verify() returned false')
  }
}

async function getPort(_default = 3000) {
  const desc = { name: 'env', variable: 'PORT' } as const;
  const { state } = await Deno.permissions.query(desc);
  if (state === 'granted') {
    const port = parseInt(Deno.env.get('PORT') ?? '');
    return Number.isInteger(port) ? port : _default;
  } else {
    return _default;
  }
}

async function makeCsrfToken(): Promise<string> {
  return await sign(`csrf:${new Date().valueOf()}`);
}

async function verifyCsrfToken(csrfToken: string): Promise<boolean> {
  const token = await verify(csrfToken);
  const parts = token.split(':');
  if (parts.length === 2 && parts[0] === 'csrf') {
    const ts = parseInt(parts[1]);
    if (Number.isInteger(ts)) {
      const tenMinutesAgo = new Date().valueOf() - (10 * 60 * 1000);
      return ts >= tenMinutesAgo;
    }
  }
  return false;
}

const port = await getPort();
const signInTemplate = await Deno.readTextFile(
  new URL('./templates/sign_in.html', import.meta.url)
);
listenAndServe(`:${port}`, async (request: Request) => {
  try {
    if (request.method === 'POST') {
      const formData = await request.formData();
      const csrfToken = formData.get('csrf_token');
      if (typeof csrfToken !== 'string') {
        throw new Error('Missing CSRF token');
      }
      const result = await verifyCsrfToken(csrfToken);
      return new Response(
        result ? 'Ready to redirect' : 'error',
        {headers: {'Content-Type': 'text/html; charset=UTF-8'}}
      )
    } else {
      const csrfToken = await makeCsrfToken();
      const html = signInTemplate.replace('{csrfToken}', csrfToken);
      return new Response(html, {headers: {'Content-Type': 'text/html; charset=UTF-8'}});
    }
  } catch (err) {
    console.error(`Error handling request: ${err}`);
    return new Response('<h1>Internal Server Error</h1>', {
      status: 500,
      headers: {'Content-Type': 'text/html; charset=UTF-8'},
    });
  }
});
```

In the above code, there is a general-purpose sign and verify function, as well
as a function for generating a CSRF token and a function for verifying it. The
general-purpose sign and verify functions will be used for the state parameter
in the next step.

Run the example (set SECRET_KEY to a random string):

```bash
cd example/0
deno run --allow-read=./templates --allow-net=:3000 --allow-env=SECRET_KEY sign_in.ts 3000
```

If you go to the web server, it will show a form, and you can enter anything into
the input and submit it (it isn't used yet) and submit it, and if it's been less
than ten minutes since you loaded the page, it should say "Ready to redirect".

## Getting the authorization_endpoint link

The `authorization_endpoint` `link` `rel` needs to be retrieved from the HTTP
headers or from the HTML at the location provided in the form.

To attempt to get it from the HTTP headers, a HEAD request will be made. These are
third-party web servers, so a User Agent will need to be given for some, as some
will reject requests without a User Agent. Not only that, some will reject all
that aren't coming from the browser, so using the same User Agent a browser would
use is something to consider. This will default to DenoIndieAuth as the user agent
and allow it to be overridden.

##### `example/1/get_link_from_header.ts`

```ts
const linkRegexp = /^<([^>]*)>(.*)$/;

export function linkFromHeaders(
  headers: Headers,
  rel: string
): string | undefined {
  for (const linkText of (headers.get('link') ?? '').split(',')) {
    const [href, remainingText] = (linkText.trim().match(linkRegexp) ?? []).slice(1);
    if (href !== undefined && remainingText !== undefined) {
      for (const s of remainingText.split(';')) {
        const equalsIndex = s.indexOf('=');
        const key = s.substr(0, equalsIndex).trim();
        const value = s.substr(equalsIndex + 1).trim();
        if (key === 'rel' && [`"${rel}"`, rel].includes(value)) {
          return href;
        }
      }
    }
  }
}

export default async function getLinkFromHeader(
  url: string,
  rel: string,
  userAgent = 'DenoIndieAuth'
): Promise<string | undefined> {
  const resp = await fetch(url, {
    method: 'HEAD',
    headers: {
      'User-Agent': userAgent,
    }
  });
  if (resp.ok) {
    return linkFromHeaders(resp.headers, rel);
  }
}
```

##### `example/1/try_get_link_from_header.ts`

```ts
import getLinkFromHeader from './get_link_from_header.ts';

const [url, rel] = Deno.args;
if (![url, rel].every(a => typeof a === 'string' && a.length > 0)) {
  throw new Error('Usage: $0 <url> <rel>');
}

const result = await getLinkFromHeader(url, rel);
```

To get the endpoint from the HTML, an HTML parser is needed. There is one on npm
called [htmlparser2][htmlparser2] that is well maintained. To import them into
Deno, [jspm with import maps can be used](jspm-import-maps). This was made with
the jspm import map generator:

##### `example/1/import-map.json`

```json
{
  "imports": {
    "htmlparser2": "https://ga.jspm.io/npm:htmlparser2@7.1.2/lib/index.js"
  },
  "scopes": {
    "https://ga.jspm.io/": {
      "dom-serializer": "https://ga.jspm.io/npm:dom-serializer@1.3.2/lib/index.js",
      "domelementtype": "https://ga.jspm.io/npm:domelementtype@2.2.0/lib/index.js",
      "domhandler": "https://ga.jspm.io/npm:domhandler@4.2.2/lib/index.js",
      "domutils": "https://ga.jspm.io/npm:domutils@2.8.0/lib/index.js",
      "entities": "https://ga.jspm.io/npm:entities@2.2.0/lib/index.js",
      "entities/lib/decode": "https://ga.jspm.io/npm:entities@3.0.1/lib/decode.js",
      "entities/lib/decode_codepoint": "https://ga.jspm.io/npm:entities@3.0.1/lib/decode_codepoint.js"
    }
  }
}
```

[It's available here, with the parameters encoded in the URL.](https://generator.jspm.io/#a+JhYGBkdEpMSs1RcK1IzC3ISWXIKMnNKUgsKk4tMnIw1zPUMwIASzJmfSUA)

Here is a script that shows it working in Deno:

##### `example/1/try_htmlparser2.ts`

```ts
import { Parser } from 'htmlparser2';
const html = `<!doctype html>
<html>
  <head>
    <title>Test</title>
    <link rel="authorization_endpoint" href="https://example.com/wp-json/indieauth/1.0/auth">
  </head>
  <body>
    <h1>Test</h1>
  </body>
</html>`;
function onopentag(name: string, attributes: any) {
  if (name === 'link') console.log({name, attributes});
}
const parser = new Parser({onopentag});
parser.write(html);
```

It can be run with the `--import-map` flag passed to it (be sure to cd into the
`example/1` directory):

```bash
deno run --import-map import-map.json
```

Here's the output:

```bash
{
  name: "link",
  attributes: {
    rel: "authorization_endpoint",
    href: "https://example.com/wp-json/indieauth/1.0/auth"
  }
}
```

To make a method for getting the endpoint, we'll take the href from the first
value, and ignore the rest.

##### `example/1/get_link_from_html.ts`

```ts
import { Parser } from 'htmlparser2';

export default function getLinkFromHtml(
  html: string,
  rel: string
): Promise<string | undefined> {
  return new Promise((resolve, reject) => {
    let done = false;
    const onopentag = (tag: string, attrs: {[key: string]: any}) => {
      if (!done && tag === 'link' && attrs['rel'] === rel) {
        done = true;
        resolve(typeof attrs['href'] === 'string' ? attrs['href'] : undefined);
      }
    }
    const onend = () => {
      if (!done) resolve(undefined);
    }
    const parser = new Parser({onopentag, onend});
    parser.write(html);
    parser.end();
  })
}
```

##### `example/1/get_link_from_html_test.ts`

```ts
import { assertEquals } from "https://deno.land/std@0.110.0/testing/asserts.ts";
import getLinkFromHtml from './get_link_from_html.ts';

const html = `<!doctype html>
<html>
  <head>
    <title>Test</title>
    <link rel="authorization_endpoint" href="https://example.com/wp-json/indieauth/1.0/auth">
  </head>
  <body>
    <h1>Test</h1>
  </body>
</html>`;

Deno.test('example', async () => {
  const result = await getLinkFromHtml(html, 'authorization_endpoint');
  assertEquals(result, 'https://example.com/wp-json/indieauth/1.0/auth');
});
```

## PKCE

PKCE uses a code challenge method to ensure that the client requesting an
authorization code and the client using the authorization code are the
same. The code challenge method is SHA-256. Deno has support for this in
its standard library's [hash module][hash-module].

## Redirecting and checking the state

Here we'll start by taking features from above and putting them into modules
with tests. These will go in the main directory, and a new example will use
them.

### Signing and verifying

The signer and verifier takes a secret key, which is stored in a private
variable of a class.

##### `sign.ts`

```ts
import {encode, decode} from "https://deno.land/std/encoding/base64url.ts";

export async function createSigner(key: string): Promise<Signer> {
  const cryptoKey = await crypto.subtle.importKey(
    'raw',
    new TextEncoder().encode(key),
    {name: 'HMAC', hash: 'SHA-256'},
    false,
    ['sign', 'verify']
  );
  return new Signer(cryptoKey);
}

class Signer {
  #key: CryptoKey

  constructor(key: CryptoKey) {
    this.#key = key;
  }

  async sign(text: string): Promise<string> {
    const signature = await crypto.subtle.sign(
      {name: "HMAC"},
      this.#key,
      new TextEncoder().encode(text)
    );
    const encodedSignature = encode(new Uint8Array(signature));
    return `${text}|${encodedSignature}`;
  }

  async verify(signedText: string): Promise<string> {
    const index = signedText.lastIndexOf('|');
    if (index === -1) {
      throw new Error('Verification failed: signature not found');
    }
    const signature = decode(signedText.substr(index + 1));
    const text = signedText.substr(0, index);
    const verified = await crypto.subtle.verify(
      {name: "HMAC"},
      this.#key,
      signature,
      new TextEncoder().encode(text)
    );
    if (verified) {
      return text
    } else {
      throw new Error('Verification failed: crypto.subtle.verify() returned false')
    }
  }
}
```

##### `sign_test.ts`

```ts
import {
  assert,
  assertEquals,
  assertStringIncludes,
  assertRejects
} from "https://deno.land/std@0.110.0/testing/asserts.ts";
import { createSigner } from "./sign.ts";

Deno.test('signs and verifies successfully', async () => {
  const input = 'hello';
  const signer = await createSigner('abcweaerewjnlnwej9302432n423ajfwnenwjewjajjfajwl');
  const signed = await signer.sign(input);
  assertStringIncludes(signed, '|');
  const verified = await signer.verify(signed);
  assertEquals(input, verified);
})

Deno.test("verification fails with missing or wrong signature", async () => {
  const input = 'hello';
  const signer = await createSigner('abcweaerewjnlnwej9302432n423ajfwnenwjewjajjfajwl');
  const signed = await signer.sign(input);
  await assertRejects(() => signer.verify(input));
  const signed2 = await signer.sign('world');
  const switched = `${signed.split('|')[0]}|${signed2.split('|')[1]}`;
  await assertRejects(() => signer.verify(switched));
})
```

### HTML Links

This uses an HTML parser library from npm, via jspm, to get the requested
`<link>` values.

##### `import-map.json`

```json
{
  "imports": {
    "htmlparser2": "https://ga.jspm.io/npm:htmlparser2@7.1.2/lib/index.js"
  },
  "scopes": {
    "https://ga.jspm.io/": {
      "dom-serializer": "https://ga.jspm.io/npm:dom-serializer@1.3.2/lib/index.js",
      "domelementtype": "https://ga.jspm.io/npm:domelementtype@2.2.0/lib/index.js",
      "domhandler": "https://ga.jspm.io/npm:domhandler@4.2.2/lib/index.js",
      "domutils": "https://ga.jspm.io/npm:domutils@2.8.0/lib/index.js",
      "entities": "https://ga.jspm.io/npm:entities@2.2.0/lib/index.js",
      "entities/lib/decode": "https://ga.jspm.io/npm:entities@3.0.1/lib/decode.js",
      "entities/lib/decode_codepoint": "https://ga.jspm.io/npm:entities@3.0.1/lib/decode_codepoint.js"
    }
  }
}
```

[Source](https://generator.jspm.io/#a+JhYGBkdEpMSs1RcK1IzC3ISWXIKMnNKUgsKk4tMnIw1zPUMwIASzJmfSUA)

##### `get_html_links.ts`

```ts
import { Parser } from 'htmlparser2';

export default async function getHtmlLinks(
  response: Response,
  rels: string[]
): Promise<{[key: string]: string}> {
  const html = await response.text();
  return await new Promise((resolve, reject) => {
    let done = false;
    const links: {[key: string]: string} = {};
    const checkAndResolve = (force = false) => {
      if (force || rels.every(rel => links[rel] !== undefined)) {
        done = true;
        resolve(links);
      }
    }
    const onopentag = (tag: string, attrs: {[key: string]: any}) => {
      if (
        !done &&
        tag === 'link' &&
        rels.includes(attrs['rel']) &&
        links[attrs['rel']] === undefined
      ) {
        links[attrs['rel']] = attrs['href'] ?? ''
        checkAndResolve();
      }
    }
    const onend = () => {
      if (!done) checkAndResolve(true);
    }
    const parser = new Parser({onopentag, onend});
    parser.write(html);
    parser.end();
  });
}
```

##### `get_html_links_test.ts`

```ts
import { assertEquals } from "https://deno.land/std@0.110.0/testing/asserts.ts";
import getHtmlLinks from './get_html_links.ts';

const html = `<!doctype html>
<html>
  <head>
    <title>Test</title>
    <link rel="authorization_endpoint" href="https://example.com/wp-json/indieauth/1.0/auth">
  </head>
  <body>
    <h1>Test</h1>
  </body>
</html>`;

Deno.test('read html link', async () => {
  const links = await getHtmlLinks(new Response(html), ['authorization_endpoint']);
  assertEquals(
    links['authorization_endpoint'],
    'https://example.com/wp-json/indieauth/1.0/auth'
  );
});
```

### Links from headers or HTML

This gets link values from the headers or the HTML.

##### `get_links.ts`

```ts
const linkRegexp = /^<([^>]*)>(.*)$/;

export function getHeaderLinks(
  headers: Headers,
  rels: string[]
): {[key: string]: string} {
  const result: {[key: string]: string} = {};
  for (const linkText of (headers.get('link') ?? '').split(',')) {
    const [href, remainingText] = (linkText.trim().match(linkRegexp) ?? []).slice(1);
    if (href !== undefined && remainingText !== undefined) {
      for (const s of remainingText.split(';')) {
        const equalsIndex = s.indexOf('=');
        const key = s.substr(0, equalsIndex).trim();
        const value = s.substr(equalsIndex + 1).trim();
        if (key === 'rel') {
          for (const rel of rels) {
            if (result[rel] === undefined && [`"${rel}"`, rel].includes(value)) {
              result[rel] = href;
              if (rels.every(rel => result[rel] !== undefined)) {
                return result;
              }
            }
          }
          break;
        }
      }
    }
  }
  return result;
}

// Client for customizing request and helping with testing
type Client = (request: Request) => Promise<Response>

function makeDefaultClient(userAgent: string): Client {
  return (request) => {
    const newReq = request.clone();
    if (!newReq.headers.get('User-Agent')) {
      newReq.headers.set('User-Agent', userAgent);
    }
    return fetch(newReq);
  }
}

type LinkReader = (
  response: Response,
  rels: string[]
) => Promise<{[key: string]: string}>

type GetLinksOptions = {
  getHtmlLinks: LinkReader,
  userAgent?: string,
  makeHeadRequest?: boolean,
  client?: Client
};
export default async function getLinks(
  url: string,
  rels: string[],
  {
    getHtmlLinks,
    userAgent = 'DenoIndieAuth',
    makeHeadRequest = true,
    client: clientOption
  }: GetLinksOptions,
): Promise<{[key: string]: string}> {
  const client = clientOption ?? makeDefaultClient(userAgent);
  const linkResults: {[key: string]: string}[] = [];
  const getResult = (partial: boolean = false) => {
    const result = linkResults.reduceRight(
      (acc, v) => ({...acc, ...v}), {}
    )
    if (partial || rels.every(rel => result[rel])) {
      return result;
    }
  }
  if (makeHeadRequest) {
    const resp = await client(new Request(url, {
      method: 'HEAD',
      headers: {
        'User-Agent': userAgent,
      }
    }));
    if (resp.ok) {
      linkResults.push(getHeaderLinks(resp.headers, rels));
      if (Object.keys(linkResults.at(-1) ?? {}).length > 0) {
        const result = getResult();
        if (result !== undefined) {
          return result;
        }
      }
    }
  }
  const resp = await client(new Request(url, {
    headers: {
      'Accept': 'text/html',
      'User-Agent': userAgent,
    },
  }));
  if (resp.ok) {
    linkResults.push(getHeaderLinks(resp.headers, rels));
    if (Object.keys(linkResults.at(-1) ?? {}).length > 0) {
      const result = getResult();
      if (result !== undefined) {
        return result;
      }
    }
    try {
      const links = await getHtmlLinks(resp, rels);
    } catch (err) {
      // do nothing
    }
  }
  return getResult(true) ?? {};
}
```

##### `get_links_test.ts`

```ts
import { assertEquals } from "https://deno.land/std@0.110.0/testing/asserts.ts";
import getHtmlLinks from './get_html_links.ts';
import getLinks from './get_links.ts';

Deno.test('from head', async () => {
  const client: (request: Request) => Promise<Response> = async (request) => {
    const link = '<https://example.com/auth> rel="authorization_endpoint"';
    if (request.method.toUpperCase() === 'HEAD') {
      return new Response(undefined, { headers: {'Link': link} });
    } else {
      return new Response(undefined, { status: 500 });
    }
  }
  const links = await getLinks(
    'https://example.com/testuser',
    ['authorization_endpoint', 'misc'],
    {client, getHtmlLinks}
  );
  assertEquals(links['authorization_endpoint'], 'https://example.com/auth');
});

Deno.test('from html', async () => {
  const client: (request: Request) => Promise<Response> = async (request) => {
    const link = '<https://example.com/auth> rel="authorization_endpoint"';
    if (request.method.toUpperCase() === 'HEAD') {
      return new Response(undefined, { headers: {'Link': link} });
    } else {
      return new Response(undefined, { status: 500 });
    }
  }
  const links = await getLinks(
    'https://example.com/testuser',
    ['authorization_endpoint', 'misc'],
    {client, getHtmlLinks}
  );
  assertEquals(links['authorization_endpoint'], 'https://example.com/auth');
});

Deno.test('from both', async () => {
  const client: (request: Request) => Promise<Response> = async (request) => {
    const link = '<https://example.com/auth> rel="authorization_endpoint"';
    if (request.method.toUpperCase() === 'HEAD') {
      return new Response(undefined, { headers: {'Link': link} });
    } else {
      return new Response(undefined, { status: 500 });
    }
  }
  const links = await getLinks(
    'https://example.com/testuser',
    ['authorization_endpoint', 'misc'],
    {client, getHtmlLinks}
  );
  assertEquals(links['authorization_endpoint'], 'https://example.com/auth');
});

Deno.test('from head', async () => {
  const client: (request: Request) => Promise<Response> = async (request) => {
    const link = '<https://example.com/auth> rel="authorization_endpoint"';
    if (request.method.toUpperCase() === 'HEAD') {
      return new Response(undefined, { headers: {'Link': link} });
    } else {
      return new Response(undefined, { status: 500 });
    }
  }
  const links = await getLinks(
    'https://example.com/testuser',
    ['authorization_endpoint', 'misc'],
    {client, getHtmlLinks}
  );
  assertEquals(links['authorization_endpoint'], 'https://example.com/auth');
});
```

### Constructing the redirect URL

### Redirecting with cookies for the state and code challenge

[md_unpack_simple]: https://deno.land/x/md_unpack_simple
[indieauth]: https://indieauth.net/
[deno-dom-noinit]: https://deno.land/x/deno_dom@v0.1.15-alpha/deno-dom-wasm-noinit.ts
[sign-verify-deno]: https://medium.com/deno-the-complete-reference/sign-verify-jwt-hmac-sha256-4aa72b27042a
[htmlparser2]: https://www.npmjs.com/package/htmlparser2
[jspm-import-maps]: https://jspm.org/import-map-cdn
[pkce]: https://indieweb.org/PKCE
[hash-module]: https://deno.land/std@0.112.0/hash