# MCS Phase 6 — Frontend (GitHub Pages) Agent Prompt (v4)

## Context

You are building the frontend dashboard for MCS (Music Creator Studio). This is Phase 6. Phases 0–5 are complete and verified.

Phase 6 builds five things:
- **`frontend/index.html`** — Home page: system state, active sessions summary, navigation entry point
- **`frontend/queue.html`** — Prompt queue: copyable stem prompts per session (requires session ID parameter)
- **`frontend/review.html`** — Candidate review: technical status, approval payload generator
- **`frontend/toggle.html`** — Track toggle interface (State 2): stem IDs and technical status only
- **`frontend/status.html`** — Pipeline status: checkpoint logs, session health, reconciliation state
- **`.github/workflows/mcs_approve.yml`** — Approval execution workflow

Plus one pipeline patch:
- **`backend/run_daily.py` extension** — writes `data/sessions/index.json` on every run so frontend can discover sessions

**No Python business logic in frontend. No server. No credentials in frontend. No creative content in frontend. No Drive links in frontend. No song concepts, seed themes, or titles in frontend. Vanilla JS only. No frameworks.**

**Stop on first failure. Print `[OK]` / `[FAIL]` diagnostics. Do not continue to the next task.**

---

## What Phase 6 Does (Read Before Writing Any Code)

The frontend is a technical control panel and prompt delivery tool. It reads JSON from the repo via `fetch()` and displays pipeline status and prompts. The only creative content anywhere in the frontend is on the queue page — and only when a valid session ID is provided as a URL parameter.

**Frontend displays:**
- Pipeline health, checkpoint states, session statuses
- Stem prompts with copy buttons (queue page only, session ID required)
- Approve/reject controls tied to session IDs (no song content)
- Stem IDs and technical verification status (no playback, no Drive links)

**Frontend never displays:**
- Song concepts, seed themes, expanded concepts
- Song titles beyond session IDs and technical slugs
- Drive links or any playable content
- Library entries or completed song metadata
- Any content a third party could extract and use

**Two routes into the queue page:**
- From email — direct link with session ID parameter (`queue.html?session=abc123`)
- From home page — session cards with "View Prompts" button per active session

**Queue page with no session parameter** — shows an empty state: "No session selected — open from home page or use the link in your email." Nothing useful to a random visitor.

**Execution bridge:** Francisco copies the approval payload from the review page → opens GitHub Actions → triggers `mcs_approve.yml` manually → workflow validates payload version → workflow calls `module9.run()` → session JSON updated → next fetch reflects new state.

---

## Architectural Rules (Read Before Writing Any Code)

1. **Multi-page navigation only.** Each page is a standalone HTML file. Navigation uses `<a href>` links. No SPA routing.
2. **All data via `fetch()`.** Pages fetch JSON at runtime. No data baked into HTML.
3. **Session index file is the discovery mechanism.** `data/sessions/index.json` lists all sessions with IDs and statuses. Frontend reads this first.
4. **Vanilla JS only.** No jQuery, no React, no Vue, no build step.
5. **No credentials in frontend.** The approval workflow requires GitHub Actions manual dispatch — the frontend generates the payload only.
6. **No Drive links anywhere in frontend.** Not on any page under any circumstance.
7. **No song concepts, seed themes, or creative content on any page except queue.** Queue page shows prompts only — nothing else creative.
8. **Queue page requires session ID parameter.** Without `?session=<id>` in the URL it shows an empty state. Never auto-selects a session on the queue page.
9. **Home page auto-selects the most recent active session for display.** This is safe because home page shows only technical metadata — no creative content.
10. **CSS variables for theming.** All colors from MCS palette defined as CSS vars.
11. **Every `fetch()` call has a catch handler** that displays a visible error message — never silent failure.
12. **`fetch()` paths are configurable via one constant block at the top of each page.**
13. **Every content area initializes with a visible loading state** before any `fetch()` resolves.
14. **Each page is fully self-contained.** Helper functions are defined inside each page's `<script>` block. `nav.js` is the only shared JS file and its only job is highlighting the active nav link.
15. **Approval payload must include `payload_version: "v1"`.** Always the first key in the payload object.
16. **P score inputs must be clamped** to 1–10 via HTML attributes AND via an `oninput` clamp handler. Payload generation validates all three scores before building payload.
17. **`track_type` defaulting.** If an asset is missing `track_type`, default to `"core"` and log a console warning. Never throw or block.
18. **No library page.** Completed song metadata is never exposed in the frontend.
19. **Status page replaces library page.** Shows pipeline health, checkpoint logs, and reconciliation state only.

---

## GitHub Pages Configuration

GitHub Pages serves from the **`/docs` folder** on the `main` branch.

- Frontend files live at `docs/index.html`, `docs/queue.html`, etc.
- Data files live at `data/sessions/index.json`, etc. in the repo root

**Data copy rule:** `run_daily.py` copies data files into `docs/data/` on every pipeline run so they are within the served directory.

**Fetch paths in all frontend pages:**
```javascript
const PATHS = {
    index:   'data/sessions/index.json',
    session: id => `data/sessions/${id}.json`,
};
```

No `../` prefix. Always relative to `docs/` which is the served root.

**`git add` in both workflows must include `docs/data/`:**
```yaml
git add data/sessions/ data/library/ docs/data/
```

**`.nojekyll` note:** `docs/.nojekyll` must exist or GitHub Pages will silently drop `docs/data/`. Create it as an empty file. Confirm `.gitignore` does not exclude `docs/data/` or `*.json` under `docs/` before first push.

---

## MCS Visual Identity

```css
:root {
    --pu: #9b59ff;
    --mg: #c850c0;
    --bl: #5c6fff;
    --up: #39e8a0;
    --dn: #ff4d6d;
    --nt: #ff9a3c;
    --bg: #0d0d0f;
    --surface: #16161a;
    --border: #2a2a35;
    --text: #e8e8f0;
    --muted: #6b6b80;
}
```

Typography: headings use `Orbitron`, monospace/data uses `Share Tech Mono`, body uses system-ui fallback.

---

## Data Schemas (Read Before Writing Any HTML/JS)

### `data/sessions/index.json`

```json
{
  "updated_at": "2026-04-01T12:00:00Z",
  "sessions": [
    {
      "session_id": "abc123",
      "status": "INITIALIZED" | "ACTIVE" | "BLOCKED" | "COMPLETE" | "DISCARDED",
      "mood": "dark",
      "genre": "trap",
      "tempo_bpm": 128,
      "created_at": "2026-04-01T12:00:00Z"
    }
  ]
}
```

**Note:** `seed_theme` and song titles are deliberately excluded from the index. The index contains only technical identifiers and pipeline status fields.

Sessions array is always sorted newest first by `created_at` descending by the writer. Frontend must also sort before filtering.

### Individual session file: `data/sessions/<session_id>.json`

Key fields the frontend uses — **technical fields only**:

```json
{
  "session_id": "abc123",
  "status": "ACTIVE",
  "created_at": "...",
  "song_plan": {
    "mood": "dark",
    "energy": "driving",
    "genre": "trap",
    "tempo_bpm": 128,
    "target_duration_seconds": 180,
    "harmonic_center": "C"
  },
  "prompt_metadata": [
    {
      "stem_id": "drums",
      "rendered_prompt": "...",
      "execution_priority": 1,
      "blueprint": { "role": "foundational" }
    }
  ],
  "assets": [
    {
      "stem_id": "drums",
      "sha256": "...",
      "status": "verified",
      "track_type": "core"
    }
  ],
  "checkpoints": [],
  "identity_vector": {}
}
```

**The frontend never reads or displays:** `seed_theme`, `notes`, `title`, `web_view_link`, `drive_link`, `releases_link`, `expanded_concept`, `diversity_vector`, or any field from `song_library.json`.

---

## Pipeline Patch — Session Index Writer and Data Mirror

Before writing any HTML, patch `backend/run_daily.py` with the following two additions.

### Addition 1 — `_write_session_index()`

```python
def _write_session_index() -> None:
    """
    Writes data/sessions/index.json and mirrors it to docs/data/sessions/index.json.
    Reads all session files and builds summary list sorted newest first.
    Contains only technical fields — no creative content.
    Never raises — prints warn on failure.
    """
    import json, shutil
    from datetime import datetime, timezone

    INDEX_FILE      = ROOT / "data" / "sessions" / "index.json"
    DOCS_INDEX_FILE = ROOT / "docs" / "data" / "sessions" / "index.json"
    try:
        sessions = m13.list_sessions()
        entries  = []
        for s in sessions:
            plan = s.get("song_plan", {})
            entries.append({
                "session_id": s.get("session_id", ""),
                "status":     s.get("status", ""),
                "mood":       plan.get("mood", ""),
                "genre":      plan.get("genre", ""),
                "tempo_bpm":  plan.get("tempo_bpm", ""),
                "created_at": s.get("created_at", "")
            })
        entries.sort(key=lambda x: x["created_at"], reverse=True)
        index = {
            "updated_at": datetime.now(timezone.utc).isoformat(),
            "sessions":   entries
        }
        INDEX_FILE.parent.mkdir(parents=True, exist_ok=True)
        tmp = INDEX_FILE.with_suffix(".tmp")
        tmp.write_text(json.dumps(index, indent=2))
        tmp.replace(INDEX_FILE)
        print(f"[OK] Session index written — {len(entries)} sessions")
        DOCS_INDEX_FILE.parent.mkdir(parents=True, exist_ok=True)
        shutil.copy2(INDEX_FILE, DOCS_INDEX_FILE)
        print(f"[OK] Session index mirrored to docs/data/")
    except Exception as e:
        print(f"[WARN] Could not write session index — {e}")
```

### Addition 2 — Mirror individual session files to docs/data

Also mirror individual session JSON files so the queue page can fetch them:

```python
def _mirror_sessions() -> None:
    """
    Mirrors all files in data/sessions/ to docs/data/sessions/.
    Never raises — prints warn on failure.
    """
    import shutil

    SRC_DIR  = ROOT / "data" / "sessions"
    DEST_DIR = ROOT / "docs" / "data" / "sessions"
    try:
        DEST_DIR.mkdir(parents=True, exist_ok=True)
        for f in SRC_DIR.glob("*.json"):
            shutil.copy2(f, DEST_DIR / f.name)
        print(f"[OK] Session files mirrored to docs/data/sessions/")
    except Exception as e:
        print(f"[WARN] Could not mirror session files — {e}")
```

### Call order in `_run_pipeline()`

Insert both calls after the Module 4 email step, before any `sys.exit(0)`:

```python
        # Phase 6 — Write and mirror frontend data files
        _write_session_index()
        _mirror_sessions()
```

---

## Task 1 — Jekyll Disable, Shared CSS, and Navigation JS

**First**, create an empty file at `docs/.nojekyll`.

Then create `docs/style.css`:

```css
@import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@400;600;700&family=Share+Tech+Mono&display=swap');

:root {
    --pu: #9b59ff;
    --mg: #c850c0;
    --bl: #5c6fff;
    --up: #39e8a0;
    --dn: #ff4d6d;
    --nt: #ff9a3c;
    --bg: #0d0d0f;
    --surface: #16161a;
    --border: #2a2a35;
    --text: #e8e8f0;
    --muted: #6b6b80;
}

*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

body {
    background: var(--bg);
    color: var(--text);
    font-family: system-ui, sans-serif;
    font-size: 15px;
    line-height: 1.6;
    min-height: 100vh;
}

a { color: var(--pu); text-decoration: none; }
a:hover { text-decoration: underline; }

.container { max-width: 900px; margin: 0 auto; padding: 2rem 1.5rem; }

.nav {
    border-bottom: 1px solid var(--border);
    padding: 1rem 1.5rem;
    display: flex;
    align-items: center;
    gap: 2rem;
    overflow-x: auto;
}
.nav-brand {
    font-family: Orbitron, sans-serif;
    font-size: 0.85rem;
    font-weight: 700;
    letter-spacing: 0.2em;
    color: var(--pu);
    text-transform: uppercase;
    white-space: nowrap;
}
.nav-links { display: flex; gap: 1.5rem; }
.nav-links a {
    font-size: 0.8rem;
    color: var(--muted);
    letter-spacing: 0.05em;
    white-space: nowrap;
    transition: color 0.15s;
}
.nav-links a:hover, .nav-links a.active { color: var(--text); text-decoration: none; }

.section-label {
    font-family: Orbitron, sans-serif;
    font-size: 0.65rem;
    letter-spacing: 0.2em;
    text-transform: uppercase;
    color: var(--muted);
    margin-bottom: 1rem;
}

.card {
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 6px;
    padding: 1.5rem;
    margin-bottom: 1rem;
}
.card-title {
    font-family: Orbitron, sans-serif;
    font-size: 0.8rem;
    font-weight: 600;
    letter-spacing: 0.1em;
    color: var(--text);
    margin-bottom: 0.75rem;
}

.pill {
    display: inline-block;
    font-family: 'Share Tech Mono', monospace;
    font-size: 0.7rem;
    letter-spacing: 0.08em;
    padding: 0.2rem 0.6rem;
    border-radius: 3px;
    border: 1px solid currentColor;
}
.pill-initialized { color: var(--nt); }
.pill-active      { color: var(--up); }
.pill-blocked     { color: var(--dn); }
.pill-complete    { color: var(--bl); }
.pill-discarded   { color: var(--muted); }
.pill-state1      { color: var(--pu); border-color: var(--pu); }
.pill-state2      { color: var(--mg); border-color: var(--mg); }

.btn {
    display: inline-block;
    font-family: 'Share Tech Mono', monospace;
    font-size: 0.75rem;
    letter-spacing: 0.08em;
    padding: 0.5rem 1rem;
    border-radius: 4px;
    border: 1px solid var(--pu);
    color: var(--pu);
    background: transparent;
    cursor: pointer;
    transition: background 0.15s, color 0.15s;
}
.btn:hover { background: rgba(155, 89, 255, 0.12); }
.btn-up { border-color: var(--up); color: var(--up); }
.btn-up:hover { background: rgba(57, 232, 160, 0.12); }
.btn-dn { border-color: var(--dn); color: var(--dn); }
.btn-dn:hover { background: rgba(255, 77, 109, 0.12); }
.btn.copied { border-color: var(--up); color: var(--up); }

.mono {
    font-family: 'Share Tech Mono', monospace;
    font-size: 0.8rem;
    color: var(--muted);
}
.code-block {
    background: var(--bg);
    border: 1px solid var(--border);
    border-radius: 4px;
    padding: 1rem;
    font-family: 'Share Tech Mono', monospace;
    font-size: 0.78rem;
    white-space: pre-wrap;
    word-break: break-all;
    color: var(--text);
}

.meta-grid {
    display: grid;
    grid-template-columns: auto 1fr;
    gap: 0.3rem 1.5rem;
    font-size: 0.82rem;
}
.meta-label { color: var(--muted); font-family: 'Share Tech Mono', monospace; font-size: 0.75rem; }
.meta-value { color: var(--text); }

.score-row { display: flex; align-items: center; gap: 1rem; margin-bottom: 0.75rem; }
.score-label { font-family: 'Share Tech Mono', monospace; font-size: 0.75rem; color: var(--muted); width: 100px; }
.score-input {
    width: 60px;
    background: var(--bg);
    border: 1px solid var(--border);
    border-radius: 4px;
    color: var(--text);
    font-family: 'Share Tech Mono', monospace;
    font-size: 0.85rem;
    padding: 0.35rem 0.5rem;
    text-align: center;
}
.score-input:focus { outline: none; border-color: var(--pu); }
.score-input.invalid { border-color: var(--dn); }

.state-msg {
    font-family: 'Share Tech Mono', monospace;
    font-size: 0.82rem;
    color: var(--muted);
    padding: 2rem;
    text-align: center;
}
.state-error { color: var(--dn); }
.state-warn  { color: var(--nt); }

.stem-row {
    display: flex;
    align-items: center;
    gap: 1rem;
    padding: 0.75rem 0;
    border-bottom: 1px solid var(--border);
}
.stem-row:last-child { border-bottom: none; }
.stem-id {
    font-family: 'Share Tech Mono', monospace;
    font-size: 0.8rem;
    color: var(--pu);
    width: 120px;
    flex-shrink: 0;
}

.divider { border: none; border-top: 1px solid var(--border); margin: 1.5rem 0; }

.payload-wrapper { position: relative; }
.payload-copy-btn { position: absolute; top: 0.5rem; right: 0.5rem; }
.payload-error {
    font-family: 'Share Tech Mono', monospace;
    font-size: 0.78rem;
    color: var(--dn);
    margin-top: 0.5rem;
}

.checkpoint-row {
    display: flex;
    gap: 1rem;
    padding: 0.5rem 0;
    border-bottom: 1px solid var(--border);
    font-family: 'Share Tech Mono', monospace;
    font-size: 0.75rem;
}
.checkpoint-row:last-child { border-bottom: none; }
.checkpoint-module { color: var(--pu); width: 140px; flex-shrink: 0; }
.checkpoint-time   { color: var(--muted); width: 140px; flex-shrink: 0; }
.checkpoint-status-ok   { color: var(--up); }
.checkpoint-status-fail { color: var(--dn); }
.checkpoint-status-warn { color: var(--nt); }
```

Create `docs/nav.js`:

```javascript
(function () {
    const page = location.pathname.split('/').pop() || 'index.html';
    document.querySelectorAll('.nav-links a').forEach(a => {
        const href = a.getAttribute('href').split('/').pop().split('?')[0];
        if (href === page) a.classList.add('active');
    });
})();
```

---

## Task 2 — Home Page (`docs/index.html`)

Displays system state and active sessions. Each session card shows only technical metadata — mood, genre, BPM, status, created date — and a "View Prompts" button linking to `queue.html?session=<id>`. No seed themes. No song titles beyond the session ID slug.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MCS — Home</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>

<nav class="nav">
    <span class="nav-brand">MCS</span>
    <div class="nav-links">
        <a href="index.html">Home</a>
        <a href="review.html">Review</a>
        <a href="toggle.html">Toggle</a>
        <a href="status.html">Status</a>
    </div>
</nav>

<div class="container">

    <div id="state-banner" class="card" style="margin-bottom:2rem;">
        <div class="section-label">System State</div>
        <div id="state-display"><div class="state-msg">Loading...</div></div>
    </div>

    <div class="section-label">Active Sessions</div>
    <div id="sessions-list"><div class="state-msg">Loading...</div></div>

    <div id="error-banner" class="state-msg state-error" style="display:none;"></div>

</div>

<script>
const PATHS = {
    index:   'data/sessions/index.json',
    session: id => `data/sessions/${id}.json`,
};

function escHtml(str) {
    return String(str)
        .replace(/&/g, '&amp;').replace(/</g, '&lt;')
        .replace(/>/g, '&gt;').replace(/"/g, '&quot;');
}

function statusPill(status) {
    const cls = {
        INITIALIZED: 'pill-initialized',
        ACTIVE:      'pill-active',
        BLOCKED:     'pill-blocked',
        COMPLETE:    'pill-complete',
        DISCARDED:   'pill-discarded',
    }[status] || '';
    return `<span class="pill ${cls}">${escHtml(status)}</span>`;
}

function deriveState(sessions) {
    return sessions.some(s => s.status === 'ACTIVE') ? 2 : 1;
}

function renderSessionCard(s) {
    return `
    <div class="card">
        <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:0.75rem;">
            <span class="card-title mono">${escHtml(s.session_id)}</span>
            ${statusPill(s.status)}
        </div>
        <div class="meta-grid" style="margin-bottom:1rem;">
            <span class="meta-label">Mood</span>
            <span class="meta-value">${escHtml(s.mood || '—')}</span>
            <span class="meta-label">Genre</span>
            <span class="meta-value">${escHtml(s.genre || '—')}</span>
            <span class="meta-label">BPM</span>
            <span class="meta-value">${escHtml(String(s.tempo_bpm || '—'))}</span>
            <span class="meta-label">Created</span>
            <span class="meta-value mono">${s.created_at ? s.created_at.slice(0,16).replace('T',' ') : '—'}</span>
        </div>
        <div style="display:flex;gap:0.75rem;flex-wrap:wrap;">
            <a class="btn" href="queue.html?session=${escHtml(s.session_id)}">View Prompts</a>
            <a class="btn btn-up" href="review.html?session=${escHtml(s.session_id)}">Review</a>
            <a class="btn" href="toggle.html?session=${escHtml(s.session_id)}">Toggle</a>
            <a class="btn" href="status.html?session=${escHtml(s.session_id)}">Status</a>
        </div>
    </div>`;
}

async function init() {
    try {
        const resp = await fetch(PATHS.index);
        if (!resp.ok) throw new Error(`HTTP ${resp.status}`);
        const data = await resp.json();

        const sessions = (data.sessions || []).slice();
        sessions.sort((a, b) => (b.created_at || '').localeCompare(a.created_at || ''));

        const active   = sessions.filter(s => !['COMPLETE','DISCARDED'].includes(s.status));
        const sysState = deriveState(sessions);

        document.getElementById('state-display').innerHTML = `
            <span class="pill ${sysState === 2 ? 'pill-state2' : 'pill-state1'}"
                  style="font-size:0.9rem;padding:0.4rem 1rem;">
                STATE ${sysState} — ${sysState === 1 ? 'Candidate Generation' : 'Refinement'}
            </span>
            <span style="margin-left:1rem;font-size:0.8rem;color:var(--muted);">
                ${active.length} active session${active.length !== 1 ? 's' : ''}
                &nbsp;·&nbsp; Updated ${data.updated_at ? data.updated_at.slice(0,16).replace('T',' ') : '—'}
            </span>`;

        const list = document.getElementById('sessions-list');
        list.innerHTML = active.length === 0
            ? '<div class="state-msg">No active sessions — pipeline will generate a new plan at next scheduled run.</div>'
            : active.map(renderSessionCard).join('');

    } catch (err) {
        document.getElementById('error-banner').style.display = 'block';
        document.getElementById('error-banner').textContent   = `Failed to load session index: ${err.message}`;
        document.getElementById('state-display').innerHTML    = '';
        document.getElementById('sessions-list').innerHTML    = '';
    }
}

init();
</script>
<script src="nav.js"></script>
</body>
</html>
```

---

## Task 3 — Prompt Queue Page (`docs/queue.html`)

Displays stem prompts for a specific session. **Requires `?session=<id>` in URL.** Without it shows empty state. This is the only page that shows creative prompt content.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MCS — Prompt Queue</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>

<nav class="nav">
    <span class="nav-brand">MCS</span>
    <div class="nav-links">
        <a href="index.html">Home</a>
        <a href="review.html">Review</a>
        <a href="toggle.html">Toggle</a>
        <a href="status.html">Status</a>
    </div>
</nav>

<div class="container">
    <div id="header-area"><div class="state-msg">Loading...</div></div>
    <div id="queue-list"></div>
    <div id="error-banner" class="state-msg state-error" style="display:none;"></div>
</div>

<script>
const PATHS = {
    session: id => `data/sessions/${id}.json`,
};

function escHtml(str) {
    return String(str)
        .replace(/&/g, '&amp;').replace(/</g, '&lt;')
        .replace(/>/g, '&gt;').replace(/"/g, '&quot;');
}

function getSessionId() {
    return new URLSearchParams(location.search).get('session');
}

function priorityLabel(p) {
    return ['', 'Foundational', 'Harmonic', 'Texture'][p] || `P${p}`;
}

function copyPrompt(id, btn) {
    const text = document.getElementById(id).textContent;
    navigator.clipboard.writeText(text).then(() => {
        btn.textContent = 'Copied';
        btn.classList.add('copied');
        setTimeout(() => { btn.textContent = 'Copy'; btn.classList.remove('copied'); }, 1500);
    });
}

function renderPromptCard(entry, i) {
    const bp       = entry.blueprint || {};
    const promptId = `prompt-${i}`;
    return `
    <div class="card">
        <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:0.75rem;">
            <span class="card-title">STEM ${i + 1} — ${escHtml(entry.stem_id.toUpperCase())}</span>
            <span class="mono" style="font-size:0.72rem;">
                Priority ${entry.execution_priority || '?'} · ${priorityLabel(entry.execution_priority)}
            </span>
        </div>
        <div class="meta-grid" style="margin-bottom:1rem;">
            <span class="meta-label">Role</span>
            <span class="meta-value">${escHtml(bp.role || '—')}</span>
        </div>
        <div class="payload-wrapper">
            <pre class="code-block" id="${promptId}">${escHtml(entry.rendered_prompt || '[No rendered prompt found]')}</pre>
            <button class="btn payload-copy-btn" onclick="copyPrompt('${promptId}', this)">Copy</button>
        </div>
    </div>`;
}

async function init() {
    const sid = getSessionId();

    if (!sid) {
        document.getElementById('header-area').innerHTML = `
            <div class="state-msg">
                No session selected.<br>
                Open from the <a href="index.html">home page</a> or use the link in your email.
            </div>`;
        return;
    }

    try {
        const resp    = await fetch(PATHS.session(sid));
        if (!resp.ok) throw new Error(`Session fetch failed: HTTP ${resp.status}`);
        const session = await resp.json();

        const plan  = session.song_plan      || {};
        const queue = session.prompt_metadata || [];

        document.getElementById('header-area').innerHTML = `
            <div class="card" style="margin-bottom:1.5rem;">
                <div class="section-label">Prompt Queue</div>
                <div class="meta-grid">
                    <span class="meta-label">Session</span>
                    <span class="meta-value mono">${escHtml(sid)}</span>
                    <span class="meta-label">Mood</span>
                    <span class="meta-value">${escHtml(plan.mood || '—')}</span>
                    <span class="meta-label">Energy</span>
                    <span class="meta-value">${escHtml(plan.energy || '—')}</span>
                    <span class="meta-label">Genre</span>
                    <span class="meta-value">${escHtml(plan.genre || '—')}</span>
                    <span class="meta-label">BPM</span>
                    <span class="meta-value">${escHtml(String(plan.tempo_bpm || '—'))}</span>
                    <span class="meta-label">Stems</span>
                    <span class="meta-value">${queue.length}</span>
                </div>
                <p style="margin-top:1rem;font-size:0.8rem;color:var(--muted);">
                    Copy each prompt → paste into Gemini App → download audio → upload stems within 24 hours.
                </p>
            </div>`;

        const list = document.getElementById('queue-list');
        if (queue.length === 0) {
            list.innerHTML = '<div class="state-msg">No prompts in queue for this session.</div>';
        } else {
            const sorted = [...queue].sort((a, b) => (a.execution_priority || 0) - (b.execution_priority || 0));
            list.innerHTML = sorted.map((entry, i) => renderPromptCard(entry, i)).join('');
        }

    } catch (err) {
        document.getElementById('header-area').innerHTML = '';
        document.getElementById('error-banner').style.display  = 'block';
        document.getElementById('error-banner').textContent    = `Error: ${err.message}`;
    }
}

init();
</script>
<script src="nav.js"></script>
</body>
</html>
```

---

## Task 4 — Candidate Review Page (`docs/review.html`)

Displays stems with technical status only. No Drive links. No song content. Generates approval payload. Uses session ID from URL parameter, falls back to most recent active session from index.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MCS — Review</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>

<nav class="nav">
    <span class="nav-brand">MCS</span>
    <div class="nav-links">
        <a href="index.html">Home</a>
        <a href="review.html">Review</a>
        <a href="toggle.html">Toggle</a>
        <a href="status.html">Status</a>
    </div>
</nav>

<div class="container">
    <div id="header-area"><div class="state-msg">Loading...</div></div>
    <div id="stems-area" style="margin-bottom:1.5rem;"></div>
    <div id="score-area" style="margin-bottom:1.5rem;"></div>
    <div id="payload-area"></div>
    <div id="error-banner" class="state-msg state-error" style="display:none;"></div>
</div>

<script>
const PATHS = {
    index:   'data/sessions/index.json',
    session: id => `data/sessions/${id}.json`,
};

function escHtml(str) {
    return String(str)
        .replace(/&/g, '&amp;').replace(/</g, '&lt;')
        .replace(/>/g, '&gt;').replace(/"/g, '&quot;');
}

async function resolveSessionId() {
    const explicit = new URLSearchParams(location.search).get('session');
    if (explicit) return explicit;
    const resp = await fetch(PATHS.index);
    if (!resp.ok) throw new Error(`Index fetch failed: HTTP ${resp.status}`);
    const data = await resp.json();
    const sessions = (data.sessions || []).slice();
    sessions.sort((a, b) => (b.created_at || '').localeCompare(a.created_at || ''));
    const candidate = sessions.find(s => ['INITIALIZED','ACTIVE'].includes(s.status));
    if (!candidate) throw new Error('No active session found');
    return candidate.session_id;
}

function clampScore(input) {
    let v = parseInt(input.value);
    if (isNaN(v)) { input.classList.add('invalid'); return; }
    if (v < 1)  { v = 1;  input.value = 1; }
    if (v > 10) { v = 10; input.value = 10; }
    input.classList.remove('invalid');
    refreshPayload();
}

const approvedStems = new Set();
let _session = null;

function toggleStem(stemId, checked) {
    if (checked) approvedStems.add(stemId);
    else approvedStems.delete(stemId);
    refreshPayload();
}

function renderStemRow(asset) {
    const sid    = asset.stem_id;
    const status = asset.status || 'pending';
    const statusCls = status === 'verified' ? 'pill-active' : 'pill-initialized';
    return `
    <div class="stem-row">
        <input type="checkbox" id="cb-${escHtml(sid)}"
               onchange="toggleStem('${escHtml(sid)}', this.checked)"
               style="accent-color:var(--up);width:16px;height:16px;cursor:pointer;flex-shrink:0;">
        <span class="stem-id">${escHtml(sid)}</span>
        <span class="mono" style="font-size:0.72rem;color:var(--muted);">
            ${asset.track_type || 'core'}
        </span>
        <span class="pill ${statusCls}" style="margin-left:auto;">${escHtml(status)}</span>
    </div>`;
}

function refreshPayload() {
    if (!_session) return;

    const vibeEl   = document.getElementById('score-vibe');
    const origEl   = document.getElementById('score-originality');
    const cohEl    = document.getElementById('score-coherence');
    const flagEl   = document.getElementById('score-flag');
    const preEl    = document.getElementById('payload-json');
    const errEl    = document.getElementById('payload-inline-error');

    const vibe        = vibeEl ? parseInt(vibeEl.value)  : NaN;
    const originality = origEl ? parseInt(origEl.value)  : NaN;
    const coherence   = cohEl  ? parseInt(cohEl.value)   : NaN;
    const pflag       = flagEl ? flagEl.value             : 'safe';

    const validScore  = v => Number.isInteger(v) && v >= 1 && v <= 10;
    const scoresValid = validScore(vibe) && validScore(originality) && validScore(coherence);

    if (!scoresValid) {
        if (preEl) preEl.textContent = '{}';
        if (errEl) errEl.textContent = 'Enter valid P scores (1–10) to generate payload.';
        return;
    }
    if (errEl) errEl.textContent = '';

    const approvedIds = [...approvedStems];
    const stemPaths   = {};
    approvedIds.forEach(sid => { stemPaths[sid] = `data/audio/${sid}.mp3`; });

    const payload = {
        payload_version:   "v1",
        session_id:        _session.session_id,
        approved_stem_ids: approvedIds,
        stem_local_paths:  stemPaths,
        perceptual_score: { vibe, originality, coherence, flag: pflag }
    };

    if (preEl) preEl.textContent = JSON.stringify(payload, null, 2);
}

function copyPayload(btn) {
    const text = document.getElementById('payload-json').textContent;
    if (text === '{}') return;
    navigator.clipboard.writeText(text).then(() => {
        btn.textContent = 'Copied';
        btn.classList.add('copied');
        setTimeout(() => { btn.textContent = 'Copy Payload'; btn.classList.remove('copied'); }, 1500);
    });
}

async function init() {
    try {
        const sid  = await resolveSessionId();
        const resp = await fetch(PATHS.session(sid));
        if (!resp.ok) throw new Error(`Session fetch failed: HTTP ${resp.status}`);
        _session   = await resp.json();

        const plan   = _session.song_plan || {};
        const assets = _session.assets    || [];

        document.getElementById('header-area').innerHTML = `
            <div class="card" style="margin-bottom:1.5rem;">
                <div class="section-label">Candidate Review</div>
                <div class="meta-grid">
                    <span class="meta-label">Session</span>
                    <span class="meta-value mono">${escHtml(sid)}</span>
                    <span class="meta-label">Status</span>
                    <span class="meta-value">${escHtml(_session.status || '—')}</span>
                    <span class="meta-label">Mood</span>
                    <span class="meta-value">${escHtml(plan.mood || '—')} · ${escHtml(plan.energy || '—')} · ${escHtml(plan.genre || '—')}</span>
                    <span class="meta-label">BPM</span>
                    <span class="meta-value">${escHtml(String(plan.tempo_bpm || '—'))}</span>
                    <span class="meta-label">Duration</span>
                    <span class="meta-value">${escHtml(String(plan.target_duration_seconds || '—'))}s</span>
                </div>
            </div>`;

        const stemsHtml = assets.length === 0
            ? '<div class="state-msg">No stems uploaded yet.</div>'
            : assets.map(a => renderStemRow(a)).join('');

        document.getElementById('stems-area').innerHTML = `
            <div class="section-label">Stems</div>
            <div class="card" style="padding:0.5rem 1.5rem;">
                <p style="font-size:0.75rem;color:var(--muted);padding:0.5rem 0 0.75rem;
                           border-bottom:1px solid var(--border);margin-bottom:0.25rem;">
                    Check stems to include in approval payload.
                </p>
                ${stemsHtml}
            </div>`;

        document.getElementById('score-area').innerHTML = `
            <div class="section-label">Perceptual Score</div>
            <div class="card">
                <div class="score-row">
                    <span class="score-label">Vibe</span>
                    <input class="score-input" id="score-vibe" type="number"
                           min="1" max="10" placeholder="1–10" oninput="clampScore(this)">
                </div>
                <div class="score-row">
                    <span class="score-label">Originality</span>
                    <input class="score-input" id="score-originality" type="number"
                           min="1" max="10" placeholder="1–10" oninput="clampScore(this)">
                </div>
                <div class="score-row">
                    <span class="score-label">Coherence</span>
                    <input class="score-input" id="score-coherence" type="number"
                           min="1" max="10" placeholder="1–10" oninput="clampScore(this)">
                </div>
                <div class="score-row">
                    <span class="score-label">Flag</span>
                    <select id="score-flag" onchange="refreshPayload()"
                        style="background:var(--bg);border:1px solid var(--border);
                               color:var(--text);font-family:'Share Tech Mono',monospace;
                               font-size:0.8rem;border-radius:4px;padding:0.35rem 0.5rem;cursor:pointer;">
                        <option value="safe">safe</option>
                        <option value="stretch">stretch</option>
                    </select>
                </div>
            </div>`;

        document.getElementById('payload-area').innerHTML = `
            <div class="section-label">Approval Payload</div>
            <div class="card">
                <p style="font-size:0.8rem;color:var(--muted);margin-bottom:1rem;">
                    1. Check stems above &nbsp;·&nbsp;
                    2. Enter P scores &nbsp;·&nbsp;
                    3. Copy payload &nbsp;·&nbsp;
                    4. GitHub Actions → <strong style="color:var(--text);">mcs_approve</strong> → Run workflow → paste payload
                </p>
                <div class="payload-wrapper">
                    <pre class="code-block" id="payload-json">{}</pre>
                    <button class="btn payload-copy-btn" onclick="copyPayload(this)">Copy Payload</button>
                </div>
                <div class="payload-error" id="payload-inline-error"></div>
            </div>`;

        refreshPayload();

    } catch (err) {
        document.getElementById('header-area').innerHTML = '';
        document.getElementById('error-banner').style.display = 'block';
        document.getElementById('error-banner').textContent   = `Error: ${err.message}`;
    }
}

init();
</script>
<script src="nav.js"></script>
</body>
</html>
```

---

## Task 5 — Track Toggle Interface (`docs/toggle.html`)

State 2 interface. Separates core vs refinement stems by `track_type`. Technical status only. No Drive links. No playback. Generates refinement approval payload.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MCS — Toggle</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>

<nav class="nav">
    <span class="nav-brand">MCS</span>
    <div class="nav-links">
        <a href="index.html">Home</a>
        <a href="review.html">Review</a>
        <a href="toggle.html">Toggle</a>
        <a href="status.html">Status</a>
    </div>
</nav>

<div class="container">
    <div id="header-area"><div class="state-msg">Loading...</div></div>
    <div id="core-area" style="margin-bottom:1.5rem;"></div>
    <div id="refinement-area" style="margin-bottom:1.5rem;"></div>
    <div id="payload-area"></div>
    <div id="error-banner" class="state-msg state-error" style="display:none;"></div>
</div>

<script>
const PATHS = {
    index:   'data/sessions/index.json',
    session: id => `data/sessions/${id}.json`,
};

function escHtml(str) {
    return String(str)
        .replace(/&/g, '&amp;').replace(/</g, '&lt;')
        .replace(/>/g, '&gt;').replace(/"/g, '&quot;');
}

async function resolveSessionId() {
    const explicit = new URLSearchParams(location.search).get('session');
    if (explicit) return explicit;
    const resp = await fetch(PATHS.index);
    if (!resp.ok) throw new Error(`Index fetch failed: HTTP ${resp.status}`);
    const data = await resp.json();
    const sessions = (data.sessions || []).slice();
    sessions.sort((a, b) => (b.created_at || '').localeCompare(a.created_at || ''));
    const candidate = sessions.find(s => s.status === 'ACTIVE');
    if (!candidate) throw new Error('No ACTIVE session found');
    return candidate.session_id;
}

function resolveTrackType(asset) {
    if (!asset.track_type) {
        console.warn(`[WARN] asset ${asset.stem_id} missing track_type — defaulting to core`);
        return 'core';
    }
    return asset.track_type;
}

const approvedRefinements = new Set();
let _session = null;

function toggleRefinement(stemId, checked) {
    if (checked) approvedRefinements.add(stemId);
    else approvedRefinements.delete(stemId);
    refreshPayload();
}

function renderStemRow(asset, isRefinement) {
    const sid    = asset.stem_id;
    const status = asset.status || 'pending';
    const statusCls = status === 'verified' ? 'pill-active' : 'pill-initialized';
    const checkbox = isRefinement
        ? `<input type="checkbox" id="cb-${escHtml(sid)}"
               onchange="toggleRefinement('${escHtml(sid)}', this.checked)"
               style="accent-color:var(--up);width:16px;height:16px;cursor:pointer;flex-shrink:0;">`
        : `<span style="width:16px;display:inline-block;flex-shrink:0;"></span>`;
    return `
    <div class="stem-row">
        ${checkbox}
        <span class="stem-id">${escHtml(sid)}</span>
        <span class="mono" style="font-size:0.72rem;color:var(--muted);">
            ${isRefinement ? 'refinement' : 'core'}
        </span>
        <span class="pill ${statusCls}" style="margin-left:auto;">${escHtml(status)}</span>
    </div>`;
}

function refreshPayload() {
    if (!_session) return;
    const approvedIds = [...approvedRefinements];
    const stemPaths   = {};
    approvedIds.forEach(sid => { stemPaths[sid] = `data/audio/${sid}.mp3`; });
    const payload = {
        payload_version:   "v1",
        session_id:        _session.session_id,
        approved_stem_ids: approvedIds,
        stem_local_paths:  stemPaths,
        refinement:        true
    };
    document.getElementById('payload-json').textContent = JSON.stringify(payload, null, 2);
}

function copyPayload(btn) {
    if (approvedRefinements.size === 0) {
        btn.textContent = 'Select a stem first';
        setTimeout(() => { btn.textContent = 'Copy Payload'; }, 1500);
        return;
    }
    const text = document.getElementById('payload-json').textContent;
    navigator.clipboard.writeText(text).then(() => {
        btn.textContent = 'Copied';
        btn.classList.add('copied');
        setTimeout(() => { btn.textContent = 'Copy Payload'; btn.classList.remove('copied'); }, 1500);
    });
}

async function init() {
    try {
        const sid  = await resolveSessionId();
        const resp = await fetch(PATHS.session(sid));
        if (!resp.ok) throw new Error(`Session fetch failed: HTTP ${resp.status}`);
        _session   = await resp.json();

        const plan   = _session.song_plan || {};
        const assets = (_session.assets || []).map(a => ({
            ...a,
            track_type: resolveTrackType(a)
        }));

        const core        = assets.filter(a => a.track_type !== 'refinement');
        const refinements = assets.filter(a => a.track_type === 'refinement');

        document.getElementById('header-area').innerHTML = `
            <div class="card" style="margin-bottom:1.5rem;">
                <div class="section-label">Track Toggle — State 2</div>
                <div class="meta-grid">
                    <span class="meta-label">Session</span>
                    <span class="meta-value mono">${escHtml(sid)}</span>
                    <span class="meta-label">Status</span>
                    <span class="meta-value">${escHtml(_session.status || '—')}</span>
                    <span class="meta-label">Mood</span>
                    <span class="meta-value">${escHtml(plan.mood || '—')} · ${escHtml(plan.energy || '—')}</span>
                    <span class="meta-label">Core stems</span>
                    <span class="meta-value">${core.length}</span>
                    <span class="meta-label">Refinements</span>
                    <span class="meta-value">${refinements.length}</span>
                </div>
            </div>`;

        document.getElementById('core-area').innerHTML = `
            <div class="section-label">Core Tracks (Locked)</div>
            <div class="card" style="padding:0.5rem 1.5rem;">
                ${core.length === 0
                    ? '<div class="state-msg">No core stems uploaded yet.</div>'
                    : core.map(a => renderStemRow(a, false)).join('')}
            </div>`;

        const refinementBody = refinements.length === 0
            ? `<div class="state-msg state-warn">
                   No refinement stems tagged — ensure Module 13 sets track_type on all assets.
               </div>`
            : refinements.map(a => renderStemRow(a, true)).join('');

        document.getElementById('refinement-area').innerHTML = `
            <div class="section-label">Refinement Tracks</div>
            <div class="card" style="padding:0.5rem 1.5rem;">
                ${refinementBody}
            </div>`;

        document.getElementById('payload-area').innerHTML = `
            <div class="section-label">Refinement Approval Payload</div>
            <div class="card">
                <p style="font-size:0.8rem;color:var(--muted);margin-bottom:1rem;">
                    Check refinement stems → Copy payload → GitHub Actions → mcs_approve → Run workflow
                </p>
                <div class="payload-wrapper">
                    <pre class="code-block" id="payload-json">{}</pre>
                    <button class="btn payload-copy-btn" onclick="copyPayload(this)">Copy Payload</button>
                </div>
            </div>`;

        refreshPayload();

    } catch (err) {
        document.getElementById('header-area').innerHTML = '';
        document.getElementById('error-banner').style.display = 'block';
        document.getElementById('error-banner').textContent   = `Error: ${err.message}`;
    }
}

init();
</script>
<script src="nav.js"></script>
</body>
</html>
```

---

## Task 6 — Pipeline Status Page (`docs/status.html`)

Shows checkpoint logs, session health, and reconciliation state. Purely technical. No creative content.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MCS — Status</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>

<nav class="nav">
    <span class="nav-brand">MCS</span>
    <div class="nav-links">
        <a href="index.html">Home</a>
        <a href="review.html">Review</a>
        <a href="toggle.html">Toggle</a>
        <a href="status.html">Status</a>
    </div>
</nav>

<div class="container">
    <div id="header-area"><div class="state-msg">Loading...</div></div>
    <div id="checkpoints-area" style="margin-bottom:1.5rem;"></div>
    <div id="assets-area" style="margin-bottom:1.5rem;"></div>
    <div id="error-banner" class="state-msg state-error" style="display:none;"></div>
</div>

<script>
const PATHS = {
    index:   'data/sessions/index.json',
    session: id => `data/sessions/${id}.json`,
};

function escHtml(str) {
    return String(str)
        .replace(/&/g, '&amp;').replace(/</g, '&lt;')
        .replace(/>/g, '&gt;').replace(/"/g, '&quot;');
}

async function resolveSessionId() {
    const explicit = new URLSearchParams(location.search).get('session');
    if (explicit) return explicit;
    const resp = await fetch(PATHS.index);
    if (!resp.ok) throw new Error(`Index fetch failed: HTTP ${resp.status}`);
    const data = await resp.json();
    const sessions = (data.sessions || []).slice();
    sessions.sort((a, b) => (b.created_at || '').localeCompare(a.created_at || ''));
    const candidate = sessions.find(s => !['COMPLETE','DISCARDED'].includes(s.status));
    if (!candidate) throw new Error('No active session found');
    return candidate.session_id;
}

function checkpointStatusClass(data) {
    const s = (data.status || '').toLowerCase();
    if (s === 'ok')   return 'checkpoint-status-ok';
    if (s === 'fail') return 'checkpoint-status-fail';
    if (s === 'warn') return 'checkpoint-status-warn';
    return '';
}

function renderCheckpointRow(cp) {
    const ts     = cp.timestamp ? cp.timestamp.slice(0,16).replace('T',' ') : '—';
    const status = (cp.data || {}).status || '';
    const cls    = checkpointStatusClass(cp.data || {});
    return `
    <div class="checkpoint-row">
        <span class="checkpoint-module">${escHtml(cp.module || '—')}</span>
        <span class="checkpoint-time">${escHtml(ts)}</span>
        <span class="${cls}">${escHtml(status || (cp.data || {}).event || '—')}</span>
    </div>`;
}

function renderAssetRow(asset) {
    const status    = asset.status || 'pending';
    const statusCls = status === 'verified' ? 'pill-active' : 'pill-initialized';
    return `
    <div class="stem-row">
        <span class="stem-id">${escHtml(asset.stem_id || '—')}</span>
        <span class="mono" style="font-size:0.72rem;color:var(--muted);">
            ${escHtml(asset.track_type || 'core')}
        </span>
        <span class="mono" style="font-size:0.7rem;color:var(--muted);margin-left:auto;margin-right:1rem;">
            ${asset.sha256 ? asset.sha256.slice(0,12) + '...' : 'no hash'}
        </span>
        <span class="pill ${statusCls}">${escHtml(status)}</span>
    </div>`;
}

async function init() {
    try {
        const sid  = await resolveSessionId();
        const resp = await fetch(PATHS.session(sid));
        if (!resp.ok) throw new Error(`Session fetch failed: HTTP ${resp.status}`);
        const session = await resp.json();

        const plan        = session.song_plan   || {};
        const checkpoints = session.checkpoints || [];
        const assets      = session.assets      || [];

        document.getElementById('header-area').innerHTML = `
            <div class="card" style="margin-bottom:1.5rem;">
                <div class="section-label">Session Status</div>
                <div class="meta-grid">
                    <span class="meta-label">Session</span>
                    <span class="meta-value mono">${escHtml(sid)}</span>
                    <span class="meta-label">Status</span>
                    <span class="meta-value">${escHtml(session.status || '—')}</span>
                    <span class="meta-label">Mood</span>
                    <span class="meta-value">${escHtml(plan.mood || '—')} · ${escHtml(plan.energy || '—')} · ${escHtml(plan.genre || '—')}</span>
                    <span class="meta-label">BPM</span>
                    <span class="meta-value">${escHtml(String(plan.tempo_bpm || '—'))}</span>
                    <span class="meta-label">Created</span>
                    <span class="meta-value mono">${session.created_at ? session.created_at.slice(0,16).replace('T',' ') : '—'}</span>
                    <span class="meta-label">Updated</span>
                    <span class="meta-value mono">${session.updated_at ? session.updated_at.slice(0,16).replace('T',' ') : '—'}</span>
                    <span class="meta-label">Checkpoints</span>
                    <span class="meta-value">${checkpoints.length}</span>
                    <span class="meta-label">Assets</span>
                    <span class="meta-value">${assets.length}</span>
                </div>
            </div>`;

        document.getElementById('checkpoints-area').innerHTML = `
            <div class="section-label">Checkpoint Log</div>
            <div class="card" style="padding:0.5rem 1.5rem;">
                ${checkpoints.length === 0
                    ? '<div class="state-msg">No checkpoints recorded yet.</div>'
                    : [...checkpoints].reverse().map(renderCheckpointRow).join('')}
            </div>`;

        document.getElementById('assets-area').innerHTML = `
            <div class="section-label">Asset Registry</div>
            <div class="card" style="padding:0.5rem 1.5rem;">
                ${assets.length === 0
                    ? '<div class="state-msg">No assets registered yet.</div>'
                    : assets.map(renderAssetRow).join('')}
            </div>`;

    } catch (err) {
        document.getElementById('header-area').innerHTML = '';
        document.getElementById('error-banner').style.display = 'block';
        document.getElementById('error-banner').textContent   = `Error: ${err.message}`;
    }
}

init();
</script>
<script src="nav.js"></script>
</body>
</html>
```

---

## Task 7 — Approval Execution Workflow

Create `.github/workflows/mcs_approve.yml`:

```yaml
name: MCS Approval Execution

on:
  workflow_dispatch:
    inputs:
      approval_payload:
        description: "JSON payload from MCS review page"
        required: true

jobs:
  approve:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Create required directories
        run: mkdir -p data/sessions data/audio data/library docs/data/sessions temp

      - name: Run approval
        env:
          GEMINI_API_KEY:          ${{ secrets.GEMINI_API_KEY }}
          MCS_DRIVE_CREDENTIALS_1: ${{ secrets.MCS_DRIVE_CREDENTIALS_1 }}
          MCS_DRIVE_CREDENTIALS_2: ${{ secrets.MCS_DRIVE_CREDENTIALS_2 }}
          MCS_DRIVE_CREDENTIALS_3: ${{ secrets.MCS_DRIVE_CREDENTIALS_3 }}
          APPROVAL_PAYLOAD:        ${{ github.event.inputs.approval_payload }}
        run: |
          python3.11 - <<'EOF'
          import os, json, sys
          sys.path.insert(0, '.')
          import backend.module9 as m9

          raw = os.environ.get("APPROVAL_PAYLOAD", "").strip()
          if not raw:
              print("[FAIL] APPROVAL_PAYLOAD is empty")
              sys.exit(1)

          try:
              payload = json.loads(raw)
          except json.JSONDecodeError as e:
              print(f"[FAIL] Invalid JSON in APPROVAL_PAYLOAD: {e}")
              sys.exit(1)

          version = payload.get("payload_version")
          if version != "v1":
              print(f"[FAIL] Invalid or missing payload_version — expected 'v1', got {repr(version)}")
              sys.exit(1)
          print(f"[OK] payload_version: {version}")

          session_id        = payload.get("session_id")
          approved_stem_ids = payload.get("approved_stem_ids", [])
          stem_local_paths  = payload.get("stem_local_paths", {})

          if not session_id:
              print("[FAIL] Missing session_id in payload")
              sys.exit(1)

          print(f"Running approval — session: {session_id}, stems: {approved_stem_ids}")
          result = m9.run(session_id, approved_stem_ids, stem_local_paths)
          print(json.dumps(result, indent=2))

          if result.get("status") == "fail":
              print("[FAIL] Approval execution failed")
              sys.exit(1)
          print("[OK] Approval complete")
          EOF

      - name: Commit session updates
        run: |
          git config user.name "mcs-bot"
          git config user.email "mcs-bot@users.noreply.github.com"
          git add data/sessions/ data/library/ docs/data/
          git diff --staged --quiet || (git pull --rebase --autostash && git commit -m "chore: approval update [$(date -u +%Y-%m-%dT%H:%M:%SZ)]" && git push)
```

---

## Task 8 — Verification Script

Create `docs/verify_phase6.py` (run from repo root):

```python
from pathlib import Path
import sys, json

ROOT   = Path(__file__).resolve().parent.parent
failed = False

print("=" * 60)
print("MCS Phase 6 — Verification (v4)")
print("=" * 60)
print()

# ── Frontend files exist ───────────────────────────────────────────────────
frontend_files = [
    "docs/.nojekyll",
    "docs/style.css",
    "docs/nav.js",
    "docs/index.html",
    "docs/queue.html",
    "docs/review.html",
    "docs/toggle.html",
    "docs/status.html",
]
for f in frontend_files:
    path = ROOT / f
    if path.exists():
        print(f"[OK] {f} — exists")
    else:
        print(f"[FAIL] {f} — missing")
        failed = True

# ── No library.html ────────────────────────────────────────────────────────
lib_path = ROOT / "docs" / "library.html"
if not lib_path.exists():
    print("[OK] library.html — correctly absent")
else:
    print("[FAIL] library.html — must not exist (library page removed for privacy)")
    failed = True

# ── Workflow file ──────────────────────────────────────────────────────────
wf = ROOT / ".github" / "workflows" / "mcs_approve.yml"
if wf.exists():
    content = wf.read_text()
    checks = [
        ("workflow_dispatch",       "workflow_dispatch trigger"),
        ("approval_payload",        "approval_payload input"),
        ("APPROVAL_PAYLOAD",        "APPROVAL_PAYLOAD env var"),
        ("payload_version",         "payload_version validation"),
        ("module9",                 "module9 reference"),
        ("git pull --rebase",       "rebase push"),
        ("--autostash",             "autostash"),
        ("MCS_DRIVE_CREDENTIALS_1", "Drive credentials"),
        ("docs/data/",              "docs/data/ in git add"),
    ]
    for needle, label in checks:
        if needle in content:
            print(f"[OK] mcs_approve.yml — {label} present")
        else:
            print(f"[FAIL] mcs_approve.yml — {label} missing")
            failed = True
else:
    print("[FAIL] .github/workflows/mcs_approve.yml — missing")
    failed = True

# ── CSS variables and required classes ────────────────────────────────────
css_path = ROOT / "docs" / "style.css"
css = css_path.read_text() if css_path.exists() else ""
for var in ["--pu", "--mg", "--bl", "--up", "--dn", "--nt", "--bg", "--surface", "--border"]:
    if var in css:
        print(f"[OK] style.css — {var} defined")
    else:
        print(f"[FAIL] style.css — {var} missing")
        failed = True
for cls in [".state-warn", ".state-error", ".payload-error", ".score-input",
            ".checkpoint-row", ".checkpoint-module"]:
    if cls in css:
        print(f"[OK] style.css — {cls} defined")
    else:
        print(f"[FAIL] style.css — {cls} missing")
        failed = True

# ── Per-page checks ────────────────────────────────────────────────────────
page_checks = {
    "index.html": {
        "must_contain": [
            "fetch(", "nav.js", "style.css", "Loading...",
            "created_at", "data/sessions/index.json",
            "View Prompts", "queue.html?session=",
        ],
        "must_not_contain": [
            "seed_theme", "expanded_concept", "song_library",
            "drive_link", "web_view_link", "library.html",
        ],
    },
    "queue.html": {
        "must_contain": [
            "fetch(", "nav.js", "style.css", "Loading...",
            "getSessionId", "No session selected",
            "index.html", "execution_priority",
            "rendered_prompt", "Copy",
        ],
        "must_not_contain": [
            "seed_theme", "expanded_concept", "resolveSessionId",
            "drive_link", "web_view_link",
        ],
    },
    "review.html": {
        "must_contain": [
            "fetch(", "nav.js", "style.css", "Loading...",
            "resolveSessionId", "payload_version",
            "clampScore", "payload-inline-error",
            "data/sessions/index.json",
        ],
        "must_not_contain": [
            "seed_theme", "expanded_concept", "web_view_link",
            "drive_link", "Open in Drive", "library.html",
        ],
    },
    "toggle.html": {
        "must_contain": [
            "fetch(", "nav.js", "style.css", "Loading...",
            "resolveSessionId", "payload_version",
            "track_type", "resolveTrackType",
            "approvedRefinements.size === 0",
            "data/sessions/index.json",
        ],
        "must_not_contain": [
            "seed_theme", "expanded_concept", "web_view_link",
            "drive_link", "Open in Drive", "library.html",
        ],
    },
    "status.html": {
        "must_contain": [
            "fetch(", "nav.js", "style.css", "Loading...",
            "resolveSessionId", "checkpoint",
            "sha256", "track_type",
            "data/sessions/index.json",
        ],
        "must_not_contain": [
            "seed_theme", "expanded_concept", "web_view_link",
            "drive_link", "song_library", "library.html",
        ],
    },
}

for page, rules in page_checks.items():
    path = ROOT / "docs" / page
    if not path.exists():
        print(f"[FAIL] docs/{page} — file missing")
        failed = True
        continue
    content = path.read_text()
    issues  = []

    for needle in rules["must_contain"]:
        if needle not in content:
            issues.append(f"missing: {repr(needle)}")

    for forbidden in rules["must_not_contain"]:
        if forbidden in content:
            issues.append(f"forbidden content: {repr(forbidden)}")

    if "../data/" in content:
        issues.append("uses ../data/ prefix — should be data/ (no ../)")
    if "MCS_EMAIL_PASSWORD" in content or "GEMINI_API_KEY" in content:
        issues.append("credential string found in frontend")

    if issues:
        print(f"[FAIL] docs/{page} — {', '.join(issues)}")
        failed = True
    else:
        print(f"[OK] docs/{page} — all checks passed")

# ── queue.html: requires session param, no auto-select ─────────────────────
try:
    q = (ROOT / "docs" / "queue.html").read_text()
    assert "getSessionId" in q,      "getSessionId function missing"
    assert "resolveSessionId" not in q, "queue page must not auto-select session"
    assert "No session selected" in q,  "empty state message missing"
    print("[OK] queue.html — requires session param, no auto-select")
except AssertionError as e:
    print(f"[FAIL] queue.html session param — {e}")
    failed = True

# ── nav.js: clean ──────────────────────────────────────────────────────────
nav_path = ROOT / "docs" / "nav.js"
if nav_path.exists():
    nav_content = nav_path.read_text()
    if "fetch(" in nav_content or "payload" in nav_content:
        print("[FAIL] nav.js — contains unexpected logic")
        failed = True
    else:
        print("[OK] nav.js — clean navigation only")

# ── run_daily.py patched ───────────────────────────────────────────────────
run_daily = ROOT / "backend" / "run_daily.py"
if run_daily.exists():
    rd = run_daily.read_text()
    for fn in ["_write_session_index", "_mirror_sessions", "docs/data"]:
        if fn in rd:
            print(f"[OK] run_daily.py — {repr(fn)} present")
        else:
            print(f"[FAIL] run_daily.py — {repr(fn)} missing")
            failed = True
    # Confirm seed_theme not written to index
    idx_fn_start = rd.find("def _write_session_index")
    idx_fn_end   = rd.find("\ndef ", idx_fn_start + 1)
    idx_fn_body  = rd[idx_fn_start:idx_fn_end] if idx_fn_end > 0 else rd[idx_fn_start:]
    if "seed_theme" in idx_fn_body:
        print("[FAIL] run_daily._write_session_index — seed_theme must not be written to index")
        failed = True
    else:
        print("[OK] run_daily._write_session_index — seed_theme correctly excluded from index")
else:
    print("[FAIL] backend/run_daily.py — not found")
    failed = True

# ── docs/data mirror directories ──────────────────────────────────────────
for d in ["docs/data/sessions"]:
    p = ROOT / d
    if p.exists():
        print(f"[OK] {d}/ — directory exists")
    else:
        print(f"[WARN] {d}/ — not yet created (created on first pipeline run)")

print()
if failed:
    print("[FAIL] Phase 6 verification failed — fix all [FAIL] items above")
    sys.exit(1)
else:
    print("[OK] Phase 6 verification complete")
    sys.exit(0)
```

Run from repo root:

```bash
python3.11 docs/verify_phase6.py
```

---

## Completion Gate

Phase 6 is complete when ALL of the following are true:

1. `verify_phase6.py` exits code 0 — all `[OK]`
2. `docs/.nojekyll` exists and is empty
3. `.gitignore` manually confirmed to not exclude `docs/data/`
4. `library.html` does not exist in `docs/`
5. All 5 HTML pages open in browser without console errors
6. `index.html` shows system state and session cards with technical metadata only — no seed themes, no song titles
7. `queue.html` shows empty state when no `?session=` parameter present; shows prompts with copy buttons when valid session ID provided
8. `review.html` shows stems with technical status only — no Drive links, no song content; generates payload with `payload_version: "v1"` when scores valid
9. `toggle.html` separates core vs refinement by `track_type`; no Drive links; Copy Payload blocked when zero refinement stems selected
10. `status.html` shows checkpoint log and asset registry — technical fields only
11. `mcs_approve.yml` rejects payloads missing `payload_version: "v1"`
12. `_write_session_index()` excludes `seed_theme` from index entries
13. `_mirror_sessions()` mirrors individual session files to `docs/data/sessions/`
14. Both functions called unconditionally inside `_run_pipeline()` after Module 4 step
15. All `fetch()` paths use `data/...` with no `../` prefix
16. No credentials, Drive links, song concepts, or creative content in any frontend file
17. Navigation highlights active page via `nav.js`
18. `git add data/sessions/ data/library/ docs/data/` in both workflow commit steps

---

## What You Must Not Do

- Do not render seed themes, song concepts, expanded concepts, or any creative content on any page
- Do not render Drive links or playback links on any page under any circumstance
- Do not build a library page — it is explicitly removed
- Do not auto-select a session on the queue page — session ID must come from URL parameter
- Do not include `seed_theme` in the session index JSON written by `_write_session_index()`
- Do not put business logic in frontend files
- Do not make GitHub API calls from browser
- Do not hardcode session IDs, paths, or credentials in HTML/JS
- Do not use any JS framework or build tool
- Do not silently swallow `fetch()` errors
- Do not scatter JSON path strings through JS logic
- Do not commit audio files or stems
- Do not call `module9.run()` from the frontend
- Do not render a silent empty state on toggle page when no refinement stems tagged — show explicit warning
- Do not skip `payload_version: "v1"` in any generated payload
- Do not create `utils.js` or any shared JS file other than `nav.js`
- Do not skip creating `docs/.nojekyll`
- Do not allow Copy Payload in `toggle.html` when zero refinement stems selected
- Do not use `../data/` path prefix in any `fetch()` call
- Do not leave content areas blank while fetching — initialize with `Loading...`