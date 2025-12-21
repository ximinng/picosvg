# Feature: `--allow_all_defs` Flag

## Problem

picosvg rejects `<filter>`, `<mask>`, `<pattern>`, `<switch>`, `<symbol>` elements with "BadElement" errors, even when users want to preserve them as-is.

## Solution

New flag `--allow_all_defs` enables pass-through for these elements without processing.

## Usage

```bash
# CLI
picosvg --allow_all_defs input.svg

# Python API
svg = SVG.parse("input.svg")
result = svg.topicosvg(allow_all_defs=True)
```

## What's Preserved

| Location | Elements |
|----------|----------|
| Inside `<defs>` | filter, mask, pattern, clipPath, marker, symbol, and all children |
| Root level | switch, symbol, foreignObject, use, style, pattern, mask |

## Files Changed

### `src/picosvg/svg.py`

**`checkpicosvg()`** - Added regex patterns to `path_allowlist`:
```python
if allow_all_defs:
    # Allow any element in defs with arbitrary nesting
    path_allowlist.add(r"^/svg\[0\]/defs\[0\]/[a-zA-Z]+\[\d+\](/[a-zA-Z]+\[\d+\])*$")
    # Allow switch/symbol/foreignObject/use at root level
    path_allowlist.add(r"^/svg\[0\](/(switch|symbol|foreignObject|use)\[\d+\])+(/[a-zA-Z]+\[\d+\])*$")
    # Allow style/pattern/mask at root level
    path_allowlist.add(r"^/svg\[0\]/(style|pattern|mask|clipPath)\[\d+\](/[a-zA-Z]+\[\d+\])*$")
```

**`_simplify()`** - Skip removing non-gradient defs:
```python
if not allow_all_defs:
    for unused_el in [el for el in defs if not _is_gradient(el)]:
        defs.remove(unused_el)
```

**`_resolve_clip_path()`** - Bug fix:
```python
if not clip_paths:
    return SVGPath()  # was returning None
```

### `src/picosvg/picosvg.py`

```python
flags.DEFINE_bool("allow_all_defs", False, "Allow all defs elements...")
svg.topicosvg(allow_text=FLAGS.allow_text, allow_all_defs=FLAGS.allow_all_defs, ...)
```

### `tests/svg_test.py`

Added 26+ test cases covering filter primitives, masks, patterns, symbols, switch elements.

## Bug Fix

Fixed `_resolve_clip_path()` returning `None` when clip path contains no shapes, now returns empty `SVGPath()`.
