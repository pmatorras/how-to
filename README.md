# Dev Docs

Internal documentation guides.

## Guides

### `vscode-tunel/`
Step-by-step guide for setting up VS Code Tunnels (avoiding TSPlus issues).

| File | Description |
|------|-------------|
| `es.html` | Spanish version |
| `en.html` | English version |
| `style.css` | Shared stylesheet (light/dark theme, layout) |
| `images/` | Screenshots shared by all language versions |

To add a new language: copy `en.html` to e.g. `fr.html`, translate the text and the JS strings (`"Dark Mode"` / `"Light Mode"` / `"Copied!"`), and update `<html lang="...">`. No images or CSS to duplicate.
