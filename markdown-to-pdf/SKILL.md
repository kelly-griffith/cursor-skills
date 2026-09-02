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

Recommended pipeline: pandoc + the [pandoc-ext/diagram](https://github.com/pandoc-ext/diagram) Lua filter + a LaTeX engine. The filter converts diagram code fences to images at build time, and for PDF output it requests **vector PDF** from each diagram tool — which sidesteps the classic mermaid problem of labels vanishing when its SVG is consumed outside a browser. Do not use the npm package `mermaid-filter` (unmaintained since 2023, pins a deprecated Puppeteer, `npm i -g` hangs).

## Check tools, install what's missing

| Need | Check | Install |
|---|---|---|
| pandoc ≥ 3.0 | `pandoc --version` | `brew install pandoc`; no package manager → `.pkg`/`.deb`/`.msi` from [pandoc releases](https://github.com/jgm/pandoc/releases) |
| PDF engine | `lualatex --version` | any TeX Live, incl. [TinyTeX](https://yihui.org/tinytex/); or go LaTeX-free with `typst` (≥ 0.14) or `tectonic` (auto-fetches packages) |
| the filter | `diagram.lua` on disk | `curl -fsSL -o diagram.lua https://raw.githubusercontent.com/pandoc-ext/diagram/main/_extensions/diagram/diagram.lua` (releases carry no `.lua` asset; use the raw URL) |
| mermaid renderer | `mmdc --version` | `npm install -g @mermaid-js/mermaid-cli` — needs Node; first run downloads a headless Chromium (slow once). The brew formula is dead. No Node → Docker `ghcr.io/mermaid-js/mermaid-cli/mermaid-cli` (mounts `/data`) or the Kroki fallback below |
| others, as used | `dot`, `plantuml`, `d2`, `asy` | `brew install graphviz plantuml d2 asymptote` — only what the document's fences use |

For repeat use, drop `diagram.lua` into the `filters/` subdirectory of pandoc's user data directory (`pandoc --version` prints it, e.g. `~/.local/share/pandoc`); `--lua-filter diagram.lua` then resolves by name from any directory.

## Convert

```bash
pandoc doc.md -f gfm+attributes -o doc.pdf \
  --lua-filter diagram.lua --pdf-engine=lualatex \
  -V geometry:margin=2.5cm -V colorlinks
```

- Reader: `-f gfm+attributes` for GitHub-flavored files (`+attributes` legalizes `{.mermaid width=60%}` fence attributes). Prefer the default `markdown` reader when the document uses pandoc-only features (citations, divs, raw LaTeX).
- Engine: `lualatex` copes best with Unicode; pandoc's default `pdflatex` is fine for ASCII-only docs. `--pdf-engine=typst` is a fast LaTeX-free alternative — Typst ≥ 0.14 embeds the filter's PDF images natively.
- Useful extras: `--toc`, `-N`, `-V papersize=a4`, `--resource-path=dir1:dir2` when linked images live elsewhere.
- If emitting `.tex` instead of `.pdf`, add `--extract-media=media`, otherwise generated images stay in memory and the `.tex` won't compile.

## How diagram fences are handled

The filter matches the fence's (first) class and pipes the code through the tool:

| Class | Tool | Notes |
|---|---|---|
| `mermaid` | `mmdc --pdfFit` | vector PDF for LaTeX output |
| `dot` | `dot` | Graphviz |
| `plantuml` | `plantuml` | needs Java |
| `tikz` | `pdflatex` | compiles with the `standalone` class |
| `d2` | `d2` | |
| `asymptote` | `asy` | |
| `cetz` | `typst` | |

Override an executable with env vars (`MERMAID_BIN`, `DOT_BIN`, `PLANTUML_BIN`, `PDFLATEX_BIN`, `TYPST_BIN`, `D2_BIN`) or metadata `diagram.engine.<name>.execpath`; the list form passes extra arguments, e.g. `execpath: [mmdc, -t, dark]` to theme every mermaid diagram, or `execpath: [npx, -p, "@mermaid-js/mermaid-cli", mmdc]` to avoid a global install.

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
      mermaid: true
      dot: true
      plantuml: true
      tikz: true
```

Run `pandoc -d md2pdf.yaml doc.md -o doc.pdf`; store the yaml in `<data-dir>/defaults/` to use it anywhere.

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
- GitHub's non-code-fence renderers (geoJSON, topoJSON, STL) have no local equivalent — leave those fences as code or screenshot them.

## Verify

1. The pandoc run ends warning-free — each `diagram.lua` warning means a diagram was left as literal code.
2. Inspect the PDF: every diagram is an image, node labels are present, nothing overflows the page.

## Sources

- [diagram filter README](https://github.com/pandoc-ext/diagram) and [source](https://raw.githubusercontent.com/pandoc-ext/diagram/main/_extensions/diagram/diagram.lua): classes, engines, options, security, per-format MIME selection.
- [mermaid-cli README](https://github.com/mermaid-js/mermaid-cli): install paths, markdown mode, Docker, sandbox issue.
- [pandoc MANUAL](https://pandoc.org/MANUAL.html): `--pdf-engine` values, `gfm+attributes`, `--resource-path`, font variables.
- [pandoc #11678](https://github.com/jgm/pandoc/issues/11678): trailing-colon requirement in `*fontfallback`.
- [Quarto diagrams](https://quarto.org/docs/authoring/diagrams.html) and [Chrome Headless Shell post](https://opensource.posit.co/blog/2026-04-14_chrome-headless-shell/): `{mermaid}` fences, PNG print path, `quarto install chrome-headless-shell`.
- [Kroki docs](https://docs.kroki.io/): POST API, formats, self-hosting.
- [mermaid #58](https://github.com/knsv/mermaid/issues/58) and [mermaid-cli #691](https://github.com/mermaid-js/mermaid-cli/issues/691): foreignObject label loss, `htmlLabels` workaround.
- [Typst 0.14 release](https://typst.app/docs/changelog/0.14.0/): PDFs usable as images.
