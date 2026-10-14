---
layout: post
title: "Measuring Fonts: The Terminal Typography Rabbit Hole"
date: 2026-10-14 20:30:00 +0530
categories: linux tools terminal
---

A Hacker News thread about Iosevka sent me down a two-hour font rabbit hole. I emerged with JetBrains Mono — the same font I started with — and a set of measurements I didn't know I needed.

## The trigger

[This HN thread](https://news.ycombinator.com/item?id=50084240) about Iosevka, the open-source monospace font. One commenter called it their "absolutely most loved font for years." Another said they "just don't understand how can people read this."

Both perspectives are correct. The explanation is in the numbers.

## Why Iosevka looks thinner

I measured every font with Python's `fonttools`. Each font's OS/2 table holds two key metrics:

- **advance width** — how wide each character is, as a fraction of the font size
- **x-height** — how tall lowercase letters are, same scale

At the same point size, a lower `adv` means narrower text. A lower `xh` means shorter lowercase letters.

```
JetBrainsMono Nerd Font    adv=0.600 xh=0.550
Iosevka Term               adv=0.500 xh=0.520
Cascadia Code              adv=0.586 xh=0.518
Terminus (TTF)             adv=0.500 xh=0.458
Tamzen                     adv=0.538 xh=0.462
Scientifica                adv=0.455 xh=0.455
```

Iosevka at 13 pt shows 17% narrower letters and 5% shorter lowercase than JetBrains Mono at 13 pt. To match JetBrains Mono's visual size, Iosevka needs 15.6 pt by width or 13.8 pt by height. No single size fixes both — you pick one axis and accept the other.

This is not a rendering problem. It is a design choice. Iosevka is a condensed monospace font, made to fit more text on each line. Whether that is readable to you depends on your display, your eyes, and your tolerance for a cramped screen.

## The bitmap surprise

Terminus comes in two forms: TTF outlines (what I had) and the original pixel-drawn bitmaps. The TTF file I installed actually embeds both — the outlines for scaling and the pixel copies at fixed sizes. I found this with fonttools:

```
$ uv run --with fonttools python3 -c \
  "from fontTools.ttLib import TTFont; f = TTFont('TerminusTTF-4.49.3.ttf'); print(f.keys())"
[... 'EBDT', 'EBLC', ...]
strikes(ppem): [12, 14, 16, 18, 20, 22, 24, 28, 32]
```

`EBDT` and `EBLC` are the bitmap data tables. The strikes are pixel heights. At any other pixel size, the renderer scales the outline version and applies anti-aliasing — which is why 16 pt looked soft. But at exactly 20 px (15 pt on a 96 DPI display), the font renderer uses the pixel-drawn letters directly. No smoothing. Sharp edges.

I made it work in WezTerm with:

```lua
config.font = wezterm.font_with_fallback({
  'Terminus (TTF)',
  'JetBrainsMono Nerd Font',   -- fallback for icons
  'JetBrains Mono',
  'monospace',
})
config.font_size    = 15        -- 15pt × 96/72 = 20px → exact strike
config.dpi          = 96
config.freetype_load_target   = 'Mono'   -- disable anti-aliasing
config.freetype_render_target = 'Mono'
```

The downside: Terminus bitmap strikes top out at 32 px, so zooming with Ctrl+ softens the text again. And the pixel letters felt thin — the vertical strokes are 2 pixels wide at 20 px. At 24 px they become 3 pixels, but you trade off screen real estate.

## What I learned

1. **Measure before judging.** "This font looks thin" has a numerical answer in the OS/2 table. Knowing the advance width and x-height tells you exactly why — and what point size would compensate.

2. **Bitmap strikes are still a thing in 2026.** Terminus TTF embeds them, and FreeType loads them by default unless `NO_BITMAP` is set. This matters for pixel-perfect rendering on low-DPI displays.

3. **Tamzen and Scientifica are outlines.** I installed both, expecting more bitmap fonts from the Linux console tradition. Neither has embedded strikes. The font names (6x12, 8x16) suggest pixel sizes, but those are only the design intent preserved in the outlines — the renderer still anti-aliases them.

4. **All roads lead back.** After two hours, five fonts, eight test windows, and one Xauthority bug, I am back on JetBrains Mono. It is the right weight for my eyes, at the right size for my display. The adventure was still worth it — I now know exactly why every alternative falls short, and I can measure the next one in seconds.

## The measurement script

```python
from fontTools.ttLib import TTFont

f = TTFont('/path/to/font.ttf')
upem = f['head'].unitsPerEm
os2 = f['OS/2']
adv = f['hmtx']['a'][0] / upem
xh = os2.sxHeight / upem
print(f'adv={adv:.3f} xh={xh:.3f}')
```

Save this. Next time you wonder why a font looks wrong, run it and get a number instead of a feeling.

## Source

- [Terminus TTF](https://files.ax86.net/terminus-ttf/) — Dimitar Zhekov's TTF version with embedded bitmap strikes
- [fonttools](https://github.com/fonttools/fonttools) — Python library for reading and manipulating font files
- [WezTerm freetype_load_flags](https://wezterm.org/config/lua/config/freetype_load_flags.html) — docs on bitmap loading and render targets
- [HN thread that started this](https://news.ycombinator.com/item?id=50084240)