# Xzibit App Standard

Design tokens, component stylesheet, and rules for every Xzibit app.

## Files

- `xzibit-design.css` — the single stylesheet every Xzibit app imports. Ships Inter via Google Fonts, all tokens as CSS variables (`--xz-*`), and every component class.
- `tokens.json` — design tokens in machine-readable form. Use for Tailwind configs, React themes, or design-tool imports.
- `standards.html` — the published standards site with live component previews and copy-paste HTML snippets.
- `CLAUDE.md` — rules for Claude Code, Manus, and other AI coding tools. Drop a copy into each Xzibit app repo.

## Live reference

Deploy this repo to Vercel (or another static host). Once live, every Xzibit app imports the stylesheet with a single line:

```html
<link rel="stylesheet" href="https://<YOUR-DEPLOYED-URL>/xzibit-design.css">
```

## Version

`1.0.0` — first public standard. Light-canvas, ConnectTeam-leaning interpretation of Brand System v2.0.
