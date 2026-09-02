---
name: write-lualatex
description: >-
  Writes and builds LaTeX documents for the LuaLaTeX engine: fontspec and
  unicode-math preambles, OpenType and system font loading through luaotfload,
  multilingual and bidirectional setup with babel, embedded Lua via \directlua
  and the luacode package, and lualatex/latexmk builds and their failure
  modes. Use when creating or editing documents compiled with lualatex,
  migrating a document from pdfLaTeX, or when the user mentions LuaLaTeX,
  LuaTeX, fontspec, unicode-math, luaotfload, \directlua, luacode, system
  fonts in LaTeX, or complex-script/right-to-left text.
disable-model-invocation: true
---

# Writing for LuaLaTeX

LuaLaTeX is LaTeX on the LuaHBTeX engine: LuaTeX plus an embedded HarfBuzz shaper (since TeX Live 2020), scriptable in Lua (5.3; `\directlua{tex.print(_VERSION)}` prints the running version). Source is UTF-8, fonts are OpenType loaded through luaotfload, and main memory grows on demand. Pick it when a document needs system fonts, non-Latin or bidirectional scripts, more memory than pdfTeX allows, or computation during typesetting.

## Preamble

Omit `inputenc`, `fontenc`, `textcomp`, and `luainputenc`; they exist for 8-bit engines. Canonical skeleton:

```latex
\documentclass{article}
\usepackage{amsmath}      % load before unicode-math, with any options here
\usepackage{unicode-math} % loads fontspec; replaces 8-bit math font setup
\setmainfont{TeX Gyre Pagella}
\setmathfont{TeX Gyre Pagella Math}
\usepackage{microtype}
\tracinglostchars=3       % a glyph missing from the font is an error
```

- Load `amsmath` or `mathtools` before `unicode-math`, and `unicode-math` after every other math or font package. Without the two `\set...font` lines you get Latin Modern and Latin Modern Math.
- fontspec applies `Ligatures=TeX` to the serif and sans families by default, so the pdfLaTeX input shorthands for curly quotes and dashes keep working.
- The kernel default `\tracinglostchars=2` only warns; 3 turns silently dropped glyphs into errors.
- For bold or styled math symbols use unicode-math's `\symbf`, `\symit`, `\symcal`, `\symbb`; `\mathbf` remains the text-font alphabet for words like operator names.

## Fonts

- Request fonts by family name: `\setmainfont{EB Garamond}`. Names resolve for both system and TEXMF-tree fonts (unlike XeLaTeX, which needs file names for TEXMF fonts). File names work too: `\setmainfont{texgyrepagella-regular.otf}`.
- Features go in a trailing optional argument: `\setmainfont{EB Garamond}[Numbers=OldStyle]`.
- The first name lookup builds a font database by scanning every font on the machine. Inside a run this looks like a hang; several minutes is normal, once. After installing fonts, run `luaotfload-tool --update` (add `--force` for a full rebuild). To check what a name resolves to: `luaotfload-tool --find="EB Garamond" --fuzzy`.
- For complex scripts (Arabic, Indic, Myanmar, ...) select the HarfBuzz shaper per font and set the script: `\newfontfamily\arabicfont{Amiri}[Script=Arabic, Renderer=HarfBuzz]`. The default renderer is right for Latin, Cyrillic, and Greek. Never set `Renderer=HarfBuzz` on the math font.

## Multilingual text

Use babel, not polyglossia: babel is the actively developed one on LuaTeX, with integrated bidi and per-language fonts. polyglossia targets XeTeX.

```latex
\usepackage[english, bidi=basic]{babel}
\babelprovide[import]{arabic}
\babelfont{rm}{TeX Gyre Pagella}
\babelfont[arabic]{rm}{Amiri}
```

- `\babelprovide[import]{...}` loads any of ~300 locales from babel's ini files; secondary languages also load on the fly at first use.
- `\babelfont` assigns a family per language and loads it lazily. babel sets `Script`, `Language`, and the renderer (HarfBuzz for most non-Latin scripts) itself; setting them manually is discouraged.
- `bidi=basic` enables babel's Unicode bidi algorithm for right-to-left text; it requires LuaTeX.
- Switch with `\selectlanguage{arabic}` or inline `\foreignlanguage{arabic}{...}`.
- For Japanese, use `luatexja`.

## Embedded Lua

TeX tokenizes and expands a `\directlua` argument before Lua sees it:

- `%` starts a TeX comment and eats the rest of the source line.
- Newlines collapse to spaces, so a Lua `--` comment comments out the rest of the chunk.
- Backslash sequences are expanded as TeX macros; prefix with `\string` to get a literal backslash through.

Keep `\directlua` to one-liners; put real code in the luacode package's environments or in `.lua` files.

| luacode tool | Difference from `\directlua` |
|---|---|
| `\luaexec{...}` | `\%`, `\#`, `\\`, and `~` produce the expected characters |
| `luacode` environment | `%` and `#` are literal; line breaks survive, so `--` comments are safe; TeX macros still expand |
| `luacode*` environment | verbatim Lua, no macro expansion |
| `\luastring{arg}` | expands `arg` and escapes it into a quoted Lua string (`\luastringN` unexpanded, `\luastringO` one level) |

Send material back with `tex.sprint(...)` (continues the current line) or `tex.print(...)` (each argument becomes its own input line). Nothing reaches TeX until the chunk returns.

```latex
\usepackage{luacode}
\newcommand\upper[1]{\directlua{tex.sprint(string.upper(\luastring{#1}))}}

\begin{luacode*}
function rows(path)
  for line in io.lines(path) do
    tex.sprint(line:gsub(",", " & ") .. "\\\\")
  end
end
\end{luacode*}

\begin{tabular}{lll}
  \directlua{rows("data.csv")}
\end{tabular}
```

- Plain file I/O (`io.open`, `io.lines`) needs no flags. `os.execute`, `io.popen`, `os.exec`, and `os.spawn` obey the same shell-escape policy as `\write18`.
- For a reusable library, put functions in `helpers.lua` next to the document and load once with `\directlua{require("helpers")}`; `require` resolves through kpathsea, so the working directory and the TEXMF tree both work.
- To hook the engine itself (node lists, font loading, file resolution), register with `luatexbase.add_to_callback`; the interface ships in the LaTeX kernel.

## Building

```bash
lualatex document.tex
latexmk -lualatex document.tex
```

- `latexmk -lualatex` is equivalent to `-pdflua -dvi- -ps-`. Customize the command via the `$lualatex` variable in `latexmkrc`, e.g. `$lualatex = 'lualatex --shell-escape %O %S';`.
- The CLI matches the other engines: `--shell-escape`, `--interaction=nonstopmode`, `--output-directory`, `--jobname`.
- Runs are slower than pdfLaTeX; that is normal. A pause of minutes right after installing fonts is the font database build, not a hang.
- Main memory is dynamic, so oversized TikZ/pgfplots documents that break pdfLaTeX usually compile unchanged. A few classic arrays stay fixed: on `TeX capacity exceeded, sorry [number of strings=...]`, raise the named parameter inline, `max_strings=1000000 lualatex document.tex`.

## pdfLaTeX leftovers

- LuaTeX 0.85 dropped the `\pdf*` primitives (`\pdfliteral` is now `\pdfextension literal`, `\pdfoutput` is `\outputmode`). For an old document or package that still uses them, make `\RequirePackage{luatex85}` the document's first line.
- microtype: protrusion, expansion, tracking, and ligature disabling work under LuaTeX; `spacing=` and `kerning=` adjustments are pdfTeX-only.
- In sources shared with pdfLaTeX, guard engine-specific code with the iftex package: `\ifLuaTeX ... \else ... \fi` (true on LuaHBTeX too), or fail fast with `\RequireLuaTeX`.

## Sources

- [fontspec manual](https://ctan.org/pkg/fontspec): font selection, features, renderers.
- [unicode-math manual](https://ctan.org/pkg/unicode-math): OpenType math, `\sym...` commands, load order.
- [luaotfload manual](https://ctan.org/pkg/luaotfload): font database, `luaotfload-tool`, modes.
- [babel manual](https://ctan.org/pkg/babel) and the [polyglossia migration guide](https://github.com/latex3/babel/discussions/368): `\babelprovide`, `\babelfont`, bidi, renderer choice.
- [luacode manual](https://ctan.org/pkg/luacode): `\directlua` pitfalls and the escape-safe wrappers.
- [LuaTeX manual](https://www.luatex.org/documentation.html): `tex.print`/`tex.sprint`, shell-escape gating of `os.*`, kpathsea-backed `require`.
- [latexmk manual](https://ctan.org/pkg/latexmk): `-lualatex`, `$lualatex`.
- [microtype manual](https://ctan.org/pkg/microtype): per-engine feature support.
- [TeX Live release notes](https://tug.org/texlive/bugs.html): engine versions per release.
