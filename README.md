# Hugolify theme icons

The icon layer for modern Hugolify themes — **Lucide** for UI/content icons and
**Simple Icons** for brands.

## Overview

Icons render as SVG masks (`currentColor`), not a webfont — no FOUT/tofu, no
Private Use Area glyphs. Content references Lucide names directly, and the build
ships only the icons actually used (the subset is produced natively by Hugo, with
no Node step).

It is opinionated: Lucide is the only UI set. Projects that want Bootstrap Icons
stay on `hugolify-theme-bootstrap` and do not import this module.

## Architecture

See [DESIGN.md](DESIGN.md) for the full design: mask rendering, the Hugo-native
build (`templates.Defer`), brand handling, and the Bootstrap → Lucide migration
reference.

Requires Hugo ≥ **v0.128.0** (`templates.Defer`).

> The SASS under `assets/sass/` is the legacy webfont system, kept as reference
> until the mask-based engine lands.

## Documentations

- [Lucide icons](https://lucide.dev/icons/)
- [Simple icons](https://simpleicons.org/)
- [Hugolify](https://www.hugolify.io/docs/)

## Licensing

Hugolify is free for personal or commercial projects (MIT license)
