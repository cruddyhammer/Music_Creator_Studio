# MCS Phase 4 — Email, Scheduling, and Auto-Discard Agent Prompt (v5)

## Context

You are building the scheduling layer for MCS (Music Creator Studio). This is Phase 4. Phases 0 (environment setup), 1 (data layer, Module 13), 2 (audio analysis foundation), and 3 (core backend modules, State 1 pipeline) are already complete and verified.

Phase 4 builds four things:
- **Module 4** — Email Notification (SMTP, sends song plan summary + GitHub Pages link only — no prompts in email)
- **`session_manager.py`** — Auto-discard logic, session lifecycle utilities, and manual trigger deduplication guard
- **`gemini_renderer.py` extension** — Two-step Gemini flow: Step 1 generates song concept from theme seed, Step 2 renders stem blueprints using that concept as context
- **GitHub Actions workflow** — Daily scheduled trigger (12PM UTC, 7 days/week), full State 1 chain, auto-discard check, manual dispatch support

**No frontend. No Drive. No audio processing beyond what Phase 3 already built. No new dependencies beyond `requirements.txt` and Python stdlib.**

**Stop on first failure. Print `[OK]` / `[FAIL]` diagnostics. Do not continue to the next task.**

---

## What Phase 4 Does (Read Before Writing Any Code)

Every day at 12PM UTC the GitHub Actions workflow:

1. Acquires a run lock — the workflow `concurrency` block prevents overlapping scheduled and manual triggers at the Actions level before any code runs
2. Runs the auto-discard check — any session in `INITIALIZED` status older than 24 hours with no stems uploaded is marked `DISCARDED`, checkpointed, and logged
3. Runs the full State 1 pipeline: Module 1 (stateless) → Module 2 → Module 3 → Module 3.5
4. If Module 3.5 passes: calls `gemini_renderer.generate_song_concept()` (Step 1) to expand the theme seed into a plain English song concept, then calls `gemini_renderer.render_queue()` (Step 2) to render all blueprints into natural language stem prompts
5. Calls Module 4 to send the email notification — the email contains the song concept and technical plan only, with a direct link to the frontend queue page where prompts are available
6. Writes final checkpoint to session

Francisco reads the email, follows the link to the frontend queue page to copy prompts, pastes each stem prompt into Gemini App manually, downloads stems, and uploads them. If no stems are uploaded within 24 hours, the next day's workflow run discards the session before generating a new one.

**Rendered stem prompts are never included in the email. The email is a notification and navigation tool only. All prompts live exclusively on the frontend queue page.**

**Scheduled runs always create a new session. Manual dispatch reuses an existing INITIALIZED session if one exists and is under 24 hours old — otherwise creates a new session. Session reuse is atomic: the session is immediately transitioned to `ACTIVE` via `compare_and_set_status()` before any further processing.**

---

## Architectural Rules (Read Before Writing Any Code)

1. **All session mutations must occur only through Module 13 public functions.** Never write to session dicts directly.
2. **Every module writes a checkpoint via `module13.write_checkpoint()` before passing control.** No module exits without checkpointing.
3. **No new dependencies may be added.** Use only what is already in `requirements.txt` plus Python stdlib (`smtplib`, `email`, `datetime`, `os`, `pathlib`, `fcntl`). SMTP email uses stdlib only — no third-party mail library.
4. **`DISCARDED` is a new valid status.** Add it to `VALID_STATUSES` and `VALID_TRANSITIONS` in Module 13. `INITIALIZED → DISCARDED` is the only new transition. `DISCARDED` is a terminal state — no transitions out.
5. **Auto-discard never touches candidates or approved tracks.** Only sessions in `INITIALIZED` status with zero verified assets and age > 24 hours are eligible. Sessions in `ACTIVE`, `BLOCKED`, `COMPLETE`, or already `DISCARDED` status are never touched by auto-discard.
6. **No vocals constraint is a hard rule.** Enforced in Module 3's `_build_negative_constraints` and in `gemini_renderer.py`'s `_build_render_prompt`. Never soft or advisory.
7. **Email credentials are GitHub Actions secrets only.** Read from environment variables `MCS_EMAIL_ADDRESS` and `MCS_EMAIL_PASSWORD`. Never hardcode. Sender and receiver are the same address.
8. **GitHub Pages link in email is read from environment variable `MCS_PAGES_URL`.** Never hardcode a URL.
9. **Standard failure envelope applies to Module 4.** Same shape as all Phase 3 modules.
10. **Path rule applies.** All scripts derive root from `Path(__file__).resolve().parents[N]`. Never hardcode absolute paths.
11. **Module 1 runs stateless.** It does not receive or create a session ID. It returns a constraint profile dict only. No bootstrap session is ever created.
12. **`get_active_session()` is a deduplication guard for manual triggers only.** Never called during scheduled pipeline runs. Scheduled runs always create a new session without consulting it.
13. **Gemini Step 1 has a fallback.** If `generate_song_concept()` fails, use the raw seed theme text directly as the concept. Store the sanitized error string as `fallback_reason`. The pipeline does not stop.
14. **`send_report()` never raises.** Returns a structured result dict on all code paths. `run()` reads the result and decides whether to return a fail envelope.
15. **Auto-discard writes a checkpoint per discarded session.** Not just a print statement.
16. **Concurrency is controlled at two layers:** (a) GitHub Actions `concurrency` block — prevents overlapping workflow runs at the Actions level; (b) `fcntl.flock` lock file in `run_daily.py` — prevents overlapping runs if Actions concurrency is somehow bypassed. Both are required.
17. **Session reuse via `compare_and_set_status()` is atomic.** This new Module 13 function reads, verifies, and writes status under a file-level lock in a single operation. No external code may call `update_status()` for a reuse transition — it must use `compare_and_set_status()`.
18. **If entropy dimension is missing from Module 1 checkpoints, use `"unknown"` and add `"entropy_missing"` to the session flags list.** Never silently degrade without flagging.
19. **`fallback_reason` in Gemini concept results must be a sanitized string only.** Never store raw exception objects.
20. **Rendered stem prompts are never included in the email body.** The email contains only the song concept, technical plan, stem count, pages link, and 24 hour warning. Prompts live exclusively on the frontend queue page.

---

## Path Rule (Mandatory — Read Before Writing Any Code)

Every script and module in Phase 4 must define paths derived from its own file location:

```python
ROOT         = Path(__file__).resolve().parents[1]
SESSIONS_DIR = ROOT / "data" / "sessions"
LOCK_FILE    = ROOT / "temp" / "run.lock"
```

Never hardcode absolute paths. Never assume a working directory.

---

## Standard Failure Envelope (Same as Phase 3 — All `run()` Functions Must Use)

```python
{
    "module":     str,
    "session_id": str,
    "status":     "ok" | "warn" | "fail",
    "flags":      [str, ...],
    "errors":     [str, ...],
    "data":       dict
}
```

---

## Module 13 Dependency Reference (Read Before Writing Any Code)

Phase 4 depends on the following Module 13 public functions built and verified in Phase 1:

| Function | Purpose |
|---|---|
| `create_session(song_plan: dict) -> dict` | Creates new session, returns session dict |
| `load_session(session_id: str) -> dict` | Loads session from disk |
| `update_status(session_id: str, new_status: str)` | Validates transition and writes new status |
| `write_checkpoint(session_id: str, module: str, data: dict)` | Appends checkpoint to session |
| `reconcile_state(session_id: str) -> dict` | Validates session integrity |

Phase 4 also adds two new public functions to Module 13 (`list_sessions` and `compare_and_set_status`) in Task 0.

---

## Preconditions

Confirm Phases 0–3 are complete before starting:

1. Folder structure exists: `/backend`, `/data`, `/data/sessions`, `/data/audio`, `/data/library`, `/temp`
2. `requirements.txt` present and all packages installed
3. `backend/__init__.py` exists
4. `backend/module13.py` exists and imports cleanly
5. `backend/module1.py` through `backend/module3_5.py` exist and import cleanly
6. `backend/gemini_renderer.py` exists and imports cleanly
7. `backend/config.py` exists and imports cleanly
8. Test audio file exists at: `data/audio/heavy_gauge.mp3`

If any are missing, print `[FAIL] Precondition failed — <what is missing>` and stop.

---

## Task 0 — Module 13 Extension

**Before writing any other file**, make the following surgical changes to `backend/module13.py`. Do not restructure any existing logic.

### Change 1 — `VALID_STATUSES`

```python
# Before
VALID_STATUSES = {"INITIALIZED", "ACTIVE", "BLOCKED", "COMPLETE"}

# After
VALID_STATUSES = {"INITIALIZED", "ACTIVE", "BLOCKED", "COMPLETE", "DISCARDED"}
```

### Change 2 — `VALID_TRANSITIONS`

```python
# Before
VALID_TRANSITIONS = {
    "INITIALIZED": {"ACTIVE", "BLOCKED"},
    "ACTIVE":      {"BLOCKED", "COMPLETE"},
    "BLOCKED":     {"ACTIVE"},
    "COMPLETE":    set()
}

# After
VALID_TRANSITIONS = {
    "INITIALIZED": {"ACTIVE", "BLOCKED", "DISCARDED"},
    "ACTIVE":      {"BLOCKED", "COMPLETE"},
    "BLOCKED":     {"ACTIVE"},
    "COMPLETE":    set(),
    "DISCARDED":   set()
}
```

### Change 3 — `reconcile_state` status check

The existing check in `reconcile_state` that validates `status` is in `VALID_STATUSES` will automatically cover `DISCARDED` once the constant is updated. Confirm this is the case — no logic change needed.

### Change 4 — Add `list_sessions()` public function

```python
def list_sessions() -> list:
    """
    Returns a list of all session dicts currently in SESSIONS_DIR.
    Skips files that fail to load or parse — never raises.
    Returns empty list if no sessions exist.
    """
    sessions = []
    for path in SESSIONS_DIR.glob("*.json"):
        try:
            session = _load_session_raw(path.stem)
            sessions.append(session)
        except Exception:
            pass
    return sessions
```

### Change 5 — Add `compare_and_set_status()` public function

```python
def compare_and_set_status(session_id: str, expected: str, new: str) -> bool:
    """
    Atomically transitions session status from `expected` to `new`.
    Returns True if transition succeeded.
    Returns False if current status does not match `expected` (another process claimed it first).
    Raises ValueError if the transition is not in VALID_TRANSITIONS.
    Never raises on file I/O — returns False on any load failure.
    """
    try:
        session = _load_session_raw(session_id)
    except Exception:
        return False

    if session["status"] != expected:
        return False

    if new not in VALID_TRANSITIONS.get(expected, set()):
        raise ValueError(
            f"Invalid transition: {expected} → {new}"
        )

    session["status"] = new
    _write_session(session)
    return True
```

### Verification

```bash
python3.11 -c "
import backend.module13 as m
print(m.VALID_STATUSES)
print(m.VALID_TRANSITIONS)
print(hasattr(m, 'list_sessions'))
print(hasattr(m, 'compare_and_set_status'))
"
```

Expected: `DISCARDED` in both constants, both functions present.

---

## Task 1 — Module 2 Extension: THEME_POOL

Make surgical additions to `backend/module2.py`. Do not restructure any existing logic.

### Change 1 — Add `THEME_POOL` constant

Add after existing constants:

```python
THEME_POOL = [
    "Isolation in motion",
    "Controlled emotional pressure",
    "Urban nighttime drift",
    "Memory fragments resurfacing",
    "Internal conflict loop",
    "Digital overstimulation fatigue",
    "Calm before structural collapse",
    "Mechanical emotional detachment",
    "Slow emotional decay",
    "High-speed mental racing",
    "Ambient existential stillness",
    "Submerged perception",
    "Fragmented identity loop",
    "Night drive through industrial space",
    "Emotional suppression effort",
    "System overload warning state",
    "Nostalgia without origin",
    "Gradual synchronization failure",
    "Expanding spatial awareness",
    "Isolated rhythmic persistence",
    "Controlled chaos emergence",
    "Emotional echo chamber",
    "Empty structure exploration",
    "Time distortion perception",
    "Quiet tension accumulation",
    "Synthetic emotional approximation",
    "Breaking point proximity",
    "Dissociation from environment",
    "Repetitive motion trance",
    "Post-event emotional residue"
]
```

### Change 2 — Seed theme selection in `run()`

**Scheduled mode:** Select a seed theme with `random.choice(THEME_POOL)`. Store it in the session plan as `"seed_theme"`. Set `notes` to the selected seed theme text.

**Manual mode:** Use `manual_params.get("seed_theme") or random.choice(THEME_POOL)` as the seed theme. Manual mode activates whenever any manual parameter is present — missing fields must fill from defaults, constraint_profile, or random fallback. Manual mode must never require all fields to be present.

In both modes: `"seed_theme"` is stored in the session's `song_plan` dict and persisted via Module 13.

### Verification

```bash
python3.11 -c "import backend.module2 as m; print(len(m.THEME_POOL), 'themes loaded')"
```

Expected: `30 themes loaded`

---

## Task 2 — Session Manager

Create `/backend/session_manager.py`.

```python
from pathlib import Path
from datetime import datetime, timezone
import backend.module13 as module13

ROOT            = Path(__file__).resolve().parents[1]
SESSIONS_DIR    = ROOT / "data" / "sessions"

DISCARD_AFTER_HOURS = 24
REUSE_WINDOW_HOURS  = 24
```

### `_session_age_hours(session: dict) -> float`

- Parses `session["created_at"]` with `datetime.fromisoformat()`
- Attaches UTC timezone if absent: `.replace(tzinfo=timezone.utc)`
- Returns delta from `datetime.now(timezone.utc)` in hours as float

### `_has_verified_assets(session: dict) -> bool`

- Returns `True` if `session["assets"]` contains at least one item with `"status": "verified"`
- Returns `False` if list is empty or no item is verified

### `_is_discard_eligible(session: dict) -> bool`

Returns `True` only if ALL of:
1. `session["status"] == "INITIALIZED"`
2. `_session_age_hours(session) > DISCARD_AFTER_HOURS`
3. `_has_verified_assets(session) == False`

Returns `False` otherwise. Never raises.

### `run_auto_discard() -> dict`

- Calls `module13.list_sessions()`
- For each session:
  - Calls `_is_discard_eligible(session)`
  - If eligible:
    - Rechecks status before writing: reload via `module13.load_session()` and verify `status == "INITIALIZED"`. If status has changed, skip and continue.
    - Calls `module13.update_status(session["session_id"], "DISCARDED")`
    - Calls `module13.write_checkpoint(session["session_id"], module="session_manager", data={"event": "auto_discard", "age_hours": round(_session_age_hours(session), 2), "reason": "no stems uploaded within 24h"})`
    - Prints `[OK] Session discarded — <session_id> (age: <hours>h)`
  - Wraps each session in try/except — logs error and continues, never raises
- Prints `[OK] Auto-discard complete — checked: <n>, discarded: <n>`
- Returns:

```python
{
    "checked":       int,
    "discarded":     int,
    "discarded_ids": [str, ...]
}
```

### `get_active_session() -> dict | None`

**Deduplication guard for manual triggers only. Never call this during scheduled pipeline runs.**

- Calls `module13.list_sessions()`
- Filters: `status == "INITIALIZED"` and `_session_age_hours(session) < REUSE_WINDOW_HOURS`
- Returns the most recently created session (sort by `created_at` descending), or `None` if none found
- Never raises

---

## Task 3 — Gemini Renderer Extension: Two-Step Concept Flow

Extend `backend/gemini_renderer.py` with the following additions. Do not restructure any existing logic.

### Architecture

The Gemini renderer now operates in two distinct steps:

- **Step 1 — `generate_song_concept()`**: single short Gemini API call. Receives seed theme + song plan. Returns plain English song concept (3–5 sentences). This is the creative seed for the stems and the human-readable opening of the email.
- **Step 2 — `render_queue()`**: existing function, extended to accept `song_concept` as optional parameter. Passes concept as brief context to each stem render call.

**The Gemini API is a tool inside the system. It does not define BPM, structure, or constraints — those come from Module 2 and Module 3. Step 1 is inspiration only. The system is the source of truth for all technical parameters.**

### Addition 1 — `generate_song_concept(seed_theme: str, song_plan: dict) -> dict`

**Never raises under any circumstance.**

Prompt to send to Gemini API:

```
You are a music concept writer for an instrumental music production system.

Given the following seed theme and technical parameters, write a plain English song concept
of 3 to 5 sentences. Describe what the song sounds like, what it evokes emotionally, and
how it moves. Do not include any technical audio terms, BPM values, or instrument names.
Write in present tense. This will be read by the producer before generating stems.

SEED THEME: {seed_theme}

TECHNICAL PARAMETERS:
- Mood: {song_plan["mood"]}
- Energy: {song_plan["energy"]}
- BPM: {song_plan["tempo_bpm"]}
- Duration: {song_plan["target_duration_seconds"]}s
- Harmonic center: {song_plan.get("harmonic_center", "unspecified")}

Rules:
- Instrumental only. No vocals, lyrics, or sung melody references.
- Do not suggest song structure (verse/chorus). Describe feel and motion only.
- Stay within 3–5 sentences. Do not exceed this.
```

Return value — always returned, never raises:

```python
{
    "seed_theme":        str,
    "expanded_concept":  str,
    "fallback_used":     bool,
    "fallback_reason":   str
}
```

On any failure: set `expanded_concept = seed_theme`, `fallback_used = True`, `fallback_reason = str(e)` (sanitized string only). Print `[WARN] generate_song_concept — Gemini call failed, using seed theme as fallback: <reason>`.

### Addition 2 — Extend `render_queue()` signature

Change signature to:

```python
def render_queue(blueprints: list, song_concept: dict | None = None) -> list:
```

Inside `_build_render_prompt()`, if `song_concept` is provided and `song_concept["fallback_used"]` is `False`, prepend this block before the ROLE field:

```
SONG CONCEPT CONTEXT (do not reproduce verbatim — use as creative direction only):
{song_concept["expanded_concept"]}

---
```

If `song_concept` is `None` or `fallback_used` is `True`, omit the block entirely.

### Patch — No-Vocals Directive in `_build_render_prompt`

Add immediately after "Do not remove any constraints":

```
Do not generate vocals, spoken word, or voice-like synthesis under any circumstance.
This is an instrumental-only system. This rule overrides any creative descriptor.
```

---

## Task 4 — No-Vocals Constraint (Module 3 Patch)

Surgical addition to `backend/module3.py` only. Do not restructure anything else.

### Patch — `_build_negative_constraints`

```python
# After
def _build_negative_constraints(stem_def: dict, active_stems: list) -> list:
    constraints = [
        "no vocals, no spoken word, no voice-like synthesis",   # hard rule — always first
        "no leading or trailing silence",
        "no fade in",
        "no fade out"
    ]
    ...
```

The no-vocals entry must always be first in the list, regardless of stem type or any other condition.

---

## Task 5 — Module 4: Email Notification

Create `/backend/module4.py`.

```python
from pathlib import Path
import os, smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import backend.module13 as module13

ROOT = Path(__file__).resolve().parents[1]

SMTP_HOST         = "smtp.gmail.com"
SMTP_PORT         = 587
EMAIL_ENV_VAR     = "MCS_EMAIL_ADDRESS"
PASSWORD_ENV_VAR  = "MCS_EMAIL_PASSWORD"
PAGES_URL_ENV_VAR = "MCS_PAGES_URL"
```

### `_get_credentials() -> tuple[str, str, str]`

- Reads all three environment variables
- If any are missing or empty: raises `EnvironmentError` listing which variables are missing by name
- Returns `(email_address, password, pages_url)`

### `_get_entropy_dimension(session: dict) -> tuple[str, bool]`

Returns `(dimension_str, missing_flag)`.

```python
def _get_entropy_dimension(session: dict) -> tuple[str, bool]:
    dimension = next(
        (
            c["data"].get("entropy", {}).get("dimension")
            for c in reversed(session.get("checkpoints", []))
            if c.get("module") == "module1"
               and isinstance(c.get("data", {}).get("entropy"), dict)
        ),
        None
    )
    if dimension is None:
        return "unknown", True
    return str(dimension), False
```

### `_build_subject(session: dict) -> str`

Seed theme is trimmed to fit available budget. Mood, BPM, and genre are never cut. Final subject never exceeds 90 chars.

```python
def _build_subject(session: dict) -> str:
    plan  = session.get("song_plan", {})
    mood  = plan.get("mood", "").capitalize()
    bpm   = f"@ {plan.get('tempo_bpm')}BPM" if plan.get("tempo_bpm") else ""
    genre = plan.get("genre", "").strip()

    fixed_parts = [p for p in [mood, bpm, genre] if p]
    fixed_str   = " | ".join(fixed_parts)
    seed_budget = 90 - len(fixed_str) - (3 if fixed_str else 0)
    seed_budget = max(0, seed_budget)

    seed  = plan.get("seed_theme", "")[:seed_budget].strip()
    parts = [p for p in [seed, mood, bpm, genre] if p]
    return " | ".join(parts)
```

### `_build_email_body(session: dict, song_concept: dict, stem_count: int, pages_url: str) -> str`

**Note the changed signature:** `rendered_prompts` is replaced by `stem_count: int`. The email body never contains prompts. It only tells Francisco how many stems are ready and directs him to the frontend.

Sections must appear in this exact order:

```
MCS — New Song Ready
=============================

DATE: <ISO date, UTC>
SESSION ID: <session_id>

─── SONG CONCEPT ────────────────────────────────────────
Seed Theme:  <song_plan.get("seed_theme", "unspecified")>

<song_concept["expanded_concept"]>

<If song_concept["fallback_used"] is True:>
[NOTE: Gemini concept generation failed — seed theme used as fallback]
[REASON: <song_concept["fallback_reason"]>]

─── TECHNICAL PLAN ──────────────────────────────────────
Mood:              <song_plan.get("mood", "unspecified")>
Energy:            <song_plan.get("energy", "unspecified")>
Genre:             <song_plan.get("genre", "unspecified")>
Tempo:             <song_plan.get("tempo_bpm", "unspecified")> BPM
Target Duration:   <song_plan.get("target_duration_seconds", "unspecified")>s
Harmonic Center:   <song_plan.get("harmonic_center", "unspecified")>
Entropy Dimension: <from _get_entropy_dimension()>
<If entropy missing:>
[FLAG: entropy_missing — Module 1 checkpoint did not contain entropy data]

─── STEMS READY ─────────────────────────────────────────
<stem_count> stem prompt(s) are ready and waiting for you.

Open the MCS prompt queue to copy each prompt:
<pages_url>/queue.html

Paste each prompt into Gemini App, download the audio,
and upload stems within 24 hours.

─── REMINDER ────────────────────────────────────────────
All stems are instrumental only. Do not generate any vocals,
spoken word, or voice-like synthesis in any stem.

If no stems are uploaded within 24 hours, this idea will
be automatically discarded and a new one generated tomorrow.

─── END OF REPORT ───────────────────────────────────────
```

All fields use `.get()` with explicit fallback strings. Never assume a field exists. No prompts, no Drive links, no rendered content of any kind.

### `send_report(session_id: str, song_concept: dict, stem_count: int) -> dict`

**Note the changed signature:** `rendered_prompts: list` replaced by `stem_count: int`.

**Never raises.** Returns:

```python
{
    "sent":       bool,
    "recipient":  str | None,
    "stem_count": int,
    "error":      str | None
}
```

Logic:
- Calls `_get_credentials()` — if `EnvironmentError`, catches it, returns fail dict immediately without attempting SMTP
- Loads session via `module13.load_session(session_id)`
- Calls `_get_entropy_dimension(session)` — if `missing_flag` is `True`, adds `"entropy_missing"` flag via checkpoint before building body
- Calls `_build_subject(session)`
- Calls `_build_email_body(session, song_concept, stem_count, pages_url)`
- Builds MIME multipart email, connects via `smtplib.SMTP(SMTP_HOST, SMTP_PORT)`, calls `starttls()`, `login()`, sends, calls `quit()`
- On success: prints `[OK] Email sent — session: <session_id>`, returns success dict
- On any exception: prints `[FAIL] Email send failed — <error>`, returns fail dict

### `run(session_id: str, song_concept: dict, stem_count: int) -> dict`

**Note the changed signature:** `rendered_prompts: list` replaced by `stem_count: int`.

- Calls `send_report()`
- If `result["sent"]` is `True`:
  - Writes checkpoint: `module="module4"`, `data={"event": "email_sent", "stem_count": result["stem_count"], "status": "ok", "fallback_concept_used": song_concept.get("fallback_used", False)}`
  - Prints `[OK] Module 4 complete — session: <session_id>`
  - Returns success envelope
- If `result["sent"]` is `False`:
  - Writes checkpoint: `module="module4"`, `data={"event": "email_failed", "error": result["error"], "status": "fail"}`
  - Prints `[FAIL] Module 4 — email not sent`
  - Returns fail envelope with `errors=[result["error"]]`

```python
{
    "module":     "module4",
    "session_id": session_id,
    "status":     "ok" | "fail",
    "flags":      [],
    "errors":     [str] if failed else [],
    "data": {
        "email_sent":            bool,
        "stem_count":            int,
        "recipient":             str | None,
        "fallback_concept_used": bool
    }
}
```

---

## Task 6 — Daily Pipeline Runner

Create `/backend/run_daily.py`.

```python
from pathlib import Path
import sys, os, fcntl

ROOT = Path(__file__).resolve().parents[1]
sys.path.insert(0, str(ROOT))

LOCK_FILE = ROOT / "temp" / "run.lock"
```

### Manual trigger detection

```python
def _is_manual_trigger() -> bool:
    return any([
        os.environ.get("MCS_MANUAL_TITLE"),
        os.environ.get("MCS_MANUAL_NOTES"),
        os.environ.get("MCS_MANUAL_MOOD"),
        os.environ.get("MCS_MANUAL_GENRE"),
        os.environ.get("MCS_MANUAL_ENERGY"),
    ])
```

### Concurrency lock

```python
def main():
    LOCK_FILE.parent.mkdir(parents=True, exist_ok=True)
    lock_handle = open(LOCK_FILE, "w")
    try:
        fcntl.flock(lock_handle, fcntl.LOCK_EX | fcntl.LOCK_NB)
    except BlockingIOError:
        print("[FAIL] Another pipeline run is already in progress — exiting")
        sys.exit(1)

    try:
        _run_pipeline()
    finally:
        fcntl.flock(lock_handle, fcntl.LOCK_UN)
        lock_handle.close()
```

### `_run_pipeline()`

```python
import backend.module1 as m1
import backend.module2 as m2
import backend.module3 as m3
import backend.module3_5 as m35
import backend.module4 as m4
import backend.module13 as m13
import backend.session_manager as sm
from backend.gemini_renderer import generate_song_concept, render_queue

def _run_pipeline():

    # ── Step 1: Auto-discard stale sessions ──────────────────────────────
    print("--- Step 1: Auto-Discard Check ---")
    try:
        discard_result = sm.run_auto_discard()
        print(f"  Checked: {discard_result['checked']}, Discarded: {discard_result['discarded']}")
    except Exception as e:
        print(f"[WARN] Auto-discard error — {e}")
    print()

    # ── Step 2: Constraint generation (Module 1 — stateless) ─────────────
    print("--- Step 2: Constraint Generation (Module 1) ---")
    try:
        constraint_profile = m1.run()
        print(f"  Entropy: {constraint_profile.get('entropy', 'unknown')}")
        print(f"  Preferred moods: {constraint_profile.get('preferred_moods', [])}")
    except Exception as e:
        print(f"[FAIL] Module 1 — {e}")
        sys.exit(1)
    print()

    # ── Step 3: Session setup ─────────────────────────────────────────────
    print("--- Step 3: Session Setup ---")
    session_id = None

    if _is_manual_trigger():
        print("  Manual trigger detected")
        existing = sm.get_active_session()
        if existing:
            claimed = m13.compare_and_set_status(
                existing["session_id"], expected="INITIALIZED", new="ACTIVE"
            )
            if claimed:
                session_id = existing["session_id"]
                print(f"  Reusing existing session — {session_id}")
            else:
                print("  Existing session was claimed by another process — creating new session")

    if session_id is None:
        try:
            manual_mood     = os.environ.get("MCS_MANUAL_MOOD")
            manual_energy   = os.environ.get("MCS_MANUAL_ENERGY")
            manual_genre    = os.environ.get("MCS_MANUAL_GENRE")
            manual_title    = os.environ.get("MCS_MANUAL_TITLE")
            manual_tempo    = os.environ.get("MCS_MANUAL_TEMPO")
            manual_duration = os.environ.get("MCS_MANUAL_DURATION")
            manual_harmonic = os.environ.get("MCS_MANUAL_HARMONIC")
            manual_notes    = os.environ.get("MCS_MANUAL_NOTES")

            if _is_manual_trigger():
                result_m2 = m2.run(
                    constraint_profile,
                    mode="manual",
                    manual_params={
                        "title":                   manual_title or "",
                        "mood":                    manual_mood or "",
                        "energy":                  manual_energy or "",
                        "genre":                   manual_genre or "",
                        "tempo_bpm":               int(manual_tempo) if manual_tempo else None,
                        "target_duration_seconds": int(manual_duration) if manual_duration else None,
                        "harmonic_center":         manual_harmonic or "",
                        "notes":                   manual_notes or "",
                        "seed_theme":              manual_notes or "",
                    }
                )
            else:
                result_m2 = m2.run(constraint_profile, mode="scheduled")

            if result_m2["status"] == "fail":
                print(f"[FAIL] Module 2 — {result_m2['errors']}")
                sys.exit(1)

            session_id = result_m2["data"]["session_id"]
            blueprint  = result_m2["data"]["blueprint"]
            print(f"  Session: {session_id}")
            print(f"  Seed theme: {blueprint.get('seed_theme', 'unset')}")
            print(f"  Mood: {blueprint.get('mood')} | Energy: {blueprint.get('energy')} | BPM: {blueprint.get('tempo_bpm')}")
        except Exception as e:
            print(f"[FAIL] Module 2 — {e}")
            sys.exit(1)
    print()

    session = m13.load_session(session_id)

    # ── Step 4: Prompt generation (Module 3) ─────────────────────────────
    print("--- Step 4: Prompt Generation (Module 3) ---")
    try:
        result_m3 = m3.run(session_id, constraint_profile)
        if result_m3["status"] == "fail":
            print(f"[FAIL] Module 3 — {result_m3['errors']}")
            sys.exit(1)
        blueprints = result_m3["data"]["blueprints"]
        print(f"  {len(blueprints)} blueprints generated")
    except Exception as e:
        print(f"[FAIL] Module 3 — {e}")
        sys.exit(1)
    print()

    # ── Step 5: Validation gate (Module 3.5) ─────────────────────────────
    print("--- Step 5: Validation Gate (Module 3.5) ---")
    try:
        result_m35 = m35.run(session_id, blueprints)
        if result_m35["status"] == "fail":
            print(f"[FAIL] Module 3.5 — {result_m35['errors']}")
            sys.exit(1)
        print(f"  All {result_m35['data']['total_stems']} blueprints valid")
    except Exception as e:
        print(f"[FAIL] Module 3.5 — {e}")
        sys.exit(1)
    print()

    # ── Step 6: Gemini Step 1 — Song concept ─────────────────────────────
    print("--- Step 6: Song Concept Generation (Gemini Step 1) ---")
    try:
        seed_theme   = session["song_plan"].get("seed_theme") or session["song_plan"].get("notes", "")
        song_concept = generate_song_concept(seed_theme, session["song_plan"])
        if song_concept["fallback_used"]:
            print(f"  [WARN] Fallback triggered — reason: {song_concept['fallback_reason']}")
        else:
            print(f"  Concept generated from seed: {seed_theme}")
        m13.write_checkpoint(session_id, module="gemini_step1", data={
            "seed_theme":       song_concept["seed_theme"],
            "expanded_concept": song_concept["expanded_concept"],
            "fallback_used":    song_concept["fallback_used"],
            "fallback_reason":  song_concept["fallback_reason"]
        })
    except Exception as e:
        print(f"[FAIL] Gemini Step 1 — {e}")
        sys.exit(1)
    print()

    # ── Step 7: Gemini Step 2 — Render stem prompts ───────────────────────
    print("--- Step 7: Stem Prompt Render (Gemini Step 2) ---")
    try:
        rendered_prompts = render_queue(blueprints, song_concept=song_concept)
        print(f"  {len(rendered_prompts)} prompts rendered")
    except Exception as e:
        print(f"[FAIL] Gemini Step 2 — {e}")
        sys.exit(1)
    print()

    # ── Step 8: Email report (Module 4) ──────────────────────────────────
    # Note: rendered_prompts are NOT sent in the email.
    # The email contains only the concept, technical plan, stem count, and pages link.
    # Prompts are available exclusively on the frontend queue page.
    print("--- Step 8: Email Report (Module 4) ---")
    try:
        result_m4 = m4.run(session_id, song_concept, len(rendered_prompts))
        if result_m4["status"] == "fail":
            print(f"[FAIL] Module 4 — {result_m4['errors']}")
            sys.exit(1)
        print(f"  Report sent — stems: {result_m4['data']['stem_count']}")
    except Exception as e:
        print(f"[FAIL] Module 4 — {e}")
        sys.exit(1)
    print()

    print("=" * 60)
    print(f"[OK] Daily run complete — session: {session_id}")
    sys.exit(0)

if __name__ == "__main__":
    main()
```

---

## Task 7 — GitHub Actions Workflow

Create `/.github/workflows/mcs_daily.yml`.

```yaml
name: MCS Daily Generation

concurrency:
  group: mcs-daily
  cancel-in-progress: false

on:
  schedule:
    - cron: "0 12 * * *"
  workflow_dispatch:
    inputs:
      manual_mood:
        description: "Mood override (dark/bright/melancholic/tense/euphoric/neutral)"
        required: false
        default: ""
      manual_energy:
        description: "Energy override (sparse/layered/dense/driving/ambient)"
        required: false
        default: ""
      manual_genre:
        description: "Genre override (trap/ambient/electronic/lo-fi/experimental/cinematic)"
        required: false
        default: ""
      manual_title:
        description: "Song title override"
        required: false
        default: ""
      manual_tempo:
        description: "Tempo override (integer BPM)"
        required: false
        default: ""
      manual_duration:
        description: "Duration override (seconds)"
        required: false
        default: ""
      manual_harmonic:
        description: "Harmonic center override (e.g. C, F#)"
        required: false
        default: ""
      manual_notes:
        description: "Seed theme or concept (used as theme seed if provided)"
        required: false
        default: ""

jobs:
  generate:
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
        run: |
          mkdir -p data/sessions data/audio data/library temp

      - name: Run daily generation pipeline
        env:
          GEMINI_API_KEY:       ${{ secrets.GEMINI_API_KEY }}
          MCS_EMAIL_ADDRESS:    ${{ secrets.MCS_EMAIL_ADDRESS }}
          MCS_EMAIL_PASSWORD:   ${{ secrets.MCS_EMAIL_PASSWORD }}
          MCS_PAGES_URL:        ${{ secrets.MCS_PAGES_URL }}
          MCS_MANUAL_MOOD:      ${{ github.event.inputs.manual_mood }}
          MCS_MANUAL_ENERGY:    ${{ github.event.inputs.manual_energy }}
          MCS_MANUAL_GENRE:     ${{ github.event.inputs.manual_genre }}
          MCS_MANUAL_TITLE:     ${{ github.event.inputs.manual_title }}
          MCS_MANUAL_TEMPO:     ${{ github.event.inputs.manual_tempo }}
          MCS_MANUAL_DURATION:  ${{ github.event.inputs.manual_duration }}
          MCS_MANUAL_HARMONIC:  ${{ github.event.inputs.manual_harmonic }}
          MCS_MANUAL_NOTES:     ${{ github.event.inputs.manual_notes }}
        run: python3.11 backend/run_daily.py

      - name: Commit session files
        run: |
          git config user.name "mcs-bot"
          git config user.email "mcs-bot@users.noreply.github.com"
          git add data/sessions/
          git diff --staged --quiet || (git pull --rebase --autostash && git commit -m "chore: session update [$(date -u +%Y-%m-%dT%H:%M:%SZ)]" && git push)
```

**Secrets required before first run:**

| Secret Name | Value |
|---|---|
| `GEMINI_API_KEY` | Gemini API key |
| `MCS_EMAIL_ADDRESS` | Gmail address (sender = receiver) |
| `MCS_EMAIL_PASSWORD` | Gmail app password |
| `MCS_PAGES_URL` | Full GitHub Pages URL e.g. `https://username.github.io/mcs` |

---

## Task 8 — Verification Script

Create `/backend/verify_phase4.py`:

```python
from pathlib import Path
import sys, os, importlib, inspect

ROOT   = Path(__file__).resolve().parents[1]
failed = False

print("=" * 60)
print("MCS Phase 4 — Verification")
print("=" * 60)
print()

modules_to_check = [
    ("backend.module13",        "m13"),
    ("backend.module2",         "m2"),
    ("backend.module4",         "m4"),
    ("backend.session_manager", "sm"),
]
imported = {}
for mod_path, alias in modules_to_check:
    try:
        imported[alias] = importlib.import_module(mod_path)
        print(f"[OK] {mod_path} — imported")
    except Exception as e:
        print(f"[FAIL] {mod_path} — {e}")
        failed = True

if failed:
    sys.exit(1)

m13 = imported["m13"]
m2  = imported["m2"]
m4  = imported["m4"]
sm  = imported["sm"]

# ── Module 13: DISCARDED status ────────────────────────────────────────────
try:
    assert "DISCARDED" in m13.VALID_STATUSES
    assert "DISCARDED" in m13.VALID_TRANSITIONS["INITIALIZED"]
    assert len(m13.VALID_TRANSITIONS["DISCARDED"]) == 0
    print("[OK] module13 — DISCARDED status and transitions correct")
except AssertionError as e:
    print(f"[FAIL] module13 DISCARDED — {e}")
    failed = True

# ── Module 13: list_sessions and compare_and_set_status present ───────────
try:
    assert hasattr(m13, "list_sessions")
    assert hasattr(m13, "compare_and_set_status")
    sessions = m13.list_sessions()
    assert isinstance(sessions, list)
    print(f"[OK] module13 — list_sessions and compare_and_set_status present")
except Exception as e:
    print(f"[FAIL] module13 functions — {e}")
    failed = True

# ── INITIALIZED → DISCARDED transition ────────────────────────────────────
try:
    ts = m13.create_session({
        "title": "discard_test", "mood": "neutral", "energy": "ambient",
        "tempo_bpm": 100, "target_duration_seconds": 180
    })
    sid = ts["session_id"]
    m13.update_status(sid, "DISCARDED")
    assert m13.load_session(sid)["status"] == "DISCARDED"
    print("[OK] INITIALIZED → DISCARDED transition works")
except Exception as e:
    print(f"[FAIL] INITIALIZED → DISCARDED — {e}")
    failed = True

# ── DISCARDED is terminal ──────────────────────────────────────────────────
try:
    m13.update_status(sid, "ACTIVE")
    print("[FAIL] DISCARDED → ACTIVE should be rejected")
    failed = True
except ValueError:
    print("[OK] DISCARDED is terminal — outgoing transition rejected")

# ── compare_and_set_status — success path ─────────────────────────────────
try:
    cas_sess = m13.create_session({
        "title": "cas_test", "mood": "dark", "energy": "driving",
        "tempo_bpm": 128, "target_duration_seconds": 180
    })
    cas_id = cas_sess["session_id"]
    result_cas = m13.compare_and_set_status(cas_id, expected="INITIALIZED", new="ACTIVE")
    assert result_cas is True
    assert m13.load_session(cas_id)["status"] == "ACTIVE"
    print("[OK] compare_and_set_status — INITIALIZED → ACTIVE succeeded")
except Exception as e:
    print(f"[FAIL] compare_and_set_status success path — {e}")
    failed = True

# ── compare_and_set_status — wrong expected ───────────────────────────────
try:
    result_cas2 = m13.compare_and_set_status(cas_id, expected="INITIALIZED", new="ACTIVE")
    assert result_cas2 is False
    print("[OK] compare_and_set_status — returns False when status already changed")
except Exception as e:
    print(f"[FAIL] compare_and_set_status concurrent claim — {e}")
    failed = True

# ── Module 2: THEME_POOL ───────────────────────────────────────────────────
try:
    assert hasattr(m2, "THEME_POOL")
    assert len(m2.THEME_POOL) == 30
    print("[OK] module2.THEME_POOL — 30 themes loaded")
except AssertionError as e:
    print(f"[FAIL] module2.THEME_POOL — {e}")
    failed = True

# ── Module 4: signature uses stem_count not rendered_prompts ─────────────
try:
    sig = inspect.signature(m4.run)
    assert "stem_count" in sig.parameters
    assert "rendered_prompts" not in sig.parameters
    print("[OK] module4.run — uses stem_count parameter, not rendered_prompts")
except AssertionError as e:
    print(f"[FAIL] module4.run signature — {e}")
    failed = True

# ── Module 4: send_report signature uses stem_count ──────────────────────
try:
    sig_sr = inspect.signature(m4.send_report)
    assert "stem_count" in sig_sr.parameters
    assert "rendered_prompts" not in sig_sr.parameters
    print("[OK] module4.send_report — uses stem_count parameter")
except AssertionError as e:
    print(f"[FAIL] module4.send_report signature — {e}")
    failed = True

# ── Module 4: email body contains no prompts ─────────────────────────────
try:
    test_sess = m13.create_session({
        "title": "email_body_test", "mood": "dark", "energy": "driving",
        "tempo_bpm": 128, "target_duration_seconds": 180
    })
    loaded = m13.load_session(test_sess["session_id"])
    loaded["song_plan"]["seed_theme"] = "Urban nighttime drift"
    fake_concept = {
        "seed_theme": "Urban nighttime drift",
        "expanded_concept": "A lone drive through a rain-soaked city at 2AM.",
        "fallback_used": False,
        "fallback_reason": ""
    }
    body = m4._build_email_body(loaded, fake_concept, 5, "https://example.github.io/mcs")

    # Must contain
    for check, label in [
        ("MCS",                   "MCS header"),
        ("Urban nighttime drift", "seed theme"),
        ("A lone drive",          "expanded concept"),
        ("TECHNICAL PLAN",        "technical plan section"),
        ("5 stem",                "stem count"),
        ("queue.html",            "queue page link"),
        ("24 hours",              "24h warning"),
        ("https://example",       "pages URL"),
        ("REMINDER",              "no vocals reminder"),
    ]:
        assert check in body, f"Missing: {label}"

    # Must NOT contain
    for forbidden, label in [
        ("STEM 1",           "individual stem header"),
        ("STEM 2",           "individual stem header"),
        ("rendered_prompt",  "rendered prompt content"),
        ("Priority",         "priority label from prompt"),
    ]:
        assert forbidden not in body, f"Forbidden content found: {label}"

    print("[OK] module4._build_email_body — correct content, no prompts in body")
except Exception as e:
    print(f"[FAIL] module4._build_email_body — {e}")
    failed = True

# ── Module 4: missing credentials raise EnvironmentError ──────────────────
try:
    orig = {k: os.environ.pop(k, None) for k in ["MCS_EMAIL_ADDRESS","MCS_EMAIL_PASSWORD","MCS_PAGES_URL"]}
    try:
        m4._get_credentials()
        print("[FAIL] module4 — missing credentials not caught")
        failed = True
    except EnvironmentError:
        print("[OK] module4 — missing credentials raise EnvironmentError")
    finally:
        for k, v in orig.items():
            if v: os.environ[k] = v
except Exception as e:
    print(f"[FAIL] module4 credential check — {e}")
    failed = True

# ── Module 4: send_report never raises ────────────────────────────────────
try:
    fake_concept_fb = {"seed_theme": "test", "expanded_concept": "test",
                       "fallback_used": True, "fallback_reason": "test error"}
    result_send = m4.send_report("nonexistent_id", fake_concept_fb, 0)
    assert isinstance(result_send, dict) and "sent" in result_send
    print("[OK] module4.send_report — never raises, returns dict on failure")
except Exception as e:
    print(f"[FAIL] module4.send_report raised unexpectedly — {e}")
    failed = True

# ── Module 4: _build_subject fixed parts never truncated ─────────────────
try:
    long_session = {"song_plan": {
        "seed_theme": "A" * 80, "mood": "dark", "tempo_bpm": 128, "genre": "trap"
    }}
    subj = m4._build_subject(long_session)
    assert len(subj) <= 90
    assert "Dark" in subj
    assert "128BPM" in subj or "@ 128BPM" in subj
    assert "trap" in subj
    print(f"[OK] module4._build_subject — length {len(subj)} ≤ 90, fixed parts preserved")
except Exception as e:
    print(f"[FAIL] module4._build_subject — {e}")
    failed = True

# ── session_manager: fresh session not eligible ────────────────────────────
try:
    fresh = m13.create_session({
        "title": "fresh", "mood": "dark", "energy": "driving",
        "tempo_bpm": 128, "target_duration_seconds": 180
    })
    assert not sm._is_discard_eligible(m13.load_session(fresh["session_id"]))
    print("[OK] session_manager — fresh session not discard-eligible")
except Exception as e:
    print(f"[FAIL] session_manager fresh eligibility — {e}")
    failed = True

# ── run_daily.py: stem_count passed not rendered_prompts ─────────────────
try:
    import backend.run_daily as rd
    src = inspect.getsource(rd._run_pipeline)
    assert "len(rendered_prompts)" in src
    assert "stem_count" not in src or "len(rendered_prompts)" in src
    print("[OK] run_daily._run_pipeline — passes len(rendered_prompts) as stem_count to Module 4")
except Exception as e:
    print(f"[FAIL] run_daily stem_count — {e}")
    failed = True

# ── Workflow file ──────────────────────────────────────────────────────────
try:
    wf = ROOT / ".github" / "workflows" / "mcs_daily.yml"
    assert wf.exists()
    content = wf.read_text()
    assert "0 12 * * *"        in content
    assert "workflow_dispatch"  in content
    assert "concurrency"        in content
    assert "mcs-daily"          in content
    assert "git pull --rebase"  in content
    assert "--autostash"        in content
    print("[OK] mcs_daily.yml — cron, concurrency, rebase+autostash all present")
except AssertionError as e:
    print(f"[FAIL] mcs_daily.yml — {e}")
    failed = True

print()
if failed:
    print("[FAIL] Phase 4 verification failed — fix all failures before proceeding")
    sys.exit(1)
else:
    print("[OK] Phase 4 verification complete — all checks passed")
    sys.exit(0)
```

Run it:

```bash
python3.11 backend/verify_phase4.py
```

---

## Completion Gate

Phase 4 is complete when ALL of the following are true:

1. `verify_phase4.py` exits code 0 — all `[OK]`
2. `DISCARDED` in Module 13 with correct transitions
3. `list_sessions()` and `compare_and_set_status()` present and functional
4. `module4.run()` and `module4.send_report()` use `stem_count: int` not `rendered_prompts: list`
5. Email body contains song concept, technical plan, stem count, pages link, and 24 hour warning
6. Email body contains no rendered stem prompts, no Drive links, no individual stem headers
7. Queue page link (`queue.html`) present in email body
8. `send_report()` never raises
9. Missing credentials detected before any SMTP attempt
10. Subject line: seed theme trimmed to fit, mood/BPM/genre never truncated, final subject ≤ 90 chars
11. `THEME_POOL` in Module 2 contains exactly 30 themes
12. `fcntl.flock` lock in `run_daily.py` prevents overlapping runs
13. `mcs_daily.yml` has `concurrency` block and `git pull --rebase --autostash`
14. No new dependencies added beyond `requirements.txt` and Python stdlib
15. All session mutations through Module 13 public functions

---

## What You Must Not Do

- Do not include any rendered stem prompts in the email body under any circumstance
- Do not include Drive links in the email body
- Do not include individual stem headers or prompt content in the email body
- Do not hardcode email addresses, passwords, or URLs
- Do not use any third-party email library — Python stdlib `smtplib` only
- Do not auto-discard sessions in `ACTIVE`, `BLOCKED`, `COMPLETE`, or `DISCARDED` status
- Do not add dependencies beyond `requirements.txt`
- Do not continue past any failed task — stop and print diagnostic
- Do not mutate session dicts directly — all mutations through Module 13 public functions
- Do not hardcode absolute paths
- Do not let `send_report()` raise
- Do not commit stems or audio files — commit session JSON files only
- Do not push without `git pull --rebase --autostash` first
- Do not use `update_status()` for manual trigger session reuse — use `compare_and_set_status()` only
- Do not store raw exception objects — sanitized strings only
- Do not omit the `concurrency` block from the workflow