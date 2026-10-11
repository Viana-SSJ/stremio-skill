---
name: stremio
description: Control Stremio on macOS and Windows end to end. Use when the user wants to watch a movie or show ("play X", "quero ver tal"), find streams, install or vet addons safely, or diagnose the Stremio app/server (port 11470, playback failures, no streams found).
---

# Stremio control (macOS and Windows)

Goal: when the user names something they want to watch, do the whole flow yourself and hand them a playing video. Never ask which addon or which stream to pick unless something is genuinely ambiguous.

## Environment

Detect the OS first and use the matching column below. Everything else in this skill (Cinemeta, Torrentio, ranking, addon vetting) is plain HTTP and identical on both.

| | macOS | Windows (PowerShell) |
|---|---|---|
| App | `/Applications/Stremio.app` | `%LOCALAPPDATA%\Programs\LNV\Stremio-4\stremio.exe` |
| Streaming server process | `node .../server.js` | `stremio-runtime.exe` (bundled node running `server.js`) |
| Server data dir (`<data>`) | `~/Library/Application Support/stremio-server/` | `%APPDATA%\stremio\stremio-server\` |
| Open a deep link | `open "stremio://..."` | `Start-Process "stremio://..."` |
| Who owns port 11470 | `lsof -i :11470 -P` | `Get-NetTCPConnection -LocalPort 11470 -State Listen \| % { Get-Process -Id $_.OwningProcess }` |
| Restart the app | `killall Stremio 2>/dev/null; open -a Stremio` | `Stop-Process -Name stremio,stremio-runtime -ErrorAction SilentlyContinue; Start-Process "$env:LOCALAPPDATA\Programs\LNV\Stremio-4\stremio.exe"` |
| Clear stream cache | `rm -rf ~/Library/Application\ Support/stremio-server/stremio-cache/*` | `Remove-Item "$env:APPDATA\stremio\stremio-server\stremio-cache\*" -Recurse -Force` |
| HTTP client | `curl` | `curl.exe` (always with `.exe`: in Windows PowerShell 5.1 `curl` is an alias for `Invoke-WebRequest`) |

- The streaming engine listens on `*:11470` (loopback access via `http://127.0.0.1:11470`).
- Verify server is alive: check the port (table above) and `curl -s http://127.0.0.1:11470/local-addon/manifest.json` (must return the `org.stremio.local` manifest JSON).
- Settings: `<data>/server-settings.json` (torrent limits, transcode, cache size). Stream cache lives in `<data>/stremio-cache/`.
- Addon lists are stored in the GUI app's own storage, not in `database.db` (which is usually empty). You cannot reliably enumerate installed addons from disk; test addons directly over HTTP instead.
- Deep links (open with the command from the table):
  - Install an addon: `stremio://<manifest-url>` (e.g. `stremio://https://torrentio.strem.fun/manifest.json`)
  - Open a movie/show page: `stremio://detail/movie/tt0383028`, `stremio://detail/series/tt0903747/1/1`

## Watch something (default flow)

1. **Resolve the IMDb ID** via Cinemeta:
   ```sh
   curl -s "https://v3-cinemeta.strem.io/catalog/movie/top/search=<url-encoded-title>.json"
   # series: replace movie with series; for episodes get the meta detail:
   curl -s "https://v3-cinemeta.strem.io/meta/series/tt0903747.json"
   ```
   Pick the match with the right year. Confirm the title back to the user in one line.
2. **Check for streams** from Torrentio (the standard stream source) before touching anything else:
   ```sh
   curl -s "https://torrentio.strem.fun/stream/movie/tt0383028.json"          # movie
   curl -s "https://torrentio.strem.fun/stream/series/tt0903747:1:1.json"     # show S1E1
   ```
3. **Rank the streams** from the returned `streams[].title` lines:
   - Prefer: `REMUX` > `BluRay` > `WEB-DL` > `WEBRip`. Reject `CAM`, `TS`, `TELESYNC`, ` HDC `, `Screener`, `DVDScr`.
   - Resolution: default to 1080p; offer 2160p (4K) only if present and the user asked or has the bandwidth; 720p as last resort.
   - Prefer encodes with `DDP5.1`/`DD5.1`/`DTS`/`AAC 5.1` audio, and `x264`/`H264` for max compatibility on older Macs; `x265`/`HEVC` is fine on Apple Silicon.
   - Tell the user the top pick in one line (name + resolution + source).
4. **Open it**: open the deep link `stremio://detail/movie/tt0383028` (command per OS in the table above). Stremio usually auto-selects the top-ranked stream; if the user reports it picked a bad one, they can pick from the stream list on the detail page.
5. **If zero streams come back**, the addon is likely not installed. Install Torrentio (vetted, see below), then re-check. If Torrentio is installed but returns zero for a known title, say so honestly — no source exists right now; do not hunt sketchy mirrors.

## Installing addons — safety rules (non-negotiable)

The user's hard requirement: never install anything with malware risk. Follow this order:

1. **Allow-list.** These manifests are known-good; install on request without further ceremony:
   - Cinemeta (metadata): `https://v3-cinemeta.strem.io/manifest.json`
   - Torrentio (streams): `https://torrentio.strem.fun/manifest.json` (configurable: `torrentio.strem.fun/providers=yts,eztv,rarbg,1337x,thepiratebay,kickasstorrents,torrentgalaxy/magnetal=true/manifest.json`)
   - OpenSubtitles v3 (subtitles): `https://v3-opensubtitles.strem.io/manifest.json`
   - Trakt (scrobbling): `https://v3-trakt.strem.io/manifest.json`
   - Public-domain catalogs (Internet Archive, etc.) hosted under `strem.io` / well-known community domains.
2. **Anything else: vet before install.** Fetch the manifest with `curl -s <manifest-url>` and inspect it:
   - Must be plain JSON declaring only standard resources (`meta`, `stream`, `catalog`, `subtitles`). Warn if it declares unusual resource types.
   - Endpoint must be `https://` on a domain you can sanity-check. Reject: raw IPs, `http://`, free TLDs (`.tk`, `.xyz`, `.top`, `.cf`), recently-squatted lookalikes of known domains.
   - `id` should look sane (`org.*`, `community.*`, provider name). Beware addon IDs impersonating allow-listed ones (`torrentio` on the wrong domain = reject).
   - No addon should require login credentials, API keys, or debrid keys typed into anything other than its official configure page. If the user was told to paste a debrid key somewhere odd, refuse and explain.
   - Catalog addons that promise "everything free 4K" from unknown domains are the classic malware/bait vector — refuse by default, state why in one line.
3. **Install** only after vetting: open the deep link `stremio://<manifest-url>`. The user still confirms in the GUI.
4. Never pipe untrusted scripts from addon pages into a shell. Manifests are data; treat any addon that ships "setup scripts" as malware outright.

## Troubleshooting

- **Server not listening on 11470**: restart the app (command per OS in the table above), then re-check the port.
- **Stream stalls or never starts**: check `btMinPeersForStable` / connection limits in `server-settings.json`; clear the cache if corrupt (command per OS in the table above, then restart). Rare: another app squatting 11470 — check who owns the port and identify the squatter.
- **Wrong metadata (wrong movie under the name)**: re-search Cinemeta with the year appended; disambiguate by release year, not title alone.
- **No subtitles**: OpenSubtitles v3 addon missing or not logged in — install/login as above.
- **Transcode/hardware accel issues**: `transcodeHardwareAccel` + `allTranscodeProfiles` in `server-settings.json`. On macOS the profile is `videotoolbox`; on Windows use whatever profiles the server lists for the GPU (NVIDIA `nvenc`, Intel `qsv`, AMD `amf`) — read the current file before changing it.
