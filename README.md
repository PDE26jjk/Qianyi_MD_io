# Qianyi Marvelous Designer IO

A Marvelous Designer `.zpac` importer for the Qianyi Blender add-on.

This package is developed in its own repository so that the add-on's main
repository stays free of Marvelous-Designer-specific code.

## What it does

- Opens the `.zpac` archive and reads the `.pac` payload inside it.
- Recovers every 2D pattern: name, outline points, granularity
  (`fEdgeLength`), 2D placement (`m3Matrix`) and edge topology - straight
  edges, free/Bessel edges (`iLineType == 3`) and cubic-spline edges
  (`iLineType == 2`, with their interior control points).
- Recovers sewing lines (`mapSeamLinePairGroup` -> `mapSeamLinePairList`):
  the pattern, start/end edge, start/end ratio and direction of both sides,
  including the edge-index flip needed after a pattern is re-oriented CCW.
- Builds the result into the active Qianyi node tree via the add-on's
  `IOFormatBase` handler contract (`ext = ".zpac"`).

## Installation

The package uses the add-on's relative imports (`..`, `....`), so it has to
sit inside the add-on package tree:

```
Qianyi/extras/io/marvelous/    <- the contents of this repository
```

No registration code is needed: `Qianyi/extras/io/__init__.py` walks the `io`
package, imports every module and registers each `IOFormatBase` subclass that
declares an `ext`, so dropping this folder into place is enough.

## File format notes

The `.pac` payload is a nested key/value map. Every map starts with a one-byte
property count, then one byte per property name length, one byte per property
type, four bytes per value length, then the names and values; a nested map
(type 22) or a list of maps (type 20) is re-parsed from its own byte range.
`utils.parse_marvelous_designer_map` returns each leaf as
`(type_id, offset, length)` so the importer can decode values on demand.

Property names are ASCII, string values are UTF-8 (`qsNameUTF8`), and the
importer currently accepts `uiVersion >= 1001`.

## Version support

The outer container, the panel payload and the sewing payload are byte-for-byte
the same shape across the versions checked so far; only the entries inside
`listPattern` and the meaning of a few sewing fields changed. Verified in
Blender 4.5.6 (which ships NumPy 1.26.4):

- **uiVersion 1001, Marvelous Designer 2024/2025 export**: parses and imports.
- **uiVersion 1001, Marvelous Designer 2026 export**: fails, for three
  independent reasons:
  1. `np.bool` no longer exists (NumPy removed it in 1.24), so every sewing
     read raises `AttributeError: module 'numpy' has no attribute 'bool'`.
     This one is not version specific - it breaks older exports too. Use
     `np.bool_`.
  2. `listPattern` also holds non-2D entries - button heads, with
     `enPatternType == 2` / `uiPatternType == 3` - which carry neither
     `fEdgeLength` nor `mapShape2D`, so the pattern loop raises
     `KeyError: 'fEdgeLength'`. Skip the entries without `mapShape2D`, and map
     the `listPattern` index to the created-pattern index so the sewing payload
     still resolves.
  3. Sewings that run along an internal line carry `uiDepth == 2` and a
     `uiShapeIndex`. The importer ignores both and reads `uiStartIndex` /
     `uiEndIndex` as outline edge indices, so those seams land on the outline
     and collide with the real ones: `calc_all_sewings_sections()` raises
     `Sewing overlap!!!`. Reading them properly needs the `uiShapeIndex` ->
     internal line mapping; skipping them imports the project but drops those
     seams. Note that older exports use `uiDepth == 2` too - it is a latent
     mis-read there, not a new field.
- **uiVersion < 1001** (for example Marvelous Designer 2020, uiVersion 1000):
  rejected with `Marvelous designer version N not supported!`.

## Requirements

- `numpy`
- A Qianyi add-on checkout (the parent package) to provide `IOFormatBase`,
  `QianyiProject`, `Pattern` and the Blender node-tree helpers.

