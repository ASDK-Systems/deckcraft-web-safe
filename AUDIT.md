# Audit: deckcraft-web v0.18.1

This repo re-hosts DeckCraft's web/WASM build for embedding in [VARKIS](https://github.com/ASDK-Systems)'s
Office Studio, and this file records a static security audit of that build, done
before it was uploaded here. Companion to [wordcraft-web-safe](https://github.com/ASDK-Systems/wordcraft-web-safe)
and [gridcraft-web-safe](https://github.com/ASDK-Systems/gridcraft-web-safe)'s own
`AUDIT.md`, same methodology.

## What was checked

Downloaded `deckcraft-web-0.18.1.zip` directly from
[storytold/deckcraft](https://github.com/storytold/deckcraft)'s `v0.18.1` release,
verified it against that release's own `SHA256SUMS.txt`, then read every line of:

- `index.html` (1.8 KB)
- `deckcraft-web-f863e8c546b80fd1.js` - the wasm-bindgen glue (175 KB)
- `snippets/eframe-d31a8450ea12dfc1/inline0.js` (3 lines)
- `_headers`, `HOSTING.md`

and grepped the raw `.wasm` binary's string table (33 MB) for any `http(s)://`-shaped
string, since the glue JS is the only surface through which compiled Rust can reach
the network at all - a `.wasm` module has no ambient network access of its own.

## Findings

**No outbound network call reachable from any JS import.** The glue JS has exactly
one `fetch()` call - the module's own same-origin load of its `.wasm` file, the
normal wasm-bindgen `init()` sequence. No `.src=` binding at all this time (unlike
wordcraft/gridcraft, which each had one WebGPU-adjacent one). `postMessage` appears
only as the generic wasm-bindgen wrapper around the browser's own `window.postMessage`
API, used by the optional `?host=parent` embedding bridge documented in `HOSTING.md`
(the same `craft-web-host` contract gridcraft and wordcraft share, confirmed by
HOSTING.md's own text). VARKIS does not use the cross-origin mode - the iframe is
always same-origin - and VARKIS's own listener checks the message's `source`/`origin`
before acting on it.

**No Python/Pyodide/CDN dependency.** Grepped the glue JS for `pyodide`, `python`,
`cdn.`, `jsdelivr`, `unpkg`, `cloudflare` - zero matches.

**The `.wasm` binary's only URL-shaped strings are inert.** OOXML/PPTX namespace
constants (`schemas.microsoft.com`, `schemas.openxmlformats.org`), PDF/XMP export
metadata namespace URIs (from the bundled PDF-export library's own string table),
PowerPoint's internal `ppaction://` hyperlink-action URI scheme (slide-navigation
constants, not network-reachable), WGSL shader-spec documentation links, and the
usual static about-box/credit links (Discord, getartcraft.com, GitHub issues for
egui/wgpu and the PDF-export library). None of these are reachable from any JS
import.

**Conclusion: nothing to strip.** Like `wordcraft-web-safe` and `gridcraft-web-safe`,
this is a verbatim mirror of upstream's own build, not a modified one - no
Apache-2.0 4(b) "changed files" notice applies.

## What this repo hosts

`LICENSE-MIT` and `LICENSE-APACHE` are copied verbatim from
`storytold/deckcraft@v0.18.1`. The `v0.18.1` web build release asset is live:
uploaded via `gh release create`, then re-downloaded fresh and confirmed
byte-for-byte identical to the pinned SHA-256 both before upload and after.
VARKIS's own `ensureWebBuild('deckcraft')` was run directly against this repo's
release URL (cache cleared first) and verified it downloads, verifies and extracts
correctly.
