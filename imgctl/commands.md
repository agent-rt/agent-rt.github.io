---
title: Commands · imgctl
---

# Commands

`imgctl` ships 16 subcommands grouped by purpose. Every command supports `--json` for structured output and emits stable error codes on failure.

## Global options

```
--json                  Structured JSON output (default: TSV-like human)
--quiet                 Suppress informational metadata (still on stderr)
-i, --input <PATH>      Input file; `-` for stdin
-o, --output <PATH>     Output file; `-` for stdout
```

When `-o -` (stdout) is set, binary image data goes to stdout and structured metadata is automatically routed to stderr — so you can `imgctl convert ... -o - | ssh host 'cat > out.png'` while still capturing JSON metadata via `2>`.

## Edit

| Command | Purpose |
|---|---|
| `imgctl convert -i ... -o ... [--quality N]` | Re-encode between formats (PNG, JPEG, WebP, BMP, GIF, TIFF, ICO) |
| `imgctl resize -i ... -o ... [--width W --height H --fit contain/cover/fill]` | Scale with aspect-ratio aware fit modes |
| `imgctl crop -i ... -o ... --x X --y Y --w W --h H` | Rectangular crop |
| `imgctl compose -i base ... --over overlay --x X --y Y` | Layer one image over another |
| `imgctl info -i ...` | Width, height, format, color space, byte size |
| `imgctl metadata -i ...` | EXIF read (camera, exposure, GPS, …) |

## Annotate

| Command | Purpose |
|---|---|
| `imgctl annotate text --x X --y Y --text "..." [--font-size N --color C]` | Overlay text |
| `imgctl annotate arrow --from X,Y --to X,Y [--color C --width N]` | Draw arrow with arrowhead |
| `imgctl annotate box --x X --y Y --w W --h H [--stroke C --width N]` | Draw rectangle (outline or filled) |
| `imgctl annotate blur --x X --y Y --w W --h H [--sigma N]` | Gaussian blur a region (privacy / focus) |
| `imgctl annotate composite ...` | Stack multiple annotations in a single command (less I/O) |

## Analyze

| Command | Purpose |
|---|---|
| `imgctl compare -i a -i b --metric ssim/psnr/mse` | Quantitative similarity |
| `imgctl diff -i baseline -i current -o diff.png --threshold T` | Visual diff with pass/fail exit codes (CI-ready) |
| `imgctl histogram -i ... [--channels rgb/luminance]` | Per-channel value distribution |
| `imgctl palette -i ... [--colors N]` | Dominant colors via k-means |

## Diagram

| Command | Purpose |
|---|---|
| `imgctl mermaid -i diagram.mmd -o ... [--width W --theme T]` | Render Mermaid (headless Chrome under the hood) |

## Output formats

### Default (TSV-like human)

```
OPERATION  INPUT       OUTPUT      STATUS  WIDTH  HEIGHT  BYTES
resize     in.png      out.png     ok      400    300     45123
```

Columns are tab-separated, header on first line — directly consumable by `cut`, `awk`, or `column -t`.

### `--json`

```json
{
  "operation": "resize",
  "input": "in.png",
  "output": "out.png",
  "status": "ok",
  "width": 400,
  "height": 300,
  "bytes": 45123
}
```

Errors:

```json
{
  "error": {
    "code": "INVALID_DIMENSIONS",
    "message": "width must be > 0",
    "argument": "width"
  }
}
```

## Stable error codes

| Code | When |
|---|---|
| `IO_ERROR` | File missing, permission denied, disk full, etc. |
| `UNSUPPORTED_FORMAT` | Format extension or magic bytes not recognized |
| `DECODE_ERROR` | File looks like format X but bytes don't parse |
| `INVALID_DIMENSIONS` | Width/height/x/y out of range or zero where positive required |
| `INVALID_COLOR` | Color string not parseable (`#abc`, `rgb(...)`, named) |
| `CHROME_TIMEOUT` | Headless Chrome process exceeded mermaid render budget |
| `CHROME_NOT_FOUND` | mermaid invoked but no Chrome/Chromium binary on `PATH` |
| `MERMAID_PARSE_ERROR` | Mermaid source rejected by the renderer |

Exit code is `1` for most errors, `2` for usage/argument errors.

## Dual-channel I/O example

Pipe binary out, capture metadata separately:

```sh
imgctl convert -i in.png -o - --quality 80 --json 2> meta.json | \
  curl -X POST -H 'Content-Type: image/jpeg' --data-binary @- https://example.com/upload

cat meta.json
# {"operation":"convert","output":"-","status":"ok","width":...}
```
