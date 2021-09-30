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
   requested. It also contains a scope parameter that is used to check
   that the user that completes sign-in is the same one who initiated
   sign-in.
3. When the user is redirected, the authorization server gets information
   about the client, so it can use that information when asking if the user
   would like to sign into the client with their identity, and with access
   permissions. It shows a sign-in form to the user. When the user approves
   sign-in, it redirects back to the client, with a temporary
   **authorization code**.
4. The client receives the redirect with the authorization code and uses
   the code to get an access token as well as a profile url. The client
   also checks the state parameter. If it doesn't match, it denies access.
   The client checks that the user's profile URL declares the same
   authorization server.
5. The client redirects the user, and stores the access token for future use,
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

## Getting `Link` values from HTTP and HTML headers

With IndieAuth, you sign in with your URL. It then gets the authorization endpoint from the header or the
HTML (typically through the HTML, though the header takes precedence).
This means that it needs to parse the `<link>` out of the HTML.

Deno doesn't have an HTML parser in the standard library, so a third
party library is needed. There is one that uses a Rust library that
works as a Deno plugin or with WebAssembly. This will use WebAssembly
so minimal Deno permissions are needed.

##### `deno_dom_link.ts`

```ts
import { DOMParser, Element } from "https://deno.land/x/deno_dom/deno-dom-wasm.ts";
import { parse } from "https://deno.land/std@0.108.0/flags/mod.ts";

async function run() {
  const flags = parse(Deno.args);
  const [url, linkRel] = flags._;
  if (flags._.length !== 2 || typeof url !== 'string' || typeof linkRel !== 'string') {
    throw new Error('Usage: deno_dom_link <url> <link-rel>');
  }
  const resp = await fetch(url);
  if (!resp.ok) {
    throw new Error(`Error getting web page: ${resp.status}`);
  }
  const text = await resp.text();
  const doc = new DOMParser().parseFromString(text, "text/html");
  if (doc === null) {
    throw new Error('Error parsing document');
  }
  const link = doc.querySelector(`link[rel=${linkRel}]`);
  if (link !== null) {
    console.log(link.getAttribute('href'));
  } else {
    throw new Error('Link not found');
  }
}

if (import.meta.main) {
  run();
}
```

It takes about half a second to load the WebAssembly (similar on repeated invocations):

```
dinosaurio@denotastic:~/macchiato/auth/indieauth$ time deno run --allow-net=benatkin.com deno_dom_link.ts https://benatkin.com/ authorization_endpoint
https://benatkin.com/wp-json/indieauth/1.0/auth

real	0m0.454s
user	0m0.445s
sys	0m0.095s
```

This cost is incurred when starting the program. It can be incurred after starting
the program by importing a [different deno_dom module][deno-dom-noinit].

This is needed to construct the URL. A state parameter is also needed. This can be
generated and set as a cookie when redirecting. The cookie should be signed to
ensure it's coming from the same place where the initial redirect is coming from.

## Step 2: 

[md_unpack_simple]: https://deno.land/x/md_unpack_simple
[indieauth]: https://indieauth.net/
[deno-dom-noinit]: https://deno.land/x/deno_dom@v0.1.15-alpha/deno-dom-wasm-noinit.ts
[sign-verify-deno]: https://medium.com/deno-the-complete-reference/sign-verify-jwt-hmac-sha256-4aa72b27042a