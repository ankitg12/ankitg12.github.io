---
layout: post
title: "Building a 234-page PDF book from a 795-file mkdocs site"
date: 2026-05-30
categories: ai tools productivity windows
---

I wanted a printable PDF of [agentpatterns.ai](https://agentpatterns.ai) — 795 markdown files across 16 sections, maintained as a mkdocs site. This is the bug log of getting that working on Windows with pandoc + xelatex.

---

## The goal

- Single PDF, book-formatted (chapters, TOC, page numbers)
- Colored chapter and section headings — readable on a color printer
- Mermaid diagrams rendered as actual diagrams, not raw `flowchart LR` text
- Reproducible build: `python build.py` → PDF

---

## Why pandoc + xelatex

mkdocs generates a website. For print-quality PDF from markdown, pandoc + a LaTeX engine is the standard path. [Eisvogel](https://github.com/enhuiz/eisvogel) is the go-to pandoc LaTeX template — book layout, chapter headings, code highlighting, title page.

Alternatives I tried and abandoned:

| Tool | Why it failed |
|------|--------------|
| `mkdocs-with-pdf` | Uses WeasyPrint which needs GTK on Windows — heavy native dependency |
| Browser print-to-PDF | 795-page HTML file hangs Chrome before it finishes loading |
| Typst (default engine) | Pandoc converts `[@cite]` markdown to `#cite()` typst; fails without a bibliography |

xelatex is the boring choice. It works.

---

## Bug log

### WinError 206: filename too long

Pandoc accepts input files as CLI arguments. 795 paths × ~80 chars each = 63k chars. Windows caps `CreateProcess` at ~32k chars.

```
FileNotFoundError: [WinError 206] The filename or extension is too long
```

**Fix:** Use pandoc's `--defaults` file. Move `input-files` into a YAML file and pass `pandoc --defaults=tmpfile.yaml`. The file path stays short; the content is unlimited.

```python
defaults = {
    "input-files": [str(f) for f in files],   # 795 paths, in the YAML file
    "from": "markdown",
    "to": "pdf",
    ...
}
defaults_file.write_text(yaml.dump(defaults))
subprocess.run(["pandoc", f"--defaults={defaults_file}"])
```

### `lua-filters` is not a valid defaults key

```
Error in $: Unknown option "lua-filters"
```

The pandoc defaults file does not accept `lua-filters` (plural) or `lua-filter` (singular) as keys. Pass lua filters as CLI arguments instead:

```python
extra_args = [f"--lua-filter={path_to_filter}"]
subprocess.run(["pandoc", f"--defaults={defaults_file}"] + extra_args)
```

### cp1252 decode crash

MiKTeX outputs non-cp1252 bytes to stderr. `subprocess.run` with `text=True` uses the system default encoding:

```
UnicodeDecodeError: 'charmap' codec can't decode byte 0x9d
```

**Fix:** Explicit encoding:
```python
subprocess.run([...], encoding="utf-8", errors="replace")
```

### Bell character in source markdown

```
! Text line contains an invalid character.
l.100460 ...7;notify;Claude Code;Build finished^^G
```

A code block in the source contained a bell character (0x07 / `^^G`) — likely copied from terminal output. The existing lua filter stripped control characters from `Str` nodes but not `Code` or `CodeBlock` nodes.

**Fix:** Extend the lua filter:

```lua
local CTRL = "[\x00-\x08\x0b\x0c\x0e-\x1f\x7f]"

function Str(el)      el.text = el.text:gsub(CTRL, "") ; return el end
function Code(el)     el.text = el.text:gsub(CTRL, "") ; return el end
function CodeBlock(el) el.text = el.text:gsub(CTRL, "") ; return el end
```

The root cause matters: the character was in a `CodeBlock`, not a `Str`. Extending the filter to only `Str` would have left the bug in place.

### Typst: `#cite` without bibliography

Typst is faster than xelatex (~8s vs ~45s per section). But pandoc converts `[@label]` markdown citation syntax to `#cite(<label>)` in typst, which requires a bibliography declaration.

**Fix:** Add `citeproc: true` to the pandoc defaults. Pandoc processes citations to inline text before generating typst output — no `#cite()` calls reach the typst engine.

```python
defaults["citeproc"] = True
```

### Mermaid diagrams rendered as code blocks

Mermaid fenced blocks (```` ```mermaid ```` ) pass through pandoc as code blocks. xelatex renders them as monospaced text.

**Fix:** Pre-process with `mmdc` (mermaid CLI). For each markdown file containing mermaid blocks, run:

```bash
mmdc -i input.md --outputFormat=pdf --pdfFit -o preprocessed.md
```

`mmdc` replaces mermaid blocks with PDF image references. The preprocessed file and images land in the same directory. On Windows, `mmdc` is an npm `.cmd` shim — subprocess needs the full path:

```python
subprocess.run(
    [r"C:\Users\...\AppData\Roaming\npm\mmdc.cmd",
     "-i", str(path), "--outputFormat=pdf", "--pdfFit", "-o", str(out_md)],
    ...
)
```

### Mermaid images not embedded

The preprocessed markdown references images as `./name-1.pdf` (relative to the preprocessed file in `mermaid-cache/`). Pandoc resolves image paths relative to CWD, not the source file, even with `--file-scope`.

**Fix:** Add the cache directory to `resource-path`:

```python
defaults["resource-path"] = [str(mermaid_cache_dir), str(repo_path)]
```

---

## Colored chapter and section headings

Eisvogel supports LaTeX customization via `header-includes` in the metadata YAML. Add `sectsty`:

```yaml
header-includes:
  - \usepackage[HTML]{xcolor}
  - \definecolor{ap-orange}{HTML}{E8381A}
  - \definecolor{ap-blue}{HTML}{1A5276}
  - \usepackage{sectsty}
  - \chapterfont{\color{ap-orange}}
  - \sectionfont{\color{ap-blue}}
  - \subsectionfont{\color{ap-blue}}
```

---

## Result

```
python build.py
# Sections: 16, Files: 795
# Mermaid: pre-rendered N files via mmdc
# Done in 9m30s -> agentpatterns-encyclopedia.pdf
```

- 234 pages, 10.2 MB
- Orange chapter headings, blue section headings
- Mermaid flowcharts rendered as vector diagrams
- Reproducible: one command

---

## What I'd do differently

**Don't start with the default pandoc approach** for large multi-file inputs on Windows. The WinError 206 trap is invisible until you hit it. Start with `--defaults` from day one.

**Pre-process mermaid before the build**, not inside the build. The mmdc step is slow (~3s/file) and sequential. It belongs in a separate cacheable step with a content-hash cache key.

**Verify each fix before moving to the next one.** I spent 30 minutes trying mkdocs-with-pdf and browser print-to-PDF because I hadn't confirmed the mermaid embedding was working. A 2-minute visual check would have saved the detour.

---

## Source

Build scripts: `agentpatterns-build/` (private — available on request).
Dependencies: `pandoc 3.9`, `xelatex` (MiKTeX), `mmdc 11.12`, `python 3.14`, `pyyaml`.
