# Studio

This is a *studio*, initially for editing Markdown files.

Design draft: make tabs
[look like this](https://quasar.dev/vue-components/tabs#dynamic-update), just
icons and more compact, though. Show a dropdown menu on the selected tab, that
can be used to view the name of the tab, close the tab, and navigate within
the tab (perhaps require clicking an icon and just show a tiny popup at first).
Single click to switch between tabs - it's just the active tab where you can
see the menu. Maybe a triple dot at the end of the tabs. On the left side have
a triple line menu to show a slide out menu. The slide out menu will have a list
of documents, and allow opening them. They will open in a tab.

##### `src/App.vue`

At left, icon 

```html
<script setup>
import { ref } from 'vue'
import { useQuasar } from 'quasar'
import HelloWorld from './components/HelloWorld.vue'
const $q = useQuasar()
$q.dark.set(true)

const tab = ref('mails')
const splitterModel = ref(80)
</script>

<template>
  <div>
    <q-splitter
      v-model="splitterModel"
      style="height: 250px"
    >
      <template v-slot:before>
        <q-tabs
          v-model="tab"
          vertical
          class="text-teal"
        >
          <q-tab name="mails" icon="mail" label="Mails" />
          <q-tab name="alarms" icon="alarm" label="Alarms" />
          <q-tab name="movies" icon="movie" label="Movies" />
        </q-tabs>
      </template>

      <template v-slot:after>
        <q-tab-panels
          v-model="tab"
          animated
          swipeable
          vertical
          transition-prev="jump-up"
          transition-next="jump-up"
        >
          <q-tab-panel name="mails">
            <div class="text-h4 q-mb-md">Mails</div>
            <p>Lorem ipsum dolor sit, amet consectetur adipisicing elit. Quis praesentium cumque magnam odio iure quidem, quod illum numquam possimus obcaecati commodi minima assumenda consectetur culpa fuga nulla ullam. In, libero.</p>
            <p>Lorem ipsum dolor sit, amet consectetur adipisicing elit. Quis praesentium cumque magnam odio iure quidem, quod illum numquam possimus obcaecati commodi minima assumenda consectetur culpa fuga nulla ullam. In, libero.</p>
          </q-tab-panel>

          <q-tab-panel name="alarms">
            <div class="text-h4 q-mb-md">Alarms</div>
            <p>Lorem ipsum dolor sit, amet consectetur adipisicing elit. Quis praesentium cumque magnam odio iure quidem, quod illum numquam possimus obcaecati commodi minima assumenda consectetur culpa fuga nulla ullam. In, libero.</p>
            <p>Lorem ipsum dolor sit, amet consectetur adipisicing elit. Quis praesentium cumque magnam odio iure quidem, quod illum numquam possimus obcaecati commodi minima assumenda consectetur culpa fuga nulla ullam. In, libero.</p>
          </q-tab-panel>

          <q-tab-panel name="movies">
            <div class="text-h4 q-mb-md">Movies</div>
            <p>Lorem ipsum dolor sit, amet consectetur adipisicing elit. Quis praesentium cumque magnam odio iure quidem, quod illum numquam possimus obcaecati commodi minima assumenda consectetur culpa fuga nulla ullam. In, libero.</p>
            <p>Lorem ipsum dolor sit, amet consectetur adipisicing elit. Quis praesentium cumque magnam odio iure quidem, quod illum numquam possimus obcaecati commodi minima assumenda consectetur culpa fuga nulla ullam. In, libero.</p>
            <p>Lorem ipsum dolor sit, amet consectetur adipisicing elit. Quis praesentium cumque magnam odio iure quidem, quod illum numquam possimus obcaecati commodi minima assumenda consectetur culpa fuga nulla ullam. In, libero.</p>
          </q-tab-panel>
        </q-tab-panels>
      </template>

    </q-splitter>
  </div>
</template>
```

##### `src/components/HelloWorld.vue`

```html
<script setup>
import { ref } from 'vue'

defineProps({
  msg: String
})

const count = ref(0)
</script>

<template>
  <h1>ShibaSlap</h1>

  <button type="button" @click="count++">Slap count is: {{ count }}</button>
</template>

<style scoped>
a {
  color: #42b983;
}
</style>
```

##### `src/main.js`

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
    <title>Macchiato Studio</title>
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
  "name": "macchiato-studio",
  "version": "0.0.0",
  "scripts": {
    "dev": "vite --port 3000 --host 0.0.0.0",
    "build": "vite build"
  },
  "dependencies": {
    "@quasar/extras": "^1.12.1",
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
