# MCS Phase 3 — Core Backend Modules (State 1) Agent Prompt (v3)

## Context

You are building the core backend modules for MCS (Music Creator Studio). This is Phase 3. Phase 0 (environment setup), Phase 1 (data layer, Module 13, session JSON schema), and Phase 2 (audio analysis foundation, vector extractors, similarity comparators) are already complete and verified.

Phase 3 builds eight system modules, one adapter layer, and one shared config file:
- **`config.py`** — Tunable threshold constants (built first — all modules import from here)
- **Module 1** — Pre-Generation Constraint Engine
- **Module 2** — Song Planner
- **Module 3** — Prompt Generator (deterministic only)
- **Module 3.5** — Prompt Validator
- **Module 5** — Stem Intake
- **Module 6** — Sample Extender
- **Module 7** — Song Identity Validator
- **Module 8** — Diversity Filter
- **`gemini_renderer.py`** — Isolated Gemini API adapter (separate from all pipeline logic)

Plus one unverified assumption test:
- **Test 3** — Gemini prompt variation (documented result, not pass/fail)

And one hybrid integration test verifying generation readiness across the full State 1 pipeline.

**No frontend. No email. No Drive. No new dependencies beyond `requirements.txt`.**

**Stop on first failure. Print `[OK]` / `[FAIL]` diagnostics. Do not continue to the next task.**

---

## v3 Changes From v2 (Read Before Writing Any Code)

Three targeted changes are incorporated in this version. Everything else from v2 is unchanged.

1. **Module 3 ordering assumption documented** — The `active_stems` list is built sequentially from `STEM_CATALOG` order. This creates an implicit ordering dependency: a stem's `global_context` depends on which stems were processed before it. This is intentional and correct for Phase 3. A future improvement (`active_stems_snapshot` frozen per stem iteration) is explicitly flagged as a v3.0 deferred item. Do not restructure Module 3 — document this note in the module docstring only.

2. **Module 6 fallback structurally tracked** — When `detect_stable_regions()` returns empty and the full stem is used as the loop segment, add `"low_stability_source_used"` to the module's `flags` list in the returned failure envelope. The `[WARN]` print already existed in v2 — this change ensures the fallback is also machine-readable in the envelope, not just human-readable in stdout.

3. **Integration test Module 13 assumption documented** — Add a comment block at the top of `integration_test_phase3.py` explicitly stating that the integration test assumes Module 13 is correct (verified in Phase 1). If Module 13 has a latent bug, downstream failures in this test may be false positives. This is a known architectural dependency, not a defect.

---

## Architectural Rules (Read Before Writing Any Code)

1. **All session mutations must occur only through Module 13 public functions.** Import and call `backend.module13` — never write to session dicts directly.
2. **Every module writes a checkpoint via `module13.write_checkpoint()` before passing control.** No module exits without checkpointing.
3. **No new dependencies may be added.** Use only what is already in `requirements.txt`: librosa, pydub, numpy, scipy, google-generativeai (already installed in Phase 0).
4. **All librosa calls must use `SAMPLE_RATE` and `HOP_LENGTH` constants from `audio_analysis.py`.** Never pass numeric literals to librosa functions. Import the constants directly: `from backend.audio_analysis import SAMPLE_RATE, HOP_LENGTH`.
5. **Module 3 is deterministic and algorithmic only.** It produces a structured `prompt_blueprint` dict. No Gemini API calls occur inside Module 3. The Gemini adapter (`gemini_renderer.py`) is the only place API calls are made, and it is isolated from all pipeline logic.
6. **Flags are advisory — no module auto-rejects stems.** Modules 5, 7, and 8 surface flags. Human review is the final authority. Auto-rejection is forbidden.
7. **Curve-shape cosine distance only.** All similarity comparisons operate on full feature curves via `backend.similarity`. Never compare scalar averages.
8. **Integration test verifies generation readiness, not generation outcome.** The Gemini audio execution step is manual. The test does not simulate or mock Gemini audio output.
9. **Module 7 is read-only.** It never initializes or mutates the identity vector. If the vector is missing or unpopulated, it returns a flag. Vector initialization is the pipeline's responsibility upstream.
10. **All thresholds come from `config.py`.** No module defines a threshold constant locally. Import from `backend.config`.

---

## Path Rule (Mandatory — Read Before Writing Any Code)

Every script and module in Phase 3 that needs the repository root or audio directory must define these lines at the top, derived from its own file location:

```python
ROOT = Path(__file__).resolve().parents[1]
AUDIO_DIR = ROOT / "data" / "audio"
SESSIONS_DIR = ROOT / "data" / "sessions"
```

Never hardcode absolute paths. Never assume a working directory. This rule applies to all modules and scripts in this phase.

---

## Standard Failure Envelope (Mandatory — All `run()` Functions Must Use)

Every module `run()` function must return a result dict that conforms to this envelope. This is the contract Module 13 uses to classify system health. Define this shape once per module — do not improvise per-module return shapes.

```python
{
    "module": str,           # module name, e.g. "module5"
    "session_id": str,
    "status": "ok" | "warn" | "fail",
    "flags": [str, ...],     # advisory flags — may be non-empty even when status is "ok"
    "errors": [str, ...],    # blocking errors — non-empty means status is "fail"
    "data": dict             # module-specific output payload — see each module spec below
}
```

Rules:
- `status = "ok"` — pipeline ran cleanly. `errors` is empty. `flags` may contain advisory notices.
- `status = "warn"` — pipeline ran but advisory flags are present that the human should review.
- `status = "fail"` — a blocking error occurred. `errors` is non-empty. Pipeline should not continue past this module without human intervention.
- Every module that writes a checkpoint must include `"status"` in the checkpoint `data` field.
- Modules that currently return plain dicts (Modules 1, 2, 3, 3.5) must also wrap their output in this envelope.

---

## Preconditions

Confirm Phase 0, 1, and 2 are complete before starting:

1. Folder structure exists: `/backend`, `/data`, `/data/sessions`, `/data/audio`, `/data/library`, `/temp`
2. `requirements.txt` present and all packages installed
3. `backend/__init__.py` exists
4. `backend/module13.py` exists and imports cleanly
5. `backend/audio_analysis.py`, `backend/diversity_vector.py`, `backend/identity_vector.py`, `backend/similarity.py` all exist and import cleanly
6. Test audio file exists at: `data/audio/heavy_gauge.mp3`
7. `data/library/` directory exists (create if missing — needed by Module 1 and 8)

If any are missing, print `[FAIL] Precondition failed — <what is missing>` and stop.

---

## Task 0 — Shared Config File

Create `/backend/config.py` **before writing any other module**. All Phase 3 modules import thresholds from here. No module defines threshold constants locally.

```python
"""
MCS Phase 3 — Shared Tunable Configuration

All threshold values live here. No module hardcodes these.
To adjust a threshold, change it here only — it propagates everywhere.

Thresholds marked PROVISIONAL are calibrated against an empty library.
They should be reviewed after 5+ approved songs exist in the library.
"""

# ── Audio analysis ─────────────────────────────────────────────────────────
BPM_TOLERANCE = 3.0                  # acceptable BPM delta for stem identity match (Module 5)
SILENCE_THRESHOLD_DB = -60.0         # dBFS below which audio is considered silence (Module 5)
PEAK_LEVEL_TARGET_DB = -1.0          # target peak level in dBFS (Module 5)
PEAK_LEVEL_TOLERANCE_DB = 1.0        # acceptable delta from target (Module 5) — range: -2.0 to 0.0 dBFS
DURATION_TOLERANCE_SECONDS = 2.0     # acceptable duration delta from target (Module 5)

# ── Sample extender ────────────────────────────────────────────────────────
CROSSFADE_MS = 50                    # crossfade duration in milliseconds (Module 6)
MIN_SEGMENT_BARS = 4                 # minimum bars for a stable loop segment (Module 6)
RMS_STABILITY_THRESHOLD = 0.15       # max RMS variance for a region to qualify as stable (Module 6)
                                     # PROVISIONAL — tune after testing on real Gemini stems

# ── Identity validation ────────────────────────────────────────────────────
ENERGY_SLOPE_TOLERANCE = 0.3         # max absolute slope delta for soft pass (Module 7)
PEAK_TIMING_TOLERANCE = 0.25         # max absolute peak timing delta for soft pass (Module 7)

# ── Diversity filter ───────────────────────────────────────────────────────
DIVERSITY_HIGH_SIMILARITY_THRESHOLD = 0.92   # per-dimension flag threshold (Module 8)
                                              # PROVISIONAL — meaningless below 5 library songs
DIVERSITY_OVERALL_THRESHOLD = 0.90           # overall similarity flag threshold (Module 8)
                                              # PROVISIONAL — review after library reaches 5+ songs
```

---

## Task 1 — Module 1: Pre-Generation Constraint Engine

Create `/backend/module1.py`.

Module 1 runs at the start of every generation cycle. It scans the song library, builds a constraint profile for the new session, and injects entropy. It feeds into Module 2.

### Constants

```python
from pathlib import Path
import json, random
from backend import config

ROOT = Path(__file__).resolve().parents[1]
AUDIO_DIR = ROOT / "data" / "audio"
SESSIONS_DIR = ROOT / "data" / "sessions"
LIBRARY_FILE = ROOT / "data" / "library" / "song_library.json"

VALID_ENTROPY_DIMENSIONS = {"identity", "rhythm", "energy_trajectory", "timbre", "harmonic"}
VALID_MOODS = {"dark", "bright", "melancholic", "tense", "euphoric", "neutral"}
VALID_ENERGIES = {"sparse", "layered", "dense", "driving", "ambient"}
VALID_GENRES = {"trap", "ambient", "electronic", "lo-fi", "experimental", "cinematic"}

# v3.0 DEFERRED: per-stem entropy adaptation
# Current design: one entropy injection per generation cycle, session-wide.
# This is intentional — entropy sets a creative bias for the whole session.
# Per-stem entropy would break the one-injection-per-cycle rule and
# undermine reproducibility. Do not implement per-stem entropy here.
```

### `load_library() -> list`

- Loads `LIBRARY_FILE` if it exists — returns list of song records
- If file does not exist or is empty: returns `[]` and prints `[WARN] Library empty — no prior songs to analyze`
- Never raises on missing file — empty library is a valid starting state

### `analyze_library_gaps(library: list) -> dict`

- Scans all song records in library for mood, energy, genre, and harmonic_center distributions
- Returns a gap report:

```python
{
    "mood_counts": {"dark": int, "bright": int, ...},
    "energy_counts": {"sparse": int, ...},
    "genre_counts": {"trap": int, ...},
    "underrepresented_moods": [str, ...],      # moods with 0 songs
    "underrepresented_energies": [str, ...],   # energies with 0 songs
    "total_songs": int
}
```

- If library is empty: all counts are 0, all valid moods and energies are underrepresented
- Never raises

### `generate_entropy(session_id: str, forced_dimension: str = None) -> dict`

- If `forced_dimension` is None: randomly selects one dimension from `VALID_ENTROPY_DIMENSIONS`
- If `forced_dimension` provided: validates it is in `VALID_ENTROPY_DIMENSIONS` — if not raise `ValueError`
- Generates:
  - `intensity`: random float in [0.1, 0.9] rounded to 2 decimal places
  - `seed`: random integer in [1000, 9999]
- Returns entropy object:

```python
{
    "dimension": str,
    "intensity": float,
    "seed": int
}
```

- Logs entropy to session via `module13.write_checkpoint(session_id, "module1", {"event": "entropy_injected", "entropy": entropy_obj, "status": "ok"})`
- Prints `[OK] Entropy injected — dimension: <dimension>, intensity: <intensity>, seed: <seed>`

### `build_constraint_profile(library: list, entropy: dict) -> dict`

- Calls `analyze_library_gaps(library)`
- Builds constraint profile:

```python
{
    "preferred_moods": [str, ...],
    "preferred_energies": [str, ...],
    "entropy": entropy,
    "library_size": int,
    "gap_report": dict
}
```

- Prints `[OK] Constraint profile built — library size: <n>, preferred moods: <list>`
- Returns the constraint profile dict

### `run(session_id: str, forced_dimension: str = None) -> dict`

- Loads library
- Generates entropy via `generate_entropy(session_id, forced_dimension)`
- Builds constraint profile via `build_constraint_profile(library, entropy)`
- Writes checkpoint: `module="module1"`, `data={"event": "constraint_profile_ready", "library_size": n, "status": "ok"}`
- Prints `[OK] Module 1 complete`
- Returns standard failure envelope:

```python
{
    "module": "module1",
    "session_id": session_id,
    "status": "ok",
    "flags": [],
    "errors": [],
    "data": constraint_profile
}
```

---

## Task 2 — Module 2: Song Planner

Create `/backend/module2.py`.

Module 2 generates a song blueprint from the constraint profile produced by Module 1. It writes the blueprint to a new session via Module 13. It supports two modes: scheduled (auto-select from constraint profile) and manual (explicit parameters).

### Constants

```python
from pathlib import Path
import json, random
from backend import config

ROOT = Path(__file__).resolve().parents[1]

VALID_MOODS = {"dark", "bright", "melancholic", "tense", "euphoric", "neutral"}
VALID_ENERGIES = {"sparse", "layered", "dense", "driving", "ambient"}
VALID_GENRES = {"trap", "ambient", "electronic", "lo-fi", "experimental", "cinematic"}

DEFAULT_TEMPO_RANGE = (80, 140)
DEFAULT_DURATION = 180
```

### `generate_blueprint_scheduled(constraint_profile: dict) -> dict`

- Selects mood from `constraint_profile["preferred_moods"]` — random choice if multiple
- Selects energy from `constraint_profile["preferred_energies"]` — random choice if multiple
- Selects genre randomly from `VALID_GENRES`
- Generates tempo_bpm: random integer in `DEFAULT_TEMPO_RANGE`
- Returns blueprint:

```python
{
    "title": f"{mood}_{genre}_{tempo_bpm}bpm",
    "mood": str,
    "energy": str,
    "genre": str,
    "tempo_bpm": int,
    "target_duration_seconds": DEFAULT_DURATION,
    "harmonic_center": "",
    "notes": "auto-generated",
    "mode": "scheduled"
}
```

### `generate_blueprint_manual(title: str, mood: str, energy: str, genre: str, tempo_bpm: int, target_duration_seconds: int = DEFAULT_DURATION, harmonic_center: str = "", notes: str = "") -> dict`

- Validates `mood` in `VALID_MOODS` — raise `ValueError` if not
- Validates `energy` in `VALID_ENERGIES` — raise `ValueError` if not
- Validates `genre` in `VALID_GENRES` — raise `ValueError` if not
- Validates `tempo_bpm` is a positive integer — raise `ValueError` if not
- Returns blueprint dict with `"mode": "manual"` added

### `run(constraint_profile: dict, mode: str = "scheduled", manual_params: dict = None) -> dict`

- If `mode == "scheduled"`: calls `generate_blueprint_scheduled(constraint_profile)`
- If `mode == "manual"`: calls `generate_blueprint_manual(**manual_params)` — raise `ValueError` if `manual_params` is None
- Builds `song_plan` dict matching Module 13's required keys exactly
- Creates session via `module13.create_session(song_plan)`
- Writes checkpoint: `module="module2"`, `data={"event": "blueprint_created", "mode": mode, "genre": blueprint["genre"], "status": "ok"}`
- Prints `[OK] Module 2 complete — session: <session_id>, mode: <mode>`
- Returns standard failure envelope:

```python
{
    "module": "module2",
    "session_id": session_id,
    "status": "ok",
    "flags": [],
    "errors": [],
    "data": {"session_id": session_id, "blueprint": blueprint}
}
```

---

## Task 3 — Module 3: Prompt Generator

Create `/backend/module3.py`.

Module 3 is **deterministic and algorithmic only**. It produces a structured `prompt_blueprint` for each stem. No Gemini API calls occur here.

### Architecture Note

Module 3 produces `prompt_blueprint` dicts — structured intermediate representations. `gemini_renderer.py` converts these into natural language downstream. Module 3 never calls `gemini_renderer.py`.

### Ordering Dependency Note (v3.0 Deferred)

The `active_stems` list is built sequentially from `STEM_CATALOG` order. Each stem's `global_context` field depends on which stems were processed before it in the iteration. This ordering is intentional and correct for Phase 3 — `STEM_CATALOG` order is the authority. A future improvement (freezing an `active_stems_snapshot` per stem iteration to make context fully reproducible regardless of runtime state) is explicitly deferred to v3.0. Do not restructure Module 3 to address this now.

**v3.0 DEFERRED:** Multi-section prompt schemas (verse/chorus/bridge), adaptive form-based generation, and per-stem entropy adaptation are explicitly out of scope. The 8-field schema is correct and complete for Phase 3. Do not extend it here.

### Constants

```python
from pathlib import Path
import json, random
from backend import config

ROOT = Path(__file__).resolve().parents[1]

SCHEMA_VERSION = "1.0"

VALID_MOODS = {"dark", "bright", "melancholic", "tense", "euphoric", "neutral"}
VALID_TEXTURES = {"grainy", "smooth", "distorted", "clean", "warm", "cold"}
VALID_ENERGIES = {"sparse", "layered", "dense", "driving", "ambient"}
VALID_DYNAMICS = {"static", "building", "declining", "volatile"}

STEM_CATALOG = [
    {"stem_id": "drums",   "role": "foundational", "priority": 1, "frequency_range": "sub-bass to upper-mid", "dependencies": []},
    {"stem_id": "bass",    "role": "foundational", "priority": 1, "frequency_range": "sub-bass to low-mid",   "dependencies": ["drums"]},
    {"stem_id": "chords",  "role": "harmonic",     "priority": 2, "frequency_range": "low-mid to high",       "dependencies": ["drums", "bass"]},
    {"stem_id": "melody",  "role": "harmonic",     "priority": 2, "frequency_range": "mid to high",           "dependencies": ["drums", "bass", "chords"]},
    {"stem_id": "texture", "role": "texture",      "priority": 3, "frequency_range": "full spectrum",         "dependencies": ["drums", "bass", "chords", "melody"]},
]

MOOD_TEXTURE_MAP = {
    "dark":        {"texture": "grainy",    "dynamic": "volatile"},
    "bright":      {"texture": "clean",     "dynamic": "building"},
    "melancholic": {"texture": "warm",      "dynamic": "declining"},
    "tense":       {"texture": "distorted", "dynamic": "volatile"},
    "euphoric":    {"texture": "smooth",    "dynamic": "building"},
    "neutral":     {"texture": "clean",     "dynamic": "static"},
}

ENERGY_DESCRIPTOR_MAP = {
    "sparse":   "minimal arrangement, wide stereo space, long reverb tails",
    "layered":  "multiple interlocking elements, mid-range density",
    "dense":    "full frequency spectrum, heavy layering, tight mix",
    "driving":  "strong rhythmic pulse, forward motion, punchy transients",
    "ambient":  "slow evolving textures, no sharp transients, heavy reverb",
}
```

### `_build_global_context(active_stems: list) -> str`

- If `active_stems` is empty: returns `""` (first stem — no context needed)
- Otherwise builds formatted string listing all active stems with frequency range and role
- Returns the formatted string

### `_apply_entropy(base_descriptor: str, entropy: dict) -> str`

- If `entropy["intensity"]` >= 0.7: appends `" [HIGH ENTROPY — intentional variation expected]"`
- If `entropy["intensity"]` >= 0.4: appends `" [MODERATE ENTROPY — subtle variation]"`
- Otherwise: returns `base_descriptor` unchanged
- This is the only place free-form language is permitted — ENTROPY INJECTION field only

### `_compute_loop_bars(tempo_bpm: int, duration_seconds: int) -> int`

- Returns `max(1, (tempo_bpm * duration_seconds) // (4 * 60))`

### `_build_negative_constraints(stem_def: dict, active_stems: list) -> list`

- Always includes: `"no leading or trailing silence"`, `"no fade in"`, `"no fade out"`
- For each stem in `active_stems`: adds `f"do not duplicate or mask {s['stem_id']} frequency range"`
- Returns list of strings

### `_compute_constraint_density(stem_def: dict, active_stems: list) -> int`

- Returns: 3 (base output constraints) + `len(active_stems)` + 4 (mood/texture/energy/dynamic descriptors)

### `build_prompt_blueprint(session: dict, stem_def: dict, active_stems: list, entropy: dict) -> dict`

Builds one complete `prompt_blueprint` for a single stem. All fields populated from session data and `STEM_CATALOG`. No freeform choices.

Returns the full blueprint dict containing all 8 P1 fields, global_context, execution_priority, dependency_flag, and prompt_metadata. Structure identical to v2 — no changes to blueprint shape.

### `run(session_id: str, constraint_profile: dict) -> dict`

- Loads session via `module13.load_session(session_id)`
- Extracts entropy from `constraint_profile["data"]["entropy"]` if constraint_profile uses envelope, else `constraint_profile["entropy"]`
- Iterates through `STEM_CATALOG` in priority order, building `active_stems` list sequentially
- For each stem: calls `build_prompt_blueprint()`, logs prompt metadata via `module13.log_prompt_metadata()`
- Writes checkpoint: `module="module3"`, `data={"event": "prompt_queue_ready", "stem_count": n, "status": "ok"}`
- Prints `[OK] Module 3 complete — <n> blueprints generated`
- Returns standard failure envelope:

```python
{
    "module": "module3",
    "session_id": session_id,
    "status": "ok",
    "flags": [],
    "errors": [],
    "data": {"blueprints": [list of blueprint dicts ordered by priority]}
}
```

---

## Task 4 — Module 3.5: Prompt Validator

Create `/backend/module3_5.py`.

Module 3.5 validates every blueprint from Module 3 before it reaches the queue. Nothing passes with an unresolved validation failure.

### Auto-Fix Rule (Deterministic — Read Before Writing)

Auto-fix applies **only** when both conditions are true:
1. The field is **entirely absent** from the blueprint (key does not exist)
2. The correct value is **unambiguously derivable** from session data with no interpretation required

If the field is present but incorrect: **escalate as a blocking issue — never silently correct.**

Auto-fixable fields (and their derivation source):
- `output_specification.peak_level_db` — absent → set to `-1` (P5 rule, always -1, no ambiguity)
- `structural_constraints.time_signature` — absent → set to `"4/4"` (system default, no ambiguity)

All other fields: if wrong or missing, escalate as blocking issue.

### Constants

```python
from pathlib import Path
from backend import config

ROOT = Path(__file__).resolve().parents[1]

REQUIRED_BLUEPRINT_KEYS = {
    "schema_version", "session_id", "stem_id", "execution_priority",
    "dependency_flag", "role", "stem_function", "song_identity_lock",
    "structural_constraints", "audio_characteristics", "negative_constraints",
    "entropy_injection", "output_specification", "global_context", "prompt_metadata"
}

REQUIRED_OUTPUT_SPEC_KEYS = {"peak_level_db", "no_silence", "loop_alignment", "no_fade", "duration_seconds"}
REQUIRED_SONG_IDENTITY_KEYS = {"mood", "energy", "tempo_bpm", "harmonic_center"}
REQUIRED_STRUCTURAL_KEYS = {"duration_seconds", "loop_bars", "time_signature"}
REQUIRED_PROMPT_METADATA_KEYS = {"structure_version", "entropy_config", "descriptor_set", "constraint_density_score", "global_context_stem_count"}

VALID_MOODS = {"dark", "bright", "melancholic", "tense", "euphoric", "neutral"}
VALID_ENERGIES = {"sparse", "layered", "dense", "driving", "ambient"}
VALID_PRIORITIES = {1, 2, 3}

# Auto-fixable fields — defined explicitly, not inferred
AUTO_FIX_RULES = {
    ("output_specification", "peak_level_db"): -1,
    ("structural_constraints", "time_signature"): "4/4"
}
```

### `validate_blueprint(blueprint: dict, stem_index: int, total_stems: int) -> dict`

Validates one blueprint. Collects ALL issues — does not stop on first failure.

Run all checks in order. For auto-fixable fields: apply fix only if field is absent. If field is present and wrong, add to `issues`.

Checks (identical to v2 list) plus:
- Check 5b: if `output_specification` exists AND `"peak_level_db"` key exists AND value != -1: add issue `"output_specification.peak_level_db present but not -1 — escalating (not auto-fixed)"`
- Check 5c: if `structural_constraints` exists AND `"time_signature"` key exists AND value != "4/4": add issue `"structural_constraints.time_signature present but not '4/4' — escalating (not auto-fixed)"`

Returns:
```python
{
    "stem_id": str,
    "valid": True | False,
    "issues": [str, ...],   # blocking issues
    "fixes": [str, ...],    # auto-fixes applied (field absent → default set)
    "warnings": [str, ...]
}
```

### `validate_queue(blueprints: list) -> dict`

- Runs `validate_blueprint()` on each blueprint
- If any blueprint has `valid: False`: entire queue is invalid
- Returns:

```python
{
    "queue_valid": True | False,
    "total_stems": int,
    "results": [per-stem validation results],
    "blocking_issues": [all issues across all invalid stems]
}
```

### `run(session_id: str, blueprints: list) -> dict`

- Calls `validate_queue(blueprints)`
- If queue valid:
  - Writes checkpoint: `module="module3_5"`, `data={"event": "validation_passed", "stem_count": n, "status": "ok"}`
  - Prints `[OK] Module 3.5 complete — all <n> blueprints valid`
  - Returns envelope with `status: "ok"`
- If queue invalid:
  - Writes checkpoint: `module="module3_5"`, `data={"event": "validation_failed", "issues": blocking_issues, "status": "fail"}`
  - Prints `[FAIL] Module 3.5 — validation failed: <issues>`
  - Returns envelope with `status: "fail"` — does NOT raise

Returns standard failure envelope:
```python
{
    "module": "module3_5",
    "session_id": session_id,
    "status": "ok" | "fail",
    "flags": [fixes applied as advisory notices],
    "errors": [blocking_issues if failed, else []],
    "data": validation_result
}
```

---

## Task 5 — Module 5: Stem Intake

Create `/backend/module5.py`.

Module 5 ingests a stem file, verifies it matches the session, checks output normalization, and registers the asset with Module 13.

**Responsibility boundary:** Module 5 checks whether the file is usable (normalization) and whether it belongs to this session (BPM identity match). It does not validate sonic coherence with the session's identity vector — that is Module 7's responsibility.

### Constants

```python
from pathlib import Path
import numpy as np
from backend import config
from backend.audio_analysis import SAMPLE_RATE, HOP_LENGTH

ROOT = Path(__file__).resolve().parents[1]
AUDIO_DIR = ROOT / "data" / "audio"
```

All threshold values (BPM_TOLERANCE, SILENCE_THRESHOLD_DB, PEAK_LEVEL_TARGET_DB, PEAK_LEVEL_TOLERANCE_DB, DURATION_TOLERANCE_SECONDS) must be imported from `backend.config`. Do not define them locally.

### `check_output_normalization(y: np.ndarray, sr: int, expected_duration_seconds: int) -> dict`

- Computes peak level in dBFS: `20 * np.log10(np.max(np.abs(y)) + 1e-9)`
- Checks for leading/trailing silence: first and last 100ms — if RMS < `config.SILENCE_THRESHOLD_DB`: flag
- Checks duration delta vs `expected_duration_seconds` — flag if > `config.DURATION_TOLERANCE_SECONDS`
- Returns normalization result dict with `flags` list

### `match_stem_to_session(y: np.ndarray, sr: int, session: dict, stem_id: str) -> dict`

- Detects BPM via `audio_analysis.detect_bpm(y, sr)`
- Compares to `session["song_plan"]["tempo_bpm"]` using `config.BPM_TOLERANCE`
- Returns match result dict with `flags` list

### `run(session_id: str, filepath: str, stem_id: str) -> dict`

- Validates `filepath` exists — if not: return envelope with `status: "fail"`, `errors: ["Stem file not found — <filepath>"]` and raise `FileNotFoundError`
- Loads session via `module13.load_session(session_id)`
- Updates session status to ACTIVE via `module13.update_status()` if currently INITIALIZED
- Loads audio once: `y, sr = audio_analysis.load_audio(filepath)` — passes `y, sr` to both check functions
- Runs normalization check and identity match
- Registers asset via `module13.register_asset()`
- Assembles all flags from both checks
- Writes checkpoint: `module="module5"`, `data={"event": "stem_ingested", "stem_id": stem_id, "flags": all_flags, "status": "warn" if all_flags else "ok"}`
- Prints `[WARN] Stem <stem_id> flagged — <flags>` if flags present (advisory only)
- Prints `[OK] Module 5 complete — stem: <stem_id>, asset: <asset_id>`
- Returns standard failure envelope:

```python
{
    "module": "module5",
    "session_id": session_id,
    "status": "warn" if all_flags else "ok",
    "flags": all_flags,
    "errors": [],
    "data": {
        "stem_id": stem_id,
        "filepath": filepath,
        "normalization": normalization_result,
        "identity_match": match_result,
        "asset_id": asset_id
    }
}
```

---

## Task 6 — Module 6: Sample Extender

Create `/backend/module6.py`.

Module 6 extends a stem algorithmically using librosa segmentation when the stem is shorter than the target duration. No AI. No audio generation.

### Constants

```python
from pathlib import Path
import numpy as np
import librosa
from pydub import AudioSegment
from backend import config
from backend.audio_analysis import SAMPLE_RATE, HOP_LENGTH

ROOT = Path(__file__).resolve().parents[1]
TEMP_DIR = ROOT / "temp"
```

All threshold values (CROSSFADE_MS, MIN_SEGMENT_BARS, RMS_STABILITY_THRESHOLD) must be imported from `backend.config`. Do not define them locally.

### `detect_stable_regions(y: np.ndarray, sr: int, tempo_bpm: int) -> list`

- Computes RMS energy profile via `audio_analysis.extract_energy_profile(y, sr)`
- Computes bar length in frames: `(60 / tempo_bpm) * 4 * sr / HOP_LENGTH`
- Scans for regions where RMS variance is below `config.RMS_STABILITY_THRESHOLD` over at least `config.MIN_SEGMENT_BARS` bars
- Returns list of `(start_frame, end_frame)` tuples — may be empty

### `extend_stem(filepath: str, target_duration_seconds: int, tempo_bpm: int) -> tuple[str, list]`

- Loads audio using pydub and librosa
- If already >= target: prints `[OK] Stem already at target duration — no extension needed`, returns `(filepath, [])`
- Calls `detect_stable_regions()` — if empty:
  - Prints `[WARN] No stable regions found — using full stem as loop segment`
  - Returns fallback flag `["low_stability_source_used"]` alongside the output path
- Selects longest stable region — falls back to full stem if none
- Builds extended audio by concatenating with `config.CROSSFADE_MS` crossfade until target duration reached
- Exports to `TEMP_DIR / f"{Path(filepath).stem}_extended.wav"`
- Prints `[OK] Sample extended — <original_duration>s → <extended_duration>s`
- Returns `(output_filepath_string, flags_list)` — flags_list is empty on clean extension, contains `"low_stability_source_used"` on fallback

### `run(session_id: str, filepath: str, stem_id: str) -> dict`

- Loads session via `module13.load_session(session_id)`
- Extracts `target_duration_seconds` and `tempo_bpm` from `session["song_plan"]`
- Runs `extend_stem()` — unpacks `(output_path, extension_flags)`
- Writes checkpoint: `module="module6"`, `data={"event": "stem_extended", "stem_id": stem_id, "output": output_path, "flags": extension_flags, "status": "warn" if extension_flags else "ok"}`
- Prints `[OK] Module 6 complete — stem: <stem_id>`
- Returns standard failure envelope:

```python
{
    "module": "module6",
    "session_id": session_id,
    "status": "warn" if extension_flags else "ok",
    "flags": extension_flags,
    "errors": [],
    "data": {"stem_id": stem_id, "output_path": output_path}
}
```

---

## Task 7 — Module 7: Song Identity Validator

Create `/backend/module7.py`.

Module 7 compares a stem's extracted identity vector against the session's stored identity vector. **Module 7 is read-only.** It never initializes or mutates the identity vector under any circumstance. If the vector is missing or unpopulated, it returns a flag and exits cleanly. Vector initialization is the pipeline's responsibility before Module 7 runs.

### Vector Precondition Contract

Before running any comparison, Module 7 must check whether the session identity vector is populated. A vector is considered **unpopulated** if all 7 fields are `None`.

If unpopulated:
- Print `[WARN] Session identity vector not populated — Module 7 cannot compare. Initialize vector before running Module 7.`
- Return envelope with `status: "warn"`, `flags: ["identity_vector_missing"]`, and `data: {"skipped": True}`
- Do NOT initialize the vector. Do NOT call `identity_vector.write_identity_vector_to_session()`. Do NOT mutate the session.

If populated (at least one non-None field):
- Proceed with comparison
- For any field that is `None` in the session vector: skip that field and add `"field <name> is None in session vector — skipped"` to `skipped_fields`

### Constants

```python
from pathlib import Path
from backend import config
from backend.audio_analysis import SAMPLE_RATE, HOP_LENGTH

ROOT = Path(__file__).resolve().parents[1]
```

All tolerance values (ENERGY_SLOPE_TOLERANCE, PEAK_TIMING_TOLERANCE) must be imported from `backend.config`.

### `_is_vector_populated(identity_vector: dict) -> bool`

- Returns `True` if at least one value in the vector dict is not `None`
- Returns `False` if all values are `None`

### `compare_identity_vectors(session_vector: dict, stem_vector: dict) -> dict`

- Compares fields present and non-None in session vector:
  - `mood`: exact match — flag if different
  - `harmonic_center`: flag if different (advisory)
  - `energy_trajectory`: flag if different (soft warning)
  - `energy_slope`: flag if absolute delta > `config.ENERGY_SLOPE_TOLERANCE`
  - `peak_timing`: flag if absolute delta > `config.PEAK_TIMING_TOLERANCE`
  - Placeholder fields (`density_trajectory`, `density_volatility`): skip — add to `skipped_fields`
- Returns comparison result dict with `flags` and `skipped_fields`

### `run(session_id: str, filepath: str, stem_id: str) -> dict`

- Loads session via `module13.load_session(session_id)`
- Calls `_is_vector_populated(session["identity_vector"])`
- If not populated: writes checkpoint with `status: "warn"`, returns warn envelope — stops here
- Extracts stem identity vector via `identity_vector.extract_identity_vector(filepath, session["song_plan"]["mood"])`
- Runs `compare_identity_vectors()`
- Writes checkpoint: `module="module7"`, `data={"event": "identity_validated", "stem_id": stem_id, "flags": flags, "status": "warn" if flags else "ok"}`
- Prints `[WARN] Identity mismatch flags for <stem_id> — <flags>` if flags (advisory only)
- Prints `[OK] Module 7 complete — stem: <stem_id>`
- Returns standard failure envelope:

```python
{
    "module": "module7",
    "session_id": session_id,
    "status": "warn" if flags else "ok",
    "flags": flags,
    "errors": [],
    "data": {
        "stem_id": stem_id,
        "skipped": False,
        "comparison": comparison_result
    }
}
```

---

## Task 8 — Module 8: Diversity Filter

Create `/backend/module8.py`.

Module 8 compares a new stem's diversity vector against library vectors. Similarity scores are advisory — never auto-rejection.

**Aggregation note:** Module 8 uses max similarity per dimension across library entries. With a small or empty library, max is the correct simple metric — top-k average or percentile aggregation is meaningless below 5 songs. Switching to a more sophisticated aggregation strategy is explicitly deferred to v3.0, to be revisited when the library has sufficient entries. The PROVISIONAL comments in `config.py` mark these thresholds accordingly.

### Vector Precondition Contract

Module 8 must verify the diversity vector extracted from the stem file is valid before comparison. A vector is considered **invalid** if extraction raised an exception or returned a dict with empty lists for all dimensions.

If the stem's diversity vector cannot be extracted:
- Print `[WARN] Diversity vector extraction failed for <stem_id> — comparison skipped`
- Return envelope with `status: "warn"`, `flags: ["diversity_vector_extraction_failed"]`, `data: {"skipped": True}`
- Do NOT raise. Do NOT block pipeline.

If library is empty:
- Print `[WARN] Library empty — diversity comparison skipped`
- Return envelope with `status: "ok"`, `flags: ["library_empty"]`, `data: {"library_empty": True}`

### Constants

```python
from pathlib import Path
import json
from backend import config

ROOT = Path(__file__).resolve().parents[1]
LIBRARY_FILE = ROOT / "data" / "library" / "song_library.json"
```

All threshold values (DIVERSITY_HIGH_SIMILARITY_THRESHOLD, DIVERSITY_OVERALL_THRESHOLD) must be imported from `backend.config`.

### `load_library_vectors() -> list`

- Loads `LIBRARY_FILE` — if missing or empty returns `[]`
- Extracts all `diversity_vector` fields from library records
- Returns list of vector dicts

### `run(session_id: str, filepath: str, stem_id: str) -> dict`

- Extracts diversity vector — wraps extraction in try/except, returns warn envelope on failure
- Loads library vectors
- If library empty: returns ok envelope with `library_empty` flag
- Compares new vector against each library vector via `similarity.compare_diversity_vectors()`
- Identifies highest similarity per dimension across all library entries
- Flags dimensions where max similarity > `config.DIVERSITY_HIGH_SIMILARITY_THRESHOLD`
- Flags if overall max > `config.DIVERSITY_OVERALL_THRESHOLD`
- Writes checkpoint: `module="module8"`, `data={"event": "diversity_checked", "stem_id": stem_id, "library_size": n, "flags": flags, "status": "warn" if flags else "ok"}`
- Returns standard failure envelope:

```python
{
    "module": "module8",
    "session_id": session_id,
    "status": "warn" if flags else "ok",
    "flags": flags,
    "errors": [],
    "data": {
        "stem_id": stem_id,
        "library_size": int,
        "library_empty": bool,
        "skipped": bool,
        "max_similarity_per_dimension": dict,
        "overall_max_similarity": float
    }
}
```

---

## Task 9 — Gemini Renderer Adapter

Create `/backend/gemini_renderer.py`.

This is the only file in Phase 3 that makes Gemini API calls. It is isolated from all pipeline logic. It never writes to sessions directly.

### Constants

```python
import os, json, time
from pathlib import Path

GEMINI_MODEL = "gemini-2.0-flash"
MAX_RETRIES = 3
RETRY_DELAY_SECONDS = 5
API_KEY_ENV_VAR = "GEMINI_API_KEY"
```

### `_build_render_prompt(blueprint: dict) -> str`

Builds the instruction string sent to Gemini. Directive — Gemini receives complete structure and renders only:

```
You are a music prompt renderer. Convert the following structured stem specification into a clear, expressive natural language prompt for Gemini audio generation.

Do not make creative decisions. Render exactly what is specified.
Do not add elements not present in the specification.
Do not remove any constraints from the specification.

OUTPUT FORMAT:
Return a single natural language prompt, 150–250 words.
Start with the stem's role and function.
Include all specified audio characteristics.
Include all negative constraints explicitly.
End with the output specification requirements.

STRUCTURED SPECIFICATION:
<JSON of blueprint fields>
```

Returns the full instruction string.

### `render_prompt(blueprint: dict) -> str`

- Reads API key from `os.environ.get(API_KEY_ENV_VAR)` — if not set raise `EnvironmentError`
- Initializes `google.generativeai` client
- Calls Gemini with retry logic — up to `MAX_RETRIES` attempts
- On success: prints `[OK] Gemini rendered — stem: <stem_id>`, returns rendered string
- On all retries exhausted: prints `[FAIL] Gemini render failed after <n> retries`, raises `RuntimeError`

### `render_queue(blueprints: list) -> list`

- Iterates blueprints in order
- Calls `render_prompt()` for each
- Returns list of `{"stem_id": str, "blueprint": dict, "rendered_prompt": str}`
- Prints `[OK] Gemini renderer complete — <n> prompts rendered`

---

## Task 10 — Unverified Assumption Test 3

Create `/backend/test_prompt_variation.py`.

Verifies whether varied descriptor language produces structurally different blueprints and (if API key available) different rendered prompts. Result documented — not pass/fail.

- Builds two minimal blueprint dicts manually:
  - Blueprint A: `mood=dark, energy=driving, texture=grainy, dynamic=volatile`
  - Blueprint B: `mood=bright, energy=ambient, texture=smooth, dynamic=static`
- Computes descriptor overlap between the two
- If `GEMINI_API_KEY` set: renders both and computes word-overlap ratio
- If key not set: skips render step, documents structural diff only

Prints:
```
[OK] Assumption Test 3 complete
Blueprint A descriptors: [list]
Blueprint B descriptors: [list]
Structural descriptor overlap: X/Y shared terms
[RESULT] STRUCTURAL VARIATION CONFIRMED | [RESULT] STRUCTURAL VARIATION LOW

[Gemini render comparison — if API key available]
Rendered prompt word-overlap ratio: X.XX
[RESULT] RENDER VARIATION CONFIRMED | [RESULT] RENDER VARIATION LOW
```

Does not assert pass/fail. Documents result for Phase 4 planning.

---

## Task 11 — Verification Script

Create `/backend/verify_phase3.py`:

```python
from pathlib import Path
import sys, importlib

ROOT = Path(__file__).resolve().parents[1]
AUDIO_DIR = ROOT / "data" / "audio"
AUDIO_FILE = AUDIO_DIR / "heavy_gauge.mp3"

failed = False

# --- Precondition ---
if not AUDIO_FILE.exists():
    print("[FAIL] heavy_gauge.mp3 not found — place the file at data/audio/heavy_gauge.mp3")
    sys.exit(1)
else:
    print("[OK] heavy_gauge.mp3 found")

# --- Import checks ---
modules_to_import = [
    ("backend.config", "cfg"),
    ("backend.module1", "m1"),
    ("backend.module2", "m2"),
    ("backend.module3", "m3"),
    ("backend.module3_5", "m35"),
    ("backend.module5", "m5"),
    ("backend.module6", "m6"),
    ("backend.module7", "m7"),
    ("backend.module8", "m8"),
    ("backend.gemini_renderer", "gr"),
    ("backend.module13", "m13"),
]
imported = {}
for mod_path, alias in modules_to_import:
    try:
        imported[alias] = importlib.import_module(mod_path)
        print(f"[OK] {mod_path} — imported successfully")
    except Exception as e:
        print(f"[FAIL] {mod_path} — {e}")
        failed = True

if failed:
    sys.exit(1)

cfg = imported["cfg"]
m1  = imported["m1"]
m2  = imported["m2"]
m3  = imported["m3"]
m35 = imported["m35"]
m5  = imported["m5"]
m6  = imported["m6"]
m7  = imported["m7"]
m8  = imported["m8"]
gr  = imported["gr"]
m13 = imported["m13"]

# --- config.py threshold contract ---
try:
    required_config_attrs = [
        "BPM_TOLERANCE", "SILENCE_THRESHOLD_DB", "PEAK_LEVEL_TARGET_DB",
        "PEAK_LEVEL_TOLERANCE_DB", "DURATION_TOLERANCE_SECONDS",
        "CROSSFADE_MS", "MIN_SEGMENT_BARS", "RMS_STABILITY_THRESHOLD",
        "ENERGY_SLOPE_TOLERANCE", "PEAK_TIMING_TOLERANCE",
        "DIVERSITY_HIGH_SIMILARITY_THRESHOLD", "DIVERSITY_OVERALL_THRESHOLD"
    ]
    for attr in required_config_attrs:
        assert hasattr(cfg, attr), f"Missing config attribute: {attr}"
    print("[OK] config.py — all required threshold attributes present")
except Exception as e:
    print(f"[FAIL] config.py — {e}")
    failed = True

# --- Standard failure envelope contract ---
def check_envelope(result, label):
    global failed
    required_keys = {"module", "session_id", "status", "flags", "errors", "data"}
    missing = required_keys - set(result.keys())
    if missing:
        print(f"[FAIL] {label} — envelope missing keys: {missing}")
        failed = True
        return False
    if result["status"] not in {"ok", "warn", "fail"}:
        print(f"[FAIL] {label} — invalid status value: {result['status']}")
        failed = True
        return False
    return True

# --- Module 1 ---
try:
    bootstrap = m13.create_session({
        "title": "verify_bootstrap", "mood": "dark", "energy": "driving",
        "tempo_bpm": 128, "target_duration_seconds": 180
    })
    bootstrap_id = bootstrap["session_id"]
    result_m1 = m1.run(bootstrap_id)
    check_envelope(result_m1, "module1.run")
    cp = result_m1["data"]
    assert "entropy" in cp
    assert cp["entropy"]["dimension"] in m1.VALID_ENTROPY_DIMENSIONS
    print(f"[OK] module1.run — entropy: {cp['entropy']['dimension']}, intensity: {cp['entropy']['intensity']}")
except Exception as e:
    print(f"[FAIL] module1.run — {e}")
    failed = True

# --- Module 2 scheduled ---
try:
    dummy_cp = {
        "preferred_moods": ["dark"], "preferred_energies": ["driving"],
        "entropy": {"dimension": "rhythm", "intensity": 0.5, "seed": 1234},
        "library_size": 0, "gap_report": {}
    }
    result_m2 = m2.run(dummy_cp, mode="scheduled")
    check_envelope(result_m2, "module2.run scheduled")
    test_session_id = result_m2["data"]["session_id"]
    assert "blueprint" in result_m2["data"]
    print(f"[OK] module2.run scheduled — session: {test_session_id}")
except Exception as e:
    print(f"[FAIL] module2.run — {e}")
    failed = True
    sys.exit(1)

# --- Module 2 manual ---
try:
    result_m2_manual = m2.run(
        dummy_cp, mode="manual",
        manual_params={"title": "Test", "mood": "dark", "energy": "driving",
                       "genre": "trap", "tempo_bpm": 128, "target_duration_seconds": 180}
    )
    check_envelope(result_m2_manual, "module2.run manual")
    assert result_m2_manual["data"]["blueprint"]["mode"] == "manual"
    print("[OK] module2.run manual mode")
except Exception as e:
    print(f"[FAIL] module2.run manual — {e}")
    failed = True

# --- Module 2 invalid mood rejection ---
try:
    m2.generate_blueprint_manual("T", "invalid_mood", "driving", "trap", 120)
    print("[FAIL] module2 — invalid mood not rejected")
    failed = True
except ValueError:
    print("[OK] module2 — invalid mood correctly rejected")

# --- Module 3 ---
try:
    result_m3 = m3.run(test_session_id, dummy_cp)
    check_envelope(result_m3, "module3.run")
    blueprints = result_m3["data"]["blueprints"]
    assert len(blueprints) == len(m3.STEM_CATALOG)
    print(f"[OK] module3.run — {len(blueprints)} blueprints generated")
except Exception as e:
    print(f"[FAIL] module3.run — {e}")
    failed = True
    blueprints = []

try:
    if blueprints:
        for bp in blueprints:
            assert "schema_version" in bp
            assert bp["output_specification"]["peak_level_db"] == -1
            assert "entropy_injection" in bp
        print("[OK] module3 — all blueprints contain required fields")
        priorities = [bp["execution_priority"] for bp in blueprints]
        assert priorities == sorted(priorities)
        print("[OK] module3 — blueprints ordered by priority")
        if len(blueprints) > 1:
            assert blueprints[1]["global_context"] != ""
            print("[OK] module3 — global context populated for non-first stems")
except Exception as e:
    print(f"[FAIL] module3 blueprint checks — {e}")
    failed = True

# --- Module 3.5 ---
try:
    if blueprints:
        result_m35 = m35.run(test_session_id, blueprints)
        check_envelope(result_m35, "module3_5.run")
        if result_m35["data"]["queue_valid"]:
            print(f"[OK] module3_5.run — queue valid")
        else:
            print(f"[WARN] module3_5 — issues: {result_m35['errors']}")
except Exception as e:
    print(f"[FAIL] module3_5.run — {e}")
    failed = True

try:
    bad_bp = {"stem_id": "test", "schema_version": "1.0"}
    bad_result = m35.validate_blueprint(bad_bp, 0, 1)
    assert not bad_result["valid"]
    print("[OK] module3_5 — incomplete blueprint correctly flagged")
except Exception as e:
    print(f"[FAIL] module3_5 bad blueprint — {e}")
    failed = True

# --- Auto-fix determinism test ---
try:
    bp_missing_peak = dict(blueprints[0]) if blueprints else {}
    if bp_missing_peak:
        bp_missing_peak["output_specification"] = {k: v for k, v in bp_missing_peak["output_specification"].items() if k != "peak_level_db"}
        fix_result = m35.validate_blueprint(bp_missing_peak, 0, 1)
        assert "peak_level_db" in str(fix_result["fixes"]) or fix_result["valid"]
        print("[OK] module3_5 — absent peak_level_db triggers auto-fix")
    bp_wrong_peak = dict(blueprints[0]) if blueprints else {}
    if bp_wrong_peak:
        bp_wrong_peak["output_specification"] = {**bp_wrong_peak["output_specification"], "peak_level_db": -6}
        wrong_result = m35.validate_blueprint(bp_wrong_peak, 0, 1)
        assert not wrong_result["valid"], "Wrong peak_level_db should be a blocking issue"
        print("[OK] module3_5 — wrong peak_level_db escalates as blocking issue (not auto-fixed)")
except Exception as e:
    print(f"[FAIL] module3_5 auto-fix determinism — {e}")
    failed = True

# --- Module 5 ---
try:
    result_m5 = m5.run(test_session_id, str(AUDIO_FILE), "drums")
    check_envelope(result_m5, "module5.run")
    assert "asset_id" in result_m5["data"]
    print(f"[OK] module5.run — asset: {result_m5['data']['asset_id']}, flags: {result_m5['flags'] or 'none'}")
except Exception as e:
    print(f"[FAIL] module5.run — {e}")
    failed = True

try:
    m5.run(test_session_id, "/nonexistent/stem.wav", "bass")
    print("[FAIL] module5 — missing file not caught")
    failed = True
except FileNotFoundError:
    print("[OK] module5 — missing file raises FileNotFoundError")

# --- Module 6 ---
try:
    result_m6 = m6.run(test_session_id, str(AUDIO_FILE), "drums")
    check_envelope(result_m6, "module6.run")
    assert "output_path" in result_m6["data"]
    # Verify low_stability_source_used flag surfaces in envelope when fallback triggers
    if "low_stability_source_used" in result_m6["flags"]:
        print(f"[OK] module6.run — fallback triggered and structurally tracked in envelope flags")
    else:
        print(f"[OK] module6.run — stable regions found, output: {result_m6['data']['output_path']}")
except Exception as e:
    print(f"[FAIL] module6.run — {e}")
    failed = True

# --- Module 7 — read-only contract ---
try:
    session_before = m13.load_session(test_session_id)
    all_none = all(v is None for v in session_before["identity_vector"].values())
    result_m7 = m7.run(test_session_id, str(AUDIO_FILE), "drums")
    check_envelope(result_m7, "module7.run")
    session_after = m13.load_session(test_session_id)
    if all_none:
        still_all_none = all(v is None for v in session_after["identity_vector"].values())
        assert still_all_none, "Module 7 mutated identity vector — violates read-only contract"
        assert result_m7["status"] == "warn"
        assert "identity_vector_missing" in result_m7["flags"]
        print("[OK] module7 — read-only contract enforced: unpopulated vector returns warn, not initialized")
    else:
        print(f"[OK] module7.run — comparison complete, flags: {result_m7['flags'] or 'none'}")
except Exception as e:
    print(f"[FAIL] module7.run — {e}")
    failed = True

# --- Module 7 — populated vector test ---
try:
    import backend.identity_vector as iv
    iv.write_identity_vector_to_session(test_session_id, str(AUDIO_FILE), mood="dark")
    result_m7_populated = m7.run(test_session_id, str(AUDIO_FILE), "drums")
    check_envelope(result_m7_populated, "module7.run populated")
    assert result_m7_populated["data"].get("skipped") is not True
    print(f"[OK] module7.run populated vector — flags: {result_m7_populated['flags'] or 'none'}")
except Exception as e:
    print(f"[FAIL] module7.run populated — {e}")
    failed = True

# --- Module 8 ---
try:
    result_m8 = m8.run(test_session_id, str(AUDIO_FILE), "drums")
    check_envelope(result_m8, "module8.run")
    assert "library_empty" in result_m8["data"]
    print(f"[OK] module8.run — library_empty: {result_m8['data']['library_empty']}, flags: {result_m8['flags'] or 'none'}")
except Exception as e:
    print(f"[FAIL] module8.run — {e}")
    failed = True

# --- Gemini renderer isolated (no API call) ---
try:
    if blueprints:
        render_str = gr._build_render_prompt(blueprints[0])
        assert isinstance(render_str, str) and len(render_str) > 50
        print("[OK] gemini_renderer._build_render_prompt — builds string without API call")
except Exception as e:
    print(f"[FAIL] gemini_renderer — {e}")
    failed = True

# --- Full checkpoint audit ---
try:
    final_session = m13.load_session(test_session_id)
    checkpoint_modules = {cp["module"] for cp in final_session["checkpoints"]}
    expected = {"module13", "module1", "module2", "module3", "module3_5", "module5", "module6", "module7", "module8"}
    missing_cp = expected - checkpoint_modules
    if missing_cp:
        print(f"[FAIL] Missing checkpoints: {missing_cp}")
        failed = True
    else:
        print(f"[OK] All module checkpoints present — {len(final_session['checkpoints'])} total")
    reconcile = m13.reconcile_state(test_session_id)
    assert reconcile["valid"], f"Reconciliation failed: {reconcile['issues']}"
    print("[OK] reconcile_state — session valid after full pipeline")
except Exception as e:
    print(f"[FAIL] Session audit — {e}")
    failed = True

print()
if failed:
    print("[FAIL] Phase 3 verification failed — fix all failures before proceeding")
    sys.exit(1)
else:
    print("[OK] Phase 3 verification complete — all checks passed")
    sys.exit(0)
```

Run it:

```bash
python3.11 backend/verify_phase3.py
```

Then run the assumption test:

```bash
python3.11 backend/test_prompt_variation.py
```

---

## Task 12 — Integration Test: Generation Readiness

Create `/backend/integration_test_phase3.py`:

```python
"""
MCS Phase 3 — Integration Test: Generation Readiness

Verifies the full State 1 pipeline from Module 1 through Module 8.

ASSUMPTION: This test assumes backend.module13 is correct and verified
(completed in Phase 1). If Module 13 has a latent bug, failures in this
test may be false positives — the cascade originates from Module 13, not
the module reporting the failure. If unexpected failures occur across
multiple unrelated modules, verify Module 13 independently first.

MANUAL HANDOFF POINT (not tested here):
  1. Copy rendered prompts from output of this script
  2. Paste into Gemini App manually
  3. Drop generated stems into data/audio/
  4. Run Module 5 on each received stem file

This script verifies generation readiness only — not generation outcome.
"""

from pathlib import Path
import sys

ROOT = Path(__file__).resolve().parents[1]
AUDIO_FILE = ROOT / "data" / "audio" / "heavy_gauge.mp3"

failed = False

print("=" * 60)
print("MCS Phase 3 — Integration Test: Generation Readiness")
print("=" * 60)
print()

if not AUDIO_FILE.exists():
    print("[FAIL] heavy_gauge.mp3 not found")
    sys.exit(1)

import backend.module1 as m1
import backend.module2 as m2
import backend.module3 as m3
import backend.module3_5 as m35
import backend.module5 as m5
import backend.module6 as m6
import backend.module7 as m7
import backend.module8 as m8
import backend.module13 as m13
import backend.identity_vector as iv
from backend.gemini_renderer import _build_render_prompt

def check_stage(result, stage_name):
    global failed
    if result.get("status") == "fail":
        print(f"[FAIL] {stage_name} — errors: {result['errors']}")
        failed = True
        return False
    return True

print("--- Stage 1: Constraint Generation (Module 1) ---")
try:
    bootstrap = m13.create_session({
        "title": "integration_test_bootstrap", "mood": "dark",
        "energy": "driving", "tempo_bpm": 128, "target_duration_seconds": 180
    })
    result_m1 = m1.run(bootstrap["session_id"])
    check_stage(result_m1, "Module 1")
    cp = result_m1["data"]
    print(f"  Library size: {cp['library_size']}")
    print(f"  Preferred moods: {cp['preferred_moods']}")
    print(f"  Entropy: {cp['entropy']}")
    print()
except Exception as e:
    print(f"[FAIL] Stage 1 — {e}")
    failed = True

print("--- Stage 2: Song Planning (Module 2) ---")
try:
    result_m2 = m2.run(cp, mode="scheduled")
    check_stage(result_m2, "Module 2")
    session_id = result_m2["data"]["session_id"]
    blueprint = result_m2["data"]["blueprint"]
    print(f"  Session ID: {session_id}")
    print(f"  Mood: {blueprint['mood']} | Energy: {blueprint['energy']} | BPM: {blueprint['tempo_bpm']}")
    print()
except Exception as e:
    print(f"[FAIL] Stage 2 — {e}")
    failed = True
    sys.exit(1)

print("--- Stage 2b: Identity Vector Initialization ---")
print("  Note: initializing identity vector now so Module 7 can compare")
try:
    iv.write_identity_vector_to_session(session_id, str(AUDIO_FILE), mood=blueprint["mood"])
    loaded = m13.load_session(session_id)
    assert any(v is not None for v in loaded["identity_vector"].values())
    print("  Identity vector initialized")
    print()
except Exception as e:
    print(f"[WARN] Stage 2b — identity vector init failed: {e} — Module 7 will return warn")
    print()

print("--- Stage 3: Prompt Generation (Module 3) ---")
try:
    result_m3 = m3.run(session_id, cp)
    check_stage(result_m3, "Module 3")
    blueprints = result_m3["data"]["blueprints"]
    print(f"  {len(blueprints)} blueprints generated")
    for bp in blueprints:
        print(f"  [{bp['execution_priority']}] {bp['stem_id']} — constraints: {bp['prompt_metadata']['constraint_density_score']}")
    print()
except Exception as e:
    print(f"[FAIL] Stage 3 — {e}")
    failed = True

print("--- Stage 4: Validation Gate (Module 3.5) ---")
try:
    result_m35 = m35.run(session_id, blueprints)
    check_stage(result_m35, "Module 3.5")
    if result_m35["data"]["queue_valid"]:
        print(f"  All {result_m35['data']['total_stems']} blueprints passed validation")
    else:
        print(f"  Blocking issues: {result_m35['errors']}")
        failed = True
    print()
except Exception as e:
    print(f"[FAIL] Stage 4 — {e}")
    failed = True

print("--- Stage 5: Prompt Readiness (Gemini Renderer — no API call) ---")
try:
    for bp in blueprints:
        instr = _build_render_prompt(bp)
        assert len(instr) > 100
    print(f"  {len(blueprints)} render instructions built")
    print()
except Exception as e:
    print(f"[FAIL] Stage 5 — {e}")
    failed = True

print("--- Stage 6: Stem Intake + Analysis (Modules 5–8 on analysis anchor) ---")
print("  Note: using heavy_gauge.mp3 as analysis anchor — not a real generated stem")

for mod_name, run_fn, stem_id in [
    ("Module 5", m5.run, "drums"),
    ("Module 6", m6.run, "drums"),
    ("Module 7", m7.run, "drums"),
    ("Module 8", m8.run, "drums"),
]:
    try:
        result = run_fn(session_id, str(AUDIO_FILE), stem_id)
        status_str = f"status={result['status']}, flags={result['flags'] or 'none'}"
        print(f"  {mod_name} — {status_str}")
    except Exception as e:
        print(f"[FAIL] {mod_name} — {e}")
        failed = True

print()
print("--- Stage 7: Session State Audit ---")
try:
    final = m13.load_session(session_id)
    print(f"  Status: {final['status']}")
    print(f"  Checkpoints: {len(final['checkpoints'])}")
    print(f"  Assets: {len(final['assets'])}")
    print(f"  Prompt metadata entries: {len(final['prompt_metadata'])}")
    reconcile = m13.reconcile_state(session_id)
    assert reconcile["valid"], f"Reconciliation issues: {reconcile['issues']}"
    print("  Reconciliation: PASSED")
    print()
except Exception as e:
    print(f"[FAIL] Stage 7 — {e}")
    failed = True

print("=" * 60)
if failed:
    print("[FAIL] Integration test failed — fix all failures before proceeding to Phase 4")
    sys.exit(1)
else:
    print("[OK] Integration test passed — State 1 pipeline generation-ready")
    print()
    print("MANUAL HANDOFF POINT:")
    print("  Validated prompt blueprints are ready.")
    print("  Next: render via gemini_renderer.render_queue(blueprints)")
    print("  Paste rendered prompts into Gemini App to generate audio stems.")
    print("  Once stems received: run Module 5 on each stem file.")
    sys.exit(0)
```

Run it:

```bash
python3.11 backend/integration_test_phase3.py
```

---

## Completion Gate

Phase 3 is complete when ALL of the following are true:

1. `verify_phase3.py` exits code 0 — all `[OK]`
2. `integration_test_phase3.py` exits code 0 — all stages pass
3. A real session JSON from the integration test exists in `/data/sessions/` with checkpoints from all eight modules
4. `reconcile_state` returns `valid: True` on the integration test session
5. All blueprints from Module 3 pass Module 3.5 validation without manual fixes
6. Gemini renderer builds instruction strings without API key — correctly isolated
7. `test_prompt_variation.py` result documented
8. No new dependencies added beyond `requirements.txt`
9. No auto-rejection in any module — all flags advisory only
10. Module 7 read-only contract verified — unpopulated vector returns warn without mutating session
11. All modules return the standard failure envelope shape
12. All threshold constants sourced from `backend.config` — none hardcoded in module files
13. Module 6 fallback (`low_stability_source_used`) surfaces in envelope flags, not stdout only

---

## What You Must Not Do

- Do not make Gemini API calls inside any module except `gemini_renderer.py`
- Do not auto-reject stems in Modules 5, 7, or 8 — flags are advisory only
- Do not initialize or mutate the identity vector inside Module 7 — read-only
- Do not define threshold constants locally in any module — import from `backend.config`
- Do not hardcode absolute paths — always derive from `ROOT = Path(__file__).resolve().parents[1]`
- Do not pass numeric literals to librosa — always use `SAMPLE_RATE` and `HOP_LENGTH` from `audio_analysis.py`
- Do not use scalar averages in cosine comparisons — curve shape only via `backend.similarity`
- Do not mutate session dicts directly — all mutations through Module 13 public functions
- Do not add dependencies beyond `requirements.txt`
- Do not continue past any failed task — stop and print diagnostic
- Do not simulate or mock Gemini audio output in the integration test
- Do not call `module13._save_session` outside of `identity_vector.py` where it is explicitly permitted
- Do not implement per-stem entropy adaptation — this is a v3.0 deferred item
- Do not extend the 8-field prompt schema — multi-section schemas are a v3.0 deferred item
- Do not silently auto-fix a field that is present but incorrect — escalate as blocking issue
- Do not restructure Module 3's active_stems ordering — the v3.0 deferred note is documentation only
- Do not switch Module 8 to top-k or percentile aggregation — max similarity is correct for Phase 3 library size