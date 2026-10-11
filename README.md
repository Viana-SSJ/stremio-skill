# stremio-skill

A [pi coding agent](https://github.com/badlogic/pi-mono) skill that turns "I want to watch *X*" into a playing video. It drives Stremio on macOS and Windows end to end: finds the title, checks real stream availability, ranks sources, opens the app, and installs stream addons — with strict malware vetting before anything is ever installed.

## What you say → what happens

| You say | The agent does |
|---|---|
| "I want to watch Synecdoche, New York" | Resolves the IMDb ID via Cinemeta, confirms the year, queries Torrentio, ranks 16 streams, picks a 1080p BluRay, opens Stremio on the movie page |
| "install Torrentio" | Recognizes it's on the allow-list, installs via `stremio://` deep link |
| "should I install this catalog addon someone sent me?" | Fetches and vets the manifest, refuses if it looks like bait/malware, explains why in one line |
| "the stream keeps stalling" | Checks the server on port 11470, inspects torrent settings, clears cache, relaunches |

## How it works

1. **Resolve** — Cinemeta search API → IMDb ID (movies, series, and per-episode IDs).
2. **Check streams** — queries stream addons directly over HTTP (Torrentio by default).
3. **Rank** — `REMUX > BluRay > WEB-DL > WEBRip`; rejects CAM/TS/Screener; defaults to 1080p, offers 4K on request; prefers DD/DTS 5.1 audio.
4. **Open** — `stremio://detail/movie/tt0383028` deep link. No GUI scripting, no hacks.
5. **Self-heal** — dead server on `:11470`, corrupt stream cache, wrong metadata match: diagnosed and fixed from the skill's runbook.

## Addon safety model

The skill treats every addon manifest as **untrusted data**:

- **Allow-list** (install without ceremony): Cinemeta, Torrentio, OpenSubtitles v3, Trakt, well-known public-domain catalogs.
- **Everything else gets vetted first**: manifest must be standard JSON with standard resources; HTTPS on a sane domain only — raw IPs, `http://`, free TLDs (`.tk`, `.xyz`, `.top`), and domain lookalikes are rejected; impersonated addon IDs are rejected; addons that want credentials typed anywhere but their official page are refused.
- Catalog addons promising "everything free in 4K" from unknown domains are refused by default — that's the classic malware vector.

## Install

```sh
git clone git@github.com:Viana-SSJ/stremio-skill.git ~/.agents/skills/stremio
```

(Any discovered skills location works — `~/.agents/skills/` or `<project>/.agents/skills/`.) Restart pi or run `/reload`, then check `/skill:stremio`.

**Requirements:** macOS or Windows 10/11, Stremio desktop installed, `curl` (bundled with Windows 10+ as `curl.exe`). No extra dependencies.

On Windows the same clone works in PowerShell:

```powershell
git clone https://github.com/Viana-SSJ/stremio-skill.git "$HOME\.agents\skills\stremio"
```
