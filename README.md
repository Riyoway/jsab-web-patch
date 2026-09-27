# jsab-web-patch

A patch for the **Just Shapes & Beats** web port.
No game data is included. You apply it to an unmodified copy of the web port with `patch.py`.

## What it fixes

| Problem | Cause and fix |
| --- | --- |
| Every stage has a plain white background | A placeholder shader (`Unlit/TestShaderDangerZone`) draws a full-screen white quad. Its output is masked (colMask 15 → 0), so the stage background is black as intended. |
| The pause and settings screens have a white/grey background | `Custom/GaussianBlur` never gets its input texture on WebGL. Its quad is no longer drawn (colMask 0, zWrite 0), so the frozen game shows through, dimmed, as intended. |
| Settings and progress are never saved | On load, `MetaResolution.cleanToBestResolutionIfInvalid` reads `Screen.resolutions`. That list is empty on WebGL, so the call throws and the game discards the save as corrupted. The function now returns immediately (a 2-byte change in the wasm). |
| `index.html` loads its files from an external CDN | `index.html` now uses `<base href="./">`, so it loads the files next to it. It revalidates the split files (`cache: "no-cache"`) and stores the patched data under new Cache Storage keys (`.v2`). Older cached copies are deleted. |

Files changed:
- `Build/jsab.data.unityweb.part1`–`part4`
- `Build/jsab.wasm.code.unityweb.part1`
- `StreamingAssets/aa/WebGL/modelfla-jsb_interfaces_mainmenu_assets_all_*.bundle`
- `StreamingAssets/aa/catalog.json` (bundle CRC)
- `index.html`

## Usage

Requirements: Python 3.8+ (standard library only).

```sh
python patch.py <web-port-folder> --check   # verify only, changes nothing
python patch.py <web-port-folder>

python -m http.server 8000 -d <web-port-folder>   # then open http://localhost:8000/
```

- **Checks:**
  - The patch targets one specific version of the web port.
  - Every input file is verified by SHA-256 before anything is written. If the version differs or a file was modified, the patch stops with an error and changes nothing.
  - The results are verified too. Running the patch twice is harmless, because finished steps are skipped.
- **Line endings:** text files that were checked out with CRLF line endings (git `core.autocrlf`) are converted back to LF before patching.
- **Hosting:**
  - The patched folder works on any static host, e.g. Cloudflare R2, GitHub Pages or `python -m http.server`.
  - Browsers that already played the unpatched version on the same URL get the new data automatically, because of the new cache keys.

## How it works

`patch/manifest.json` lists each step. For every step it records:
- the target file
- the edit positions
- the original and new bytes
- the file hashes before and after

Inside the `UnityWebData1.0` pack, the patch re-compresses only the edited LZ4 blocks of each UnityFS bundle. After an Addressables bundle is edited, its CRC in `catalog.json` is updated.

## License

The scripts and the patch description are released under the MIT License (see `LICENSE`). Just Shapes & Beats and its assets belong to Berzerk Studio. None of them are included here.
