# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Relatório Gerencial dos Entregáveis — ZPE Maranhão.** A 36-slide deck presenting what each directorate of ZPE Maranhão (Brazilian Special Export Processing Zone) delivered between November 2025 and September 2026: 174 deliverables from five directorates plus 23 from the Assessoria da Presidência (197 in total).

One static HTML file, with no build step, no framework and no runtime dependencies. `playwright` and `pdf-lib` are dev-only, used by the scripts.

It was split out of `../apresentacao-consad` (where it was `index_entregaveis.html`) and inherits that deck's stage, tokens, themes and chrome unchanged.

```
index.html          the deck (HTML + CSS + JS, all inline)
assets/             only the files index.html references
  images/           bg_hero_blue_compressed.jpg, world_dotted.svg, world_dotted_raster.png
  logos/            zpe_white.png, investe-ma-logo-branco.png, governo-ma-logo.png
  videos/           video_{capa,card,enc}_web.{mp4,webm} + _poster.jpg
design_system/      brand source of truth (copy of the ZPE design system)
documentos/         Relatório Gerencial dos entregáveis ZPEMA.Rev.01.docx — content source
scripts/            verificar.js, export-pdf.js
```

## How to Run

Open `index.html` in a browser, or serve the folder:

```bash
npm run serve        # python -m http.server 8000
npm install          # once, for the scripts
npm run verificar    # all slides × 5 window sizes: clipped text, console, 404s
npm run pdf          # one 16:9 page per slide → Relatorio-Gerencial-Entregaveis.pdf
```

## Configuration

Flags in the `<script>` block at the top of `index.html`:

- **`PERIODO`** — reference period, shown on the cover.
- **`DATA_APRESENTACAO`** — presentation date on the cover; empty string hides it.
- **`MOSTRAR_MARCA_GOVERNO`** — election-period toggle for the "Governo do Maranhão" logo (`false` hides it). URL override: `?governo=off` / `?governo=on`. "Investe Maranhão" is not affected.

## Architecture

### Fixed 16:9 stage (NON-NEGOTIABLE)

Slides are authored at **1920×1080** inside `.deck-stage` and scaled by `fitStage()`, which **always scales by width** (`scale = vw / 1920`). The slide touches both side edges; in windows wider than 16:9 (a maximised window minus the taskbar, e.g. 1920×968) it overflows ~56px above and below. The slide padding `150px 168px 190px` and `.foot { bottom: 138px }` are the vertical safe zone that keeps content clear of that band — **do not reduce them**.

- Fixed px at the design size. No responsive breakpoints for slide content.
- Never switch slides with `display:none`; visibility is `.active` (`visibility`/`opacity`/`pointer-events`).
- Every content slide needs `.pad > .body` (flex:1) or its content floats in the middle.

### Deliverables data — `CAPS` is the single source of truth

`CAPS` (in the main `<script>`) holds every deliverable, verbatim from the Relatório Gerencial Rev. 01 (spelling corrected), as `capítulo → setores → grupos → itens`. An item is a string, or `{ t, st }` where `st` is `'and'` (Em andamento) or `'cont'` (Contínuo).

- **Counts are computed, never typed.** Any element with `data-conta="<id>"` (a chapter, setor or group id, or a sum like `jur-proj+jur-soc`) gets its number from `CONTA`. The number written in the HTML is a fallback; if it disagrees with the data the console warns.
- `CONTA.diretorias` excludes the Presidência (the report counts it separately, section 3); `CONTA.todos` includes it.

### Lista de entregáveis — generated slides

Each sector ends with the full list of its deliverables. These slides are **generated from `CAPS`** by `montarLista()`, driven by `PAGINAS_LISTA`:

```js
{ apos: 'eng', cap: 'deti', titulo: '…', idx: 'Lista: Engenharia', grupos: 'deti-eng', cols: 2, lg: true, secao: 'seção 4.1' }
```

- `apos` names the anchor: the last slide of the sector carries `data-lista="eng"`, and its pages are inserted right after it, in order. Anchors: `eng`, `ma`, `ops`, `com`, `mkt`, `daf`, `jur`, `pres`.
- `grupos` is a group id list, or a setor id (expands to all its groups).
- `cols` (2 or 3) sets the CSS multicol count; `lg: true` enlarges type on short pages so they don't sit half-empty.
- Items are numbered continuously within the chapter.

**Pagination is fixed, not automatic** — it was measured at 1920×1080 with the real fonts. Two runtime guards warn in the console: pages must cover every group **in `CAPS` order** (otherwise numbering jumps between pages), and after `document.fonts.ready` any page whose list overflows its body is reported. If you edit `CAPS` text, run `npm run verificar`; if a page overflows, move groups between pages or add a page.

Slide titles for the index overlay (`G`) live in `TITULOS` for the static slides and are written to `data-titulo` **before** the list is interleaved; generated slides carry their own `data-titulo`. When adding or removing a static slide, update `TITULOS`.

### Dormant drawer

The side drawer (`#dw`, "PAINEL DE ENTREGÁVEIS") is still in the file but **no button opens it** — the client had the "Ver os N entregáveis" / "Todos os entregáveis" buttons removed because they competed with the list slides. Do not reintroduce them without asking. The drawer can be deleted if it is confirmed it will not return.

### Themes and rhythm

`.s-dark`, `.s-deep` (chapter openers), `.s-light`, `.capa`. `.grain` is mandatory on every surface (`.grain-light` on light). List slides are always `.s-light` — reference material reads as a document. On light slides the eyebrow is `#24651f` (contrast), and every component needs an override in the light-theme CSS blocks.

### Animation

`.anim` + `.d1`–`.d8` reveal on slide entry (Corporate motion: `cubic-bezier(.2,0,0,1)`, no bounce). Bars and counters run in `runSlideAnimations()` and honour `prefers-reduced-motion`.

### Background videos (memory budget)

Only the cover video (`#capaVideo`) autoplays. Chapter openers and the CZPE card have `preload="none"` with URLs in `data-src-webm` / `data-src-mp4`; `montarFontes()` mounts the sources on entry and `descarregarVideo()` strips them on exit. This is a crash fix for iOS Safari (too many simultaneous decoders), not an optimisation — do not add `autoplay`/`preload="auto"` to the secondaries.

### Navigation

Arrows / Space / PageUp-Down, Home, End, click on either half, swipe, `F` fullscreen, `G` slide index. The HUD (bloco, dots, counter) auto-hides.

## Front-end Standards

- **`design_system/` is the source of truth.** Read `design_system/readme.md` and `design_system/SKILL.md` before changing UI. Tokens come from `design_system/tokens/*.css`, mirrored as custom properties in `:root`.
- Blue `#244582` structure (~70%), green `#4cac4b` accent only (~10%; never a large number or title in green over dark blue), canvas `#f9f9ff` (~20%). Text on dark uses the tinted ramp `--w-100…--w-30`, never translucent white; nothing a reader needs below `--w-45`.
- Fonts: **Barlow** (display), **Inter** (body — brand-mandated, not a default), **JetBrains Mono** (eyebrows, labels, figures), via Google Fonts.
- Eyebrows: mono, uppercase, green, **no leading dash or rule**.

## Verifying changes

Render and look — the deck is presented to a board. `npm run verificar` must report no clipped text and a clean console/network at all five sizes (1920×968 first, then 1920×1080, 1366×728, 2560×1330, 1440×900). `SHOTS=1 npm run verificar` saves screenshots to `.shots/` for review.

## Content Rules

All data comes from `documentos/Relatório Gerencial dos entregáveis ZPEMA.Rev.01.docx`. **Do not alter numbers, names or item wording without asking.** The summary slides may shorten an item; the list slides always show the full wording from `CAPS`.

Filenames with accents must be NFC-normalised (an NFD `Ç` once broke a link on a byte-exact server).

## Language

All user-facing text is **Brazilian Portuguese**. Code comments and class names are in English or Portuguese, matching the surrounding code.
