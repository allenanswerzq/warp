# Fonts — Font Loading & Glyph Pipeline

> Source: `crates/warpui_core/src/fonts.rs` + `crates/warpui_core/src/fonts/`

---

## Key Types

```
FamilyId  → identifies a font family ("JetBrains Mono", "Inter")
FontId    → identifies a specific font (family + weight + style)
GlyphId   → identifies a specific character shape within a font
Weight    → Thin | Light | Normal | Medium | Bold | ... (9 levels)
Properties → weight + style (italic/normal) for font selection
Metrics   → ascent, descent, line height, underline position, ...
```

---

## The Font Pipeline

```
"Hello" needs to be drawn:

1. FamilyId = font_db.family_id_for_name("JetBrains Mono")
2. FontId   = font_db.select_font(family_id, Properties { weight: Bold })
3. For each char:
     GlyphId  = font_db.glyph_for_char(font_id, 'H')
     advance  = font_db.glyph_advance(font_id, glyph_id)  → move cursor
     bitmap   = font_db.rasterize_glyph(font_id, size, glyph_id, ...)
4. Bitmaps go into glyph cache (texture atlas on GPU)
5. Glyph shader draws them on screen
```

---

## Font Fallback

When a font doesn't have a glyph (e.g., emoji in a monospace font):

```
font_db.glyph_for_char(jetbrains_mono, '😀') → None
font_db.fallback_fonts('😀', jetbrains_mono) → [apple_emoji, noto_emoji, ...]
  → try each until one has the glyph
```

---

## Platform Implementations

```
macOS:         Core Text (native)     → mac/fonts.rs
Win/Linux:     cosmic-text + fontdb   → uses fontdb crate
WASM:          cosmic-text            → same as Win/Linux
```
