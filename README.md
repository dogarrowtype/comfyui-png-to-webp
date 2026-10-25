# comfyui-png-to-webp

Have a large comfyui backlog? Need to turn those PNGs into WEBPs to save space, while keeping the workflow? Use this.

Batch-convert ComfyUI PNG generations to lossy WebP (quality 100, slowest
encoder) for archiving, dumping each image's embedded ComfyUI **workflow** to
a sidecar `.json` file. Built for very large folders: it encodes in parallel
across all CPU cores, is safe to interrupt and re-run, and only deletes a
source PNG after its replacements are verified.

```
photo.png  ──►  photo.webp   (lossy, q100, method 6 = slowest)
           └─►  photo.json   (embedded workflow, verbatim)
           (photo.png deleted after verification — use --keep to retain it)
```

The output settings mirror ComfyUI's own WebP save node at
`quality: 100, lossless: false, method: slowest` (`cwebp -q 100 -m 6`).
Typical size reduction vs. PNG is ~80–90%. The sidecar `.json` can be dragged
into the ComfyUI UI to restore the workflow.

## Requirements

* Linux or macOS, bash 3.2+
* `perl`, `curl` or `wget`, `tar` — all present on a stock system
* **No other installs needed**: `exiftool` and `cwebp` are downloaded
  automatically (checksum-verified) into `./tools` on first run. See
  [Dependency bootstrap](#dependency-bootstrap).

## Usage

```bash
./comfyui-png-to-webp /path/to/generations          # convert + delete PNGs
./comfyui-png-to-webp --keep /path/to/generations   # convert, keep PNGs
./comfyui-png-to-webp -j 16 /path/to/generations    # 16 parallel encoders
./comfyui-png-to-webp --dry-run /path/to/generations
```

| Option | Description |
| --- | --- |
| `-j, --jobs N` | Parallel encoders (default: all CPU cores) |
| `-k, --keep` | Keep source PNGs after successful conversion |
| `-q, --quality N` | WebP quality 0–100 (default: 100) |
| `-m, --method N` | cwebp encoding effort 0–6 (default: 6 = slowest) |
| `-n, --dry-run` | Print files that would be processed, change nothing |
| `--no-bootstrap` | Fail instead of downloading missing tools |
| `-h, --help` | Show help |

The directory argument defaults to the current folder. Processing is
**non-recursive**; run once per folder. Exit status is `0` only if every file
succeeded or was skipped, so runs can be chained from other scripts.

## Behavior and safety

* **Verified before deletion** — each `.webp` is decode-checked with
  `webpinfo` after encoding; the source PNG is deleted only when both the
  WebP *and* the sidecar JSON exist and verify. With `--keep`, nothing is
  ever deleted.
* **Crash-safe** — outputs are written to temporary files and atomically
  renamed into place. Interrupting a run (Ctrl-C, power loss) never leaves a
  half-written `.webp`/`.json` behind and never loses a source PNG.
* **Idempotent** — re-running skips files where both outputs already exist,
  and repairs partial states (e.g. JSON present but WebP missing). Large
  folders can be processed incrementally.
* **Fail-safe on metadata** — a PNG without an embedded workflow is reported
  as `[FAIL]`, left untouched, and the run exits non-zero, rather than
  silently archiving without its workflow.
* **Filenames** — spaces, unicode, and newlines are handled (NUL-delimited
  piping throughout).

## Dependency bootstrap

If `exiftool`/`cwebp` are not found on `PATH`, they are fetched once into
`tools/` next to the script:

| Tool | Source | Notes |
| --- | --- | --- |
| `cwebp`, `webpinfo` (libwebp 1.6.0) | downloads.webmproject.org | Official prebuilt **static** binaries for linux-x86-64, linux-aarch64, mac-x86-64, mac-arm64 — no libc issues |
| `exiftool` 13.59 | exiftool.org, fallback: GitHub | Runs on the system `perl` via a relocatable wrapper |

Downloads are **SHA-256 verified against hashes pinned in the script**;
verification failure aborts before anything is executed. The `tools/`
directory is laid out per-platform, so the script folder can be shared
between machines with different operating systems or architectures (each
machine fetches only its own binaries). Concurrent first runs are guarded by
a lock.

* Relocate the tools directory with `CPTW_TOOLS_DIR=/some/path`.
* `--no-bootstrap` disables network access entirely (also useful for
  air-gapped machines: copy `tools/` over from another machine).
* To slim `tools/`, everything inside `tools/libwebp-*/` except
  `bin/cwebp` and `bin/webpinfo` can be deleted.

## How it works

1. `exiftool -PNG:Workflow` extracts the workflow tEXt chunk verbatim to
   `<name>.json`.
2. `cwebp -q 100 -m 6` encodes `<name>.webp` (lossy, maximum effort).
3. `webpinfo` verifies the encoded file decodes.
4. The source PNG is deleted (unless `--keep`).

Parallelism is fan-out per file via `xargs -0 -P`, one PNG per process — the
encoder is single-threaded per image and throughput scales with core count.

## Sample output

```
$ ./comfyui-png-to-webp -j 8 ~/comfy/archive/2026-10
comfyui-png-to-webp: 1240 PNG file(s) in ~/comfy/archive/2026-10; jobs=8 quality=100 method=6 keep=0 dry_run=0
[OK] ~/comfy/archive/2026-10/ComfyUI_02516_.png
[OK] ~/comfy/archive/2026-10/ComfyUI_02517_.png
[FAIL] ~/comfy/archive/2026-10/manual_export.png: no workflow metadata found
...
comfyui-png-to-webp: done. ok=1238 skip=1 fail=1 (total=1240)
```
