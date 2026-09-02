---
name: markdown-to-pdf
description: >-
  Converts Markdown to PDF with embedded diagram code blocks rendered as
  images: pandoc plus the pandoc-ext diagram Lua filter and a LaTeX or Typst
  engine, rendering Mermaid (via mermaid-cli), Graphviz/DOT, PlantUML, TikZ,
  D2, and Asymptote fences to vector PDF; includes tool checks and installs,
  per-diagram captions and sizing, Kroki and Quarto fallbacks, and fixes for
  vanished mermaid labels, missing LaTeX packages, and Unicode or emoji. Use
  when converting, exporting, compiling, or printing Markdown (READMEs,
  design docs, notes) to PDF, especially when the markdown contains mermaid
  or other diagram code fences.
---

# Markdown to PDF

Recommended pipeline: pandoc + the [pandoc-ext/diagram](https://github.com/pandoc-ext/diagram) Lua filter + a LaTeX engine. The filter converts diagram code fences to images at build time, and for LaTeX output it requests **vector PDF** from each diagram tool — which sidesteps the classic mermaid problem of labels vanishing when its SVG is consumed outside a browser. Typst reaches the same place but needs a one-line filter patch first ([Typst engine](#typst-engine)). Do not use the npm package `mermaid-filter` (unmaintained since 2023, pins a deprecated Puppeteer, `npm i -g` hangs).

## Check tools, install what's missing

| Need | Check | Install |
|---|---|---|
| pandoc ≥ 3.0 | `pandoc --version` | `brew install pandoc`; no package manager → `.pkg`/`.deb`/`.msi` from [pandoc releases](https://github.com/jgm/pandoc/releases), or the tarball route below |
| PDF engine | `lualatex --version` or `typst --version` | any TeX Live, incl. [TinyTeX](https://yihui.org/tinytex/); or go LaTeX-free with `typst` (≥ 0.14) or `tectonic` (auto-fetches packages) |
| the filter | `grep 'local pdf2svg = name' diagram.lua` | pinned `curl`, below — and patch it for Typst |
| mermaid renderer | render a fence, below — **not** `mmdc --version` | `npm install -g @mermaid-js/mermaid-cli` — needs Node; first install downloads a headless Chromium (slow once). The brew formula is dead. No Node → Docker `ghcr.io/mermaid-js/mermaid-cli/mermaid-cli` (mounts `/data`) or the Kroki fallback below |
| others, as used | `dot`, `plantuml`, `d2`, `asy` | `brew install graphviz plantuml d2 asymptote` — only what the document's fences use |

For repeat use, drop `diagram.lua` into the `filters/` subdirectory of pandoc's user data directory (`pandoc --version` prints it, e.g. `~/.local/share/pandoc`); `--lua-filter diagram.lua` then resolves by name from any directory. Keep a patched copy under its own name (`diagram-typst.lua`) rather than editing the shared one in place, so the next `curl` does not silently revert the build.

### Pin the filter

`main` is not reproducible and the tags lag it: `v1.2.0` (Oct 2024) predates the D2 engine, and both files self-report version `1.2.0`, so the version string cannot tell them apart. Pin a commit instead.

```bash
SHA=5aaf35e6f775ddf501045fe28b03642ee73db2f4   # main as of Oct 2025; has d2
curl -fsSL -o diagram.lua \
  "https://raw.githubusercontent.com/pandoc-ext/diagram/$SHA/_extensions/diagram/diagram.lua"
```

Resolve a newer one with `curl -fsSL 'https://api.github.com/repos/pandoc-ext/diagram/commits?path=_extensions/diagram/diagram.lua&per_page=1'`.

### Prove mermaid renders, don't trust `--version`

`mmdc --version` prints a version even when Puppeteer's Chromium was never downloaded, so the version check passes on an installation that cannot render. npm ≥ 11 blocks install scripts by default and says so in a line easy to scroll past (`allow-scripts ... puppeteer@N (postinstall: node install.mjs)`). Fetch the browser explicitly, into a directory that survives the shell:

```bash
export PUPPETEER_CACHE_DIR="$HOME/.cache/puppeteer"   # also needed at render time
npx puppeteer browsers install chrome

printf 'flowchart LR\n  A[Client] --> B[Server]\n' > /tmp/smoke.mmd
mmdc -i /tmp/smoke.mmd -o /tmp/smoke.pdf --pdfFit    # must exit 0 and write a real PDF
```

`npm approve-scripts puppeteer` does not work here — puppeteer is a transitive dependency, so npm answers `ENOMATCH`. Without `PUPPETEER_CACHE_DIR` the download can land in a per-process temp directory and vanish before the next run.

### No package manager

```bash
mkdir -p ~/.local/bin
# pandoc — the zip's top directory carries the arch, not just the version
curl -fsSL -o /tmp/pandoc.zip https://github.com/jgm/pandoc/releases/download/3.11/pandoc-3.11-arm64-macOS.zip
unzip -qo /tmp/pandoc.zip -d /tmp && cp /tmp/pandoc-3.11-arm64/bin/pandoc ~/.local/bin/
# typst
curl -fsSL https://github.com/typst/typst/releases/download/v0.15.1/typst-aarch64-apple-darwin.tar.xz \
  | tar -xJ -C /tmp && cp /tmp/typst-aarch64-apple-darwin/typst ~/.local/bin/
export PATH="$HOME/.local/bin:$PATH"    # persist this in the shell rc
```

Match the asset to `uname -m` (`arm64`/`aarch64` vs `x86_64`). This route is the reason to prefer Typst on a machine with no TeX tree: one 30 MB binary against a TeX Live install.

## Convert

```bash
pandoc doc.md -f gfm+attributes -o doc.pdf \
  --lua-filter diagram.lua --pdf-engine=lualatex \
  -V geometry:margin=2.5cm -V colorlinks
```

- Reader: `-f gfm+attributes` for GitHub-flavored files (`+attributes` legalizes `{.mermaid width=60%}` fence attributes). Prefer the default `markdown` reader when the document uses pandoc-only features (citations, divs, raw LaTeX).
- Engine: `lualatex` copes best with Unicode; pandoc's default `pdflatex` is fine for ASCII-only docs. `--pdf-engine=typst` is a fast LaTeX-free alternative — Typst ≥ 0.14 embeds the filter's PDF images natively, but read the next section before using it.
- Useful extras: `--toc`, `-N`, `-V papersize=a4`, `--resource-path=dir1:dir2` when linked images live elsewhere.
- If emitting `.tex` instead of `.pdf`, add `--extract-media=media`, otherwise generated images stay in memory and the `.tex` won't compile.

## Typst engine

Two things do not carry over from the LaTeX route, and both fail quietly.

**Patch the filter, or mermaid arrives as SVG.** `diagram.lua` decides format from the output target, and only LaTeX and ConTeXt are exempt from its SVG preference — Typst is not, despite embedding PDF natively. Mermaid then goes through SVG and Typst warns `image contains foreign object`, which is exactly the label-loss failure this pipeline exists to avoid. Pandoc still exits 0, so the build looks clean and the PDF is broken. One line fixes it, idempotently:

```bash
perl -pi -e "s/(local pdf2svg = name ~= 'latex' and name ~= 'context')\$/\$1 and name ~= 'typst'/" diagram.lua
grep -n "local pdf2svg = name" diagram.lua    # must end: and name ~= 'typst'
```

Setting `diagram.engine.mermaid.mime-type: application/pdf` is not an alternative. The filter reads that as "PDF, then convert", routes the image through its `pdf2svg` helper — which shells out to **Inkscape** (`inkscape: createProcess ... does not exist` when absent, aborting the run) — and lands back on the SVG that loses labels. With the patch, no `mime-type` setting is needed at all.

**Margins are a map, and `-V` cannot express one.** Pandoc's typst template iterates `margin/pairs`, so `margin` has to arrive as YAML. Every command-line spelling fails, two of them without saying so:

| Attempt | Result |
|---|---|
| `-V geometry:margin=2.5cm` | silently ignored — `geometry` is a LaTeX-only variable |
| `-V margin.x=2.5cm` | silently ignored — `-V` does not build nested values |
| `-V margin:2.5cm` | hard error, `unexpected comma ... margin: (: ,)` |
| map in a defaults file or the document's YAML block | works |

```yaml
variables:          # in a defaults file; at the top level in a document's YAML block
  margin:
    x: 2.5cm
    y: 2.5cm
```

Check it took rather than assuming: at 2.5cm the text block starts 70.9pt from the page edge, against Typst's 90pt default on US Letter. Unchanged 90pt means the setting never arrived.

## How diagram fences are handled

The filter matches the fence's (first) class and pipes the code through the tool:

| Class | Tool | Notes |
|---|---|---|
| `mermaid` | `mmdc --pdfFit` | vector PDF for LaTeX output, and for Typst once patched |
| `dot` | `dot` | Graphviz |
| `plantuml` | `plantuml` | needs Java |
| `tikz` | `pdflatex` | compiles with the `standalone` class |
| `d2` | `d2` | |
| `asymptote` | `asy` | |
| `cetz` | `typst` | |

Override an executable with env vars (`MERMAID_BIN`, `DOT_BIN`, `PLANTUML_BIN`, `PDFLATEX_BIN`, `TYPST_BIN`, `D2_BIN`) or metadata `diagram.engine.<name>.execpath`; the list form passes extra arguments, e.g. `execpath: [mmdc, -t, dark]` to theme every mermaid diagram, or `execpath: [npx, -p, "@mermaid-js/mermaid-cli", mmdc]` to avoid a global install. Arguments from the list are prepended to the ones the filter already builds, so don't re-add `--pdfFit`, `--input`, or `--output`.

- A missing tool logs `[WARNING] ... mmdc: createProcess ... does not exist` and the fence lands in the PDF as literal code. Treat every `diagram.lua` warning as a failed diagram; fix and re-run until the build is warning-free.
- Per-diagram options go on comment-pipe lines inside the fence (mermaid comments are `%%`, dot `//`, plantuml `'`):

````markdown
```mermaid
%%| fig-cap: Request flow
%%| width: 70%
flowchart LR
  A[Client] --> B[Server]
```
````

  `fig-cap` (caption), `label` (cross-reference id), `filename`, `alt`; any other key becomes an image attribute (`width`, `height`). Same keys work as fence attributes with `+attributes`.
- Security: document metadata can point `execpath` at any binary, so don't convert untrusted markdown without pinning the engines (defaults file below; see the filter README's security section).
- A defaults file bundles the setup, enables caching, and pins engines. `md2pdf.yaml`:

```yaml
from: gfm+attributes
pdf-engine: lualatex
filters: [diagram.lua]
metadata:
  diagram:
    cache: true
    engine:   # pinning prevents execpath injection from the document
      mermaid: {}   # NOT `true` — see below
      dot: {}
      plantuml: {}
      tikz: {}
```

Run `pandoc -d md2pdf.yaml doc.md -o doc.pdf`; store the yaml in `<data-dir>/defaults/` to use it anywhere. The Typst variant swaps the engine and the filter, and carries the margins that `-V` cannot:

```yaml
from: gfm+attributes
pdf-engine: typst
filters: [diagram-typst.lua]   # the patched copy
variables:
  margin:
    x: 2.5cm
    y: 2.5cm
metadata:
  diagram:
    cache: true
    engine:
      mermaid: {}
      dot: {}
```

**Pin engines with `{}`, never `true`.** The filter documents `true` as "use default options", but it short-circuits before computing the engine's output MIME type, and the nil that leaves behind breaks two ways:

- with `cache: true`, the run dies on `diagram.lua:NNN: attempt to concatenate a nil value (field '?')` — the cache builds its filename from that MIME type
- with `cache: false`, mermaid falls back to its internal SVG default, so `: true` quietly undoes the Typst patch and the labels go with it

An empty map takes the normal configuration path, negotiates PDF, and still pins the executable: a defaults file's `metadata` replaces the document's YAML wholesale rather than merging into it, so once it sets `diagram`, nothing the document says under that key survives.

Engines you leave off the list are inert: their fences land in the PDF as literal code, with no warning. List every class the document actually uses.

## No local renderer: Kroki

[Kroki](https://kroki.io) renders ~25 diagram languages server-side (mermaid, plantuml, graphviz, d2, excalidraw, bpmn, ...). Render each fence body, then replace the fence with `![](file)`:

```bash
curl -fsSL -X POST -H 'Content-Type: text/plain' \
  --data-binary @diagram.mmd -o diagram.png https://kroki.io/mermaid/png
```

URL is `https://kroki.io/<type>/<format>`. For the LaTeX route request `pdf` (graphviz, plantuml, ...) or `png` (mermaid) — avoid `svg`, which needs `rsvg-convert` and hits the mermaid label problem. Caveats: the diagram source goes to a third-party server, and the free instance's mermaid endpoint is flaky (observed 500s while graphviz worked). For confidential sources self-host: `docker run -p 8000:8000 yuzutech/kroki`, plus the `yuzutech/kroki-mermaid` companion container for mermaid.

## Quarto alternative

If Quarto is installed it bundles pandoc and mermaid — no npm needed. Rewrite fences from ` ```mermaid ` to ` ```{mermaid} `, then `quarto render doc.md --to pdf`. Print formats rasterize diagrams to PNG through Chrome or Edge (auto-detected); without either, run `quarto install chrome-headless-shell` (Quarto ≥ 1.9; supersedes the deprecated `quarto install chromium`).

## Pitfalls

- **Mermaid labels vanish** whenever its SVG is consumed outside a browser (librsvg, Inkscape, LaTeX): mermaid emits `<foreignObject>` HTML labels. The pipeline above never touches SVG; if SVG is unavoidable, set `{"htmlLabels": false, "flowchart": {"htmlLabels": false}}` in a mermaid config file (`mmdc -c cfg.json`) and still expect residual SVG2-CSS breakage.
- **Minimal TeX installs**: the TikZ engine needs `tlmgr install standalone pgf` on TinyTeX. Any other missing `.sty` in the main build → `tlmgr install <package>`.
- **Unicode/emoji dropped** by LaTeX: set a capable main font plus fallbacks — LuaLaTeX-only, and each fallback name must end with `:` or luaotfload crashes:

```yaml
mainfont: "TeX Gyre Pagella"
mainfontfallback: ["FreeSans:", "NotoColorEmoji:mode=harf"]
```

  Or switch to `--pdf-engine=typst`, which font-falls-back automatically.
- **Diagram overflows the page**: add `%%| width: 80%` (the PDF is already crop-fit via `--pdfFit`). Raster mermaid (`diagram.engine.mermaid.mime-type: image/png`) looks blurry at default resolution; scale up with `execpath: [mmdc, -s, "3"]`.
- **Chromium sandbox on Linux/CI (root)**: mmdc needs a Puppeteer config `{"args": ["--no-sandbox"]}` passed via `execpath: [mmdc, -p, puppeteer.json]`.
- **Stale diagram cache**: the cache key is the fence text plus the output extension and nothing else, so a changed mermaid theme, `-s` scale, or mmdc version reuses the old image. Clear `${XDG_CACHE_HOME:-~/.cache}/pandoc-diagram-filter` after changing anything about the renderer rather than the diagram.
- **macOS `sips` renders Typst PDFs black**: it rasterizes the mermaid PDFs from Chromium correctly but returns solid black pages for Typst output, which reads as a broken PDF that is in fact fine. Rasterize with `pypdfium2` instead (`page.render(scale=2)`), or just open the PDF.
- GitHub's non-code-fence renderers (geoJSON, topoJSON, STL) have no local equivalent — leave those fences as code or screenshot them.

## Verify

Exit status proves nothing here: a missing renderer, an SVG mermaid diagram under Typst, and an ignored margin setting all exit 0.

1. The whole log is warning-free. Each `diagram.lua` warning means a diagram was left as literal code; each Typst `image contains foreign object` means a diagram is losing its labels.
2. Diagram count matches — one image per fence. With `cache: true` the cache directory holds one file per distinct diagram:

```bash
grep -cE '^ *```(mermaid|dot|plantuml|tikz|d2)' doc.md
ls "${XDG_CACHE_HOME:-$HOME/.cache}/pandoc-diagram-filter" | wc -l
```

3. Open the diagram-bearing pages, not just page 1. Every diagram is an image, node labels are present, nothing runs off the page.

## Sources

- [diagram filter README](https://github.com/pandoc-ext/diagram) and [source](https://raw.githubusercontent.com/pandoc-ext/diagram/main/_extensions/diagram/diagram.lua): classes, engines, options, security, per-format MIME selection. `format_options` is where the Typst/SVG choice is made, `pdf2svg` is the Inkscape call, `cache_image` shows the cache key.
- [mermaid-cli README](https://github.com/mermaid-js/mermaid-cli): install paths, markdown mode, Docker, sandbox issue.
- [pandoc MANUAL](https://pandoc.org/MANUAL.html): `--pdf-engine` values, `gfm+attributes`, `--resource-path`, font variables.
- [pandoc #11678](https://github.com/jgm/pandoc/issues/11678): trailing-colon requirement in `*fontfallback`.
- [Quarto diagrams](https://quarto.org/docs/authoring/diagrams.html) and [Chrome Headless Shell post](https://opensource.posit.co/blog/2026-04-14_chrome-headless-shell/): `{mermaid}` fences, PNG print path, `quarto install chrome-headless-shell`.
- [Kroki docs](https://docs.kroki.io/): POST API, formats, self-hosting.
- [mermaid #58](https://github.com/knsv/mermaid/issues/58) and [mermaid-cli #691](https://github.com/mermaid-js/mermaid-cli/issues/691): foreignObject label loss, `htmlLabels` workaround.
- [Typst 0.14 release](https://typst.app/docs/changelog/0.14.0/): PDFs usable as images. [typst#1421](https://github.com/typst/typst/issues/1421) is the `foreign object` warning's own reference.
- `pandoc -D typst`: the template that iterates `margin/pairs`, and the authority on which variables the Typst writer reads.
