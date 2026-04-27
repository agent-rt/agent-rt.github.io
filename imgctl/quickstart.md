---
title: Quickstart · imgctl
---

# Quickstart

Goal: in two minutes, convert a format, resize with structured output, annotate, and render a Mermaid diagram. Then see how an agent consumes the results.

Prereq: `imgctl` installed (see [Install](/imgctl/install/)).

## 1. Inspect an image

```sh
imgctl info -i photo.jpg --json
# {
#   "path": "photo.jpg",
#   "width": 1920,
#   "height": 1080,
#   "format": "jpeg",
#   "color": "rgb",
#   "bytes": 287456
# }
```

Stable shape: agents can `jq '.width'` without parsing prose.

## 2. Convert + resize

```sh
imgctl convert -i photo.jpg -o photo.webp --quality 85
imgctl resize  -i photo.webp -o thumb.webp --width 400 --fit contain
```

Default human output is one TSV line per operation:

```
OPERATION  INPUT       OUTPUT      STATUS
convert    photo.jpg   photo.webp  ok
resize     photo.webp  thumb.webp  ok
```

`--json` returns the same data as a JSON object per command. `--quiet` suppresses metadata entirely (still emits to stderr).

## 3. Annotate

```sh
imgctl annotate text -i thumb.webp -o thumb.png \
  --x 20 --y 30 --text "Q4 launch" --font-size 24 --color red

imgctl annotate arrow -i thumb.png -o thumb.png \
  --from 100,50 --to 200,150 --color "#0070f3" --width 3

imgctl annotate box -i thumb.png -o thumb.png \
  --x 80 --y 60 --w 140 --h 100 --stroke red --width 2
```

All annotate subcommands compose — pipe one into the next via shared file paths, or chain with stdin/stdout (`-` for I/O).

## 4. Mermaid → PNG

```sh
cat > /tmp/flow.mmd <<'EOF'
graph LR
  Agent -->|skillctl list| Gateway
  Gateway -->|TSV| Agent
  Agent -->|skillctl show id| Gateway
  Gateway -->|SKILL.md| Agent
EOF

imgctl mermaid -i /tmp/flow.mmd -o /tmp/flow.png --width 1200
```

This spawns headless Chrome via `chromiumoxide`, renders the Mermaid SVG to PNG, returns metadata about the result.

## 5. Visual diff (agent-friendly screenshot regression)

```sh
imgctl diff -i baseline.png -i current.png -o diff.png --threshold 0.05 --json
# {
#   "changed_pixels": 1247,
#   "total_pixels": 2073600,
#   "ratio": 0.0006,
#   "threshold": 0.05,
#   "exceeded": false,
#   "diff_path": "diff.png"
# }
```

Exit code is `0` when within threshold, `1` when exceeded — directly drives CI/CD gates.

## 6. Stable error codes

```sh
imgctl convert -i nonexistent.png -o out.png --json
# {"error":{"code":"IO_ERROR","message":"...","path":"nonexistent.png"}}
echo $?   # → non-zero
```

Codes are documented and stable. Agents can branch on `code` without prose matching:

| Code | Meaning |
|---|---|
| `IO_ERROR` | File I/O failure (missing, permissions, …) |
| `UNSUPPORTED_FORMAT` | Format not recognized |
| `INVALID_DIMENSIONS` | Width/height/x/y out of range |
| `CHROME_TIMEOUT` | Headless Chrome did not respond (mermaid only) |
| `DECODE_ERROR` | Bytes do not parse as the claimed format |

## What's next

- [Commands](/imgctl/commands/) — full reference for all 16 subcommands.
- Use alongside [`memctl`](/memctl/) — capture visual-diff thresholds as `feedback` entries; recall them across project work.
