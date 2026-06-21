# docs — brand & launch assets

On-brand visual assets for `code-os`, generated with the yempik design + copy skills.
Each PNG ships with its editable source (HTML or SVG) so it can be regenerated or tweaked.

## Assets

| File | Size | Use |
|---|---|---|
| `banner.png` | 2560×840 (@2×) | README header image. Rendered from `banner.html`. |
| `social-preview.png` | 2560×1280 (@2×) | GitHub social preview / Open Graph card. Upload under **Settings → Social preview**. Rendered from `social-preview.html`. |
| `launch-card.png` | 2160×2160 (@2×) | LinkedIn launch card (1080² @2×). Generated from `assets/launch-card.json` via the `yempik-linkedin-post` skill. |

Sources live alongside the PNGs: `banner.html`, `social-preview.html`, `assets/launch-card.{json,html,svg}`, the decorative node-path motifs in `assets/*-motif.svg`, and the yempik logo set in `assets/logo/`.

## Brand tokens (the one accent rule)

| Token | Value |
|---|---|
| Accent (coral) | `#E35B2D` — the **one** accent. Wordmark dot, kicker, links, key emphasis. Use it sparingly. |
| Ink / dark bg | `#0A0A0A` |
| Paper / light bg | `#FAFAF8` |
| Line / node (grey) | `#6F7173` |
| Type | **Geist** (sans), Geist Mono for code |

Guardrails: no second accent, no gradients, no robot/brain/sparkle imagery. **Never render Claude or Anthropic logos** — text only ("for Claude Code"). The yempik wordmark is `yempik` + a coral square "period"; `code-os` echoes it with a coral accent block.

## Regenerating

The decorative motifs come from the `yempik-graphics` skill (pure-Python SVG, no deps):

```bash
python generate.py connector --seed 7 --accent-count 0 --stroke "#6F7173" \
  --width 560 --height 440 --bg transparent --out assets/banner-motif.svg
```

The banner and social card are HTML (real Geist via Google Fonts), rasterized with headless Chrome:

```bash
chrome --headless=new --hide-scrollbars --force-device-scale-factor=2 \
  --virtual-time-budget=4000 --window-size=1280,420 \
  --screenshot=banner.png file://$PWD/banner.html
```

The LinkedIn card uses the `yempik-linkedin-post` skill (Node, no deps):

```bash
node gen_post.js assets/launch-card.json assets/launch-card   # writes .html + .svg
```
