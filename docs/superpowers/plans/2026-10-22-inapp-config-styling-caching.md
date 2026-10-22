# In-app Config, Styling & Tiered Caching Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use subagent-driven-development (recommended) or executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let users pick languages and subtitle styling inside Stremio's native addon config, add configurable subtitle styling, tiered persistent caching (memory → Upstash → disk), and fix three quality bugs.

**Architecture:** Additive modules inside the existing Express + addonBuilder app: `lib/styleRenderer.js` (pure rendering), `lib/subtitleCache.js` (3-tier cache), new config keys in the manifest, config-value parsing in server.js supporting both the website's `A|B` form and Stremio's native JSON form, and style carried to the `.srt` endpoint as a compact query code (same pattern as the existing video params).

**Tech Stack:** Node.js 18+, Express 4, stremio-addon-sdk 1.6, axios, Upstash Redis REST (optional), `node:assert`-based test harness (`node test.js`).

**Working directory:** `C:\Users\admin\stremio-dual-subtitles` (branch `feature/inapp-config-styling-caching`).

**Source spec:** `docs/superpowers/specs/2026-10-22-inapp-config-styling-caching-design.md`

**Legacy output contract (verified from test.js lines 687-694 and addon.js lines 582-583):** with default style, a merged cue renders as `<b>MAIN</b>\n\u203a <i><font color='#94a3b8'>TRANS</font></i>` (italic outermost, `<font>` inside it, marker `›` + space, gray `#94a3b8`). Existing tests must stay green.

---

### Task 1: `lib/styleRenderer.js` — pure style module (TDD)

**Files:**
- Create: `lib/styleRenderer.js`
- Test: append to `test.js` (section before the `RESULTS` banner at line ~893)

- [ ] **Step 1: Write the failing tests**

Append this block to `test.js` immediately before the `// ===== RESULTS =====` section:

```javascript
// ============================================================================
// styleRenderer
// ============================================================================
console.log('\n--- styleRenderer ---');

const {
  DEFAULT_STYLE,
  COLOR_PRESETS,
  MARKER_OPTIONS,
  parseStyleCode,
  buildStyleCode,
  applyStyle
} = require('./lib/styleRenderer');

test('applyStyle: legacy defaults are byte-identical to v1 formatting', () => {
  assert.strictEqual(
    applyStyle('Hello', 'World'),
    "<b>Hello</b>\n\u203a <i><font color='#94a3b8'>World</font></i>"
  );
});

test('applyStyle: empty trans text renders main line only', () => {
  assert.strictEqual(applyStyle('Hello', ''), '<b>Hello</b>');
  assert.strictEqual(applyStyle('Hello', ''), '<b>Hello</b>');
});

test('applyStyle: custom toggles and marker, no bold', () => {
  assert.strictEqual(
    applyStyle('Hi', 'Yo', { mainBold: false, transItalic: false, marker: '::', transColor: '#ff0000' }),
    "Hi\n<i><font color='#ff0000'>:: Yo</font></i>"
  );
});

test('applyStyle: swapLines puts secondary line on top', () => {
  assert.strictEqual(
    applyStyle('Hi', 'Yo', { swapLines: true }),
    "\u203a <i><font color='#94a3b8'>Yo</font></i>\n<b>Hi</b>"
  );
});

test('applyStyle: mainColor wraps primary line in font tag', () => {
  assert.strictEqual(
    applyStyle('Hi', 'Yo', { mainColor: '#00ff00' }),
    "<font color='#00ff00'><b>Hi</b></font>\n\u203a <i><font color='#94a3b8'>Yo</font></i>"
  );
});

test('applyStyle: preset label extracts hex (config value from Stremio)', () => {
  assert.strictEqual(
    applyStyle('Hi', 'Yo', { transColor: 'Red (#f43f5e)' }),
    "<b>Hi</b>\n\u203a <i><font color='#f43f5e'>Yo</font></i>"
  );
});

test('applyStyle: invalid color falls back to legacy gray, not raw injection', () => {
  assert.ok(applyStyle('Hi', 'Yo', { transColor: '<script>' }).includes('#94a3b8'));
  assert.ok(!applyStyle('Hi', 'Yo', { transColor: '<script>' }).includes('<script>'));
});

test('applyStyle: no marker renders without prefix', () => {
  assert.strictEqual(
    applyStyle('Hi', 'Yo', { marker: 'none', transColor: '', transItalic: false }),
    "<b>Hi</b>\nYo"
  );
});

test('buildStyleCode + parseStyleCode: default config roundtrip', () => {
  const code = buildStyleCode({});
  assert.strictEqual(code, 'none|#94a3b8|1|1|\u203a|0');
  const s = parseStyleCode(code);
  assert.strictEqual(s.mainColor, '');
  assert.strictEqual(s.transColor, '#94a3b8');
  assert.strictEqual(s.mainBold, true);
  assert.strictEqual(s.transItalic, true);
  assert.strictEqual(s.marker, '\u203a');
  assert.strictEqual(s.swapLines, false);
});

test('buildStyleCode + parseStyleCode: full custom config roundtrip', () => {
  const code = buildStyleCode({
    mainColor: '#00ff00', transColor: '#ff0000', mainBold: false,
    transItalic: false, marker: '::', swapLines: true
  });
  assert.strictEqual(code, '#00ff00|#ff0000|0|0|::|1');
  const s = parseStyleCode(code);
  assert.strictEqual(s.mainColor, '#00ff00');
  assert.strictEqual(s.transColor, '#ff0000');
  assert.strictEqual(s.mainBold, false);
  assert.strictEqual(s.transItalic, false);
  assert.strictEqual(s.marker, '::');
  assert.strictEqual(s.swapLines, true);
});

test('parseStyleCode: invalid fields fall back to defaults', () => {
  const s = parseStyleCode('zzz|zzz|9|9|<script>|9');
  assert.strictEqual(s.transColor, '#94a3b8', 'invalid color -> default gray');
  assert.strictEqual(s.mainBold, true);
  assert.strictEqual(s.transItalic, true);
  assert.strictEqual(s.marker, '\u203a');
  assert.strictEqual(s.swapLines, false);
});

test('parseStyleCode: null/empty code returns defaults', () => {
  const s = parseStyleCode(null);
  assert.strictEqual(s.transColor, '#94a3b8');
  assert.strictEqual(s.mainBold, true);
  const e = parseStyleCode('');
  assert.strictEqual(e.transColor, '#94a3b8');
});

test('preset/marker option lists are exposed for the manifest', () => {
  assert.ok(Array.isArray(COLOR_PRESETS) && COLOR_PRESETS.length >= 6);
  assert.ok(MARKER_OPTIONS.includes('\u203a') && !MARKER_OPTIONS.includes('|'));
});
```

- [ ] **Step 2: Run tests, verify they fail**

Run: `cd C:\Users\admin\stremio-dual-subtitles && npm test`
Expected: FAIL — `Cannot find module './lib/styleRenderer'` (require throws; every test in file errors). Exit code 1.

- [ ] **Step 3: Implement `lib/styleRenderer.js`**

Create `lib/styleRenderer.js` with exactly:

```javascript
'use strict';

/**
 * Style rendering for merged dual-subtitle cues.
 *
 * User style settings travel from the addon config (set inside Stremio's
 * native configuration screen, or the website) to the .srt endpoint as a
 * compact 6-field, pipe-separated code:
 *
 *   <mainColor>|<transColor>|<mainBold>|<transItalic>|<marker>|<swapLines>
 *   e.g. "none|#94a3b8|1|1|›|0"      (fields come from buildStyleCode)
 *
 * Colors are hex or `none` (unset). Booleans are '1'/'0'. '|' cannot appear
 * in a field because the marker options deliberately exclude it — that is
 * why MARKER_OPTIONS does not offer '|'.
 *
 * applyStyle() receives HTML-ESCAPED text (addon.js escapes before calling).
 * With default settings the output is byte-identical to the pre-styling
 * hardcoded format:
 *   <b>MAIN</b>\n› <i><font color='#94a3b8'>TRANS</font></i>
 */

const DEFAULT_STYLE = Object.freeze({
  mainColor: '',        // '' = no <font> tag around the primary line
  transColor: '#94a3b8',
  mainBold: true,
  transItalic: true,
  marker: '\u203a',     // ›
  swapLines: false
});

// 'none' = marker omitted; '|' is excluded on purpose (style-code separator)
const MARKER_OPTIONS = ['\u203a', '::', '\u2014', 'none'];

const COLOR_PRESETS = [
  'None (player default)',
  'Gray (#94a3b8)',
  'Red (#f43f5e)',
  'Blue (#38bdf8)',
  'Green (#22c55e)',
  'Yellow (#eab308)',
  'Purple (#a855f7)',
  'White (#ffffff)'
];

const SUB_COLOR_PRESETS = COLOR_PRESETS.slice(1);

const HEX_RE = /^#[0-9a-fA-F]{6}$/;

function boolish(value, fallback) {
  if (value === true) return true;
  if (value === false) return false;
  if (value === 'true' || value === '1') return true;
  if (value === 'false' || value === '0') return false;
  return fallback;
}

/**
 * Extract a hex color from any config string (raw hex, preset label like
 * "Red (#f43f5e)", or "None (player default)"). Returns '' when the value
 * resolves to no color. `fallback` is used only when value is null/undefined.
 */
function extractColor(value, fallback) {
  if (value == null) return fallback;
  const s = String(value).trim();
  if (!s) return fallback;
  const m = s.match(/#[0-9a-fA-F]{6}\b/);
  return m ? m[0] : '';
}

/** Clamp an arbitrary config object (or style object) to a valid style. */
function normalizeStyle(input = {}) {
  const c = input || {};
  const style = { ...DEFAULT_STYLE };
  if ('mainColor' in c) style.mainColor = extractColor(c.mainColor, '');
  if ('transColor' in c) style.transColor = extractColor(c.transColor, DEFAULT_STYLE.transColor);
  if ('mainBold' in c) style.mainBold = boolish(c.mainBold, DEFAULT_STYLE.mainBold);
  if ('transItalic' in c) style.transItalic = boolish(c.transItalic, DEFAULT_STYLE.transItalic);
  if ('marker' in c && MARKER_OPTIONS.includes(c.marker)) style.marker = c.marker;
  if ('swapLines' in c) style.swapLines = boolish(c.swapLines, DEFAULT_STYLE.swapLines);
  return style;
}

/** Parse a compact style code (or style config object) into a valid style. */
function parseStyleCode(code) {
  if (code == null) return normalizeStyle({});
  if (typeof code === 'object') return normalizeStyle(code);
  const parts = String(code).split('|');
  const cfg = {};
  const fields = ['mainColor', 'transColor'];
  for (let i = 0; i < fields.length; i++) {
    const raw = parts[i];
    if (raw === undefined || raw === '') continue;        // keep default
    if (raw === 'none') { cfg[fields[i]] = ''; continue; }
    if (HEX_RE.test(raw)) { cfg[fields[i]] = raw; continue; }
    // invalid -> leave default
  }
  const [ , , mainBold, transItalic, marker, swap] = parts;
  if (mainBold !== undefined && mainBold !== '') cfg.mainBold = mainBold;
  if (transItalic !== undefined && transItalic !== '') cfg.transItalic = transItalic;
  if (marker !== undefined && marker !== '') cfg.marker = marker;
  if (swap !== undefined && swap !== '') cfg.swapLines = swap;
  return normalizeStyle(cfg);
}

/** Serialize a config object/settings into the compact code for .srt URLs. */
function buildStyleCode(config = {}) {
  const s = normalizeStyle(config);
  return [
    s.mainColor === '' ? 'none' : s.mainColor,
    s.transColor === '' ? 'none' : s.transColor,
    s.mainBold ? '1' : '0',
    s.transItalic ? '1' : '0',
    s.marker,
    s.swapLines ? '1' : '0'
  ].join('|');
}

/**
 * Render one merged cue's text from escaped main/trans strings.
 * Tag order is fixed so style presets stay readable on players:
 *   primary:  <font color='mainColor'><b>main</b></font>
 *   secondary: marker + <i><font color='transColor'>…</font></i> (legacy order)
 */
function applyStyle(encMainText, encTransText, styleInput = {}) {
  const s = normalizeStyle(styleInput);

  let main = encMainText;
  if (s.mainBold) main = `<b>${main}</b>`;
  if (s.mainColor) main = `<font color='${s.mainColor}'>${main}</font>`;

  if (encTransText == null || encTransText.length === 0) {
    return main;
  }

  const prefix = s.marker && s.marker !== 'none' ? `${s.marker} ` : '';
  let secondary = `${prefix}${encTransText}`;
  if (s.transColor) secondary = `<font color='${s.transColor}'>${secondary}</font>`;
  if (s.transItalic) secondary = `<i>${secondary}</i>`;

  return s.swapLines ? `${secondary}\n${main}` : `${main}\n${secondary}`;
}

module.exports = {
  DEFAULT_STYLE,
  MARKER_OPTIONS,
  COLOR_PRESETS,
  SUB_COLOR_PRESETS,
  normalizeStyle,
  parseStyleCode,
  buildStyleCode,
  applyStyle
};
```

Note: `parseStyleCode` on `'zzz|zzz|9|9|<script>|9'` — invalid colors keep defaults because we only set `cfg` for `none`/valid-hex; `'9'` for booleans passes boolish default-true since '9' is truthy-limit? **No** — `boolish('9', fallback)` returns fallback because '9' matches no accepted string. Wait: `'9'` is a string that hits none of the four accepted values → returns fallback (default). Verified.

- [ ] **Step 4: Run tests, verify pass**

Run: `npm test`
Expected: PASS — all styleRenderer tests green AND all pre-existing tests still green (they don't exercise styleRenderer yet). Exit code 0.

- [ ] **Step 5: Commit**

```bash
git add lib/styleRenderer.js test.js
git commit -m "feat: add styleRenderer module with pure style parsing/rendering (TDD)"
```

---

### Task 2: Wire `mergeSubtitles` to `applyStyle` (legacy output byte-identical)

**Files:**
- Modify: `addon.js` (lines ~487-488 constant, ~512-619 `mergeSubtitles`, ~125 import block)
- Test: append to `test.js`, then re-run existing section

- [ ] **Step 1: Write the failing tests**

Append before the `RESULTS` banner in `test.js`:

```javascript
// ============================================================================
// mergeSubtitles + style wiring
// ============================================================================
console.log('\n--- mergeSubtitles style wiring ---');

test('mergeSubtitles: default output byte-identical to legacy formatting', () => {
  const main = [
    { id: '1', startTime: '00:00:01,000', endTime: '00:00:02,000', text: 'one' },
    { id: '2', startTime: '00:00:03,000', endTime: '00:00:04,000', text: 'two' }
  ];
  const trans = [
    { id: '1', startTime: '00:00:01,000', endTime: '00:00:02,000', text: 'uno' }
  ];
  const merged = mergeSubtitles(main, trans, { mainLang: 'eng', transLang: 'tur' });
  assert.strictEqual(
    merged[0].text,
    "<b>one</b>\n\u203a <i><font color='#94a3b8'>uno</font></i>"
  );
  assert.strictEqual(merged[1].text, '<b>two</b>');
});

test('mergeSubtitles: style option changes rendering', () => {
  const main = [
    { id: '1', startTime: '00:00:01,000', endTime: '00:00:02,000', text: 'one' }
  ];
  const trans = [
    { id: '1', startTime: '00:00:01,000', endTime: '00:00:02,000', text: 'uno' }
  ];
  const merged = mergeSubtitles(main, trans, {
    mainLang: 'eng', transLang: 'tur',
    style: { mainBold: false, transItalic: false, marker: 'none', transColor: '', swapLines: true }
  });
  assert.strictEqual(merged[0].text, 'uno\none');
});

test('mergeSubtitles: style code string is accepted too', () => {
  const main = [
    { id: '1', startTime: '00:00:01,000', endTime: '00:00:02,000', text: 'one' }
  ];
  const trans = [
    { id: '1', startTime: '00:00:01,000', endTime: '00:00:02,000', text: 'uno' }
  ];
  const merged = mergeSubtitles(main, trans, {
    mainLang: 'eng', transLang: 'tur',
    style: 'none|#94a3b8|1|1|\u203a|1'
  });
  assert.strictEqual(
    merged[0].text,
    "\u203a <i><font color='#94a3b8'>uno</font></i>\n<b>one</b>"
  );
});
```

- [ ] **Step 2: Run, verify fail**

Run: `npm test`
Expected: FAIL — first new test fails because output is currently produced by hardcoded assembly **and actually passes** identical by accident? No: check `style` option — currently ignored. First test may PASS (legacy markup identical). The two `style`-option tests FAIL (option ignored, output still legacy format). That is the expected failure signature.

- [ ] **Step 3: Implement**

In `addon.js`:

3a. Replace the import line `const { alignAndMatch } = require('./lib/syncEngine');` context — after it, add:

```javascript
const { parseStyleCode, applyStyle } = require('./lib/styleRenderer');
```

3b. Delete the constant block (lines ~487-488):

```javascript
/** Muted color for secondary line; players that ignore <font> still have <b> + › marker. */
const DUAL_SUB_TRANS_COLOR = '#94a3b8';
```

3c. In `mergeSubtitles`, add `style` to the opts destructure (line ~520):

```javascript
  const {
    mainLang = null,
    transLang = null,
    matchThresholdMs = 1500,
    allowMultiTrans = true,
    enableOffset = true,
    enableDrift = true,
    style = null
  } = opts;
```

3d. Replace the merged-text assembly (lines ~565-589, from `let mergedText;` through the `if (mergedText === undefined)` block) with:

```javascript
    let mergedText;
    const styleCfg = style == null ? DEFAULT_STYLE_CLONE : parseStyleCode(style);
    const transIdxs = matches.get(mi);
    if (transIdxs && transIdxs.length > 0) {
      const transParts = [];
      for (const ti of transIdxs) {
        const t = transTimed[ti];
        if (!t) continue;
        const piece = joinSubtitleLines(
          sanitize(t.text, { allowedTags: [], allowedAttributes: {} }),
          transLang
        );
        if (piece) transParts.push(piece);
      }
      if (transParts.length > 0) {
        const cleanTransText = transParts.join(transJoiner);
        const encMain = htmlEncodeSrt(cleanMainText);
        const encTrans = htmlEncodeSrt(cleanTransText);
        mergedText = applyStyle(encMain, encTrans, styleCfg);
      }
    }

    if (mergedText === undefined) {
      mergedText = applyStyle(htmlEncodeSrt(cleanMainText), '', styleCfg);
    }
```

And define once, at top of file after imports (cheap shared default — `parseStyleCode(null)` allocates; avoid in hot loop):

```javascript
const DEFAULT_STYLE_CLONE = { ...parseStyleCode(null) };
```

3e. Sanity: `parseStyleCode(true)` non-string non-null object? Booleans arrive only via JS misuse, not config paths; acceptable.

- [ ] **Step 4: Run, verify pass**

Run: `npm test`
Expected: PASS — all pre-existing merge tests (Sopranos, duplicate guard, emits-all-cues) still pass with identical markup, plus the three new tests. Exit code 0.

- [ ] **Step 5: Commit**

```bash
git add addon.js test.js
git commit -m "feat: mergeSubtitles delegates cue formatting to styleRenderer"
```

---

### Task 3: Manifest style config keys + version bump

**Files:**
- Modify: `addon.js` (manifest `config` array ~line 194, `ADDON_VERSION` ~line 171)
- Modify: `package.json` (`"version"`)
- Test: append to `test.js`

- [ ] **Step 1: Write the failing tests**

Append before `RESULTS` banner:

```javascript
// ============================================================================
// manifest config
// ============================================================================
console.log('\n--- manifest config ---');

test('manifest: version bumped to 1.2.0', () => {
  assert.strictEqual(manifest.version, '1.2.0');
});

test('manifest: language + style config keys present', () => {
  const keys = manifest.config.map(c => c.key);
  for (const key of ['mainLang', 'transLang', 'mainColor', 'transColor',
                     'mainBold', 'transItalic', 'marker', 'swapLines']) {
    assert.ok(keys.includes(key), `missing config key: ${key}`);
  }
});

test('manifest: style defaults are safe', () => {
  const byKey = Object.fromEntries(manifest.config.map(c => [c.key, c]));
  assert.strictEqual(byKey.mainColor.default, 'None (player default)');
  assert.strictEqual(byKey.transColor.default, 'Gray (#94a3b8)');
  assert.strictEqual(byKey.mainBold.default, 'true');
  assert.strictEqual(byKey.transItalic.default, 'true');
  assert.strictEqual(byKey.marker.default, '\u203a');
  assert.strictEqual(byKey.swapLines.default, 'false');
  for (const key of ['mainColor', 'transColor', 'marker']) {
    assert.ok(Array.isArray(byKey[key].options) && byKey[key].options.length > 0,
      `${key} must offer options`);
  }
});
```

- [ ] **Step 2: Run, verify fail**

Run: `npm test`
Expected: FAIL — version is '1.0.0', keys missing.

- [ ] **Step 3: Implement**

3a. In `addon.js`, bump:

```javascript
const ADDON_VERSION = '1.2.0';
```

3b. Import option lists — extend the styleRenderer import from Task 2:

```javascript
const { parseStyleCode, applyStyle, COLOR_PRESETS, SUB_COLOR_PRESETS, MARKER_OPTIONS } = require('./lib/styleRenderer');
```

3c. Replace the whole `config: [...]` array in the manifest with:

```javascript
  config: [
    {
      key: 'mainLang',
      type: 'select',
      title: 'Primary Language (Audio/Learning Language)',
      options: getLanguageOptions(),
      required: true,
      default: 'English [eng]'
    },
    {
      key: 'transLang',
      type: 'select',
      title: 'Secondary Language (Your Native Language)',
      options: getLanguageOptions(),
      required: true,
      default: 'Turkish [tur]'
    },
    {
      key: 'mainColor',
      type: 'select',
      title: 'Primary Line Color',
      options: COLOR_PRESETS,
      required: false,
      default: 'None (player default)'
    },
    {
      key: 'transColor',
      type: 'select',
      title: 'Secondary Line Color',
      options: SUB_COLOR_PRESETS,
      required: false,
      default: 'Gray (#94a3b8)'
    },
    {
      key: 'mainBold',
      type: 'checkbox',
      title: 'Bold Primary Line',
      required: false,
      default: 'true'
    },
    {
      key: 'transItalic',
      type: 'checkbox',
      title: 'Italic Secondary Line',
      required: false,
      default: 'true'
    },
    {
      key: 'marker',
      type: 'select',
      title: 'Secondary Line Marker',
      options: MARKER_OPTIONS,
      required: false,
      default: '\u203a'
    },
    {
      key: 'swapLines',
      type: 'checkbox',
      title: 'Show Secondary Line Above Primary',
      required: false,
      default: 'false'
    }
  ]
```

3d. In `package.json`, `"version": "1.0.0"` → `"version": "1.2.0"`.

- [ ] **Step 4: Run, verify pass**

Run: `npm test`
Expected: PASS all. Exit code 0.

- [ ] **Step 5: Commit**

```bash
git add addon.js package.json test.js
git commit -m "feat: expose language+style settings in manifest config; bump to 1.2.0"
```

---

### Task 4: Carry style from config to the `.srt` endpoint

**Files:**
- Modify: `addon.js`: `subtitlesHandler` (~759-874), `selectAndMergeBestPair` (~692-756), `generateDynamicSubtitle` (~886-981)
- Test: `test.js` additions (roundtrip covered by Task 1; here verify URL plumbing via handler unit test with stubbed fetch)

- [ ] **Step 1: Write the failing test**

Append before `RESULTS` banner. This stubs `axios` internals indirectly — instead, exploit a seam: `subtitlesHandler` reads `config` and constructs URLs. Stub the network by monkey-patching `require('axios').get` before calling:

```javascript
// ============================================================================
// subtitles URL plumbing (style + video params)
// ============================================================================
console.log('\n--- subtitles URL plumbing ---');

const axios = require('axios');
const { subtitlesHandler } = require('./addon');

await testAsync('subtitlesHandler: .srt URL carries style param and video params', async () => {
  // Stub axios.get; stub returns one (main, trans) pair. parseSrt on the
  // fake subtitle FILE is never reached (only URL construction happens in
  // the handler), so this stub is enough.
  const origGet = axios.get;
  axios.get = async () => ({
    data: {
      subtitles: [
        { id: '111', url: 'https://example.com/main.srt', lang: 'eng', downloads: 500 },
        { id: '222', url: 'https://example.com/trans.srt', lang: 'tur', downloads: 400 }
      ]
    }
  });

  const result = await subtitlesHandler({
    type: 'movie',
    id: 'tt0111161',
    extra: { filename: 'Shawshank.1994.mp4' },
    config: {
      mainLang: 'English [eng]',
      transLang: 'Turkish [tur]',
      mainColor: 'Red (#f43f5e)',
      mainBold: 'false'
    }
  });
  axios.get = origGet;

  assert.strictEqual(result.subtitles.length, 1);
  const url = result.subtitles[0].url;
  // imdbId: 'tt0111161'.replace('tt','') === '0111161' (first 'tt' only)
  assert.ok(url.startsWith('{{ADDON_URL}}/subs/movie/0111161/0/0/eng/tur/111/222.srt?'), `unexpected path: ${url}`);
  const q = url.split('?')[1];
  const params = new URLSearchParams(q);
  assert.ok(params.has('style'), 'style param missing');
  const styleCfg = parseStyleCode(params.get('style'));
  assert.strictEqual(styleCfg.mainColor, '#f43f5e');
  assert.strictEqual(styleCfg.mainBold, false);
  assert.strictEqual(params.get('filename'), 'Shawshank.1994.mp4');
});

await testAsync('subtitlesHandler: defaults used when config has only languages', async () => {
  const origGet = axios.get;
  axios.get = async () => ({
    data: { subtitles: [
      { id: '111', url: 'https://example.com/main.srt', lang: 'eng', downloads: 500 },
      { id: '222', url: 'https://example.com/trans.srt', lang: 'tur', downloads: 400 }
    ] }
  });
  const result = await subtitlesHandler({
    type: 'movie', id: 'tt0111161', extra: {},
    config: { mainLang: 'English [eng]', transLang: 'Turkish [tur]' }
  });
  axios.get = origGet;
  const params = new URLSearchParams(result.subtitles[0].url.split('?')[1]);
  const styleCfg = parseStyleCode(params.get('style'));
  assert.strictEqual(styleCfg.transColor, '#94a3b8');
  assert.strictEqual(styleCfg.mainBold, true);
});
```

Put `parseStyleCode` in scope at top of this section: `const { parseStyleCode } = require('./lib/styleRenderer');` (already imported in Task 1 — reuse that binding).

Note: `test()` harness is synchronous — the harness `function test(name, fn)` runs `fn()` and catches sync errors; async handler promises resolve within the same event loop turn only if awaited — they are not awaited by `test()`. So add an async-aware variant. **Add `testAsync` next to `test` in test.js** (before new tests):

```javascript
async function testAsync(name, fn) {
  try {
    await fn();
    passed++;
    console.log(`  PASS  ${name}`);
  } catch (err) {
    failed++;
    console.error(`  FAIL  ${name}`);
    console.error(`        ${err.message}`);
  }
}
```

Then make the RESULTS block await-aware: rename the two tests above to `await testAsync(...)`. And wrap the final section so all async tests complete before results print: introduce at the very end of the file (replacing the plain RESULTS block):

```javascript
(async () => {
  await testAsync('subtitlesHandler: .srt URL carries style param and video params', async () => { /* body */ });
  await testAsync('subtitlesHandler: defaults used when config has only languages', async () => { /* body */ });

  console.log('\n========================================');
  console.log(`  Results: ${passed} passed, ${failed} failed`);
  if (failed > 0) process.exit(1);
})();
```

Implementation detail for the implementing agent: move the RESULTS banner + exit check into that IIFE, keeping the sync `test()` calls before the IIFE as-is (they are all registered synchronously before the IIFE runs).

- [ ] **Step 2: Run, verify fail**

Run: `npm test`
Expected: FAIL — style param missing (URL has only video params).

- [ ] **Step 3: Implement in `addon.js`**

3a. In `selectAndMergeBestPair`, add 4th parameter and use it:

```javascript
async function selectAndMergeBestPair(candidatePairs, mainLang, transLang, styleCfg = null) {
```

and inside the loop replace:

```javascript
    const merged = mergeSubtitles(mainParsed, transParsed, { mainLang, transLang });
```

with:

```javascript
    const merged = mergeSubtitles(mainParsed, transParsed, { mainLang, transLang, style: styleCfg });
```

3b. In `subtitlesHandler`, after `const videoQuery = serializeVideoParams(videoParams);` add:

```javascript
    // Style travels with the .srt URL (same pattern as video params).
    const styleCode = buildStyleCode(config || {});
```

(extend the styleRenderer import in `addon.js` with `buildStyleCode`.)

Then replace the URL assembly block (currently ending with the `{{ADDON_URL}}` template literal) with:

```javascript
    const queryParts = [];
    if (videoQuery) queryParts.push(videoQuery);
    queryParts.push(`style=${encodeURIComponent(styleCode)}`);

    const finalSubtitles = [{
      id: `dual-${best.main.id}-${best.trans.id}`,
      url: `{{ADDON_URL}}/subs/${dynamicParams}.srt?${queryParts.join('&')}`,
      lang: mainLang,
      SubtitlesName:
        `Dual (${mainLang.toUpperCase()}+${transLang.toUpperCase()}) - ` +
        `${getLanguageName(mainLang)} + ${getLanguageName(transLang)}`
    }];
```

3c. In `generateDynamicSubtitle`, new signature + cache key + style use (replace function head and cache-key/build lines):

```javascript
async function generateDynamicSubtitle(
  type,
  imdbId,
  season,
  episode,
  mainLang,
  transLang,
  mainSubId,
  transSubId,
  videoParams = {},
  styleCode = null
) {
```

cache key becomes:

```javascript
  const cacheKey =
    `${imdbId}_${season || ''}_${episode || ''}` +
    `_${mainLang}_${transLang}_${mainSubId}_${transSubId}` +
    `_${videoCacheFragment || ''}_${styleCode || ''}`;
```

and the merge call becomes:

```javascript
    const best = await selectAndMergeBestPair(orderedPairs, mainLang, transLang, styleCode == null ? null : parseStyleCode(styleCode));
```

- [ ] **Step 4: Run, verify pass**

Run: `npm test`
Expected: PASS all including new plumbing tests. Exit code 0.

- [ ] **Step 5: Commit**

```bash
git add addon.js test.js
git commit -m "feat: embed style code in .srt URLs and respect it during merge"
```

---

### Task 5: Redis helpers in `lib/persistence.js`

**Files:**
- Modify: `lib/persistence.js` (append near public API section)

- [ ] **Step 1: Implement** (no behavior change without Upstash env — safe; keep code-first, verify below)

Add before `module.exports`:

```javascript
// ── Generic cache access (subtitle cache uses these) ────────────────────────

/** Read a raw string value. Returns null when disabled or missing. */
async function redisGet(key) {
  if (!ENABLED) return null;
  const result = await redis(['GET', key]);
  return typeof result === 'string' ? result : null;
}

/** Fire-and-forget write with TTL. Never rejects. */
function redisSet(key, value, ttlSeconds) {
  if (!ENABLED) return Promise.resolve();
  return redis(['SET', key, value, 'EX', String(ttlSeconds)]).catch(() => {});
}
```

Extend `module.exports` to add `redisGet, redisSet,`.

- [ ] **Step 2: Verify**

Run: `npm test` (module is imported transitively via addon.js; must load without error when Upstash env is unset).
Expected: PASS, exit 0.

- [ ] **Step 3: Commit**

```bash
git add lib/persistence.js
git commit -m "feat: add generic redisGet/redisSet helpers for cache tiers"
```

---

### Task 6: `lib/subtitleCache.js` — 3-tier cache + wire into addon.js

**Files:**
- Create: `lib/subtitleCache.js`
- Modify: `addon.js` (cache block ~635-672, `getSubtitle` call in `generateDynamicSubtitle` ~904, exports)
- Modify: `server.js` (legacy `/subtitles/:filename` route must await — line ~281)
- Test: append to `test.js`

- [ ] **Step 1: Write the failing tests**

Append before `RESULTS` banner (respecting the IIFE structure from Task 4 — put tests inside the same IIFE before results print, using `testAsync`):

```javascript
// Subtitle cache tiers
console.log('\n--- subtitleCache ---');

const subtitleCache = require('./lib/subtitleCache');

await testAsync('subtitleCache: put/get roundtrip in memory', async () => {
  subtitleCache.clearMem();
  subtitleCache.put('testkey1', 'A\nB');
  const got = await subtitleCache.get('testkey1');
  assert.strictEqual(got, 'A\nB');
});

await testAsync('subtitleCache: miss returns null', async () => {
  subtitleCache.clearMem();
  assert.strictEqual(await subtitleCache.get('nope_missing_key'), null);
});

await testAsync('subtitleCache: disk tier off on Vercel, honors explicit override', async () => {
  const orig = process.env.VERCEL;
  delete process.env.VERCEL;
  delete process.env.SUBTITLE_DISK_CACHE;
  assert.strictEqual(subtitleCache._test.isDiskEnabled(), true, 'local default on');
  process.env.VERCEL = '1';
  assert.strictEqual(subtitleCache._test.isDiskEnabled(), false, 'vercel auto-off');
  process.env.SUBTITLE_DISK_CACHE = 'on';
  assert.strictEqual(subtitleCache._test.isDiskEnabled(), true, 'explicit on wins');
  delete process.env.SUBTITLE_DISK_CACHE;
  process.env.VERCEL = orig;
});

await testAsync('subtitleCache: disk tier write/read/expiry (dir override)', async () => {
  const fs = require('fs');
  const path = require('path');
  const os = require('os');
  const tmp = fs.mkdtempSync(path.join(os.tmpdir(), 'dualsub-cache-'));
  const origDir = process.env.SUBTITLE_DISK_DIR;
  const origAuto = process.env.SUBTITLE_DISK_CACHE;
  process.env.SUBTITLE_DISK_DIR = tmp;
  process.env.SUBTITLE_DISK_CACHE = 'on';

  // path is sha1-based: same key twice -> same file
  const p1 = subtitleCache._test.diskPath('cachecheck');
  const p2 = subtitleCache._test.diskPath('cachecheck');
  assert.strictEqual(p1, p2);

  subtitleCache.clearMem();
  subtitleCache.put('cachecheck', 'disk-content');
  assert.ok(fs.existsSync(p1), 'file written');
  assert.strictEqual(await subtitleCache.get('cachecheck'), 'disk-content');

  // expire: backdate file mtime beyond TTL is not directly injectable via TTL reset — simulate by touching mtime
  const TTL_MS = 6 * 60 * 60 * 1000;
  const old = new Date(Date.now() - TTL_MS - 1000);
  fs.utimesSync(p1, old, old);
  subtitleCache.clearMem();
  assert.strictEqual(await subtitleCache.get('cachecheck'), null, 'expired on disk');
  assert.ok(!fs.existsSync(p1), 'expired file removed');

  if (origDir === undefined) delete process.env.SUBTITLE_DISK_DIR; else process.env.SUBTITLE_DISK_DIR = origDir;
  if (origAuto === undefined) delete process.env.SUBTITLE_DISK_CACHE; else process.env.SUBTITLE_DISK_CACHE = origAuto;
  fs.rmSync(tmp, { recursive: true, force: true });
});
```

- [ ] **Step 2: Run, verify fail**

Run: `npm test`
Expected: FAIL — `Cannot find module './lib/subtitleCache'`.

- [ ] **Step 3: Create `lib/subtitleCache.js`**

```javascript
'use strict';

/**
 * Tiered cache for merged subtitle SRT files.
 *
 * L1 — in-memory Map (instance-local, fastest)
 * L2 — Upstash Redis (survives serverless invocations; optional via env)
 * L3 — disk (local deployments only; auto-disabled on Vercel)
 *
 * Every tier degrades gracefully: disabled/errored layers fall through to
 * the next, and a fully-down cache only costs re-generation.
 */

const fs = require('fs');
const path = require('path');
const crypto = require('crypto');
const { redisGet, redisSet } = require('./persistence');
const { debugServer } = require('./debug');

const TTL_MS = 6 * 60 * 60 * 1000;               // 6 hours, mirrors old behavior
const REDIS_TTL_SECONDS = Math.floor(TTL_MS / 1000);
const REDIS_MAX_BYTES = 512 * 1024;              // Upstash free-tier friendly cap

const memCache = new Map();

function isDiskEnabled() {
  const v = String(process.env.SUBTITLE_DISK_CACHE || '').toLowerCase();
  if (v === 'off') return false;
  if (v === 'on') return true;
  return !process.env.VERCEL;                    // auto: no persistent FS on Vercel
}

function diskDir() {
  return process.env.SUBTITLE_DISK_DIR || path.join(__dirname, '..', '.cache', 'subtitles');
}

function diskPath(key) {
  const file = crypto.createHash('sha1').update(key).digest('hex') + '.srt';
  return path.join(diskDir(), file);
}

function diskGet(key) {
  if (!isDiskEnabled()) return null;
  try {
    const file = diskPath(key);
    const stat = fs.statSync(file);
    if (Date.now() - stat.mtimeMs > TTL_MS) {
      fs.rmSync(file, { force: true });
      return null;
    }
    return fs.readFileSync(file, 'utf8');
  } catch (_) {
    return null;
  }
}

function diskPut(key, srt) {
  if (!isDiskEnabled()) return;
  try {
    const dir = diskDir();
    fs.mkdirSync(dir, { recursive: true });
    const target = diskPath(key);
    const tmp = `${target}.tmp`;
    fs.writeFileSync(tmp, srt, 'utf8');
    fs.renameSync(tmp, target);                  // atomic-ish replace
  } catch (err) {
    debugServer.warn(`Subtitle disk cache write failed: ${err && err.message}`);
  }
}

function pruneExpired() {
  const now = Date.now();
  for (const [k, v] of memCache.entries()) {
    if (now - v.timestamp > TTL_MS) memCache.delete(k);
  }
}

async function get(key) {
  // L1: memory
  const entry = memCache.get(key);
  if (entry) {
    if (Date.now() - entry.timestamp <= TTL_MS) return entry.content;
    memCache.delete(key);
  }

  // L2: Redis
  try {
    const redisValue = await redisGet(key);
    if (redisValue) {
      memCache.set(key, { content: redisValue, timestamp: Date.now() });
      debugServer.log(`Cache L2 hit: ${key}`);
      return redisValue;
    }
  } catch (_) { /* fall through */ }

  // L3: disk
  const diskValue = diskGet(key);
  if (diskValue) {
    memCache.set(key, { content: diskValue, timestamp: Date.now() });
    debugServer.log(`Cache L3 hit: ${key}`);
    return diskValue;
  }

  return null;
}

function put(key, srt) {
  memCache.set(key, { content: srt, timestamp: Date.now() });
  if (srt.length <= REDIS_MAX_BYTES) {
    redisSet(key, srt, REDIS_TTL_SECONDS).catch(() => {});
  }
  diskPut(key, srt);
  pruneExpired();
}

function clearMem() {
  memCache.clear();
}

module.exports = {
  get,
  put,
  clearMem,
  _test: { memCache, isDiskEnabled, diskPath }
};
```

- [ ] **Step 4: Wire into `addon.js`**

4a. Replace the cache block (lines ~635-672: `const subtitleCache = new Map();` through end of `function getSubtitle`) with:

```javascript
// Tiered cache for merged subtitles (memory → Upstash Redis → disk)
const subtitleCache = require('./lib/subtitleCache');

/**
 * Store subtitle in the tiered cache and return the key.
 */
function storeSubtitle(key, srtContent) {
  subtitleCache.put(key, srtContent);
  return key;
}

/**
 * Get subtitle from the tiered cache (async: may hit Redis/disk).
 */
async function getSubtitle(key) {
  return subtitleCache.get(key);
}
```

4b. In `generateDynamicSubtitle`, change `const cached = getSubtitle(cacheKey);` → `const cached = await getSubtitle(cacheKey);`.

4c. Exports: `subtitleCache` now exports the module object (keep key name — `module.exports = { builder, manifest, getSubtitle, subtitleCache, ... }` unchanged).

- [ ] **Step 5: Wire into `server.js` (async route)**

Replace route handler at `/subtitles/:filename` (line ~281) with an async body:

```javascript
app.get('/subtitles/:filename', async (req, res) => {
  const filename = req.params.filename;
  const cacheKey = filename.replace('.srt', '');

  const content = await getSubtitle(cacheKey);

  if (!content) {
    debugServer.warn(`Subtitle not found in cache: ${cacheKey}`);
    return res.status(404).send('Subtitle not found or expired');
  }

  trackSubtitleServed();

  res.setHeader('Content-Type', 'text/srt; charset=utf-8');
  res.setHeader('Cache-Control', 'public, max-age=21600');
  res.setHeader('Content-Disposition', `inline; filename="${filename}"`);
  res.send(content);
});
```

- [ ] **Step 6: Run, verify pass; smoke-test startup**

Run: `npm test`
Expected: PASS all, exit 0.

Then brief startup check:

```bash
node -e "require('./server.js')" against timeout
```

In PowerShell:

```powershell
node -e "require('./addon.js'); console.log('addon loads OK')"
node -e "const app=require('./server.js'); console.log('server loads OK')"
```

Expected: both print OK lines, exit 0 (server does not listen when VERCEL set is absent — it will listen briefly; the `-e` process stays alive ~ until Express listener keeps event loop busy; use instead):

```powershell
node -e "process.env.PORT='7999'; require('./server.js'); setTimeout(()=>process.exit(0), 1500)"
```

Expected: logs `Dual Subtitles Addon Started` + OK exit.

- [ ] **Step 7: Commit**

```bash
git add lib/subtitleCache.js addon.js server.js test.js
git commit -m "feat: tiered subtitle cache (memory/Upstash/disk) replaces in-memory Map"
```

---

### Task 7: `server.js` — parse both config encodings (in-app native config)

**Files:**
- Modify: `server.js` (add helper near `parseExtra`; refactor `/:config/manifest.json` ~348 and `/:config/subtitles/...` ~381 routes)
- Test: append to `test.js`

- [ ] **Step 1: Write the failing tests**

Append inside the IIFE (after Task 4/6 tests, before results):

```javascript
// server.js config parsing
console.log('\n--- server config parsing ---');

const serverMod = require('./server');
const { _test: serverTest } = serverMod;

await testAsync('parseConfigParam: compact pipe form', async () => {
  assert.deepStrictEqual(
    serverTest.parseConfigParam(encodeURIComponent('English [eng]|Turkish [tur]')),
    { mainLang: 'English [eng]', transLang: 'Turkish [tur]' }
  );
});

await testAsync('parseConfigParam: native JSON form (Stremio in-app config)', async () => {
  const cfg = { mainLang: 'English [eng]', transLang: 'Turkish [tur]', mainColor: 'Red (#f43f5e)', mainBold: 'false' };
  assert.deepStrictEqual(
    serverTest.parseConfigParam(encodeURIComponent(JSON.stringify(cfg))),
    cfg
  );
});

await testAsync('parseConfigParam: garbage returns null', async () => {
  assert.strictEqual(serverTest.parseConfigParam('plain-garbage-no-pipe'), null);
  assert.strictEqual(serverTest.parseConfigParam(''), null);
});
```

Note: `require('./server')` inside test.js — server.js starts listening when `!process.env.VERCEL`; set **`process.env.VERCEL = '1'` at the top of test.js** (below the requires) so requiring it never binds a port:

```javascript
process.env.VERCEL = '1'; // never bind ports in tests
```

and extend `module.exports = app;` at the bottom of `server.js` to:

```javascript
module.exports = app;
module.exports._test = { parseConfigParam, getClientIP };
```

- [ ] **Step 2: Run, verify fail**

Run: `npm test`
Expected: FAIL — `parseConfigParam` undefined.

- [ ] **Step 3: Implement in `server.js`**

3a. Add helper above the `/:config/manifest.json` route:

```javascript
// Parse the config path segment. Two encodings are supported:
//  1) compact : "English [eng]|Turkish [tur]"  (website-generated URLs)
//  2) native  : URL-encoded JSON {"mainLang": "...", ...}  (Stremio builds
//               this when the user configures the addon inside the app)
// Returns a config object (languages + optional style keys) or null.
function parseConfigParam(raw) {
  if (!raw) return null;
  let decoded;
  try {
    decoded = decodeURIComponent(raw);
  } catch (_) {
    return null;
  }
  if (decoded.trim().startsWith('{')) {
    try {
      const obj = JSON.parse(decoded);
      if (obj && typeof obj === 'object' && obj.mainLang && obj.transLang) return obj;
      return null;
    } catch (_) {
      return null;
    }
  }
  const [mainLang, transLang] = decoded.split('|');
  if (mainLang && transLang) return { mainLang, transLang };
  return null;
}
```

3b. Refactor `/:config/manifest.json` route body:

```javascript
app.get('/:config/manifest.json', (req, res) => {
  try {
    const parsed = parseConfigParam(req.params.config);
    if (!parsed) {
      return res.status(400).json({ error: 'Invalid configuration' });
    }

    const mainCode = parseLangCode(parsed.mainLang);
    const transCode = parseLangCode(parsed.transLang);
    const baseUrl = getExternalUrl(req);

    const configuredManifest = {
      ...manifest,
      id: `${manifest.id}.${mainCode}.${transCode}`,
      name: `${manifest.name} (${mainCode.toUpperCase()}+${transCode.toUpperCase()})`,
      logo: manifest.logo.startsWith('http') ? manifest.logo : `${baseUrl}${manifest.logo}`,
      behaviorHints: {
        ...manifest.behaviorHints,
        configurationRequired: false
      }
    };

    res.json(configuredManifest);
  } catch (error) {
    debugServer.error('Error generating manifest:', sanitizeForLogging(error.message));
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

3c. Refactor `/:config/subtitles/...` route head:

```javascript
app.get('/:config/subtitles/:type/:id/:extra?.json', async (req, res) => {
  try {
    const parsed = parseConfigParam(req.params.config);
    if (!parsed) {
      return res.status(400).json({ subtitles: [] });
    }

    const { type, id } = req.params;
    const extra = req.params.extra ? parseExtra(req.params.extra) : {};

    // parsed carries mainLang/transLang plus any style keys; the handler
    // reads exactly what it needs.
    const config = parsed;

    trackSubtitleRequest(parseLangCode(config.mainLang), parseLangCode(config.transLang), type, id);
    debugServer.log(`Subtitle request: ${type}/${id}, langs: ${parseLangCode(config.mainLang)}+${parseLangCode(config.transLang)}`);
```

(remaining body unchanged; `result.subtitles` mapping unchanged).

- [ ] **Step 4: Run, verify pass**

Run: `npm test`
Expected: PASS all, exit 0.

- [ ] **Step 5: Commit**

```bash
git add server.js test.js
git commit -m "feat: support Stremio native (JSON) addon config alongside pipe form"
```

---

### Task 8: `server.js` — pass style through `/subs` route

**Files:**
- Modify: `server.js` (`/subs/:...:transSubId.srt` route ~302-340)

- [ ] **Step 1: Implement**

In the `/subs/...` route, after the videoParams block add:

```javascript
  // Style code rides in the query (see styleRenderer docs). Website-
  // configured installs don't send it -> undefined -> defaults used.
  const styleCode = req.query && typeof req.query.style === 'string' ? req.query.style : undefined;
```

and pass it to the generator:

```javascript
    const content = await generateDynamicSubtitle(
      type, imdbId, season, episode, mainLang, transLang, mainSubId, transSubId, videoParams, styleCode
    );
```

- [ ] **Step 2: Verify**

Run: `npm test` — expected PASS (no new unit; startup check below covers loading).

Then live smoke test (server on port 7999):

```powershell
$env:PORT='7999'
node -e "require('./server.js')"
```

This starts a real listener; run the health check from a **second shell** or backgrounded:

```powershell
Start-Job -ScriptBlock { $env:PORT='7999'; Set-Location 'C:\Users\admin\stremio-dual-subtitles'; node server.js }
Start-Sleep -Seconds 2
curl.exe -s http://127.0.0.1:7999/health
Get-Job | Remove-Job -Force
```

Expected: `{"status":"ok","version":"1.2.0","uptime":...}`.

- [ ] **Step 3: Commit**

```bash
git add server.js
git commit -m "feat: /subs route forwards style code to dynamic generation"
```

---

### Task 9: Rate-limit store sweep (memory leak fix)

**Files:**
- Modify: `server.js` (after rate limit middleware block, ~line 128)
- Test: `test.js` addition

- [ ] **Step 1: Write the failing test**

Append inside IIFE:

```javascript
await testAsync('rate limiter: expired entries are swept', async () => {
  // Sweep implementation detail: exported sweep function must drop entries past resetAt.
  const sweepRateLimitStore = serverMod._test.sweepRateLimitStore;
  assert.strictEqual(typeof sweepRateLimitStore, 'function', 'sweepRateLimitStore missing');
  const store = serverMod._test.rateLimitStore;
  const now = Date.now();
  store.set('1.1.1.1', { count: 5, resetAt: now - 1000 });   // expired
  store.set('2.2.2.2', { count: 5, resetAt: now + 60000 });  // active
  sweepRateLimitStore();
  assert.ok(!store.has('1.1.1.1'), 'expired entry must be deleted');
  assert.ok(store.has('2.2.2.2'), 'active entry must survive');
  store.delete('2.2.2.2');
});
```

- [ ] **Step 2: Run, verify fail**

Run: `npm test` — expected FAIL ('sweepRateLimitStore missing').

- [ ] **Step 3: Implement**

In `server.js`, refactor the interval + expose internals. After the rate-limit middleware app.use block add:

```javascript
// Periodically drop expired IP buckets so the Map doesn't grow forever.
function sweepRateLimitStore() {
  const now = Date.now();
  for (const [ip, entry] of rateLimitStore) {
    if (now > entry.resetAt) rateLimitStore.delete(ip);
  }
}
const rateLimitSweeperTimer = setInterval(sweepRateLimitStore, RATE_LIMIT_WINDOW_MS);
rateLimitSweeperTimer.unref(); // don't keep the process alive just for sweeping
```

And in `server.js` bottom exports (Task 7 touched them):

```javascript
module.exports = app;
module.exports._test = { parseConfigParam, getClientIP, sweepRateLimitStore, rateLimitStore };
```

- [ ] **Step 4: Run, verify pass**

Run: `npm test` — PASS all, exit 0.

- [ ] **Step 5: Commit**

```bash
git add server.js test.js
git commit -m "fix: sweep expired rate-limit entries to prevent unbounded memory growth"
```

---

### Task 10: Keep-alive HTTP agents (axios)

**Files:**
- Modify: `addon.js` (imports + agent setup after `const axios = require('axios')` line 9)

- [ ] **Step 1: Implement**

In `addon.js`, after the axios require:

```javascript
const http = require('http');
const https = require('https');

// Reuse TLS connections/sockets across OpenSubtitles calls — a measurable
// cold-start win on serverless, worthwhile locally too.
const keepAliveHttpAgent = new http.Agent({ keepAlive: true, maxSockets: 32 });
const keepAliveHttpsAgent = new https.Agent({ keepAlive: true, maxSockets: 32 });
axios.defaults.httpAgent = keepAliveHttpAgent;
axios.defaults.httpsAgent = keepAliveHttpsAgent;
```

- [ ] **Step 2: Verify**

```powershell
node -e "const a=require('./addon.js'); const axios=require('axios'); console.log('keepAlive:', axios.defaults.httpsAgent.options.keepAlive)"
```

Expected output: `keepAlive: true`.

`npm test` — PASS, exit 0.

- [ ] **Step 3: Commit**

```bash
git add addon.js
git commit -m "perf: reuse keep-alive HTTP(S) agents for OpenSubtitles requests"
```

---

### Task 11: Docs — `.env.example`, README, CHANGELOG

**Files:**
- Modify: `.env.example` (append section)
- Modify: `README.md` (Add an "In-app configuration" subsection under Usage Guide)
- Modify: `CHANGELOG.md` (add 1.2.0 entry at top of changelog list)

- [ ] **Step 1: `.env.example` — append at end:**

```
# ============================================================================
# SUBTITLE CACHING (optional)
# ============================================================================

# Disk cache tier for merged subtitles. "auto" (default) enables it when
# self-hosting and disables it on Vercel (non-persistent filesystem).
# Values: auto (default) | on | off
# SUBTITLE_DISK_CACHE=auto

# Where the disk cache lives (only used when the disk tier is active).
# SUBTITLE_DISK_DIR=./.cache/subtitles
```

- [ ] **Step 2: README — insert after the "Configuration" section intro (or as new subsection right before "Network Access Setup"):**

```markdown
### In-App Configuration (No Website Needed)

After installing from the **base manifest** (`https://your-domain/manifest.json`),
Stremio shows its own configuration screen inside the app where you pick:

- Primary and Secondary languages
- Subtitle styling: primary/secondary color, bold/italic, the secondary
  line marker, and line order

Your choices are saved with the addon — no external website required. Color
picking uses preset names (e.g. `Red (#f43f5e)`); pick `None (player
default)` for the primary line to leave it unstyled.

> **Player caveat:** styling relies on SRT tags (`<font color>`, `<b>`,
> `<i>`). Clients that ignore those tags render plain text — this is a
> limitation of the player, not the addon.
```

- [ ] **Step 3: CHANGELOG — add at the top of the version list:**

```markdown
### v1.2.0 (2026-10-22)

**Added**
- Native in-app configuration: choose languages and subtitle style inside Stremio (no website redirect needed)
- Custom subtitle styling: primary/secondary line colors, bold/italic, secondary-line marker, line order swap
- Tiered subtitle caching: in-memory → Upstash Redis → disk (auto-detects environment)

**Fixed**
- `{{ADDON_URL}}` placeholder not resolved for native-config installs (blank subtitles when configuring in-app)
- Rate-limit store kept expired IP entries forever (memory leak)

**Changed**
- Keep-alive HTTP agents for upstream subtitle requests
- Version fields (manifest/package) now track 1.2.0
```

- [ ] **Step 4: Commit**

```bash
git add .env.example README.md CHANGELOG.md
git commit -m "docs: in-app configuration, styling options, cache env vars, v1.2.0 notes"
```

---

### Task 12: Full verification, push, PR

**Files:** none (verification + git operations)

- [ ] **Step 1: Full test suite**

Run: `npm test`
Expected: PASS all sections (`styleRenderer`, `mergeSubtitles style wiring`, `manifest config`, `subtitles URL plumbing`, `server config parsing`, `subtitleCache`, existing sections), `Results: N passed, 0 failed`, exit code 0.

- [ ] **Step 2: Live smoke test (languages via native-style JSON config)**

```powershell
$env:PORT='7999'
node server.js &   # background
```

Wait ~2s, then (from the second shell — same pattern as Task 8 with `Start-Job`):

```powershell
$env:PORT='7999'   # in the job's shell
node server.js     # background
```

Running via `Start-Job` in one command:

```powershell
Start-Job -ScriptBlock { $env:PORT='7999'; Set-Location 'C:\Users\admin\stremio-dual-subtitles'; node server.js }
Start-Sleep -Seconds 2
curl.exe -s http://127.0.0.1:7999/health
$cfg = [uri]::EscapeDataString('{"mainLang":"English [eng]","transLang":"Turkish [tur]"}')
curl.exe -s "http://127.0.0.1:7999/$cfg/manifest.json"
Get-Job | Remove-Job -Force
```

Expected: health returns `"version":"1.2.0"`; configured manifest returns `id: community.dualsubtitles.eng.tur` and the full config list incl. style keys.

- [ ] **Step 3: Push the branch to your fork**

```bash
git push -u origin feature/inapp-config-styling-caching
```

Expected: branch appears at `https://github.com/R3vll/stremio-dual-subtitles/tree/feature/inapp-config-styling-caching`.

- [ ] **Step 4: Open PR (fork → upstream)**

```bash
gh pr create --repo ummugulsunn/stremio-dual-subtitles \
  --base master --head R3vll:feature/inapp-config-styling-caching \
  --title "Native in-app configuration, custom styling, tiered caching (v1.2.0)" \
  --body "Implements: Stremio-native addon config (language + style selection inside the app), `{{ADDON_URL}}` resolution fix for native installs, configurable dual-subtitle styling (colors, bold/italic, marker, line order), 3-tier subtitle cache (memory/Upstash/disk with graceful fallback), rate-limit sweep fix, keep-alive agents, docs + changelog. All tests pass; legacy default output byte-identical to v1 formatting."
```

Expected: PR URL printed. (If fork restrictions block PR creation, the branch push alone satisfies the deliverable; report the PR URL or the restriction.)

- [ ] **Step 5: Report to user** — full test output summary, smoke-test health output, branch/PR URLs, plus Vercel deploy reminder (import fork → deploy → install via `https://<vercel-url>/manifest.json`, configure in-app).
