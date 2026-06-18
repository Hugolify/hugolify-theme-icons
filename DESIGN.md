# Hugolify Theme Icons — Design

The icon layer for modern Hugolify themes. It renders **Lucide** for UI/content
icons and **Simple Icons** for brands, as SVG masks (`currentColor`), with no
webfont and no project-side build step.

It is **opinionated**: Lucide is the only UI set. Projects that want Bootstrap
Icons stay on `hugolify-theme-bootstrap` (which keeps its native webfont system)
and do **not** import this module.

> Status: **design**. The SASS files currently under `assets/sass/` are the
> legacy webfont system copied from `hugolify-theme-bootstrap`, kept only as a
> reference. They are removed when the mask-based engine lands.

---

## 1. Goals

- **Modern rendering.** SVG masks, not a webfont: no FOUT/tofu, no Private Use
  Area glyphs, baseline-aligned, inherits `currentColor`.
- **Lightweight DOM.** The rendered output is a single `<i class="icon icon-NAME">`
  — no inline SVG, no inline style.
- **No project-side build step.** Shipping only the icons actually used (the
  "subset") is produced natively by Hugo — no Node, no `subset-font`.
- **Direct naming.** Content references Lucide names (`map-pin`). There is no
  translation layer.

## 2. Naming — Lucide, directly

Content uses Lucide names as-is. `{{ partial "icon" "map-pin" }}` →
`icons/ui/map-pin.svg`. There is **no mapping table** (`data/icons.yaml` does not
exist): the name *is* the filename.

Legacy content authored with Bootstrap Icons names (`pin-map-fill`) is **not**
bridged at runtime. Hugolify v2 rewrites that content to Lucide names
(`icon-map-pin`) as part of the migration — see the reference table in §8.

## 3. Why mask, not webfont

Icon fonts are the legacy approach (accessibility issues, FOUT/tofu, PUA glyphs,
text-hinting blur, single color with none of the upsides). Lucide is SVG-native.
Every icon is rendered the same way:

```css
@layer components {
  .icon {
    background-color: currentColor;
    display: inline-block;
    width: var(--icon-size, 1em);
    height: var(--icon-size, 1em);
    mask: var(--icon-glyph) center / contain no-repeat;
    vertical-align: -0.125em;
  }
}
```

Icons differ only by their `--icon-glyph` (an inline SVG data-URI). The SVG is an
**alpha mask**, so its own colors are irrelevant — the final color comes from
`background-color: currentColor`.

Brand icons therefore render **monochrome**, which is the norm for social rows. If
colored logos are ever required, those specific icons would need inline SVG
instead of a mask.

## 4. Architecture

```text
content: icon-map-pin                         ← Lucide name, directly
        │
partial "icon" "map-pin"                      ← emits markup + registers the name
        │
templates.Defer (in baseof)                   ← after full render: resolve → SVG
        │   resources.Get "icons/ui/map-pin.svg" → data-URI
        ▼
css/icons.css (fingerprinted, subset)         ← one cached file, only used icons
        +
<i class="icon icon-map-pin">                 ← lightweight DOM
```

Two sources, mounted by this module:

- `assets/icons/ui/` ← Lucide
- `assets/icons/brands/` ← Simple Icons

The UI mount target is the **role** (`ui`), not the library name, so swapping
Lucide for another set later (Phosphor, Heroicons…) is a one-line mount change
with no engine change. There is no runtime selection and no `params.icons.ui`.

## 5. Module config — `hugo.yaml`

```yaml
module:
  mounts:
    - { source: assets, target: assets }
    - { source: layouts, target: layouts }
  imports:
    - path: github.com/lucide-icons/lucide        # UI/content icons
      ignoreConfig: true
      mounts: [{ source: icons, target: assets/icons/ui }]
    - path: github.com/simple-icons/simple-icons  # brands
      ignoreConfig: true
      mounts: [{ source: icons, target: assets/icons/brands }]
```

A consuming project just imports the module:

```yaml
module:
  imports:
    - path: github.com/hugolify/hugolify-theme-icons
```

## 6. Brands — resolved by context

A brand only appears in a known place: the social menu (`social.yml`). That
partial prefixes the name with `brand:` so it resolves against Simple Icons; the
author writes nothing special.

```go-html-template
{{/* social menu partial */}}
{{ partial "icon" (printf "brand:%s" .title) }}
```

This also disambiguates names that exist in both sets:

- `apple` in content → `icons/ui/apple.svg` (the Lucide apple).
- `apple` in the social menu → `brand:apple` → `icons/brands/apple.svg` (the
  Apple logo).

Brand names must match Simple Icons slugs (e.g. use `x`, not `twitter`). There is
no alias layer; slug accuracy is a `social.yml` content convention.

## 7. Implementation

### Partial — `layouts/partials/icon.html`

Emits markup only and registers the name for the deferred pass.

```go-html-template
{{- $dir := "ui" }}{{ $name := . -}}
{{- if strings.HasPrefix . "brand:" -}}
  {{- $dir = "brands" }}{{ $name = strings.TrimPrefix "brand:" . -}}
{{- end -}}
{{- site.Store.Add "usedIcons" (slice (dict "dir" $dir "name" $name)) -}}
<i class="icon icon-{{ $name }}" aria-hidden="true"></i>
```

### Engine — `templates.Defer` in `baseof.html`

Runs after all pages render, so `site.Store` holds every icon used site-wide.
Produces one fingerprinted CSS file containing only those icons. No lookup table.

```go-html-template
{{ with (templates.Defer (dict "key" "iconset")) }}
  {{- $css := slice -}}
  {{- range site.Store.Get "usedIcons" | uniq -}}
    {{- $name := .name }}{{ $dir := .dir -}}
    {{- with resources.Get (printf "icons/%s/%s.svg" $dir $name) -}}
      {{- $clean := .Content | replaceRE `currentColor` `#000` | replaceRE `\s*\n\s*` `` -}}
      {{- $css = $css | append (printf ".icon-%s{--icon-glyph:url('data:image/svg+xml;base64,%s')}" $name ($clean | base64Encode)) -}}
    {{- else -}}
      {{- warnf "icon %q not found in %q" $name $dir -}}
    {{- end -}}
  {{- end -}}
  {{- $res := resources.FromString "css/icons.css" (printf "@layer components{%s}" (delimit $css "")) | minify | fingerprint -}}
  <link rel="stylesheet" href="{{ $res.RelPermalink }}" integrity="{{ $res.Data.Integrity }}">
{{ end }}
```

### Base CSS — `assets/css/base/icon.css`

See §3.

### Base-theme call sites

`hugolify-theme-icons` is the **sole** provider of `partials/icon.html`, and it is
**opt-in**. There is intentionally **no default `icon` partial in the base theme**:
a default would always satisfy `templates.Exists` (blocking the fallback below) and
would double-render Bootstrap's social icons — its `.nav-social .github a::before`
glyph *plus* the `<i>`. So call sites guard with `templates.Exists` and adapt when
the module is absent.

Content icons (`key-features`, `alert`, `comparison`, `informations`):

```go-html-template
{{ if templates.Exists "partials/icon.html" }}{{ partial "icon" . }}{{ else }}<i class="icon icon-{{ . }}"></i>{{ end }}
```

Social menu (`nav/social.html`) — the fallback shows the **label** and keeps the
markup Bootstrap's `_nav-social.sass` expects:

```go-html-template
{{ if templates.Exists "partials/icon.html" }}
  <a href="{{ .url | safeURL }}" title="{{ .title }}">
    {{ partial "icon" (printf "brand:%s" (lower .title)) }}<span class="visually-hidden">{{ .title }}</span>
  </a>
{{ else }}
  {{ partial "nav/link.html" (dict "link" . "span" true) }}
{{ end }}
```

Resulting behaviour:

- **theme-icons present** → SVG mask; the icon is registered for the deferred CSS.
- **Bootstrap** → the `else` markup; its webfont renders `.icon-NAME` (content) or the
  `.nav-social … ::before` glyph (social). **That theme is untouched.**
- **Neither** → content shows an empty `<i>`; the social menu shows the text label.

CSS adapts to the same condition: layout and icon sizing (`.icon { --icon-size }`)
live in the consuming theme (e.g.
`hugolify-theme-design-system/assets/css/components/nav-social.css`); when no icon is
rendered the link simply shows its text label. Never the glyphs.

## 8. Migration & legacy

- The webfont SASS under `assets/sass/` (bootstrap-icons, icomoon, material-icons)
  is reference-only and is removed once the engine lands.
- Bootstrap Icons stays in `hugolify-theme-bootstrap`; this module is not used
  there.
- **Hugolify v2 content migration**: rewrite `icon-*` Bootstrap names to Lucide
  names. The table below is the rename reference (grounded on the 179 names
  actually used across the ecosystem, verified against `lucide-static@1.21.0`).

### Identical (no rename)

> archive, arrow-up-circle, bar-chart, binoculars, book, bookmark,
> bookmark-check, box, braces, briefcase, building, calendar-heart,
> calendar-range, check, check-circle, clock, cloud, cloud-rain, cloud-sun,
> code, cookie, eye, files, fingerprint, globe, grip-horizontal, heart,
> heart-pulse, hospital, hourglass, house, image, images, kanban, key, laptop,
> layers, link, list, map, megaphone, newspaper, palette, percent, phone,
> pie-chart, puzzle, qr-code, search, shield-check, sliders, sun, tag, tags,
> terminal, truck, wallet

### Rename (Bootstrap → Lucide)

| Bootstrap | Lucide | Bootstrap | Lucide |
| --- | --- | --- | --- |
| airplane | plane | hand-thumbs-up | thumbs-up |
| app, window-fullscreen | app-window | hdd-network | network |
| arrows-fullscreen | maximize | house-add | house-plus |
| blockquote-left | text-quote | houses | house |
| body-text | text | info-circle | info |
| box-arrow-up-right | external-link | input-cursor | text-cursor |
| braces-asterisk | braces | input-cursor-text | text-cursor-input |
| browser-chrome | chrome | journal-richtext | notebook-text |
| browser-safari | compass | language, translate | languages |
| buildings | building-2 | layout-three-columns | columns-3 |
| calendar-date | calendar | link-45deg | link |
| calendar-event | calendar-days | list-nested, tree | list-tree |
| camera-reels | clapperboard | list-ol | list-ordered |
| camera-video, videocam | video | mortarboard | graduation-cap |
| card-heading | heading | paragraph | pilcrow |
| card-image | image | text-paragraph | align-left |
| caret-down | chevron-down | patch-check | badge-check |
| chat | message-circle | patch-question | badge-help |
| chat-square-quote | message-square-quote | pc-display-horizontal, display | monitor |
| check-all | check-check | pencil-square | square-pen |
| check2-circle | check-circle | people | users |
| clipboard2-pulse | clipboard-plus | person | user |
| clock-history | history | person-bounding-box | square-user |
| cloud-arrow-up | cloud-upload | person-hearts | heart-handshake |
| clouds | cloudy | person-plus | user-plus |
| code-slash | code-xml | person-workspace | presentation |
| collection | library | pin-map-fill, geo-alt | map-pin |
| credit-card-2-front | credit-card | postcard, envelope | mail |
| currency-euro | euro | envelope-at | at-sign |
| database-add, database-check | database | puzzle-fill | puzzle |
| emoji-smile | smile | search-heart | search |
| exclamation-diamond, warning | triangle-alert | shield-slash | shield-off |
| exclamation-octagon | octagon-alert | shop | store |
| eyeglasses | glasses | soundwave | audio-waveform |
| file-earmark | file | speedometer, speedometer2 | gauge |
| file-earmark-code, filetype-* | file-code | telephone | phone |
| file-earmark-image | file-image | tools | wrench |
| file-earmark-richtext, -text | file-text | type-h2 | heading-2 |
| file-font-fill | file-type | ui-checks | list-checks |
| file-zip | file-archive | universal-access-circle | accessibility |
| gear | settings | globe-europe-africa | globe |
| git | git-branch | grid-3x2-gap | layout-grid |
| graph-down-arrow | trending-down | hand-index | pointer |
| graph-up-arrow | trending-up | &nbsp; | &nbsp; |

### Brands → Simple Icons

`github, linkedin, instagram, youtube, facebook` resolve directly; `twitter` → `x`.
`vimeo`, `google`, `bluesky`, `bootstrap` have no Lucide equivalent — which is
exactly why brands go through Simple Icons.

## 9. Status & open items

The engine is in place: `hugo.yaml`, `layouts/partials/icon.html`,
`layouts/partials/head/icons.html`. The base `.icon` rule is folded into the
generated stylesheet, so no separate CSS asset is needed.

- `templates.Defer` — available since Hugo **v0.128.0**; confirmed (dev machine
  runs v0.158). The module therefore requires Hugo ≥ 0.128.0.
- **Not yet validated by a build.** Run `hugo mod get` in a consuming project and
  confirm the upstream module imports resolve and their SVGs live at
  `icons/*.svg` for `lucide-icons/lucide` and `simple-icons/simple-icons`.
- **Legacy SASS** under `assets/sass/` is now orphaned (its `twbs/icons` import was
  removed from `hugo.yaml`) and is no longer mounted. Safe to delete — identical
  files remain in `hugolify-theme-bootstrap`.
- **Class collision:** a page using both a UI `github` and a `brand:github` emits
  the same `.icon-github` twice with different glyphs. Namespace brand classes
  (e.g. `.icon-brand-github`) if this case is real.
