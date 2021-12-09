# Hello, Vite

This is an example of a minimal Vite app.

The `vite.config.js` included in the [vue][vue] template for Vite is included
for the vue plugin. If you don't need any plugins, it isn't required, and
it isn't included in the [vanilla][vanilla] plugin.

The `scripts` in `package.json` can also be left out and vite can be run
directly with `npm run vite`.

##### `package.json`

```json
{
  "name": "hello-vite",
  "version": "0.0.1",
  "dependencies": {
    "vite": "*"
  }
}
```

The HTML file loads when you go to the root of the site. It has a script tag
to include the `main.js` file. Vite will transform and bundle this file.

##### `index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <title>Hello, Vite.</title>
    <script src="/main.js" type="module"></script>
  </head>
  <body>
    <h1>Hello, Vite.</h1>
  </body>
</html>
```

##### `main.js`

This imports css, which is not built into browsers, but is supported by
vite.

```js
import "./style.css";

window.addEventListener('DOMContentLoaded', () => {
  setTimeout(() => {
    document.querySelector("h1").classList.add('red');
  }, 3000)
});
```

##### `style.css`

This gets added to the `<h1>` tag after three seconds, which shows that
the js and css successfully loaded.

```css
.red { color: red; }
```

To run this:

```sh
npm install
npx vite
```

You use pnpm with `pnpm install` and `pnpx vite`, or use `yarn` with
`yarn install` and `yarn vite`.

Now, go to [http://localhost:3000/](http://localhost:3000/) and you
should see "Hello, Vite!" and after three seconds it should change
to red.

[vue]: https://github.com/vitejs/vite/tree/main/packages/create-vite/template-vue
[vanilla]: https://github.com/vitejs/vite/tree/main/packages/create-vite/template-vanilla