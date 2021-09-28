# IndieAuth Client Library for Deno

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
hashed and verified using WebCrypto, and will expire after 10 minutes.

`example/0/templates/sign_in.html`

```html
<!doctype html>
<html charset="en">
  <head>
    <title>IndieAuth sign-in example</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/water.css@2/out/water.css">
  </head>
  <body>
    <h1>Form</h1>
    <main>
      <form action="/auth/indieauth" method="post">
        <label for="url"></label>
        <input type="text" name="url" id="url">
        <input type="submit" value="Sign In">
        <input type="hidden" name="csrf_token" value="{csrf_token}">
      </form>
    </main>
  </body>
</html>
```

`example/0/sign_in.ts`

```ts
import { listenAndServe } from "https://deno.land/std@0.108.0/http/server.ts";

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

const port = await getPort();
const html = await Deno.readTextFile('./templates/sign_in.html');
listenAndServe(`:{$port}`, () => Response(html));
```

To run the example:

```bash
cd example/0
deno run --allow-read=./templates --allow-net=0.0.0.0:3000 sign_in.ts 3000
```

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

[indieauth]: https://indieauth.net/
[deno-dom-noinit]: https://deno.land/x/deno_dom@v0.1.15-alpha/deno-dom-wasm-noinit.ts