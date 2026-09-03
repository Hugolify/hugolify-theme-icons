# Hugolify theme icons

<img width="1280" height="640" alt="share-hugolify-theme-icons" src="https://github.com/user-attachments/assets/12f905da-ce85-4b75-8c1d-b538124aa877" />

The icon layer for modern Hugolify themes — **Lucide** for UI/content icons and
**Simple Icons** for brands.

## Overview

Icons render as SVG masks (`currentColor`), not a webfont — no FOUT/tofu, no
Private Use Area glyphs. Content references Lucide names directly, and the build
ships only the icons actually used (the subset is produced natively by Hugo, with
no Node step).

It is opinionated: Lucide is the only UI set. It is also opt-in — a project imports
the module to get icons, and renders none if it doesn't.

## Configuration

Stroke weight is a design token. Lucide's native weight is `2`, which reads heavy
at UI sizes, so **this module defaults to `1`** — importing it changes how icons
look compared to using Lucide directly. Override it per site:

```yaml
# params.yaml
icons:
  strokeWidth: 1.5   # 2 restores Lucide's own weight
```

The unit is viewBox units (icons are 24x24), so the rendered thickness is
`strokeWidth / 24 * --icon-size` — at the default 1em/16px, `1` gives 0.67px and
`1.5` gives 1px. It is a build-time param rather than a CSS variable because masks
are opaque to runtime CSS; see [DESIGN.md](DESIGN.md) §5.

## Architecture

See [DESIGN.md](DESIGN.md) for the full design: mask rendering, the Hugo-native
build (`templates.Defer`), brand handling, and content-naming migration.

Requires Hugo ≥ **v0.128.0** (`templates.Defer`).

## Documentations

- [Lucide icons](https://lucide.dev/icons/)
- [Simple icons](https://simpleicons.org/)
- [Hugolify](https://www.hugolify.io/docs/)

## Licensing

Hugolify is free for personal or commercial projects (MIT license)
