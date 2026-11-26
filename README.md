# Hugolify theme icons

<img width="1280" height="640" alt="share-hugolify-theme-icons" src="https://github.com/user-attachments/assets/12f905da-ce85-4b75-8c1d-b538124aa877" />

The icon layer for modern Hugolify themes: **Lucide** for UI and content icons,
**Simple Icons** for brands, and any other set a site adds, such as **Tabler**.

## Overview

Icons render as SVG masks (`currentColor`), not a webfont: no FOUT/tofu, no
Private Use Area glyphs. Content references the set's names directly, and the
build ships only the icons actually used (the subset is produced natively by
Hugo, with no Node step).

This module is the engine. Each icon set is a module of its own, so a site only
downloads the sets it uses:

| Module | Set | Usage |
| --- | --- | --- |
| [hugolify-theme-icons-lucide](https://github.com/hugolify/hugolify-theme-icons-lucide) | [Lucide](https://lucide.dev/icons/) | `map-pin`, imported by default |
| [hugolify-theme-icons-simple-icons](https://github.com/hugolify/hugolify-theme-icons-simple-icons) | [Simple Icons](https://simpleicons.org/) | `brand:github`, imported by default |
| [hugolify-theme-icons-tabler](https://github.com/hugolify/hugolify-theme-icons-tabler) | [Tabler](https://tabler.io/icons) | `tabler:ladder`, imported by the site |
| [hugolify-theme-icons-bootstrap](https://github.com/hugolify/hugolify-theme-icons-bootstrap) | [Bootstrap Icons](https://icons.getbootstrap.com/) | `bootstrap:geo-alt`, imported by the site |

It is opt-in: a project imports the module to get icons, and renders none if it
doesn't.

## Install

```yaml
# config/_default/module.yaml
imports:
  - path: github.com/hugolify/hugolify-theme-icons
```

## Usage

```go-html-template
{{ partial "icon" "map-pin" }}            default set (Lucide)
{{ partial "icon" "solid:star" }}         filled in
{{ partial "icon" "brand:github" }}       brand (Simple Icons)
{{ partial "icon" "tabler:ladder" }}      another set, once its module is imported
```

In content, the `icon` shortcode and the blocks' `icon` fields take the same
names: `{{< icon icon="tabler:trowel" >}}`.

## Configuration

### Default set

Importing another set's module adds it as a complement, reached with its prefix.
To make it draw the names written without a prefix, including the icons the
themes call themselves (video play and pause, vCard, rating stars), set it as
the default:

```yaml
# params.yaml
icons:
  set: tabler   # default: lucide
```

Lucide and Simple Icons then stay in the module cache, with no effect on the
published site. To skip downloading them, import this module with
`ignoreImports: true`, and import `github.com/hugolify/hugolify-theme-icons-simple-icons`
yourself if the site shows brands.

A site's own icons go in `assets/icons/ui/`: a name without a prefix is looked up
there first.

### Stroke weight

Stroke weight is a design token. Lucide's and Tabler's native weight is `2`,
which reads heavy at UI sizes, so **this module defaults to `1`**: importing it
changes how icons look compared to using the libraries directly. Override it per
site:

```yaml
# params.yaml
icons:
  strokeWidth: 1.5   # 2 restores the libraries' own weight
```

The unit is viewBox units (icons are 24x24), so the rendered thickness is
`strokeWidth / 24 * --icon-size`: at the default 1em/16px, `1` gives 0.67px and
`1.5` gives 1px. It is a build-time param rather than a CSS variable because masks
are opaque to runtime CSS; see [DESIGN.md](DESIGN.md) §5.

## Admin

With [hugolify-admin](https://github.com/hugolify/hugolify-admin), the icon
fields of Decap and Sveltia CMS show the chosen icon and open a grid of the
icons the site can show, searchable by name or keyword. The list comes from
`func/icons/GetCatalog.html` (since v1.9.0); see [DESIGN.md](DESIGN.md) §7.

## Architecture

See [DESIGN.md](DESIGN.md) for the full design: mask rendering, the Hugo-native
build (`templates.Defer`), set modules and classes, brand handling, the icon
catalog, and content-naming migration.

Requires Hugo ≥ **v0.128.0** (`templates.Defer`).

## Documentations

- [Lucide icons](https://lucide.dev/icons/)
- [Simple icons](https://simpleicons.org/)
- [Tabler icons](https://tabler.io/icons)
- [Bootstrap icons](https://icons.getbootstrap.com/)
- [Hugolify](https://www.hugolify.io/docs/)

## Licensing

Hugolify is free for personal or commercial projects (MIT license)
