# IndieAuth Sign-in with Deno

## Introduction

[IndieAuth][indieauth] is an authentication protocol for the Open Web, that
is based on OAuth 2.0.

Perhaps the biggest difference between a typical OAuth 2.0 provider and
IndieAuth is that clients are public – there is no client secret. With
regular OAuth 2.0, you need to go to the provider, such as GitHub or
GitLab and register a client with a redirect URL and obtain a client
secret. With IndieAuth the client is identified by a URL and the client's
information is provided in the HTML.

## Step 1: User enters a profile URL, clicks Sign In, and is redirected

With IndieAuth, you sign in with your URL. It then fetches the web page
at the URL and gets the authorization endpoint from the header or the
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