# MCS Phase 7 — State 2 Refinement Pipeline Agent Prompt (v3)

## Context

You are building the State 2 refinement pipeline for MCS (Music Creator Studio). This is Phase 7. Phases 0–6 are complete and verified.

Phase 7 builds four modules and one integration test:
- **Module 10** — Refinement Planner (daily layer suggestions for active projects)
- **Module 11** — Refinement Prompt Generator (same 8-field schema as Module 3, refinement-tagged)
- **Module 12 backend** — Refinement stem verification wiring (frontend already built in Phase 6)
- **Module 14** — Final Mix and Export (pydub assembly, scipy EQ, Drive upload, GitHub Releases publish, song library write)
- **`backend/integration_test_phase7.py`** — Full State 2 cycle integration test

**No new dependencies beyond `requirements.txt`. No frontend changes. No changes to any Phase 0–6 module beyond surgical additions listed explicitly.**

**Stop on first failure. Print `[OK]` / `[FAIL]` diagnostics. Do not continue to the next task.**

---

## What Phase 7 Does (Read Before Writing Any Code)

State 2 is active when at least one session has `status == "ACTIVE"`. The daily pipeline checks for active sessions first. If any exist, only the State 2 branch runs — the State 1 generation pipeline (Modules 1–3) does not run. If no active sessions exist, the State 1 pipeline runs as normal.

Each day in State 2:
1. Module 10 reads all `ACTIVE` sessions, analyzes current stem coverage, and sends one refinement suggestion per active project via Module 4's email infrastructure
2. Francisco pastes refinement stem prompts into Gemini App, downloads new stems, drops them into the intake folder
3. Module 5 (already built) handles intake — refinement stems get `track_type: "refinement"` set before Module 13 asset registration
4. Module 12 backend confirms Francisco's toggle decisions: approved refinement stems are uploaded to Drive via Module 9, rejected ones are discarded and checkpointed
5. When Francisco is satisfied, he triggers Module 14 manually — it assembles all approved stems, applies scipy EQ, exports the final mix, uploads to Drive as master, publishes to GitHub Releases, writes the completed song to `song_library.json`, and marks the session `COMPLETE`

---

## Architectural Rules (Read Before Writing Any Code)

1. **All session mutations through Module 13 public functions.** Never write to session dicts directly. The new `update_asset_field()` function is the only permitted way to modify asset records.
2. **Every module writes a checkpoint before passing control.** No module exits without checkpointing. This includes validation results — a validation run not in the checkpoint log did not happen.
3. **No new dependencies.** Use only `requirements.txt`: librosa, pydub, numpy, scipy, google-api-python-client, google-auth, python-dotenv. All already installed.
4. **Standard failure envelope on all `run()` functions.** Same shape as all prior phases: `{module, session_id, status, flags, errors, data}`.
5. **Module 11 is deterministic and algorithmic only.** No Gemini API calls inside Module 11. `gemini_renderer.py` handles rendering. Module 11 produces `prompt_blueprint` dicts only.
6. **Module 12 backend never auto-approves.** Francisco's explicit decision is the authority. Module 12 only executes what the toggle payload specifies.
7. **Module 14 never raises.** Returns structured envelope on all code paths. Partial failures (Drive, Releases) are non-blocking — session is marked `COMPLETE` regardless of delivery channel failures.
8. **scipy EQ is applied per stem category, not globally.** Drums and bass: low-pass guard. Melody and texture: high-pass guard. Chords and harmonic pads: no EQ.
9. **GitHub Releases publish uses the `gh` CLI.** Pre-installed on `ubuntu-latest` runners. Never use the GitHub REST API directly.
10. **`song_library.json` append is atomic.** Read full list → append → write to `.tmp` → rename. Never write partial lists.
11. **Refinement stems carry `track_type: "refinement"` in asset records. Core stems carry `track_type: "core"`.** Module 5 patch sets this via `module13.update_asset_field()`.
12. **Module 14 reads `local_path` from asset records only.** Never reconstruct file paths. Module 6 writes `local_path` into the asset record via `module13.update_asset_field()` after extension.
13. **State 2 gate is enforced in `run_daily.py`.** If any ACTIVE session exists: run Module 10 only, skip State 1 pipeline entirely. If no ACTIVE sessions: run State 1 pipeline as normal.
14. **All librosa calls use `SAMPLE_RATE` and `HOP_LENGTH` from `audio_analysis.py`.**
15. **All thresholds from `backend/config.py`.** No module defines threshold constants locally.
16. **Path rule applies everywhere.** All scripts derive root from `Path(__file__).resolve().parents[N]`.

---

## Path Rule (Mandatory)

```python
ROOT         = Path(__file__).resolve().parents[1]
SESSIONS_DIR = ROOT / "data" / "sessions"
LIBRARY_FILE = ROOT / "data" / "library" / "song_library.json"
TEMP_DIR     = ROOT / "temp"
```

---

## Standard Failure Envelope

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

## New Config Constants

Add to `backend/config.py` **before writing any other Phase 7 file**:

```python
# ── Final mix / EQ (Module 14) ─────────────────────────────────────────────
LOWPASS_CUTOFF_HZ   = 8000
HIGHPASS_CUTOFF_HZ  = 200
EQ_FILTER_ORDER     = 4
MIX_CROSSFADE_MS    = 100
STEM_VOLUME_DB = {
    "drums":   0.0,
    "bass":    -1.5,
    "chords":  -2.0,
    "melody":  -1.0,
    "texture": -3.0,
}
```

Verify:

```bash
python3.11 -c "from backend import config; print(config.LOWPASS_CUTOFF_HZ)"
```

Expected: `8000`

---

## Preconditions

Confirm Phases 0–6 are complete:

1. Folder structure: `/backend`, `/data`, `/data/sessions`, `/data/audio`, `/data/library`, `/temp`, `/docs`
2. `requirements.txt` present, all packages installed
3. `backend/module13.py` with all public functions including `list_sessions`, `compare_and_set_status`, `verify_asset_hashes`
4. `backend/module9.py` exists and imports cleanly
5. `backend/drive_uploader.py` exists and imports cleanly
6. `backend/gemini_renderer.py` with `render_queue(blueprints, song_concept=None)` and `generate_song_concept()`
7. `backend/module4.py` with `send_report(session_id, song_concept, rendered_prompts)` and `run()`
8. `backend/module3.py` with `STEM_CATALOG` constant accessible
9. `backend/module3_5.py` with `validate_queue(blueprints)` function
10. `backend/module6.py` exists and imports cleanly
11. `backend/config.py` exists
12. Test audio file at: `data/audio/heavy_gauge.mp3`
13. `gh` CLI available: confirm with `gh --version`

If any are missing: print `[FAIL] Precondition failed — <what is missing>` and stop.

---

## Task 0 — Config Extension

Add the constants above to `backend/config.py`. Verify with the one-liner. Do not proceed until verification passes.

---

## Task 1 — Module 13 Extension: `update_asset_field` and `update_module6_asset`

Make two surgical additions to `backend/module13.py`. Do not restructure any existing logic.

### Addition 1 — `update_asset_field()`

```python
def update_asset_field(session_id: str, asset_id: str, field: str, value) -> bool:
    """
    Updates a single field on an asset record identified by asset_id.
    Returns True if the asset was found and updated.
    Returns False if no asset with that ID exists in the session.
    Never raises — prints [FAIL] and returns False on any error.

    This is the only permitted way to mutate asset records outside of
    register_asset(). Never call _save_session() directly for this purpose.
    """
    try:
        session = load_session(session_id)
        for asset in session.get("assets", []):
            if asset.get("id") == asset_id:
                asset[field] = value
                _save_session(session)
                print(f"[OK] Asset field updated — asset: {asset_id}, field: {field}")
                return True
        print(f"[FAIL] Asset not found — {asset_id}")
        return False
    except Exception as e:
        print(f"[FAIL] update_asset_field — {e}")
        return False
```

### Verification

```bash
python3.11 -c "
import backend.module13 as m
print(hasattr(m, 'update_asset_field'))
"
```

Expected: `True`

---

## Task 2 — Module 5 Patch: `track_type` on Asset Registration

Surgical addition to `backend/module5.py`. Do not restructure any existing logic.

### Change 1 — Add `track_type` parameter

```python
# Before
def run(session_id: str, filepath: str, stem_id: str) -> dict:

# After
def run(session_id: str, filepath: str, stem_id: str, track_type: str = "core") -> dict:
```

### Change 2 — Write `track_type` into asset record after registration

After `register_asset()` returns the `asset_id`, add:

```python
module13.update_asset_field(session_id, asset_id, "track_type", track_type)
module13.write_checkpoint(session_id, "module5_track_type", {
    "event":      "track_type_assigned",
    "stem_id":    stem_id,
    "asset_id":   asset_id,
    "track_type": track_type
})
```

All existing callers that do not pass `track_type` default to `"core"` — no other module changes required.

### Verification

```bash
python3.11 -c "
import backend.module5 as m5
import inspect
sig = inspect.signature(m5.run)
print('track_type' in sig.parameters)
print(sig.parameters['track_type'].default)
"
```

Expected: `True`, `core`

---

## Task 3 — Module 6 Patch: `local_path` in Asset Record

Surgical addition to `backend/module6.py`. Do not restructure any existing logic.

Module 6's `run()` function calls `extend_stem()` and writes a checkpoint. After the checkpoint, add one call to store the output path in the asset record so Module 14 can read it directly.

### Where to insert

Find the existing checkpoint write in `module6.run()`. It looks like:

```python
module13.write_checkpoint(session_id, "module6", {
    "event":   "stem_extended",
    "stem_id": stem_id,
    "output":  output_path,
    ...
})
```

After that checkpoint, add:

```python
# Store local_path in asset record so Module 14 can load the file directly.
# Match on stem_id only — filename-based fallback is intentionally excluded
# to prevent silently writing local_path to the wrong asset record.
session = module13.load_session(session_id)
asset_to_update = next(
    (a for a in reversed(session.get("assets", [])) if a.get("stem_id") == stem_id),
    None
)
if asset_to_update:
    module13.update_asset_field(session_id, asset_to_update["id"], "local_path", output_path)
else:
    print(f"[WARN] Module 6 — could not find asset record for stem {stem_id} to store local_path")
```

### Verification

```bash
python3.11 -c "
import backend.module6 as m6
import inspect
src = inspect.getsource(m6.run)
print('local_path' in src)
print('update_asset_field' in src)
"
```

Expected: `True`, `True`

---

## Task 4 — Module 10: Refinement Planner

Create `backend/module10.py`.

```python
from pathlib import Path
import random
import backend.module13 as module13
import backend.module4  as module4
from backend import config

ROOT         = Path(__file__).resolve().parents[1]
SESSIONS_DIR = ROOT / "data" / "sessions"

LAYER_SUGGESTIONS = {
    "drums":   ["hi-hat variation layer", "percussion ghost notes", "drum fill texture"],
    "bass":    ["sub-bass weight layer", "bass counter-melody", "bass harmonic layer"],
    "chords":  ["chord voicing variation", "harmonic pad layer", "chord tension layer"],
    "melody":  ["melody counter-line", "melody harmonic doubling", "melodic response phrase"],
    "texture": ["atmospheric noise layer", "spatial reverb texture", "rhythmic texture pulse"],
}

DEFAULT_SUGGESTION = "experimental layer — choose a dimension not yet covered"
```

### `analyze_active_session(session: dict) -> dict`

- Reads `session["assets"]` for all assets with `status == "verified"`
- Reads `session["song_plan"]` for `mood`, `energy`, `genre`, `seed_theme`
- Identifies which `LAYER_SUGGESTIONS` categories are already covered by verified stem IDs
- Identifies which categories are absent
- Selects one suggestion: if absent categories exist, picks a random absent one and its suggestion list; otherwise picks `DEFAULT_SUGGESTION`
- Returns:

```python
{
    "session_id":         str,
    "verified_stems":     [str, ...],
    "absent_categories":  [str, ...],
    "suggested_layer":    str,
    "suggested_category": str | None
}
```

### `build_suggestion_email_parts(session: dict, analysis: dict) -> tuple[dict, list]`

Builds the `song_concept` dict and `rendered_prompts` list that Module 4's `send_report()` accepts.

`song_concept`:
```python
{
    "seed_theme":       session["song_plan"].get("seed_theme", ""),
    "expanded_concept": (
        f"Refinement suggestion for active project: {analysis['suggested_layer']}. "
        f"Current verified stems: {', '.join(analysis['verified_stems']) or 'none'}. "
        f"Absent categories: {', '.join(analysis['absent_categories']) or 'none'}."
    ),
    "fallback_used":    False,
    "fallback_reason":  ""
}
```

`rendered_prompts` — one entry:
```python
[
    {
        "stem_id":         analysis["suggested_category"] or "experimental",
        "blueprint":       {"execution_priority": 3, "role": "refinement"},
        "rendered_prompt": (
            f"REFINEMENT LAYER SUGGESTION\n"
            f"Session: {session['session_id']}\n"
            f"Theme: {session['song_plan'].get('seed_theme', '')}\n"
            f"Mood: {session['song_plan'].get('mood', '')} | "
            f"Energy: {session['song_plan'].get('energy', '')} | "
            f"BPM: {session['song_plan'].get('tempo_bpm', '')}\n\n"
            f"Suggested layer: {analysis['suggested_layer']}\n\n"
            f"Generate a stem that complements the existing arrangement without duplicating "
            f"the frequency space or role of: {', '.join(analysis['verified_stems']) or 'none'}.\n\n"
            f"IMPORTANT: Instrumental only. No vocals, no spoken word, no voice-like synthesis."
        )
    }
]
```

### `run(active_sessions: list) -> dict`

- If `active_sessions` is empty: prints `[OK] Module 10 — no active sessions, skipping`, returns ok envelope with `data: {"active_count": 0, "suggestions_sent": 0}`
- For each active session (wrap each in try/except — log error and continue, never raise):
  - Calls `analyze_active_session(session)`
  - Calls `build_suggestion_email_parts(session, analysis)`
  - Calls `module4.send_report(session["session_id"], song_concept, rendered_prompts)`
  - Writes checkpoint: `module="module10"`, `data={"event": "refinement_suggestion_sent", "suggested_layer": analysis["suggested_layer"], "session_id": session["session_id"], "status": "ok"}`
  - Prints `[OK] Module 10 — suggestion sent for session: <session_id> — layer: <suggested_layer>`
- Prints `[OK] Module 10 complete — active: <n>, sent: <suggestions_sent>`
- Returns standard failure envelope:

```python
{
    "module":     "module10",
    "session_id": "multiple" if len(active_sessions) > 1 else (active_sessions[0]["session_id"] if active_sessions else ""),
    "status":     "ok",
    "flags":      [],
    "errors":     [],
    "data":       {"active_count": len(active_sessions), "suggestions_sent": int}
}
```

---

## Task 5 — Module 11: Refinement Prompt Generator

Create `backend/module11.py`.

```python
from pathlib import Path
import backend.module13  as module13
import backend.module3_5 as module3_5
from backend.module3 import STEM_CATALOG  # explicit import — do not reconstruct locally
from backend import config

ROOT = Path(__file__).resolve().parents[1]

SCHEMA_VERSION = "1.0"

VALID_MOODS    = {"dark", "bright", "melancholic", "tense", "euphoric", "neutral"}
VALID_TEXTURES = {"grainy", "smooth", "distorted", "clean", "warm", "cold"}
VALID_ENERGIES = {"sparse", "layered", "dense", "driving", "ambient"}
VALID_DYNAMICS = {"static", "building", "declining", "volatile"}

REFINEMENT_STEM_CATALOG = [
    {"stem_id": "counter_melody",    "role": "refinement", "priority": 3,
     "frequency_range": "mid to high",        "dependencies": ["melody"]},
    {"stem_id": "harmonic_pad",      "role": "refinement", "priority": 3,
     "frequency_range": "low-mid to high",    "dependencies": ["chords"]},
    {"stem_id": "percussion_layer",  "role": "refinement", "priority": 3,
     "frequency_range": "sub-bass to mid",    "dependencies": ["drums"]},
    {"stem_id": "atmosphere",        "role": "refinement", "priority": 3,
     "frequency_range": "full spectrum",      "dependencies": []},
    {"stem_id": "texture_layer",     "role": "refinement", "priority": 3,
     "frequency_range": "full spectrum",      "dependencies": []},
]

MOOD_TEXTURE_MAP = {
    "dark":        {"texture": "grainy",    "dynamic": "volatile"},
    "bright":      {"texture": "clean",     "dynamic": "building"},
    "melancholic": {"texture": "warm",      "dynamic": "declining"},
    "tense":       {"texture": "distorted", "dynamic": "volatile"},
    "euphoric":    {"texture": "smooth",    "dynamic": "building"},
    "neutral":     {"texture": "clean",     "dynamic": "static"},
}

# Build a lookup from Module 3's STEM_CATALOG for frequency/role resolution
_CORE_STEM_LOOKUP = {s["stem_id"]: s for s in STEM_CATALOG}
```

### `_build_active_stems_context(session: dict) -> list`

- Reads `session["assets"]` for all assets with `status == "verified"`
- For each verified asset, resolves `frequency_range` and `role`:
  - If `stem_id` is in `_CORE_STEM_LOOKUP`: use those values
  - Otherwise: `frequency_range = "full spectrum"`, `role = "unknown"`
- Returns list of dicts:

```python
[{"stem_id": str, "frequency_range": str, "role": str}, ...]
```

### `build_refinement_blueprint(session: dict, stem_def: dict, active_stems: list, entropy: dict) -> dict`

Builds one complete `prompt_blueprint` for one refinement stem. Same 8-field P1 schema as Module 3.

Key differences from Module 3:
- `stem_function` has `"[REFINEMENT] "` prefix: `f"[REFINEMENT] {stem_def['role']} layer complementing existing arrangement"`
- `negative_constraints` — first entry is always `"no vocals, no spoken word, no voice-like synthesis"` (hard rule, always first), followed by frequency conflict guards for all active stems
- `song_identity_lock` includes extra field: `"refinement_note": "This is a refinement layer — complement existing arrangement, do not replace it"`
- `prompt_metadata` includes `"refinement": True`

Full blueprint structure (all 8 P1 fields must be present):

```python
{
    "schema_version":      SCHEMA_VERSION,
    "session_id":          session["session_id"],
    "stem_id":             stem_def["stem_id"],
    "execution_priority":  stem_def["priority"],
    "dependency_flag":     stem_def["dependencies"],
    "role":                stem_def["role"],

    # P1 Field 1
    "stem_function": f"[REFINEMENT] {stem_def['role']} layer complementing existing arrangement",

    # P1 Field 2
    "song_identity_lock": {
        "mood":             session["song_plan"].get("mood", ""),
        "energy":           session["song_plan"].get("energy", ""),
        "tempo_bpm":        session["song_plan"].get("tempo_bpm", ""),
        "harmonic_center":  session["song_plan"].get("harmonic_center", ""),
        "refinement_note":  "This is a refinement layer — complement existing arrangement, do not replace it"
    },

    # P1 Field 3
    "structural_constraints": {
        "duration_seconds": session["song_plan"].get("target_duration_seconds", 180),
        "loop_bars":        max(1, (session["song_plan"].get("tempo_bpm", 120) * session["song_plan"].get("target_duration_seconds", 180)) // (4 * 60)),
        "time_signature":   "4/4"
    },

    # P1 Field 4
    "audio_characteristics": {
        "texture": MOOD_TEXTURE_MAP.get(session["song_plan"].get("mood", "neutral"), {}).get("texture", "clean"),
        "energy":  session["song_plan"].get("energy", ""),
        "dynamic": MOOD_TEXTURE_MAP.get(session["song_plan"].get("mood", "neutral"), {}).get("dynamic", "static"),
        "frequency_range": stem_def["frequency_range"]
    },

    # P1 Field 5 — negative constraints: no-vocals always first
    "negative_constraints": (
        ["no vocals, no spoken word, no voice-like synthesis"]
        + [f"do not duplicate or mask {s['stem_id']} frequency range" for s in active_stems]
        + ["no leading or trailing silence", "no fade in", "no fade out"]
    ),

    # P1 Field 6 — global context block
    "global_context": (
        "\n".join([
            f"- {s['stem_id']}: occupies {s['frequency_range']}, role {s['role']}"
            for s in active_stems
        ]) if active_stems else ""
    ),

    # P1 Field 7 — entropy injection
    "entropy_injection": (
        f"dimension={entropy.get('dimension', 'timbre')}, "
        f"intensity={entropy.get('intensity', 0.5)}, "
        f"seed={entropy.get('seed', 0)}"
        + (" [HIGH ENTROPY — intentional variation expected]" if entropy.get("intensity", 0) >= 0.7
           else " [MODERATE ENTROPY — subtle variation]" if entropy.get("intensity", 0) >= 0.4
           else "")
    ),

    # P1 Field 8
    "output_specification": {
        "peak_level_db":  -1,
        "no_silence":     True,
        "loop_alignment": True,
        "no_fade":        True,
        "duration_seconds": session["song_plan"].get("target_duration_seconds", 180)
    },

    # Prompt metadata
    "prompt_metadata": {
        "structure_version":         SCHEMA_VERSION,
        "entropy_config":            entropy,
        "descriptor_set":            {
            "mood":    session["song_plan"].get("mood", ""),
            "texture": MOOD_TEXTURE_MAP.get(session["song_plan"].get("mood", "neutral"), {}).get("texture", "clean"),
            "energy":  session["song_plan"].get("energy", ""),
            "dynamic": MOOD_TEXTURE_MAP.get(session["song_plan"].get("mood", "neutral"), {}).get("dynamic", "static"),
        },
        "constraint_density_score":  3 + len(active_stems) + 4,
        "global_context_stem_count": len(active_stems),
        "refinement":                True
    }
}
```

### `run(session_id: str, constraint_profile: dict, requested_stem_ids: list = None) -> dict`

- Loads session via `module13.load_session(session_id)`
- Extracts entropy: `constraint_profile.get("entropy") or constraint_profile.get("data", {}).get("entropy", {})`
- If entropy is empty dict or None: use default `{"dimension": "timbre", "intensity": 0.5, "seed": 0}` and add `"entropy_missing"` to flags
- Builds `active_stems` via `_build_active_stems_context(session)`
- Determines stem list:
  - If `requested_stem_ids` is None: use all `REFINEMENT_STEM_CATALOG`
  - If provided: filter to matching `stem_id` values — for any ID not found in catalog, print `[WARN] Module 11 — unknown stem_id requested: <id>` and skip
- For each stem: calls `build_refinement_blueprint()`, logs prompt metadata via `module13.log_prompt_metadata()`
- **Runs Module 3.5 validation on all generated blueprints:**

```python
validation_result = module3_5.validate_queue(blueprints)
module13.write_checkpoint(session_id, "module11_validation", {
    "event":  "validation_result",
    "valid":  validation_result["queue_valid"],
    "issues": validation_result.get("blocking_issues", [])
})
if not validation_result["queue_valid"]:
    flags.append("validation_issues_detected")
    for issue in validation_result.get("blocking_issues", []):
        flags.append(f"validation: {issue}")
```

- Writes checkpoint: `module="module11"`, `data={"event": "refinement_prompt_queue_ready", "stem_count": n, "status": "ok"}`
- Prints `[OK] Module 11 complete — <n> refinement blueprints generated`
- Returns standard failure envelope:

```python
{
    "module":     "module11",
    "session_id": session_id,
    "status":     "warn" if flags else "ok",
    "flags":      flags,
    "errors":     [],
    "data":       {"blueprints": [list of blueprint dicts]}
}
```

Note: validation issues surface in `flags` and the checkpoint — they do not change `status` to `"fail"`. Refinement blueprints are advisory prompts, not production pipeline gates.

---

## Task 6 — Module 12 Backend: Refinement Stem Verification

Create `backend/module12.py`.

```python
from pathlib import Path
import backend.module13 as module13
import backend.module9  as module9

ROOT = Path(__file__).resolve().parents[1]
```

### `_validate_inputs(session: dict, approved_stem_ids: list, stem_local_paths: dict) -> list`

Pre-validates inputs before any upload attempt. Returns list of error strings (empty = all valid).

```python
def _validate_inputs(session: dict, approved_stem_ids: list, stem_local_paths: dict) -> list:
    errors = []
    asset_stem_ids = {a.get("stem_id") for a in session.get("assets", [])}
    for stem_id in approved_stem_ids:
        if stem_id not in asset_stem_ids:
            errors.append(f"stem_id '{stem_id}' not found in session assets")
        local_path = stem_local_paths.get(stem_id)
        if not local_path:
            errors.append(f"no local_path provided for stem '{stem_id}'")
        elif not Path(local_path).exists():
            errors.append(f"local file not found for stem '{stem_id}': {local_path}")
    return errors
```

### `run(session_id: str, approved_stem_ids: list, rejected_stem_ids: list, stem_local_paths: dict) -> dict`

1. Load session via `module13.load_session(session_id)`
2. Run `_validate_inputs()` — if errors: write checkpoint with errors, return fail envelope immediately
3. For each `stem_id` in `rejected_stem_ids`:
   - Write checkpoint: `module="module12"`, `data={"event": "refinement_stem_rejected", "stem_id": stem_id, "status": "ok"}`
   - Print `[OK] Module 12 — refinement stem rejected: <stem_id>`
4. If `approved_stem_ids` is non-empty:
   - Call `module9.run(session_id, approved_stem_ids, stem_local_paths)`
   - Record result
5. Write checkpoint: `module="module12"`, `data={"event": "refinement_verification_complete", "approved": len(approved_stem_ids), "rejected": len(rejected_stem_ids), "upload_status": module9_result["status"] if ran else "skipped", "status": "ok"}`
6. Print `[OK] Module 12 complete — approved: <n>, rejected: <n>`
7. Return:

```python
{
    "module":     "module12",
    "session_id": session_id,
    "status":     "ok" | "warn" | "fail",
    "flags":      [],
    "errors":     [],
    "data": {
        "approved_count": int,
        "rejected_count": int,
        "upload_result":  dict | None
    }
}
```

If `module9.run()` returns a fail envelope: Module 12 returns `status: "warn"` with the Module 9 errors in its own `flags`. Never raises.

---

## Task 7 — Module 14: Final Mix and Export

Create `backend/module14.py`.

```python
from pathlib import Path
import os, json, subprocess
import numpy as np
from scipy.signal import butter, sosfilt
from pydub import AudioSegment
import backend.module13     as module13
from backend.drive_uploader import upload_file
from backend               import config

ROOT         = Path(__file__).resolve().parents[1]
SESSIONS_DIR = ROOT / "data" / "sessions"
LIBRARY_FILE = ROOT / "data" / "library" / "song_library.json"
TEMP_DIR     = ROOT / "temp"

EQ_MAP = {
    "drums":            "lowpass",
    "bass":             "lowpass",
    "chords":           None,
    "melody":           "highpass",
    "texture":          "highpass",
    "counter_melody":   "highpass",
    "harmonic_pad":     None,
    "percussion_layer": "lowpass",
    "atmosphere":       "highpass",
    "texture_layer":    "highpass",
}

STEM_PRIORITY = {
    "drums": 1, "bass": 1,
    "chords": 2, "melody": 2,
    "texture": 3, "counter_melody": 3, "harmonic_pad": 3,
    "percussion_layer": 3, "atmosphere": 3, "texture_layer": 3
}
```

### `_apply_eq(audio_segment: AudioSegment, stem_id: str) -> AudioSegment`

```python
def _apply_eq(audio_segment: AudioSegment, stem_id: str) -> AudioSegment:
    eq_type = EQ_MAP.get(stem_id)
    if eq_type is None:
        return audio_segment

    sr       = audio_segment.frame_rate
    channels = audio_segment.channels
    samples  = np.array(audio_segment.get_array_of_samples(), dtype=np.float32)
    max_val  = float(2 ** (8 * audio_segment.sample_width - 1))
    samples  = samples / max_val

    if channels == 2:
        samples = samples.reshape(-1, 2)

    if eq_type == "lowpass":
        sos = butter(config.EQ_FILTER_ORDER,
                     config.LOWPASS_CUTOFF_HZ / (sr / 2),
                     btype="low", output="sos")
    else:
        sos = butter(config.EQ_FILTER_ORDER,
                     config.HIGHPASS_CUTOFF_HZ / (sr / 2),
                     btype="high", output="sos")

    if channels == 2:
        filtered = np.column_stack([
            sosfilt(sos, samples[:, 0]),
            sosfilt(sos, samples[:, 1])
        ]).flatten()
    else:
        filtered = sosfilt(sos, samples)

    filtered = np.clip(filtered, -1.0, 1.0)
    filtered = (filtered * max_val).astype(np.int16)
    return audio_segment._spawn(filtered.tobytes())
```

### `_apply_volume(audio_segment: AudioSegment, stem_id: str) -> AudioSegment`

```python
def _apply_volume(audio_segment: AudioSegment, stem_id: str) -> AudioSegment:
    db = config.STEM_VOLUME_DB.get(stem_id, 0.0)
    return audio_segment + db
```

### `_load_stems(session: dict) -> list[tuple[str, AudioSegment]]`

- Reads `session["assets"]` — only assets with `status == "verified"`
- For each verified asset:
  - Reads `local_path` from the asset record directly — `asset.get("local_path")`
  - If `local_path` is absent or file does not exist: print `[WARN] Stem file not found — <stem_id> (local_path: <value>)` and skip
  - Loads via `AudioSegment.from_file(local_path)`
- Returns list of `(stem_id, AudioSegment)` sorted by priority:
  - Priority from `STEM_PRIORITY` dict — unknown stem IDs default to priority 3
  - Within same priority: sort alphabetically by `stem_id`
- Never raises

### `_assemble_mix(stems: list[tuple[str, AudioSegment]]) -> AudioSegment`

- For each stem: apply EQ via `_apply_eq()`, then volume via `_apply_volume()`
- Start with first stem as base
- Overlay each subsequent stem at position 0 using pydub `overlay()`
- Returns assembled `AudioSegment`

### `_generate_title(session: dict) -> str`

- Reads `session["song_plan"]` for `title`, `mood`, `genre`, `tempo_bpm`
- If `title` is non-empty and does not match auto-generated pattern (starts with `f"{mood}_"` or equals `"auto-generated"`): use title directly
- Otherwise: generate from `f"{mood}_{genre}_{tempo_bpm}bpm"`
- Sanitize: replace spaces with `_`, keep only alphanumeric, `_`, `-`
- Returns sanitized string

### `_publish_to_releases(filepath: str, title: str, session_id: str) -> dict`

```python
def _publish_to_releases(filepath: str, title: str, session_id: str) -> dict:
    tag = f"song-{session_id[:8]}"
    cmd = [
        "gh", "release", "create", tag,
        filepath,
        "--title", title,
        "--notes", f"MCS auto-export — session: {session_id}",
    ]
    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=60)
        if result.returncode == 0:
            url = result.stdout.strip()
            print(f"[OK] Published to GitHub Releases — {url}")
            return {"success": True, "url": url, "error": None}
        else:
            err = result.stderr.strip()
            print(f"[FAIL] GitHub Releases publish failed — {err}")
            return {"success": False, "url": None, "error": err}
    except Exception as e:
        print(f"[FAIL] GitHub Releases publish error — {e}")
        return {"success": False, "url": None, "error": str(e)}
```

### `_append_to_library(session: dict, title: str, drive_result: dict, releases_result: dict, perceptual_score: dict) -> None`

```python
def _append_to_library(session, title, drive_result, releases_result, perceptual_score):
    from datetime import datetime, timezone
    plan  = session.get("song_plan", {})
    entry = {
        "session_id":       session["session_id"],
        "title":            title,
        "mood":             plan.get("mood", ""),
        "genre":            plan.get("genre", ""),
        "tempo_bpm":        plan.get("tempo_bpm", ""),
        "seed_theme":       plan.get("seed_theme", ""),
        "completed_at":     datetime.now(timezone.utc).isoformat(),
        "perceptual_score": perceptual_score,
        "drive_link":       drive_result.get("web_view_link"),
        "releases_link":    releases_result.get("url"),
        "stems_used":       [a["stem_id"] for a in session.get("assets", []) if a.get("status") == "verified"],
        "diversity_vector": session.get("diversity_vector", {}),
        "identity_vector":  session.get("identity_vector", {})
    }

    LIBRARY_FILE.parent.mkdir(parents=True, exist_ok=True)
    existing = []
    if LIBRARY_FILE.exists():
        try:
            existing = json.loads(LIBRARY_FILE.read_text())
        except Exception:
            existing = []

    existing.append(entry)
    tmp = LIBRARY_FILE.with_suffix(".tmp")
    tmp.write_text(json.dumps(existing, indent=2))
    tmp.replace(LIBRARY_FILE)
    print(f"[OK] Song library updated — {len(existing)} total songs")
```

### `run(session_id: str, perceptual_score: dict) -> dict`

1. Load session via `module13.load_session(session_id)`
2. Check `session["status"] == "ACTIVE"` — if not, return fail envelope: `errors: ["Session is not ACTIVE — cannot export"]`
3. Load stems via `_load_stems(session)` — if zero stems loaded, return fail envelope: `errors: ["No verified stems with local_path found — run Module 6 on all stems first"]`
4. Assemble mix via `_assemble_mix(stems)`
5. Generate title via `_generate_title(session)`
6. Export: `TEMP_DIR.mkdir(parents=True, exist_ok=True)`, then `mix.export(str(output_path), format="wav")` where `output_path = TEMP_DIR / f"{title}.wav"`
7. If export raises: return fail envelope with error — do not continue
8. Upload to Drive via `upload_file(str(output_path), filename=f"{title}.wav", mime_type="audio/wav")` — never raises, captures result
9. Publish to GitHub Releases via `_publish_to_releases()` — never raises, captures result
10. Append to library via `_append_to_library()` — wrap in try/except, add to flags on failure
11. Transition to COMPLETE via `module13.update_status(session_id, "COMPLETE")`
12. Write final checkpoint: `module="module14"`, `data={"event": "export_complete", "title": title, "drive_success": drive_result["success"], "releases_success": releases_result["success"], "stems_mixed": len(stems), "status": "ok"}`
13. Print `[OK] Module 14 complete — <title>`

Partial failure handling:
- Drive upload fails: add `"drive_upload_failed"` to flags, continue
- Releases publish fails: add `"releases_publish_failed"` to flags, continue
- Both fail: `status: "warn"`, both flags present, session still marked `COMPLETE`
- Export fails (step 7): `status: "fail"`, return immediately

```python
# Return shape
{
    "module":     "module14",
    "session_id": session_id,
    "status":     "ok" | "warn" | "fail",
    "flags":      [str, ...],
    "errors":     [str, ...],
    "data": {
        "title":            str | None,
        "output_path":      str | None,
        "drive_success":    bool,
        "releases_success": bool,
        "drive_link":       str | None,
        "releases_url":     str | None,
        "stems_mixed":      int
    }
}
```

---

## Task 8 — Module 14 CLI

Add `__main__` block to `backend/module14.py`:

```python
if __name__ == "__main__":
    import argparse, sys

    parser = argparse.ArgumentParser(description="MCS Module 14 — Final Mix and Export")
    parser.add_argument("--session-id",  required=True)
    parser.add_argument("--vibe",        required=True, type=int)
    parser.add_argument("--originality", required=True, type=int)
    parser.add_argument("--coherence",   required=True, type=int)
    parser.add_argument("--flag",        default="safe", choices=["safe", "stretch"])
    args = parser.parse_args()

    for field, val in [("vibe", args.vibe), ("originality", args.originality), ("coherence", args.coherence)]:
        if not (1 <= val <= 10):
            print(f"[FAIL] {field} must be 1–10, got {val}")
            sys.exit(1)

    perceptual_score = {
        "vibe":        args.vibe,
        "originality": args.originality,
        "coherence":   args.coherence,
        "flag":        args.flag
    }

    try:
        result = run(args.session_id, perceptual_score)
        import json
        print(json.dumps(result, indent=2))
        sys.exit(0 if result["status"] in {"ok", "warn"} else 1)
    except Exception as e:
        print(f"[FAIL] Module 14 — {e}")
        sys.exit(1)
```

Usage:

```bash
python3.11 backend/module14.py \
    --session-id <uuid> \
    --vibe 8 \
    --originality 7 \
    --coherence 9 \
    --flag safe
```

---

## Task 9 — `run_daily.py` Extension: State 2 Gate

Surgical replacement of the existing pipeline entry point in `backend/run_daily.py`. The State 2 gate must be the first check inside `_run_pipeline()`, before auto-discard.

### Replace the opening of `_run_pipeline()` with:

```python
def _run_pipeline():
    import backend.module13 as m13

    # ── State gate: check for active State 2 sessions ─────────────────────
    print("--- State Gate: Checking for Active Sessions ---")
    try:
        all_sessions    = m13.list_sessions()
        active_sessions = [s for s in all_sessions if s.get("status") == "ACTIVE"]
    except Exception as e:
        print(f"[FAIL] State gate — could not list sessions: {e}")
        sys.exit(1)

    if active_sessions:
        # ── STATE 2: Refinement branch ────────────────────────────────────
        print(f"  STATE 2 active — {len(active_sessions)} active session(s)")
        print(f"  Skipping State 1 generation pipeline (Modules 1–3)")
        print()

        import backend.module10 as m10
        print("--- State 2: Refinement Suggestions (Module 10) ---")
        try:
            result_m10 = m10.run(active_sessions)
            print(f"  Suggestions sent: {result_m10['data'].get('suggestions_sent', 0)}")
        except Exception as e:
            print(f"[WARN] Module 10 — {e}")
        print()

        # Write session index for frontend
        _write_session_index()
        _mirror_library()
        print("[OK] State 2 daily run complete")
        sys.exit(0)

    # ── STATE 1: Candidate generation pipeline ────────────────────────────
    print("  STATE 1 active — no active sessions found")
    print("  Running candidate generation pipeline")
    print()

    # ... existing State 1 pipeline continues below unchanged (Step 1 through Step 8) ...
```

The remainder of `_run_pipeline()` — Steps 1 through 8 (auto-discard through email) — remains completely unchanged.

### Verification

```bash
python3.11 -c "
import backend.run_daily as rd
import inspect
src = inspect.getsource(rd._run_pipeline)
print('STATE 2' in src)
print('STATE 1' in src)
print('Skipping State 1' in src)
"
```

Expected: `True`, `True`, `True`

---

## Task 10 — Verification Script

Create `backend/verify_phase7.py`:

```python
from pathlib import Path
import sys, importlib, inspect, json

ROOT       = Path(__file__).resolve().parents[1]
AUDIO_FILE = ROOT / "data" / "audio" / "heavy_gauge.mp3"
failed     = False

print("=" * 60)
print("MCS Phase 7 — Verification")
print("=" * 60)
print()

if not AUDIO_FILE.exists():
    print(f"[FAIL] heavy_gauge.mp3 not found at {AUDIO_FILE}")
    sys.exit(1)
else:
    print("[OK] heavy_gauge.mp3 found")

# ── Import checks ──────────────────────────────────────────────────────────
modules_to_check = [
    ("backend.config",   "cfg"),
    ("backend.module5",  "m5"),
    ("backend.module6",  "m6"),
    ("backend.module10", "m10"),
    ("backend.module11", "m11"),
    ("backend.module12", "m12"),
    ("backend.module14", "m14"),
    ("backend.module13", "m13"),
    ("backend.module9",  "m9"),
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

cfg = imported["cfg"]
m5  = imported["m5"]
m6  = imported["m6"]
m10 = imported["m10"]
m11 = imported["m11"]
m12 = imported["m12"]
m14 = imported["m14"]
m13 = imported["m13"]

# ── Config: Phase 7 constants ─────────────────────────────────────────────
try:
    for attr in ["LOWPASS_CUTOFF_HZ","HIGHPASS_CUTOFF_HZ","EQ_FILTER_ORDER",
                 "MIX_CROSSFADE_MS","STEM_VOLUME_DB"]:
        assert hasattr(cfg, attr), f"Missing: {attr}"
    assert isinstance(cfg.STEM_VOLUME_DB, dict)
    print("[OK] config.py — Phase 7 constants present")
except AssertionError as e:
    print(f"[FAIL] config.py — {e}")
    failed = True

# ── Module 13: update_asset_field present ────────────────────────────────
try:
    assert hasattr(m13, "update_asset_field")
    print("[OK] module13.update_asset_field — present")
except AssertionError:
    print("[FAIL] module13.update_asset_field — missing")
    failed = True

# ── update_asset_field: functional test ───────────────────────────────────
try:
    ts = m13.create_session({
        "title": "uaf_test", "mood": "dark", "energy": "driving",
        "tempo_bpm": 128, "target_duration_seconds": 180
    })
    sid_uaf = ts["session_id"]
    aid = m13.register_asset(sid_uaf, "test.wav", str(AUDIO_FILE))
    result = m13.update_asset_field(sid_uaf, aid, "track_type", "refinement")
    assert result is True
    loaded = m13.load_session(sid_uaf)
    asset  = next(a for a in loaded["assets"] if a["id"] == aid)
    assert asset["track_type"] == "refinement"
    print("[OK] module13.update_asset_field — writes field to asset record correctly")
except Exception as e:
    print(f"[FAIL] module13.update_asset_field functional — {e}")
    failed = True

# ── update_asset_field: nonexistent asset returns False ───────────────────
try:
    result = m13.update_asset_field(sid_uaf, "nonexistent-id", "field", "value")
    assert result is False
    print("[OK] module13.update_asset_field — returns False for missing asset")
except Exception as e:
    print(f"[FAIL] module13.update_asset_field missing asset — {e}")
    failed = True

# ── Module 5 patch: track_type parameter ──────────────────────────────────
try:
    sig = inspect.signature(m5.run)
    assert "track_type" in sig.parameters
    assert sig.parameters["track_type"].default == "core"
    print("[OK] module5.run — track_type parameter with default 'core'")
except AssertionError as e:
    print(f"[FAIL] module5 track_type — {e}")
    failed = True

# ── Module 5 patch: track_type written to asset record ────────────────────
try:
    ts2    = m13.create_session({
        "title": "m5_test", "mood": "dark", "energy": "driving",
        "tempo_bpm": 128, "target_duration_seconds": 180
    })
    sid_m5 = ts2["session_id"]
    m13.update_status(sid_m5, "ACTIVE")
    result_m5 = m5.run(sid_m5, str(AUDIO_FILE), "atmosphere", track_type="refinement")
    loaded_m5  = m13.load_session(sid_m5)
    asset_m5   = next((a for a in loaded_m5["assets"] if a.get("stem_id") == "atmosphere"), None)
    assert asset_m5 is not None
    assert asset_m5.get("track_type") == "refinement", f"Got: {asset_m5.get('track_type')}"
    print("[OK] module5 patch — track_type=refinement written to asset record")
except Exception as e:
    print(f"[FAIL] module5 track_type in asset — {e}")
    failed = True

# ── Module 6 patch: local_path written to asset record ────────────────────
try:
    src_m6 = inspect.getsource(m6.run)
    assert "local_path" in src_m6
    assert "update_asset_field" in src_m6
    print("[OK] module6.run — local_path and update_asset_field present in source")
except AssertionError as e:
    print(f"[FAIL] module6 patch — {e}")
    failed = True

# ── Module 10: empty active sessions ──────────────────────────────────────
try:
    result = m10.run([])
    assert result["status"] == "ok"
    assert result["data"]["active_count"] == 0
    assert result["data"]["suggestions_sent"] == 0
    print("[OK] module10.run — empty active sessions returns ok")
except Exception as e:
    print(f"[FAIL] module10.run empty — {e}")
    failed = True

# ── Module 10: analyze_active_session ─────────────────────────────────────
try:
    fake_session = {
        "session_id": "fake123",
        "status":     "ACTIVE",
        "song_plan":  {"mood": "dark", "energy": "driving", "genre": "trap",
                       "tempo_bpm": 128, "seed_theme": "Urban drift"},
        "assets": [
            {"stem_id": "drums", "status": "verified"},
            {"stem_id": "bass",  "status": "verified"},
        ]
    }
    analysis = m10.analyze_active_session(fake_session)
    assert "suggested_layer"    in analysis
    assert "verified_stems"     in analysis
    assert "absent_categories"  in analysis
    assert "drums" in analysis["verified_stems"]
    assert "bass"  in analysis["verified_stems"]
    print(f"[OK] module10.analyze_active_session — suggestion: {analysis['suggested_layer']}")
except Exception as e:
    print(f"[FAIL] module10.analyze_active_session — {e}")
    failed = True

# ── Module 10: no-vocals in rendered_prompt ───────────────────────────────
try:
    sc, rp = m10.build_suggestion_email_parts(fake_session, analysis)
    assert "seed_theme" in sc and "expanded_concept" in sc
    assert isinstance(rp, list) and len(rp) == 1
    prompt_text = rp[0]["rendered_prompt"].upper()
    assert "VOCAL" in prompt_text or "INSTRUMENTAL" in prompt_text
    print("[OK] module10.build_suggestion_email_parts — no-vocals present")
except Exception as e:
    print(f"[FAIL] module10.build_suggestion_email_parts — {e}")
    failed = True

# ── Module 11: run with real session ──────────────────────────────────────
try:
    ts3 = m13.create_session({
        "title": "m11_test", "mood": "dark", "energy": "driving",
        "tempo_bpm": 128, "target_duration_seconds": 180
    })
    sid_m11 = ts3["session_id"]
    m13.update_status(sid_m11, "ACTIVE")
    dummy_cp = {
        "entropy": {"dimension": "rhythm", "intensity": 0.5, "seed": 1234}
    }
    result_m11 = m11.run(sid_m11, dummy_cp)
    assert result_m11["status"] in {"ok", "warn"}
    blueprints = result_m11["data"]["blueprints"]
    assert len(blueprints) > 0
    for bp in blueprints:
        assert "[REFINEMENT]" in bp["stem_function"]
        nc = bp["negative_constraints"]
        assert "vocal" in nc[0].lower(), f"First constraint not no-vocals: {nc[0]}"
        assert bp["prompt_metadata"]["refinement"] is True
        assert bp["output_specification"]["peak_level_db"] == -1
    print(f"[OK] module11.run — {len(blueprints)} blueprints, [REFINEMENT] in function, no-vocals first, peak_level_db=-1")
except Exception as e:
    print(f"[FAIL] module11.run — {e}")
    failed = True

# ── Module 11: validation checkpoint written ──────────────────────────────
try:
    loaded_m11 = m13.load_session(sid_m11)
    val_cps = [cp for cp in loaded_m11["checkpoints"] if cp["module"] == "module11_validation"]
    assert len(val_cps) >= 1
    assert "valid" in val_cps[0]["data"]
    assert "issues" in val_cps[0]["data"]
    print("[OK] module11 — validation checkpoint written with valid and issues fields")
except Exception as e:
    print(f"[FAIL] module11 validation checkpoint — {e}")
    failed = True

# ── Module 11: requested_stem_ids filter ──────────────────────────────────
try:
    result_filtered = m11.run(sid_m11, dummy_cp, requested_stem_ids=["atmosphere"])
    bps = result_filtered["data"]["blueprints"]
    assert all(bp["stem_id"] == "atmosphere" for bp in bps)
    print("[OK] module11 — requested_stem_ids=['atmosphere'] returns only atmosphere")
except Exception as e:
    print(f"[FAIL] module11 requested_stem_ids — {e}")
    failed = True

# ── Module 11: STEM_CATALOG imported from module3 ─────────────────────────
try:
    src_m11 = inspect.getsource(m11)
    assert "from backend.module3 import STEM_CATALOG" in src_m11
    print("[OK] module11 — explicit STEM_CATALOG import from module3")
except AssertionError as e:
    print(f"[FAIL] module11 STEM_CATALOG import — {e}")
    failed = True

# ── Module 12: pre-validation rejects bad inputs ──────────────────────────
try:
    ts4    = m13.create_session({
        "title": "m12_test", "mood": "neutral", "energy": "sparse",
        "tempo_bpm": 90, "target_duration_seconds": 180
    })
    sid_m12 = ts4["session_id"]
    m13.update_status(sid_m12, "ACTIVE")
    result_bad = m12.run(
        session_id=sid_m12,
        approved_stem_ids=["nonexistent_stem"],
        rejected_stem_ids=[],
        stem_local_paths={"nonexistent_stem": "/bad/path.wav"}
    )
    assert result_bad["status"] == "fail"
    print("[OK] module12 — pre-validation rejects nonexistent stem_id")
except Exception as e:
    print(f"[FAIL] module12 pre-validation — {e}")
    failed = True

# ── Module 12: rejection checkpointed ────────────────────────────────────
try:
    result_m12 = m12.run(
        session_id=sid_m12,
        approved_stem_ids=[],
        rejected_stem_ids=["counter_melody"],
        stem_local_paths={}
    )
    assert result_m12["module"] == "module12"
    assert result_m12["data"]["rejected_count"] == 1
    loaded_m12 = m13.load_session(sid_m12)
    rej_cps = [cp for cp in loaded_m12["checkpoints"]
               if cp["data"].get("event") == "refinement_stem_rejected"]
    assert len(rej_cps) >= 1
    print("[OK] module12 — rejection checkpointed correctly")
except Exception as e:
    print(f"[FAIL] module12 rejection checkpoint — {e}")
    failed = True

# ── Module 14: _apply_eq on all categories ────────────────────────────────
try:
    from pydub import AudioSegment as AS
    test_audio = AS.from_file(str(AUDIO_FILE))
    for stem_id in ["drums", "bass", "chords", "melody", "texture"]:
        result_audio = m14._apply_eq(test_audio, stem_id)
        assert isinstance(result_audio, AS)
    print("[OK] module14._apply_eq — all stem categories processed without error")
except Exception as e:
    print(f"[FAIL] module14._apply_eq — {e}")
    failed = True

# ── Module 14: _apply_volume ──────────────────────────────────────────────
try:
    adj = m14._apply_volume(test_audio, "drums")
    assert isinstance(adj, AS)
    print("[OK] module14._apply_volume — runs without error")
except Exception as e:
    print(f"[FAIL] module14._apply_volume — {e}")
    failed = True

# ── Module 14: _generate_title sanitization ───────────────────────────────
try:
    t1 = m14._generate_title({"song_plan": {
        "title": "dark_trap_128bpm", "mood": "dark", "genre": "trap", "tempo_bpm": 128
    }})
    assert all(c.isalnum() or c in "_-" for c in t1) and " " not in t1
    t2 = m14._generate_title({"song_plan": {
        "title": "My Custom Track", "mood": "dark", "genre": "trap", "tempo_bpm": 128
    }})
    assert " " not in t2
    print(f"[OK] module14._generate_title — auto: {t1}, custom: {t2}")
except Exception as e:
    print(f"[FAIL] module14._generate_title — {e}")
    failed = True

# ── Module 14: non-ACTIVE session returns fail ────────────────────────────
try:
    ts5 = m13.create_session({
        "title": "not_active", "mood": "neutral", "energy": "sparse",
        "tempo_bpm": 90, "target_duration_seconds": 180
    })
    result_na = m14.run(ts5["session_id"], {"vibe": 7, "originality": 7, "coherence": 7, "flag": "safe"})
    assert result_na["status"] == "fail"
    assert any("not ACTIVE" in e for e in result_na["errors"])
    print("[OK] module14.run — non-ACTIVE session returns fail envelope")
except Exception as e:
    print(f"[FAIL] module14 non-ACTIVE — {e}")
    failed = True

# ── Module 14: reads local_path from asset record ────────────────────────
try:
    src_m14 = inspect.getsource(m14._load_stems)
    assert "local_path" in src_m14
    assert "reconstruct" not in src_m14.lower()
    print("[OK] module14._load_stems — reads local_path from asset record")
except AssertionError as e:
    print(f"[FAIL] module14._load_stems local_path — {e}")
    failed = True

# ── Module 14: CLI __main__ block ─────────────────────────────────────────
try:
    src_m14_full = inspect.getsource(m14)
    assert "__main__" in src_m14_full
    assert "argparse"    in src_m14_full
    assert "--session-id" in src_m14_full
    print("[OK] module14 — __main__ CLI block present")
except AssertionError as e:
    print(f"[FAIL] module14 __main__ — {e}")
    failed = True

# ── run_daily.py: State 2 gate ────────────────────────────────────────────
try:
    import backend.run_daily as rd
    src_rd = inspect.getsource(rd._run_pipeline)
    assert "STATE 2"           in src_rd
    assert "STATE 1"           in src_rd
    assert "Skipping State 1"  in src_rd
    assert "module10"          in src_rd
    print("[OK] run_daily._run_pipeline — State 2 gate present")
except Exception as e:
    print(f"[FAIL] run_daily State 2 gate — {e}")
    failed = True

# ── Checkpoint audit on module11 session ──────────────────────────────────
try:
    final = m13.load_session(sid_m11)
    cp_modules = {cp["module"] for cp in final["checkpoints"]}
    expected   = {"module13", "module11", "module11_validation"}
    missing_cp = expected - cp_modules
    if missing_cp:
        print(f"[FAIL] Missing checkpoints: {missing_cp}")
        failed = True
    else:
        print(f"[OK] Checkpoint audit — all expected modules present: {cp_modules}")
    reconcile = m13.reconcile_state(sid_m11)
    assert reconcile["valid"], f"Reconciliation failed: {reconcile['issues']}"
    print("[OK] reconcile_state — test session valid after Phase 7 run")
except Exception as e:
    print(f"[FAIL] Checkpoint audit — {e}")
    failed = True

print()
if failed:
    print("[FAIL] Phase 7 verification failed — fix all failures before proceeding")
    sys.exit(1)
else:
    print("[OK] Phase 7 verification complete — all checks passed")
    sys.exit(0)
```

Run it:

```bash
python3.11 backend/verify_phase7.py
```

---

## Task 11 — Integration Test

Create `backend/integration_test_phase7.py`:

```python
"""
MCS Phase 7 — Integration Test: Full State 2 Cycle

ASSUMPTION: Module 13 is correct and verified (Phase 1). Latent Module 13
bugs may cause false positives across multiple stages.

STATE 2 GATE BEHAVIOR: When this test runs, if any ACTIVE sessions exist
in the real data directory, run_daily.py would enter State 2 and skip
State 1. This integration test creates its own session and manages state
directly — it does not invoke run_daily.py to avoid affecting real sessions.

MANUAL HANDOFF POINTS:
  Stage 6 requires MCS_DRIVE_CREDENTIALS_* env vars and gh CLI authenticated.
  If absent, Stage 6 produces a warn envelope — this is expected and acceptable.
"""

from pathlib import Path
import sys, os, json

ROOT       = Path(__file__).resolve().parents[1]
AUDIO_FILE = ROOT / "data" / "audio" / "heavy_gauge.mp3"
failed     = False

print("=" * 60)
print("MCS Phase 7 — Integration Test: Full State 2 Cycle")
print("=" * 60)
print()

if not AUDIO_FILE.exists():
    print("[FAIL] heavy_gauge.mp3 not found")
    sys.exit(1)

import backend.module1  as m1
import backend.module10 as m10
import backend.module11 as m11
import backend.module5  as m5
import backend.module12 as m12
import backend.module14 as m14
import backend.module13 as m13

def check_stage(result, stage_name, allow_warn=False):
    global failed
    if result.get("status") == "fail":
        print(f"[FAIL] {stage_name} — errors: {result['errors']}")
        failed = True
        return False
    if result.get("status") == "warn" and not allow_warn:
        print(f"[WARN] {stage_name} — flags: {result['flags']}")
    return True

# ── Setup ─────────────────────────────────────────────────────────────────
print("--- Setup: Create and activate test session ---")
try:
    sess = m13.create_session({
        "title":                   "state2_integration_test",
        "mood":                    "dark",
        "energy":                  "driving",
        "genre":                   "trap",
        "tempo_bpm":               128,
        "target_duration_seconds": 180,
        "seed_theme":              "Integration test — urban nighttime"
    })
    sid = sess["session_id"]
    m13.update_status(sid, "ACTIVE")
    print(f"  Session: {sid}")
    print()
except Exception as e:
    print(f"[FAIL] Setup — {e}")
    sys.exit(1)

# ── Stage 1: Constraint generation ────────────────────────────────────────
print("--- Stage 1: Constraint Generation (Module 1) ---")
try:
    cp = m1.run()
    print(f"  Entropy: {cp.get('entropy', 'unknown')}")
    print()
except Exception as e:
    print(f"[FAIL] Stage 1 — {e}")
    failed = True
    cp = {"entropy": {"dimension": "timbre", "intensity": 0.5, "seed": 0}}

# ── Stage 2: Refinement planning ─────────────────────────────────────────
print("--- Stage 2: Refinement Planning (Module 10) ---")
try:
    loaded = m13.load_session(sid)
    result_m10 = m10.run([loaded])
    check_stage(result_m10, "Module 10", allow_warn=True)
    print(f"  Suggestions: {result_m10['data'].get('suggestions_sent', 0)}")
    print()
except Exception as e:
    print(f"[WARN] Stage 2 — {e} (non-fatal if email not configured)")
    print()

# ── Stage 3: Refinement prompt generation ────────────────────────────────
print("--- Stage 3: Refinement Prompt Generation (Module 11) ---")
try:
    result_m11 = m11.run(sid, cp)
    check_stage(result_m11, "Module 11", allow_warn=True)
    blueprints = result_m11["data"]["blueprints"]
    print(f"  {len(blueprints)} blueprints — validation checkpoint present:")
    loaded_m11 = m13.load_session(sid)
    val_cp = next((c for c in loaded_m11["checkpoints"] if c["module"] == "module11_validation"), None)
    print(f"  validation checkpoint valid={val_cp['data']['valid'] if val_cp else 'MISSING'}")
    print()
except Exception as e:
    print(f"[FAIL] Stage 3 — {e}")
    failed = True
    blueprints = []

# ── Stage 4: Stem intake (refinement) ────────────────────────────────────
print("--- Stage 4: Stem Intake (Module 5 — track_type=refinement) ---")
try:
    result_m5 = m5.run(sid, str(AUDIO_FILE), "atmosphere", track_type="refinement")
    check_stage(result_m5, "Module 5 refinement", allow_warn=True)
    asset_id  = result_m5["data"]["asset_id"]
    loaded    = m13.load_session(sid)
    asset     = next((a for a in loaded["assets"] if a["id"] == asset_id), None)
    assert asset is not None and asset.get("track_type") == "refinement"
    print(f"  Asset: {asset_id}, track_type: {asset['track_type']}")
    print()
except Exception as e:
    print(f"[FAIL] Stage 4 — {e}")
    failed = True

# ── Stage 5: Module 12 toggle verification ───────────────────────────────
print("--- Stage 5: Toggle Verification (Module 12) ---")
try:
    result_m12 = m12.run(
        session_id=sid,
        approved_stem_ids=[],
        rejected_stem_ids=["counter_melody"],
        stem_local_paths={}
    )
    check_stage(result_m12, "Module 12")
    print(f"  Approved: {result_m12['data']['approved_count']}, Rejected: {result_m12['data']['rejected_count']}")
    print()
except Exception as e:
    print(f"[FAIL] Stage 5 — {e}")
    failed = True

# ── Stage 6: Final mix ────────────────────────────────────────────────────
print("--- Stage 6: Final Mix and Export (Module 14) ---")
drive_set = any(os.environ.get(f"MCS_DRIVE_CREDENTIALS_{i}") for i in range(1, 4))
if not drive_set:
    print("  [WARN] MCS_DRIVE_CREDENTIALS_* not set — Drive upload will fail gracefully")

try:
    # TEST ONLY — bypasses Modules 7 and 8. In production, status is set to "verified"
    # only after Module 7 and Module 8 pass. Never replicate this pattern in production code.
    loaded = m13.load_session(sid)
    for asset in loaded["assets"]:
        if asset.get("stem_id") == "atmosphere":
            m13.update_asset_field(sid, asset["id"], "status", "verified")
            m13.update_asset_field(sid, asset["id"], "local_path", str(AUDIO_FILE))

    result_m14 = m14.run(sid, {"vibe": 8, "originality": 7, "coherence": 9, "flag": "safe"})
    check_stage(result_m14, "Module 14", allow_warn=True)
    print(f"  Status: {result_m14['status']}")
    print(f"  Title: {result_m14['data'].get('title')}")
    print(f"  Stems mixed: {result_m14['data'].get('stems_mixed')}")
    print(f"  Drive success: {result_m14['data'].get('drive_success')}")
    print(f"  Releases success: {result_m14['data'].get('releases_success')}")
    print()
except Exception as e:
    print(f"[FAIL] Stage 6 — {e}")
    failed = True

# ── Stage 7: Session state audit ─────────────────────────────────────────
print("--- Stage 7: Session State Audit ---")
try:
    final      = m13.load_session(sid)
    cp_modules = {cp["module"] for cp in final["checkpoints"]}
    print(f"  Status: {final['status']}")
    print(f"  Checkpoints: {len(final['checkpoints'])} — modules: {cp_modules}")
    reconcile  = m13.reconcile_state(sid)
    if reconcile["valid"]:
        print("  Reconciliation: PASSED")
    else:
        print(f"  Reconciliation: FAILED — {reconcile['issues']}")
        failed = True

    if final["status"] == "COMPLETE":
        lib = ROOT / "data" / "library" / "song_library.json"
        if lib.exists():
            library = json.loads(lib.read_text())
            entry   = next((s for s in library if s["session_id"] == sid), None)
            if entry:
                print(f"  Library entry: title={entry['title']}, stems={entry['stems_used']}")
            else:
                print("  [WARN] Session not found in library")
    print()
except Exception as e:
    print(f"[FAIL] Stage 7 — {e}")
    failed = True

print("=" * 60)
if failed:
    print("[FAIL] Integration test failed — fix all failures before proceeding to Phase 8")
    sys.exit(1)
else:
    print("[OK] Integration test passed — State 2 pipeline ready")
    sys.exit(0)
```

Run it:

```bash
python3.11 backend/integration_test_phase7.py
```

---

## Completion Gate

Phase 7 is complete when ALL of the following are true:

1. `verify_phase7.py` exits code 0 — all `[OK]`
2. `integration_test_phase7.py` exits code 0 — Stage 6 may warn if Drive/Releases not configured
3. `module13.update_asset_field()` present, updates asset record, returns `False` for missing asset ID
4. `module5.run()` accepts `track_type`, defaults to `"core"`, stores in asset record via `update_asset_field()`
5. `module6.run()` stores `local_path` in asset record via `update_asset_field()` after extension — stem_id match only, no filename fallback
6. `module10.run([])` returns ok envelope with `active_count: 0`, `suggestions_sent: 0`
7. `module10.analyze_active_session()` returns all required fields
8. No-vocals directive present in `module10.build_suggestion_email_parts()` rendered prompt
9. `module11.run()` produces blueprints with `[REFINEMENT]` in `stem_function`, no-vocals as first negative constraint, `refinement: True` in `prompt_metadata`, `peak_level_db: -1` in output_specification
10. Module 11 validation checkpoint written with `valid` and `issues` fields after every run
11. `from backend.module3 import STEM_CATALOG` present in `module11.py`
12. `module11.run(requested_stem_ids=["atmosphere"])` returns only `atmosphere` blueprint
13. `module12._validate_inputs()` rejects nonexistent stem IDs — `run()` returns fail envelope
14. `module12.run()` checkpoints rejection decisions
15. `module14._apply_eq()` runs on all stem categories without error
16. `module14._load_stems()` reads `local_path` from asset record — no path reconstruction
17. `module14._generate_title()` produces filename-safe strings
18. `module14.run()` on non-ACTIVE session returns fail envelope with `"not ACTIVE"` in errors
19. `module14.run()` on ACTIVE session with verified stems assembles mix, writes library entry with `diversity_vector` read from session, marks session `COMPLETE`
20. `run_daily.py` State 2 gate: skips State 1 pipeline when ACTIVE sessions exist, runs Module 10 only
21. All session mutations through Module 13 public functions — no `_save_session` calls outside Module 13 itself
22. All checkpoints written — `module11`, `module11_validation`, `module12` appear in session logs
23. `reconcile_state` returns `valid: True` on test sessions after full pipeline
24. No new dependencies added
25. All thresholds from `backend/config.py`

---

## What You Must Not Do

- Do not build Phase 8 features (Module 15, adaptive feedback)
- Do not build Phase 9 hardening tests
- Do not make any frontend changes
- Do not add dependencies beyond `requirements.txt`
- Do not make Gemini API calls inside Module 11
- Do not auto-approve any refinement stem in Module 12
- Do not let Module 14 raise — structured envelope always
- Do not apply EQ globally — per stem category via `EQ_MAP` only
- Do not define threshold constants locally — import from `backend.config`
- Do not hardcode absolute paths
- Do not pass numeric literals to librosa — use `SAMPLE_RATE` and `HOP_LENGTH` from `audio_analysis.py`
- Do not call `module13._save_session` anywhere — use `update_asset_field()` for asset mutations
- Do not reconstruct file paths in Module 14 — read `asset["local_path"]` exclusively
- Do not use filename-based fallback when looking up assets by stem_id — match on stem_id only
- Do not run State 1 pipeline when ACTIVE sessions exist — State 2 gate is hard
- Do not continue past any failed task — stop and print diagnostic