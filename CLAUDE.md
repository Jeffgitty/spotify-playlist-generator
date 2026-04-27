# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running

Open `index.html` directly in a browser, or serve locally:

```bash
npx serve spotify-playlist-generator
# or
python -m http.server 8080
```

The app is deployed at `https://jeffgitty.github.io/spotify-playlist-generator` — this exact URL is hardcoded as `REDIRECT_URI` and registered in the Spotify Developer Dashboard. Changing it breaks OAuth.

## Architecture

Single-file SPA — all CSS, HTML, and JS in `index.html`. No build step, no framework, no bundler.

### Auth: Spotify PKCE

`login()` → Spotify authorize → redirect back with `?code=` → `handleCallback()` → `storeTokens()`. Tokens live in `localStorage` (`sp_token`, `sp_expires`, `sp_refresh`). `getToken()` auto-refreshes 60 s before expiry; on failure it calls `logout()`. The PKCE verifier is temporarily stored under `pkce_v`.

### API wrapper

`api(path, opts)` is the single Spotify API call entry point — it prepends `https://api.spotify.com/v1`, injects the Bearer token, and throws on non-2xx.

### Three track lists

| Variable | Purpose |
|---|---|
| `foundTracks[]` | Tracks committed to the playlist (from paste or added from staging). Rendered in `#results-card`. |
| `stagingTracks[]` | Candidate tracks from AI generation or manual search. Rendered in `#staging-card`. |
| `playlistTracks[]` | Current contents of the selected existing playlist. Rendered in `#playlist-card`. |

### AI generation

`generateWithAI()` sends a prompt to `https://text.pollinations.ai/` (free, no API key). The response may be plain text or OpenAI-format JSON — both are handled. The result is fed through `parseTracklist()` and then `populateStaging()`.

`regenTrack(i)` / `stagingRegen(i)` regenerate a single track via the same endpoint, using `_lastGenParams` for context.

### Parser

`parseTracklist(text)` handles numbered lists, dash/colon/`by`-separated formats, and strips markdown. Returns `[{artist, title, original}]`.

`searchTrack({artist, title})` tries a strict Spotify field-filter query first, then falls back to a plain query.

### Audio

One shared `_audio = new Audio()` instance. Two index vars track which list is playing: `_playingIdx` (foundTracks) and `_stagingAudioIdx` (stagingTracks). Switching sources pauses first.

### Drag & drop

`_dragIdx` / `onDragStart/Over/Drop/End` for `foundTracks`. `_plDragIdx` / `onPLDragStart/Over/Drop/End` for `playlistTracks`. Both mutate their array in place and re-render.

### XSS

All user-controlled strings go through `esc(s)` before `innerHTML`. Never put user data in `onclick="..."` string interpolation — use `data-*` attributes.

### CSS

Dark/light theming via `data-theme` attribute on `<html>`. CSS custom properties: `--green`, `--bg`, `--surface`, `--surface2`, `--surface3`, `--text`, `--muted`, `--border`, `--radius`, `--font`. Use these everywhere — avoid hardcoded hex.

### Settings

Persisted to `localStorage`: `theme` (`dark`/`light`), `debug` (`true`/`false`). Debug mode reveals `.debug-only` elements including the `testAccess()` diagnostic tool.

### Boot sequence

`boot()` runs on page load: applies saved theme/debug, checks for `?code=` (PKCE callback), then tries to restore session from stored tokens.
