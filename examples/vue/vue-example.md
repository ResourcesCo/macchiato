# 

## Running

This is a tiny Deno app that needs `--allow-run` permission. Anything with
the allow-run permission should be carefully examined.

It runs Docker with some environment variables set.

The CLI user interface is in `cli.ts` and is run as a worker with no
permissions, so it doesn't need to be as carefully examined.

##### `run.ts`

```ts

```

##### `Dockerfile`

```docker
FROM node
```

## App

### Core

##### `app/App.vue`

`<template />`

```html
<img alt="Vue logo" src="./assets/logo.png" />
<HelloWorld msg="Hello Vue 3 + TypeScript + Vite" />
```

`<script lang="ts" />`

```ts
import { defineComponent } from 'vue'
import HelloWorld from './components/HelloWorld.vue'
export default defineComponent({
  name: 'App',
  components: {
    HelloWorld
  }
})
```

`<style />`

```css
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
```

### Setup

##### `app/index.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" href="/favicon.ico" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite App</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

##### `src/main.ts`

```ts
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#app')
```

##### `shims-vue.d.ts`

```ts
declare module '*.vue' {
  import { DefineComponent } from 'vue'
  // eslint-disable-next-line @typescript-eslint/no-explicit-any, @typescript-eslint/ban-types
  const component: DefineComponent<{}, {}, any>
  export default component
}
```

##### `vite-env.d.ts`

```ts
/// <reference types="vite/client" />
```

### Config

##### `app/package.json`

```json
{
  "name": "vite-vue-typescript-starter",
  "version": "0.0.0",
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "serve": "vite preview"
  },
  "dependencies": {
    "vue": "^3.0.5"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^1.3.0",
    "@vue/compiler-sfc": "^3.0.5",
    "typescript": "^4.3.2",
    "vite": "^2.4.4",
    "vue-tsc": "^0.2.2"
  }
}
```

##### `app/vite.config.ts`

```ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [vue()]
})
```

##### `app/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "node",
    "strict": true,
    "jsx": "preserve",
    "sourceMap": true,
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "lib": ["esnext", "dom"]
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts", "src/**/*.tsx", "src/**/*.vue"]
}
```

##### `app/.gitignore`

```
node_modules
.DS_Store
dist
dist-ssr
*.local
```