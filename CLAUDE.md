# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ink/Stitch is a professional-grade machine embroidery design platform built as an Inkscape extension. It converts SVG designs into stitch files (PES, DST, JEF, etc.) using Python algorithms for fill patterns, satin columns, and path optimization.

**License:** GPLv3 - modifications must be open-sourced under the same license.

## Build Commands

```bash
# Generate INX extension files (required for testing in Inkscape)
make inx

# Full distribution build
make dist

# Generate localization files
make locales

# Run tests
make test
pytest tests/test_specific.py  # Run single test file

# Style check (PEP8)
make style

# Type checking
make mypy
```

## Architecture

### Entry Point Flow
`inkstitch.py` → parses `--extension=<name>` → instantiates `lib/extensions/<Name>` class → calls `extension.run()`

Extension class names are derived from argument: `foo_bar` → `FooBar`

### Core Modules

**lib/extensions/** - 80+ Inkscape extension implementations. All inherit from `InkstitchExtension`.

**lib/elements/** - Embroidery element types:
- `Stroke` - Running stitch along paths
- `FillStitch` - Complex fill algorithms (auto_fill, contour_fill, meander_fill, etc.)
- `SatinColumn` - Precise column stitching with rail/rung system
- `Clone`, `TextObject`, `ImageObject` - Special element handling

**lib/stitches/** - Core stitching algorithms:
- `auto_fill.py` - Main fill algorithm with grating/graph optimization, pull compensation
- `auto_satin.py` - Automatic satin column generation
- `contour_fill.py`, `meander_fill.py`, `circular_fill.py`, `guided_fill.py` - Fill variants
- `tartan_fill.py` - Tartan/plaid pattern generation
- `cross_stitch.py` - Cross stitch pattern generation

**lib/stitch_plan/** - Data structures for stitch output:
- `StitchPlan` → `ColorBlock` → `Stitch` hierarchy
- `StitchGroup` - Stitches from single element
- Conversion: Elements → StitchGroups → StitchPlan → Output file

**lib/svg/** - SVG handling, transform normalization, path parsing

**lib/gui/** - wxPython dialogs, Flask-based simulator and UI components

**lib/threads/** - Thread color catalogs and matching algorithms

### Key Dependencies
- **inkex** - Inkscape extension framework (bundled version from Inkscape 1.4.1)
- **pystitch** - Embroidery format read/write (wraps pyembroidery, 40+ formats)
- **shapely** - Geometric computations (polygons, intersections)
- **networkx** - Graph algorithms for path optimization
- **wxPython** - Native GUI dialogs
- **Flask** - Local web server for interactive UIs

### Configuration

**Document metadata** - Stored in SVG `<metadata>` as JSON: `min_stitch_len_mm`, `collapse_len_mm`, `thread-palette`, `rotate_on_export`

**User settings** - `settings.json` in platform-specific user directory

**Development** - Copy `DEBUG_template.toml` to `DEBUG.toml`:
- Enable debugger: `debug_type = "vscode"`, `debug_enable = true`
- Enable profiler: `profiler_type = "pyinstrument"`, `profile_enable = true`
- Collect types: `profiler_type = "monkeytype"`, `profile_enable = true`

### INX Generation

Extension UI definitions are generated via Jinja2 templates:
- Templates: `lib/inx/*.py`
- Output: `inx/*.inx`
- Run `make inx` after modifying templates

## Code Style

- PEP8 with focus on readability over brevity
- Type annotations encouraged (see `mypy.ini`)
- Longer, expressive variable names preferred
- Code comments expected for complex stitch algorithms
- Libraries in `lib/stitches/` separate algorithms from element classes

## Testing

Tests use pytest, located in `tests/`. Configuration in `pytest.ini`.

```bash
pytest                           # Run all tests
pytest tests/test_output.py      # Single file
pytest -k "test_name"            # By name pattern
```

## Coordinate Systems

- SVG coordinates: pixels
- Embroidery coordinates: tenths of millimeters (for pystitch)
- Use conversion utilities when interfacing with pystitch

## Pre-commit Hook (Optional)

Add to `.git/hooks/pre-commit` for automatic style checking:
```bash
#!/bin/bash
cd $(dirname "$0")/../..
errors=$(make style 2>&1)
if [ "$?" != "0" ]; then
    echo "$errors"
    exit 1
fi
```

## Debugging Tips

- Copy `DEBUG_template.toml` to `DEBUG.toml` before debugging
- Use `INKSTITCH_OFFLINE_SCRIPT=true` environment variable for standalone testing
- Logs written to `inkstitch.log` in project directory when debug logging enabled
