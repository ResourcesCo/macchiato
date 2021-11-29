# Quasar with Dark Mode Template

This template is for creating a Vue app with Quasar and Dark Mode
support.

##### `src/App.vue`

This uses VueUse to get whether the dark system color scheme is selected, and
sets it immediately in Quasar's isDark plugin. There is another VueUse
function that supports toggling the user preference and setting it in
Local Storage.

```html
<script setup>
import { watchEffect } from 'vue'
import { usePreferredDark } from '@vueuse/core'
import { useQuasar } from 'quasar'
import HelloWorld from './components/HelloWorld.vue'

const $q = useQuasar()
const isDark = usePreferredDark()
watchEffect(() => $q.dark.set(isDark.value))
</script>

<template>
  <div>
    <HelloWorld />
  </div>
</template>
```

##### `src/components/HelloWorld.vue`

This is just an example.

```html
<script setup>
import { ref } from 'vue'

defineProps({
  msg: String
})

const count = ref(0)
</script>

<template>
  <h1>Count</h1>

  <button type="button" @click="count++">Count is: {{ count }}</button>
</template>

<style scoped>
a {
  color: #42b983;
}
</style>
```

##### `src/main.js`

This creates the Vue app and sets up Quasar.

```js
import { createApp } from 'vue'
import { Quasar } from 'quasar'

// Import icon libraries
import '@quasar/extras/material-icons/material-icons.css'

// Import Quasar css
import 'quasar/src/css/index.sass'

// Assumes your root component is App.vue
// and placed in same folder as main.js
import App from './App.vue'

const myApp = createApp(App)

myApp.use(Quasar, {
  plugins: {}, // import Quasar plugins and add here
})

// Assumes you have a <div id="app"></div> in your index.html
myApp.mount('#app')
```

##### `src/quasar-variables.scss`

See https://quasar.dev/style/theme-builder

```css
$primary   : #1976D2;
$secondary : #26A69A;
$accent    : #9C27B0;

$dark      : #1D1D1D;

$positive  : #21BA45;
$negative  : #C10015;
$info      : #31CCEC;
$warning   : #F2C037;
```

##### `index.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Hello, Quasar</title>
  </head>
  <body>
    <div id="app" class="dark"></div>
    <script type="module" src="/src/main.js"></script>
  </body>
</html>
```

##### `package.json`

```json
{
  "name": "vite-project",
  "version": "0.0.0",
  "scripts": {
    "dev": "vite --port 3000 --host 0.0.0.0",
    "build": "vite build"
  },
  "dependencies": {
    "@quasar/extras": "^1.12.1",
    "@vueuse/core": "^7.0.1",
    "vue": "^3.2.16"
  },
  "devDependencies": {
    "@quasar/vite-plugin": "^1.0.2",
    "@vitejs/plugin-vue": "^1.9.3",
    "quasar": "^2.3.3",
    "sass": "1.32.0",
    "vite": "^2.6.4"
  }
}
```

##### `vite.config.js`

This has the `server.hmr` settings for running on a remote server
which need to be changed to the host.

```js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { quasar, transformAssetUrls } from '@quasar/vite-plugin'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [
    vue({
      template: { transformAssetUrls }
    }),
    quasar({
      sassVariables: 'src/quasar-variables.scss',
    }),
  ],
  server: {
    hmr: {
      host: "vite-project.sandbox.example.com",
      port: 443,
      protocol: 'wss',
    },
  },
})
```
