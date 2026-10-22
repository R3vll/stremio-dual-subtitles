# Design: In-app Configuration, Custom Subtitle Styling, Tiered Caching

- **Date:** 2026-10-22
- **Repo:** `R3vll/stremio-dual-subtitles` (fork of `ummugulsunn/stremio-dual-subtitles`)
- **Status:** Approved conversationally; spec for implementation

---

## 1. Goals

1. **In-app configuration** — select primary/secondary language (and styling) inside Stremio's native addon-configuration UI; the external website must not be required.
2. **Custom subtitle styling** — user-configurable colors/text style for the two subtitle lines, rendered via SRT tags.
3. **Tiered subtitle caching** — merged SRT results survive cross-instance/serverless restarts and work on both Vercel and local.
4. **Quality fixes** — rate-limit store leak, stale manifest version, connection reuse.

Non-goals: third language support, font size/position (Stremio players don't honor these SRT tags), analytics changes, sync-engine algorithm changes.

## 2. Architecture

No new frameworks. Pure additive modules inside the existing Express + addonBuilder app.

```
Stremio client
  │  installs via base /manifest.json (configurationRequired → in-app Configure)
  │  requests /subtitles/:type/:id.json  (args.config = languages + styling)
  ▼
server.js: NEW /subtitles route (mounted before SDK router)
  │  runs existing subtitlesHandler; resolves {{ADDON_URL}} from request
  ▼
addon.js subtitlesHandler
  │  picks best candidate pair as today; embeds style params + langs in .srt URL query
  ▼
/subs/...srt → generateDynamicSubtitle
  │  L1 mem → L2 Upstash Redis → L3 disk cache
  ▼
mergeSubtitles → styleRenderer.applyStyle(text, mainText, transText, styleConfig)
```

## 3. Component 1 — Native in-app configuration

### Manifest (addon.js)
- Keep `behaviorHints.configurable: true`, `configurationRequired: true`.
- `config` array gains style keys after the two language selects:
  - `transColor` — select, options: `#94a3b8` (default Gray), `#f43f5e`, `#38bdf8`, `#22c55e`, `#eab308`, `#a855f7`, `#ffffff`; label = preset name + hex.
  - `mainBold` — checkbox, default `true`.
  - `transItalic` — checkbox, default `true`.
  - `marker` — select: `›` (default), `::`, `|`, `none`.
  - `swapLines` — checkbox, default `false` (secondary below primary).
- Keep `stremioAddonsConfig` untouched.

### Placeholder bug fix (server.js + addon.js)
Problem: `subtitlesHandler` returns `.srt` URLs containing literal `{{ADDON_URL}}`; only the website-config route (`/:config/subtitles`) replaces it. Native-config installs hit the SDK-mounted `/subtitles` route where no replacement happens → 404.

Fix: add an explicit Express route `GET /subtitles/:type/:id/:extra?.json` **before** `app.use(getRouter(addonInterface))` that:
1. Parses `config` from `args` (values arrive from Stremio's config URL) into `{ mainLang, transLang, ...style }`.
2. Calls the existing `subtitlesHandler({ type, id, extra, config })`.
3. Replaces `{{ADDON_URL}}` with `getExternalUrl(req)` (existing helper).
4. Emits the same `Cache-Control` headers as the website path (they currently differ only in s-maxage; use the existing same values).

`subtitlesHandler` reading `config?.mainLang || 'English [eng]'` defaults stays — required config means values are always sent by Stremio.

### Style carriage to the .srt endpoint
Same pattern as video params: subtitlesHandler builds the `.srt` URL with query params `style=<compact code>`, where the code is a deterministic string derived from the five style keys (e.g. `style=%2394a3b8|true|true|%E2%80%BA|false`, values joined by `|`, URL-encoded; use `0`/`1` for booleans, `none` for marker, `swap` etc.). `generateDynamicSubtitle` receives `styleCode` as an argument, parses it into a style config object, includes it in the cache key, and hands the object to `styleRenderer`.
- Unknown/unparsable style codes fall back to defaults (no errors).
- Website path (`/:config/...`) ignores style (defaults) — backward compatible.

## 4. Component 2 — Style rendering (lib/styleRenderer.js)

Pure module:

```
parseStyleCode(code)  -> styleConfig (defaults on missing/invalid parts)
applyStyle(mainText, transText, styleConfig) -> SRT cue text (string, both lines)
```

Rendering rules (single-line callers already sanitize/escape; renderer only wraps):
- Lines are joined with `\n` as today. Order per `swapLines`.
- Primary line: `<b>…</b>` when `mainBold`, else raw (no tags).
- Secondary line (matched content only; unmatched cues never get a second line — unchanged behavior): optional `marker/em` prefix (marker value + space, when not `none`), then `<i>`/`</i>` when `transItalic` and `<font color='#hex'>`/`</font>` when the color differs from player-default (only when a color is set; gray default keeps today's explicit color).
- Colors validated against `^#[0-9a-fA-F]{6}$`; invalid → default.
- `mergeSubtitles` in addon.js delegates final text assembly to `styleRenderer.applyStyle`, replacing the hardcoded `<b>…</b>\n› <i><font>` assembly. `DUAL_SUB_TRANS_COLOR` moves into styleRenderer defaults.
- HTML escaping stays in addon.js (before calling applyStyle).

Documented limitation (README note): players that ignore SRT `<font>/<b>/<i>` tags show plain text; this is a client-side behavior.

## 5. Component 3 — Tiered subtitle cache (lib/subtitleCache.js)

Replaces direct Map usage in addon.js (`getSubtitle`, `storeSubtitle`) behind one API:

```
get(key)  -> string|null    // L1 mem → L2 Upstash → L3 disk
put(key, srt)             // write-through: mem + fire-and-forget L2 + L3 (local only)
```

- **L1** in-memory Map, TTL 6h (existing logic, extracted as-is).
- **L2** Upstash Redis `SET subtitle:<key> <srt> EX 21600` / `GET`; reuses `lib/persistence.js` Redis REST config (new helper functions in persistence.js: `get(key)`, `set(key, value, ttl)`; keep them separate from analytics counters). Fire-and-forget writes, `MAX_CONTENT_LENGTH` 512KB guard — larger SRTs skip L2.
- **L3** disk, `.cache/subtitles/<sha1(key)>.srt`, TTL honored by file mtime check. Disk tier **disabled on Vercel** (`process.env.VERCEL` present → skip; no persistent FS).
- Cache key unchanged plus `_${styleCode}` when styling differs from defaults (style params present).
- Graceful degradation: any tier down → fall through to next; no request path depends on Redis/disk availability.

## 6. Component 4 — Quality fixes

1. **Rate-limit leak:** sweep `rateLimitStore` every RATE_LIMIT_WINDOW_MS — delete entries past `resetAt` (piggybacked on the existing middleware tick, via one `setInterval` with `.unref()`).
2. **Version sync:** manifest `ADDON_VERSION` → `1.2.0` (CHANGELOG next entry will document the feature).
3. **Keep-alive agent:** shared `https.Agent({ keepAlive: true, maxSockets: 32 })` + `http.Agent` equivalent passed as axios default agent in addon.js for OpenSubtitles and subtitle-file requests.

## 7. Testing & validation

Extend existing node test harness (`test.js`):
- styleRenderer: parseStyleCode defaults/invalid inputs; applyStyle output matrices (bold on/off × italic on/off × marker × swap × color-set/none).
- subtitleCache: mock persistence (env disabled → L1/local only); disk tier write/read/expire; Vercel mode skips disk.
- Placeholder resolution: `/subtitles` route test with a stubbed handler producing `{{ADDON_URL}}` URLs.
- Rate limiter: expired-entry sweep behavior.
- Run `npm test`; existing tests must stay green (style refactor changes merge output only when style ≠ legacy defaults — legacy defaults must produce byte-identical output to today).

## 8. Deployment & config surface

- `.env.example`: no new required vars; document optional `SUBTITLE_DISK_CACHE=off|on` (default auto: off on Vercel, on locally).
- `vercel.json` unchanged. Upstash uses existing env vars.
- README: short "In-app configuration" section + player-tag caveat.

## 9. Risks

| Risk | Mitigation |
| --- | --- |
| Stremio clients vary in native-config support | Website path untouched; both flows work |
| `<font>/<b>/<i>` ignored by some players | Defaults preserve today's look; documented caveat |
| Upstash 512KB quota pressure | Size-cap skip; TTL 6h; analytics quotas unaffected |
| Tests asserting legacy markup byte-for-byte | Legacy default path verified byte-identical before merge |
