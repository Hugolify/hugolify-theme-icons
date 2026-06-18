# Hugolify theme icons

The icon layer for modern Hugolify themes — **Lucide** for UI/content icons and
**Simple Icons** for brands.

## Overview

Icons render as SVG masks (`currentColor`), not a webfont — no FOUT/tofu, no
Private Use Area glyphs. Content references Lucide names directly, and the build
ships only the icons actually used (the subset is produced natively by Hugo, with
no Node step).

It is opinionated: Lucide is the only UI set. It is also opt-in — a project imports
the module to get icons, and renders none if it doesn't.

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
