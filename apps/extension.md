# Extension

This is a browser extension that shows a keyboard-friendly dashboard upon
pressing Command-Shift-P (same as Visual Studio Code).

##### `manifest.json`

```json
{
  "name": "Macchiato",
  "description": "Markdown document tool",
  "version": "1.0",
  "manifest_version": 3,
  "background": {
    "service_worker": "background.js"
  },
  "permissions": ["storage"],
  "action": {
    "default_popup": "popup.html"
  },
  "commands": {
    "_execute_action": {
      "suggested_key": {
        "default": "Ctrl+Shift+P",
        "mac": "Command+Shift+P"
      },
      "description": "Open Macchiato"
    }
  }
}
```

##### `background.js`

```js
```

##### `style.css`

```css
html {
  background-color: #ccc;
  width: 360px;
  height: 480px;
}

body {
  height: 100%;
}

input, textarea {
  background-color: #ddd;
}

* {
  box-sizing: border-box;
}

div, body {
  padding: 0;
  margin: 0;
}

.wrap {
  display: flex;
  flex-direction: columns;
  border: 1px solid #333;
  width: 100%;
  height: 100%;
}

@media (prefers-color-scheme: dark) {
  html {
    background-color: #171720;
    color: #f7f7f7;
  }
  input, textarea {
    background-color: #202027;
    color: #eee;
  }
  .wrap {
    border: 2px solid #333;
  }
}

.project-tabs {
  display: flex;
}

.project-tab, .project-tab:active, .project-tab:hover {
  outline: none;
  background: transparent;
}

.project-tab {
  display: block;
  margin: 2px;
  border: 2px solid #aaa0;
  border-radius: 5px;
  padding: 5px;
  text-align: center;
  width: 32px;
  height: 32px;
  font-size: 18px;
  line-height: 1;
}

.project-tab:hover {
  border: 2px solid #aaa;
}

.project-tab.active, .project-tab.active:hover {
  border-color: #88a;
}

@media (prefers-color-scheme: dark) {
  .project-tab {
    border: 2px solid #3330;
  }

  .project-tab:hover {
    border: 2px solid #333;
  }

  .project-tab.active {
    border-color: #559;
  }
}

.content {
  display: flex;
  flex-grow: 1;
  flex-direction: column;
}
```

##### `app.js`

```js
```

##### `popup.html`

```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8">
    <link rel="stylesheet" href="style.css">
    <script type="module" src="app.js"></script>
  </head>
  <body>
    <div class="wrap">
      <div class="projects">
        <button class="project-tab active" data-title="search">🔍</button>
        <button class="project-tab" data-title="new note">📝</button>
        <button class="project-tab" data-title="about">ℹ️</button>
      </div>
      <div class="content">
        <input type="text">
      </div>
    </div>
  </body>
</html>
```
