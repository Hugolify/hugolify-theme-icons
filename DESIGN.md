# Hugolify Theme Icons: Design

The icon layer for modern Hugolify themes. This module is the **engine**: it
renders icons as SVG masks (`currentColor`), with no webfont and no
project-side build step. The icons themselves come from **set modules**, one
per icon set. Two are imported by default: **Lucide** for UI and content icons,
**Simple Icons** for brands. Others, such as Tabler, are added by a site.

It is **opt-in**: a project imports the module to get icons, and renders none if
it doesn't (see §7).

---

## 1. Goals

- **Modern rendering.** SVG masks, not a webfont: no FOUT/tofu, no Private Use
  Area glyphs, baseline-aligned, inherits `currentColor`.
- **Lightweight DOM.** The rendered output is a single `<i class="icon icon-NAME">`:
  no inline SVG, no inline style.
- **No project-side build step.** Shipping only the icons actually used (the
  "subset") is produced natively by Hugo: no Node, no `subset-font`.
- **Direct naming.** Content references the set's own names (`map-pin`). There
  is no translation layer.
- **Light by default.** A site downloads only the sets it uses: the engine holds
  no icon, and each set is a module of its own.

## 2. Naming: the set's names, directly

Content uses the names of the default set as-is: `{{ partial "icon" "map-pin" }}`
gives `icons/lucide/map-pin.svg`. There is **no mapping table**
(`data/icons.yaml` does not exist): the name *is* the filename.

Another set is reached with its name as a prefix: `tabler:ladder` gives
`icons/tabler/ladder.svg`. Brands use `brand:` (§6).

Content authored with names of another set (e.g. an older Bootstrap
`pin-map-fill`) is **not** bridged at runtime: there is no alias layer. Such
content is rewritten to the default set's names ahead of time, by the Hugolify
v2 content migration (§8).

## 3. Why mask, not webfont

Icon fonts are the legacy approach (accessibility issues, FOUT/tofu, PUA glyphs,
text-hinting blur, single color with none of the upsides). Lucide is SVG-native.
Every icon is rendered the same way:

```css
@layer components {
  .icon {
    /* default glyph = empty SVG, so an unmapped icon is invisible, not a solid square */
    --icon-glyph: url('data:image/svg+xml,%3Csvg/%3E');
    background-color: var(--icon-color, currentColor);
    display: inline-block;
    width: var(--icon-size, 1em);
    height: var(--icon-size, 1em);
    mask: var(--icon-glyph) center / contain no-repeat;
    vertical-align: -0.125em;
  }
}
```

Icons differ only by their `--icon-glyph` (an inline SVG data-URI). The SVG is an
**alpha mask**, so its own colors are irrelevant: the final color comes from
`background-color: var(--icon-color, currentColor)`. This base rule is not a
separate file: it is folded into the generated stylesheet (see §7).

Brand icons therefore render **monochrome**, which is the norm for social rows. If
colored logos are ever required, those specific icons would need inline SVG
instead of a mask.

## 4. Architecture

```text
content: map-pin, tabler:ladder, brand:github
        │
partial "icon"                                ← emits markup + registers "dir:name"
        │
templates.Defer (head/icons.html)             ← after full render: resolve → SVG
        │   resources.Get "icons/<dir>/<name>.svg" → data-URI
        ▼
assets/css/icons.css (subset, fingerprinted in prod) ← one cached file, only used icons
        +
<i class="icon icon-map-pin">                 ← lightweight DOM
```

### Engine and set modules

| Module | Mounts | Prefix | Imported |
| --- | --- | --- | --- |
| `hugolify-theme-icons` | partials only | | by the site |
| `hugolify-theme-icons-lucide` | `assets/icons/lucide/` | `lucide:` | by the engine |
| `hugolify-theme-icons-simple-icons` | `assets/icons/brands/` | `brand:` | by the engine |
| `hugolify-theme-icons-tabler` | `assets/icons/tabler/` | `tabler:` | by the site, if wanted |
| `hugolify-theme-icons-bootstrap` | `assets/icons/bootstrap/` | `bootstrap:` | by the site, if wanted |

The engine resolves `set:name` to `assets/icons/<set>/<name>.svg`, so a new set
needs **no engine change**: its module mounts its icons in its own folder.
`brand:` is the one exception, its folder is `brands`.

A set module:

- mounts its SVGs to `assets/icons/<set>/` and pins its upstream source;
- **never imports the engine**: the site imports the engine, and a set module
  that did would bring the default sets back past the site's `ignoreImports`;
- if it can serve as the default set, ships the names the Hugolify themes call
  directly and it lacks, as local SVG aliases (§7, theme vocabulary).

Hugo cannot condition an import on a param (an import only takes `path`,
`version`, `disable`, `ignoreConfig`, `ignoreImports`, `noMounts`, `noVendor`,
`usePackageJSON`). Each added set therefore costs one import line. The
alternative, an engine importing every set and a param choosing among them,
would make every site download every set: rejected.

### The default set: `params.icons.set`

`params.icons.set` (default `lucide`) names the set that draws the names written
without a prefix. It is a **rendering** setting: it also switches the icons the
themes call directly (`play`, `pause`, `contact-round`, `map-pin`, `solid:star`)
to that set. Importing a set module never changes it: the module is then a
complement, reached with its prefix, until the site sets the param.

```yaml
# config/_default/module.yaml: Tabler as a complement ("tabler:ladder")
imports:
  - path: github.com/hugolify/hugolify-theme-icons-tabler
```

```yaml
# config/_default/params.yaml: Tabler as the default set ("ladder", and the themes' play, pause...)
icons:
  set: tabler
```

Lucide and Simple Icons then stay in the module cache, with no effect on the
published site. A site that wants them gone imports the engine with
`ignoreImports: true`, and imports `hugolify-theme-icons-simple-icons` again if it
wants brands.

The default is repeated in `func/icons/GetDefaultSet.html`, not only in this
module's `hugo.yaml`, so a site with its own `params.icons` can never lose it.

### The site's own icons: `assets/icons/ui/`

A name of the default set is looked up in `assets/icons/ui/` first, then in the
set's folder. No module mounts `ui` (Lucide was mounted there until v1.7.0), so
that folder holds a site's own icons, whatever the default set, and a site that
had put SVGs there keeps the same rendering.

### Classes

The class mirrors the name, prefix included, except the default set's
(`func/icons/GetClass.html`, shared by the markup and the selector):

| Call | Class |
| --- | --- |
| `map-pin` | `icon-map-pin` |
| `solid:star` | `icon-solid-star` |
| `brand:github` | `icon-brand-github` |
| `tabler:ladder` | `icon-tabler-ladder` |
| `solid:tabler:star` | `icon-solid-tabler-star` |
| `lucide:map-pin` (Lucide is the default) | `icon-map-pin` |

One stylesheet serves the whole site, and two sets can share a name (726
names between Lucide and Tabler; `x` is Lucide's cross and the X logo): without
the prefix, both would get the glyph written last.

`brand-` and `solid-` start classes of their own, so a default-set name starting
with one of them keeps its set prefix: with Tabler as the default, its
`brand-github` is `icon-tabler-brand-github`, not the `icon-brand-github` of
Simple Icons. No Lucide name starts with either, and no Lucide name starts with
`tabler-`, so the classes of a Lucide site are exactly those of v1.6.0.

## 5. Module config: `hugo.yaml`

```yaml
module:
  mounts:
    - { source: layouts, target: layouts }
  imports:
    - path: github.com/hugolify/hugolify-theme-icons-lucide        # default set
    - path: github.com/hugolify/hugolify-theme-icons-simple-icons  # brands
params:
  icons:
    set: lucide
    strokeWidth: 1
```

A consuming project just imports the module:

```yaml
module:
  imports:
    - path: github.com/hugolify/hugolify-theme-icons
```

### Stroke weight

Lucide and Tabler ship every icon at `stroke-width="2"`, which reads heavy at UI
sizes. The module treats stroke weight as a design token and **overrides the
library default**: Hugolify icons are `1` unless a site says otherwise.

```yaml
# a consuming project's params.yaml: back to the libraries' own weight
icons:
  strokeWidth: 2
```

This is a deliberate break from upstream, so it is worth stating plainly: importing
this module changes how every outline icon looks compared to using the library
directly. The value `2` is still a first-class choice, and setting it skips the
rewrite entirely: it is what the source SVGs already contain.

This is deliberately **not** a CSS variable. Glyphs are painted as `mask` from a
base64-encoded SVG (§3), and a mask exposes only its alpha channel: no runtime
custom property can reach the `stroke-width` attribute inside it. The value is
therefore substituted into the SVG source before encoding, at build time.

Two consequences:

- The unit is **viewBox units**, not pixels. Lucide and Tabler icons are 24x24,
  so the rendered thickness is `strokeWidth / 24 * --icon-size`. At the default
  `--icon-size: 1em` (16px): `1` renders as 0.67px, `1.5` as 1px, `2` as 1.33px.
  A weight of `1` is genuinely hairline at 16px and depends on the renderer's
  antialiasing: `1.5` is the safer pick for icon-dense UIs.
- The weight is **site-wide**. A per-component variant would require emitting a
  second glyph set, doubling the stylesheet: not worth it for a design token that
  should stay consistent anyway.

The substitution is exhaustive and safe on outline sets: every Lucide icon (1743
of 1743 at the pinned commit, 2123 of 2123 on `main` in November 2026) and every
Tabler outline icon (5509 of 5509 at v3.53.0) carries exactly one
`stroke-width="2"`, on the root `<svg>`. It applies to
every folder but `brands`: Simple Icons are solid fills and carry no stroke, and
on any other set without a stroke it changes nothing.

Tabler opens each file with an XML comment of metadata (tags, category). The
engine strips comments before encoding: they draw nothing and would only weigh
on the stylesheet.

### Solid: `solid:`

Lucide is outline-only: no icon ships filled, and there is no `star-fill` to ask
for (that `-fill` suffix is another set's convention, see §2). A filled glyph is
still sometimes the right one: the stars of a rating, a toggled heart or
bookmark. A mask cannot fill an outline at runtime: it only lets colour through
what the SVG draws.

So filling is a build-time rendering of the outline icon, asked for with a
prefix, the way `brand:` is:

```go-html-template
{{ partial "icon" "solid:star" }}         → <i class="icon icon-solid-star">
{{ partial "icon" "solid:tabler:star" }}  → <i class="icon icon-solid-tabler-star">
```

The name stays the set's name. The engine turns the root `fill="none"` into
`fill="currentColor"`: every Lucide icon and every Tabler outline icon carries
exactly one, on the root `<svg>`, and `currentColor` is how one fills such an icon
inline. Every shape the stroke draws is filled in, and the stroke stays on top,
keeping the outline's weight and round joins. Like every other `currentColor`, it
then becomes opaque black for the mask (§3): the colour on screen is still the
`.icon` background, `--icon-color`. `icon-star` keeps the outline, so both
renderings of one icon can be used on the same page.

It suits silhouettes: closed shapes such as `star`, `heart`, `bookmark`,
`circle`. On an icon drawn with inner detail or open strokes, filling paints over
that detail; pick another icon rather than fill that one. `solid:` is ignored on
a brand, which is a solid fill already.

A set that draws its own filled icons, named `<name>-fill` (Bootstrap Icons has
670), serves those instead: `solid:star` takes `star-fill.svg` as drawn, with no
computed fill (since v1.8.0). The computed fill stays for the names a set has no
`-fill` drawing for, and for outline sets: neither Lucide nor Tabler has a single
name ending in `-fill`, so their rendering is unchanged.

## 6. Brands: resolved by context

A brand only appears in a known place: the social menu (`social.yml`). That
partial prefixes the name with `brand:` so it resolves against Simple Icons; the
author writes nothing special.

```go-html-template
{{/* social menu partial */}}
{{ partial "icon" (printf "brand:%s" (lower .title)) }}
```

This also disambiguates names that exist in both sets, 34 at the pinned commits
(`x`, `apple`, `rss`, `bluetooth`…):

- `apple` in content → `icons/lucide/apple.svg` (the Lucide apple), class `icon-apple`.
- `apple` in the social menu → `brand:apple` → `icons/brands/apple.svg` (the
  Apple logo), class `icon-brand-apple`.

Brand names must match Simple Icons slugs (e.g. use `x`, not `twitter`). There is
no alias layer; slug accuracy is a `social.yml` content convention.

## 7. Implementation

- `layouts/partials/icon.html`: splits `solid:`, then `set:name`, emits the
  markup and registers `"dir:name"` (plus `:solid`) in `hugo.Store`.
- `layouts/partials/head/icons.html`: the deferred pass. Resolves each
  registered name (the site's `icons/ui/` first for the default set), rewrites
  the stroke, fills `solid:`, strips comments, forces `currentColor` to `#000`
  for the mask, and writes one stylesheet.
- `layouts/partials/func/icons/GetDefaultSet.html`: `params.icons.set`, `lucide`
  when unset.
- `layouts/partials/func/icons/GetClass.html`: the class rule of §4, used by both
  partials so markup and selector always agree.
- `layouts/partials/func/icons/GetCatalog.html` (since v1.9.0): every icon the
  site can show, folder by folder, named as `partials/icon.html` resolves them:
  the folder's files and their names in the same order, with up to eight tags
  each when the set ships a `<name>.json` (Lucide). A set that keeps its tags in
  the comment each SVG opens with (Tabler) is flagged instead (`comments`):
  reading thousands of SVGs would double the cost, and a consumer that draws the
  icons reads them anyway. Each set is probed on its first icons, so a set
  without tags (brands, Bootstrap) isn't read at all. hugolify-admin builds its
  icon dialog on it, in Decap and Sveltia CMS: a JSON of names and tags, and
  each set's SVGs joined in one file it draws them from.

The catalog reads thousands of assets: call it after every page has rendered
(`templates.Defer`), as hugolify-admin does. Read while pages render, it
disturbed `hugo server`'s resource cache, and the base theme's `css/main.css`
(templated, then run through PostCSS under the same name) came back as its own
PostCSS output: `main.css:9819: unexpected "<" in command`. It costs about 0.5 s
for Lucide and brands, 1 s with Tabler and Bootstrap too; most of it is reading
files, one per Lucide icon for its tags.

The store is `hugo.Store`, not `site.Store`. Every language publishes the same
`assets/css/icons.css`, and `resources.FromString` caches by target path: whichever
language builds it first wins for all of them. A per-language store dropped the
icons used in one language only. The names are sorted so the parallel render
order of the languages cannot change the fingerprint.

A missing icon renders invisible (the empty default glyph) and logs a warning
that tells a set that isn't installed (`no icons/foo/ folder, import
github.com/hugolify/hugolify-theme-icons-foo`) from a name the set doesn't have.

### Theme vocabulary

The Hugolify themes call a few icons by name: `play` and `pause` (video toggle;
css-components also targets `.icon-play` and `.icon-pause`), `contact-round`
(vCard), `map-pin`, `solid:star` (rating) and `brand:github`. A set module that
can serve as the default set must resolve all of them. Lucide does natively;
`hugolify-theme-icons-tabler` ships `play`, `pause` and `contact-round` as local
aliases of its `player-play`, `player-pause` and `user-square-rounded`;
`hugolify-theme-icons-bootstrap` ships `map-pin` and `contact-round` (and their
`-fill`) as aliases of its `geo-alt` and `person-vcard`.

### Consuming-theme call sites

This module is the **sole** provider of `partials/icon.html`, and it is **opt-in**.
A consuming theme must **not** ship a default `icon` partial: a default would always
satisfy `templates.Exists`, defeating both the guard and the fallback below. So call
sites guard with `templates.Exists` and adapt when the module is absent.

Content icons (`key-features`, `alert`, `comparison`, `informations`):

```go-html-template
{{ with .icon }}{{ if templates.Exists "partials/icon.html" }}{{ partial "icon" . }}{{ end }}{{ end }}
```

Social menu (`nav/social.html`) falls back to the plain link (its **label**):

```go-html-template
{{ if templates.Exists "partials/icon.html" }}
  <a href="{{ .url | safeURL }}" title="{{ .title }}">
    {{ partial "icon" (printf "brand:%s" (lower .title)) }}<span class="visually-hidden">{{ .title }}</span>
  </a>
{{ else }}
  {{ partial "nav/link.html" (dict "link" . "span" true) }}
{{ end }}
```

The base theme's `<head>` guards the stylesheet the same way:
`{{ if templates.Exists "_partials/head/icons.html" }}{{ partial "head/icons.html" . }}{{ end }}`.

Resulting behaviour:

- **module present**: SVG mask; the icon is registered for the deferred CSS.
- **module absent**: content renders nothing; the social menu shows the text label
  (via the `nav/link` fallback).

Layout and icon sizing (`.icon { --icon-size }`) live in the consuming theme (e.g.
`hugolify-theme-design-system/assets/css/components/nav-social.css`); when no icon is
rendered the link simply shows its text label.

## 8. Migration

- **Content naming.** A project coming from another icon set carries non-Lucide
  `icon-*` names in its content. There is no runtime alias; the Hugolify v2 content
  migration rewrites those names to their Lucide equivalents ahead of time.
- **Brands.** `github, instagram, youtube, facebook` resolve directly; `twitter`
  becomes `x`. `vimeo`, `google`, `bluesky`, `mastodon` have no Lucide equivalent,
  which is exactly why brands go through Simple Icons. `linkedin` is the exception:
  Simple Icons removed it on legal request in 2022, so
  `hugolify-theme-icons-simple-icons` ships it as a local SVG.
- **Switching the default set** (e.g. Lucide to Tabler) is a content migration
  too: of the 127 Lucide names used in the content of the Hugolify sites in
  November 2026, 68 exist under the same name in Tabler (and a same name is not
  always the same drawing).

## 9. Versions and open items

- Requires Hugo ≥ **v0.128.0** (`templates.Defer`).
- **Versions are the site's.** The set modules require the oldest upstream commit
  the engine has used (Lucide `cf0e1e44baa6`, 2026-06-18; Simple Icons
  `c3777a7e48a5`, 2026-06-07). Go selects the highest version required, so a site
  keeps the Lucide it pinned and moves it on its own, with
  `hugo mod get github.com/lucide-icons/lucide@main` run in the site. Forcing a
  newer Lucide from here would break sites: on 2026-11-03 Lucide removed or
  renamed about sixty files (`circle-alert`, `triangle-alert`, `trash-2`, `smile`,
  `history`, `heading`…), and a site still on an earlier commit would lose those
  icons on update.
- **Lucide and Simple Icons tags are stale**: `lucide-icons/lucide` stopped tagging
  around v0.289.0 and `simple-icons` at v2.17.1, so `@latest` would regress. Pin
  `@main` / `@master` (pseudo-versions), never `@latest`; see each set module's
  README. Tabler tags its releases (`v3.53.0+incompatible`), so `-u` works there.
- **Tags of this module**: releases up to 1.6.0 were tagged without the `v` that
  Go requires, so sites pinned pseudo-versions. From v1.7.0 tags carry it.
- **Class collision, resolved** (v1.6.0): a UI `x` and a `brand:x` used anywhere
  on one site emitted the same `.icon-x` twice, and both showed the glyph written
  last. Brand classes are `icon-brand-*`, filled icons `icon-solid-*`, and other
  sets `icon-<set>-*` (§4).
- **Bootstrap Icons** (`hugolify-theme-icons-bootstrap`, `github.com/twbs/icons`
  v1.13.2, tagged) are solid icons drawn on a 16x16 grid: `strokeWidth` has no
  effect on them, and `solid:` uses their `-fill` drawings (§5). With
  `icons.set: bootstrap`, a site keeps content written with Bootstrap names (a v1
  site, before its content migration). Not for sites on hugolify-theme-bootstrap,
  which ships the Bootstrap Icons webfont and its own `.icon` class.
