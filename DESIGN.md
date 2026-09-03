# Hugolify Theme Icons — Design

The icon layer for modern Hugolify themes. It renders **Lucide** for UI/content
icons and **Simple Icons** for brands, as SVG masks (`currentColor`), with no
webfont and no project-side build step.

It is **opinionated**: Lucide is the only UI set. It is also **opt-in** — a project
imports the module to get icons, and renders none if it doesn't (see §7).

> Status: **implemented, not yet validated by a build** (see §9). The mask-based
> engine — `hugo.yaml`, `layouts/partials/icon.html`,
> `layouts/partials/head/icons.html` — is in place.

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

Content authored with non-Lucide names (e.g. an older `pin-map-fill`) is **not**
bridged at runtime — there is no alias layer. Such content is rewritten to Lucide
names (`icon-map-pin`) ahead of time, by the Hugolify v2 content migration (§8).

## 3. Why mask, not webfont

Icon fonts are the legacy approach (accessibility issues, FOUT/tofu, PUA glyphs,
text-hinting blur, single color with none of the upsides). Lucide is SVG-native.
Every icon is rendered the same way:

```css
@layer components {
  .icon {
    /* default glyph = empty SVG → an unmapped icon is invisible, not a solid square */
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
**alpha mask**, so its own colors are irrelevant — the final color comes from
`background-color: var(--icon-color, currentColor)`. This base rule is not a
separate file: it is folded into the generated stylesheet (see §7).

Brand icons therefore render **monochrome**, which is the norm for social rows. If
colored logos are ever required, those specific icons would need inline SVG
instead of a mask.

## 4. Architecture

```text
content: icon-map-pin                         ← Lucide name, directly
        │
partial "icon" "map-pin"                      ← emits markup + registers the name
        │
templates.Defer (head/icons.html)             ← after full render: resolve → SVG
        │   resources.Get "icons/ui/map-pin.svg" → data-URI
        ▼
css/icons.css (subset, fingerprinted in prod) ← one cached file, only used icons
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
    - { source: layouts, target: layouts }
    # local assets win over the imported icon modules below → curated brands
    # (e.g. linkedin, dropped by both upstreams) fill the gaps. See §6 and §9.
    - { source: assets, target: assets }
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

### Stroke weight

Lucide ships every icon at `stroke-width="2"`, which reads heavy at UI sizes. The
module treats stroke weight as a design token and **overrides the library default**:
Hugolify icons are `1` unless a site says otherwise.

```yaml
# hugo.yaml (this module) — opinionated default, not Lucide's
params:
  icons:
    strokeWidth: 1
```

```yaml
# a consuming project's params.yaml — back to Lucide's own weight
icons:
  strokeWidth: 2
```

This is a deliberate break from upstream, so it is worth stating plainly: importing
this module changes how every Lucide icon looks compared to using Lucide directly.
The value `2` is still a first-class choice, and setting it skips the rewrite
entirely — it is what the source SVGs already contain.

This is deliberately **not** a CSS variable. Glyphs are painted as `mask` from a
base64-encoded SVG (§3), and a mask exposes only its alpha channel — no runtime
custom property can reach the `stroke-width` attribute inside it. The value is
therefore substituted into the SVG source before encoding, at build time.

Two consequences:

- The unit is **viewBox units**, not pixels. All Lucide icons are 24x24, so the
  rendered thickness is `strokeWidth / 24 * --icon-size`. At the default
  `--icon-size: 1em` (16px): `1` renders as 0.67px, `1.5` as 1px, `2` as 1.33px.
  A weight of `1` is genuinely hairline at 16px and depends on the renderer's
  antialiasing — `1.5` is the safer pick for icon-dense UIs.
- The weight is **site-wide**. A per-component variant would require emitting a
  second glyph set, doubling the stylesheet — not worth it for a design token that
  should stay consistent anyway.

The substitution is exhaustive and safe: all 1236 Lucide icons carry exactly one
`stroke-width="2"`, on the root `<svg>`. It is scoped to `icons/ui/` — Simple Icons
brands are solid fills and carry no stroke.

## 6. Brands — resolved by context

A brand only appears in a known place: the social menu (`social.yml`). That
partial prefixes the name with `brand:` so it resolves against Simple Icons; the
author writes nothing special.

```go-html-template
{{/* social menu partial */}}
{{ partial "icon" (printf "brand:%s" (lower .title)) }}
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
{{- site.Store.Add "usedIcons" (slice (printf "%s:%s" $dir $name)) -}}
<i class="icon icon-{{ $name }}" aria-hidden="true"></i>
```

Each used icon is registered as a `"dir:name"` string (e.g. `ui:map-pin`,
`brands:github`); the deferred pass splits it back on `:`.

### Engine — `layouts/partials/head/icons.html`

The base theme's `<head>` guards the include the same opt-in way as the call
sites — `{{ if templates.Exists "partials/head/icons.html" }}{{ partial "head/icons.html" . }}{{ end }}`
(in `hugolify-theme`'s `partials/head/head.html`) — so this partial runs **only**
when this module provides it. The `templates.Defer` block runs after all pages
render, so `site.Store` holds
every icon used site-wide. It emits one CSS file containing the base `.icon` rule
(§3, folded in) plus one `--icon-glyph` per used icon — no separate base asset, no
lookup table. The file is minified + fingerprinted **in production only**.

```go-html-template
{{ with (templates.Defer (dict "key" "hugolify-icons")) }}
  {{- /* default matches Lucide's own stroke-width, so an unset param changes nothing (§5) */ -}}
  {{- $stroke := 2 -}}
  {{- with site.Params.icons -}}{{- with .strokeWidth -}}{{- $stroke = . -}}{{- end -}}{{- end -}}
  {{- $glyphs := slice -}}
  {{- range site.Store.Get "usedIcons" | uniq -}}
    {{- $parts := split . ":" -}}
    {{- $dir := index $parts 0 }}{{ $name := index $parts 1 -}}
    {{- with resources.Get (printf "icons/%s/%s.svg" $dir $name) -}}
      {{- $svg := .Content -}}
      {{- /* every Lucide icon carries exactly one stroke-width="2", on the root <svg>; brands are solid fills and have none */ -}}
      {{- if and (eq $dir "ui") (ne $stroke 2) -}}
        {{- $svg = replaceRE `stroke-width="2"` (printf `stroke-width="%v"` $stroke) $svg -}}
      {{- end -}}
      {{- /* currentColor has no context in a mask → force opaque; collapse whitespace to ONE space (never empty: pretty-printed SVGs would glue their attributes) */ -}}
      {{- $svg = $svg | replaceRE `currentColor` `#000` | replaceRE `\s+` ` ` -}}
      {{- $glyphs = $glyphs | append (printf ".icon-%s{--icon-glyph:url('data:image/svg+xml;base64,%s')}" $name ($svg | base64Encode)) -}}
    {{- else -}}
      {{- warnf "[icons] %q not found in icons/%s/" $name $dir -}}
    {{- end -}}
  {{- end -}}
  {{- with $glyphs -}}
    {{- $base := ".icon{--icon-glyph:url('data:image/svg+xml,%3Csvg/%3E');background-color:var(--icon-color,currentColor);display:inline-block;width:var(--icon-size,1em);height:var(--icon-size,1em);mask:var(--icon-glyph) center / contain no-repeat;vertical-align:-0.125em}" -}}
    {{- $res := resources.FromString "css/icons.css" (printf "@layer components{%s%s}" $base (delimit . "")) -}}
    {{- if hugo.IsProduction }}{{ $res = $res | minify | fingerprint }}{{ end -}}
    <link rel="stylesheet" href="{{ $res.RelPermalink }}"{{ with $res.Data.Integrity }} integrity="{{ . }}"{{ end }}>
  {{- end -}}
{{ end }}
```

### Consuming-theme call sites

This module is the **sole** provider of `partials/icon.html`, and it is **opt-in**.
A consuming theme must **not** ship a default `icon` partial: a default would always
satisfy `templates.Exists`, defeating both the guard and the fallback below. So call
sites guard with `templates.Exists` and adapt when the module is absent.

Content icons (`key-features`, `alert`, `comparison`, `informations`):

```go-html-template
{{ with .icon }}{{ if templates.Exists "partials/icon.html" }}{{ partial "icon" . }}{{ end }}{{ end }}
```

Social menu (`nav/social.html`) — falls back to the plain link (its **label**):

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

- **module present** → SVG mask; the icon is registered for the deferred CSS.
- **module absent** → content renders nothing; the social menu shows the text label
  (via the `nav/link` fallback).

Layout and icon sizing (`.icon { --icon-size }`) live in the consuming theme (e.g.
`hugolify-theme-design-system/assets/css/components/nav-social.css`); when no icon is
rendered the link simply shows its text label.

## 8. Migration

- **Content naming.** A project coming from another icon set carries non-Lucide
  `icon-*` names in its content. There is no runtime alias; the Hugolify v2 content
  migration rewrites those names to their Lucide equivalents ahead of time.

### Brands → Simple Icons

`github, instagram, youtube, facebook` resolve directly; `twitter` → `x`.
`vimeo`, `google`, `bluesky`, `mastodon` have no Lucide equivalent — which is
exactly why brands go through Simple Icons.

**`linkedin` is the exception:** Simple Icons removed it (and a few other brands)
on legal request in 2022, and Lucide dropped all brand icons too — so it resolves
from neither upstream. Filling it requires a local SVG committed to this module
(see §9), which re-introduces the trademark exposure Simple Icons chose to avoid.

## 9. Status & open items

The engine is in place: `hugo.yaml`, `layouts/partials/icon.html`,
`layouts/partials/head/icons.html`. The base `.icon` rule is folded into the
generated stylesheet, so no separate CSS asset is needed.

- `templates.Defer` — available since Hugo **v0.128.0**; confirmed (dev machine
  runs v0.158). The module therefore requires Hugo ≥ 0.128.0.
- **Not yet validated by a build.** Run `hugo mod get` in a consuming project and
  confirm the upstream module imports resolve and their SVGs live at
  `icons/*.svg` for `lucide-icons/lucide` and `simple-icons/simple-icons`.
- **Lucide source caveat (important).** Go resolves modules by git tag, but the
  `lucide-icons/lucide` repo **stopped tagging around v0.289.0** (late 2023). So
  `@latest` caps at v0.289.0, which still uses legacy names (`home` not `house`,
  `alert-octagon` not `octagon-alert`, `code` not `code-xml`, `columns` not
  `columns-3`, `badge-help` not `badge-question-mark`); the `home → house` rename
  landed in ~v0.292 and was never tagged. Current icons live only on `main`/npm.
  Options:
  - pin `@main` (pseudo-version of the latest commit:
    `hugo mod get github.com/lucide-icons/lucide@main`) — note it is numerically
    *below* v0.289.0, so a later `@latest` would silently regress;
  - or source from npm `lucide-static` (properly versioned, current) or commit
    curated SVGs in this module.
  - **`simple-icons` has the same stale-tag gap, now confirmed:** highest git tag
    is `v2.17.1` (long abandoned), while `master` ships 3442 brands. Pinned to its
    default branch with `hugo mod get github.com/simple-icons/simple-icons@master`
    (pseudo-version `v0.0.0-…`, again numerically below the old tag). Re-bump with
    the same command; never `@latest`.
- **`linkedin` has no upstream source** — removed from Simple Icons on legal request
  (2022) and absent from Lucide (brand icons dropped). **Resolved here** with a local
  `assets/icons/brands/linkedin.svg`; the local `assets` mount (§5) takes precedence
  over the imports, so it fills the gap. This re-introduces the trademark exposure
  Simple Icons avoided — a deliberate, accepted call.
- **Class collision:** a page using both a UI `github` and a `brand:github` emits
  the same `.icon-github` twice with different glyphs. Namespace brand classes
  (e.g. `.icon-brand-github`) if this case is real.
