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
body {
  background-color: #ccc;
}
```

##### `popup.html`

```html
<!doctype html>
<html>
  <head>
    <link rel="stylesheet" href="style.css">
  </head>
  <body>
    <div>Projects on left</div>
    <div>Tabs on top - pinned home tab and open documents</div>
    <div>Status bar on bottom (90s-style hover tips show at bottom)</div>
  </body>
</html>
```
