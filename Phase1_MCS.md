# MCS Phase 1 — Data Layer Agent Prompt (v6 Final)

## Context

You are building the data layer for MCS (Music Creator Studio). This is Phase 1. Phase 0 (environment setup, folder structure, dependencies, Drive authentication) is already complete and verified.

Phase 1 builds three things:
- The session JSON schema — single source of truth for all pipeline state
- Module 13 (Human Decision Logger + State Authority) — core library with minimal CLI
- The checkpoint system — standardized write/read/verify pattern all future modules will use

**No audio logic. No other modules. No frontend. Data layer only.**

**Stop on first failure. Print `[OK]` / `[FAIL]` diagnostics. Do not continue to the next task.**

---

## Architectural Rule (Read Before Writing Any Code)

> All session mutations must occur only through Module 13 public functions. Direct modification of session dictionaries outside Module 13 is forbidden. This rule applies to all future modules — no module may read a session dict and write back to it directly.

This is the core integrity contract of the entire system. It must be enforced from Phase 1 forward.

---

## Preconditions

Confirm Phase 0 is complete before starting:

1. Folder structure exists: `/backend`, `/frontend`, `/data`, `/data/sessions`, `/temp`, `/docs`
2. `requirements.txt` is present and dependencies are installed
3. `backend/__init__.py` exists

If any are missing, print `[FAIL] Phase 0 incomplete — <what is missing>` and stop.

---

## Task 1 — Session JSON Schema

Create `/data/session_schema.json`. This file serves two purposes:

1. **Key contract** — the authoritative definition of all required session fields. Module 13 loads this file at runtime to extract required top-level keys and validates their presence in every session object.
2. **Human-readable reference** — value strings are annotations only and are never machine-parsed. Only the keys carry enforcement weight.

```json
{
  "schema_version": "1.0",
  "session_id": "string — UUID4",
  "status": "INITIALIZED | ACTIVE | BLOCKED | COMPLETE",
  "created_at": "ISO 8601 UTC timestamp",
  "updated_at": "ISO 8601 UTC timestamp",
  "song_plan": {
    "title": "string",
    "mood": "dark | bright | melancholic | tense | euphoric | neutral",
    "energy": "sparse | layered | dense | driving | ambient",
    "tempo_bpm": "integer",
    "target_duration_seconds": "integer — target 180, warn below 180, warn above 300",
    "harmonic_center": "string",
    "notes": "string"
  },
  "identity_vector": {
    "mood": null,
    "energy_trajectory": null,
    "density_trajectory": null,
    "harmonic_center": null,
    "energy_slope": null,
    "peak_timing": null,
    "density_volatility": null
  },
  "entropy_log": [],
  "checkpoints": [],
  "decisions": [],
  "assets": [],
  "prompt_metadata": [],
  "duration_warnings": []
}
```

**Required top-level fields** — Module 13 loads these from the schema file's top-level keys at runtime and validates all are present in every session:
`schema_version`, `session_id`, `status`, `created_at`, `updated_at`, `song_plan`, `identity_vector`, `entropy_log`, `checkpoints`, `decisions`, `assets`, `prompt_metadata`, `duration_warnings`

**Required `song_plan` subfields** — hardcoded in `REQUIRED_SONG_PLAN_KEYS`, validated in `create_session`, `load_session`, and `reconcile_state`:
`title`, `mood`, `energy`, `tempo_bpm`, `target_duration_seconds`

---

## Task 2 — Module 13 Core Library

Create `/backend/module13.py`.

This module is the single state authority. All other modules will import and call it. It must never be bypassed.

### Constants

Define these once at the top of the file. Never hardcode these values anywhere else in the module:

```python
from pathlib import Path
from datetime import datetime, timezone
import json, uuid, hashlib, sys, argparse

SUPPORTED_SCHEMA_VERSION = "1.0"
SESSIONS_DIR = Path(__file__).resolve().parents[1] / "data" / "sessions"
SCHEMA_FILE = Path(__file__).resolve().parents[1] / "data" / "session_schema.json"
DURATION_TARGET_SECONDS = 180
DURATION_WARN_MAX_SECONDS = 300

VALID_STATUSES = {"INITIALIZED", "ACTIVE", "BLOCKED", "COMPLETE"}
VALID_DECISION_TYPES = {"candidate_review", "refinement", "override"}
VALID_MOODS = {"dark", "bright", "melancholic", "tense", "euphoric", "neutral"}
VALID_ENERGIES = {"sparse", "layered", "dense", "driving", "ambient"}
REQUIRED_SONG_PLAN_KEYS = {"title", "mood", "energy", "tempo_bpm", "target_duration_seconds"}

# Valid state transitions — only these moves are permitted
# Key = current status, Value = set of allowed next statuses
VALID_TRANSITIONS = {
    "INITIALIZED": {"ACTIVE", "BLOCKED"},
    "ACTIVE":      {"BLOCKED", "COMPLETE"},
    "BLOCKED":     {"ACTIVE"},
    "COMPLETE":    set()
}
```

---

### Internal Helpers

#### `_now() -> str`
Returns current UTC time as ISO 8601 string:
```python
def _now() -> str:
    return datetime.now(timezone.utc).isoformat()
```

#### `_validate_uuid(value: str, label: str) -> None`
Validates that `value` is a valid UUID. Raises `ValueError` with clear message if not:
```python
def _validate_uuid(value: str, label: str) -> None:
    try:
        uuid.UUID(value)
    except ValueError:
        raise ValueError(f"Invalid UUID for {label}: {value}")
```

#### `_load_schema_keys() -> list`
Loads `SCHEMA_FILE` and returns its top-level keys. Keys are the authoritative contract — value strings are never parsed:
```python
def _load_schema_keys() -> list:
    with open(SCHEMA_FILE) as f:
        return list(json.load(f).keys())
```

#### `_validate_session_fields(session: dict) -> None`
Calls `_load_schema_keys()` and checks all returned keys are present in `session`. Raises `ValueError` listing any missing fields.

#### `_validate_song_plan(song_plan: dict) -> None`
Checks all keys in `REQUIRED_SONG_PLAN_KEYS` are present in `song_plan`. Raises `ValueError` listing any missing keys. This is the single shared function called by `create_session`, `load_session`, and `reconcile_state` — never duplicate this logic inline.

#### `_save_session(session: dict) -> None`
- Updates `updated_at` to `_now()`
- Writes atomically: write full JSON to `SESSIONS_DIR/<session_id>.tmp`, then rename to `<session_id>.json`
- This is an internal function — never call from outside the module

#### `_load_session_raw(session_id: str) -> dict`
- Validates `session_id` is a valid UUID via `_validate_uuid`
- Loads JSON from `SESSIONS_DIR/<session_id>.json`
- Raises `FileNotFoundError` with clear message if not found
- Returns raw dict without schema validation (used internally only)

---

### Public Functions

#### `create_session(song_plan: dict) -> dict`

- Runs `_validate_song_plan(song_plan)`
- Validates `song_plan["mood"]` is in `VALID_MOODS`
- Validates `song_plan["energy"]` is in `VALID_ENERGIES`
- Generates UUID4 session ID
- Initializes ALL fields explicitly — no field may be added implicitly later:

```python
session = {
    "schema_version": SUPPORTED_SCHEMA_VERSION,
    "session_id": str(uuid.uuid4()),
    "status": "INITIALIZED",
    "created_at": _now(),
    "updated_at": _now(),
    "song_plan": song_plan,
    "identity_vector": {
        "mood": None,
        "energy_trajectory": None,
        "density_trajectory": None,
        "harmonic_center": None,
        "energy_slope": None,
        "peak_timing": None,
        "density_volatility": None
    },
    "entropy_log": [],
    "checkpoints": [],
    "decisions": [],
    "assets": [],
    "prompt_metadata": [],
    "duration_warnings": []
}
```

- Runs duration check on `song_plan["target_duration_seconds"]`:
  - Below `DURATION_TARGET_SECONDS`: append warning to `duration_warnings`, print `[WARN] Duration below 180s target`
  - Above `DURATION_WARN_MAX_SECONDS`: append warning to `duration_warnings`, print `[WARN] Duration above 300s ideal range`
  - Never block creation regardless of duration value
- Writes initial checkpoint: `module="module13"`, `data={"event": "session_created"}`
- Saves session file via `_save_session`
- Prints `[OK] Session created — <session_id>`
- Returns full session dict

---

#### `load_session(session_id: str) -> dict`

- Validates `session_id` is a valid UUID
- Loads session from file via `_load_session_raw`
- Validates `schema_version` matches `SUPPORTED_SCHEMA_VERSION` — if mismatch raise `ValueError(f"Schema version mismatch: expected {SUPPORTED_SCHEMA_VERSION}, got <actual>")`
- Runs `_validate_session_fields` — all required top-level fields must exist
- Runs `_validate_song_plan(session["song_plan"])` — all required subfields must exist
- Returns full session dict

---

#### `write_checkpoint(session_id: str, module: str, data: dict) -> None`

- Loads session via `load_session`
- Appends to `checkpoints`:
```python
{
    "module": module,
    "timestamp": _now(),
    "data": data
}
```
- Saves session via `_save_session`
- Prints `[OK] Checkpoint written — module: <module>`

---

#### `log_decision(session_id: str, decision_type: str, payload: dict) -> None`

- Validates `decision_type` is in `VALID_DECISION_TYPES` — if not raise `ValueError` listing valid types
- Loads session
- Appends to `decisions`:
```python
{
    "type": decision_type,
    "timestamp": _now(),
    "payload": payload
}
```
- Saves session
- Prints `[OK] Decision logged — type: <decision_type>`

---

#### `update_status(session_id: str, new_status: str) -> None`

- Validates `new_status` is in `VALID_STATUSES` — if not raise `ValueError` listing valid statuses
- Loads session
- Reads `current_status = session["status"]`
- Checks transition is allowed via `VALID_TRANSITIONS`:
  - If `new_status` not in `VALID_TRANSITIONS[current_status]`: raise `ValueError(f"Invalid transition: {current_status} → {new_status}. Allowed: {VALID_TRANSITIONS[current_status]}")`
- Updates `status` field to `new_status`
- Writes checkpoint: `module="module13"`, `data={"event": "status_updated", "from": current_status, "to": new_status}`
- Saves session
- Prints `[OK] Status updated — <current_status> → <new_status>`

---

#### `register_asset(session_id: str, filename: str, filepath: str) -> str`

- Checks file exists at `filepath` — if not print `[FAIL] File not found — <filepath>` and raise `FileNotFoundError`
- Computes SHA256 hash of file
- Generates UUID4 asset ID
- Loads session
- Appends to `assets`:
```python
{
    "id": asset_id,
    "filename": filename,
    "sha256": sha256_hash,
    "status": "pending",
    "uploaded_at": _now()
}
```
- Saves session
- Prints `[OK] Asset registered — <filename> — SHA256: <hash>`
- Returns asset ID

---

#### `verify_asset(session_id: str, asset_id: str, filepath: str) -> bool`

- Validates `asset_id` is a valid UUID via `_validate_uuid`
- Checks file exists at `filepath` — if not print `[FAIL] File not found — <filepath>` and return `False`
- Loads session
- Finds asset by `asset_id` — if not found print `[FAIL] Asset not found — <asset_id>` and return `False`
- Recomputes SHA256
- If matches: update asset `status` to `"verified"`, save session, print `[OK] Asset verified — <filename>`, return `True`
- If mismatch: print `[FAIL] Hash mismatch — <filename>`, return `False`

---

#### `log_prompt_metadata(session_id: str, stem_id: str, structure_version: str, entropy_config: dict, descriptor_set: dict, constraint_density_score: float) -> None`

- Loads session
- Appends to `prompt_metadata`:
```python
{
    "stem_id": stem_id,
    "structure_version": structure_version,
    "entropy_config": entropy_config,
    "descriptor_set": descriptor_set,
    "constraint_density_score": constraint_density_score,
    "timestamp": _now()
}
```
- Saves session
- Prints `[OK] Prompt metadata logged — stem: <stem_id>`

---

#### `reconcile_state(session_id: str) -> dict`

> **Note:** `reconcile_state` is a full integrity audit — it collects ALL issues without stopping. This is intentionally different from `load_session`, which fails fast on the first error. Do not optimize away the overlapping checks — they serve a different purpose here.

- Validates `session_id` is a valid UUID via `_validate_uuid`
- Derives file path internally: `filepath = SESSIONS_DIR / f"{session_id}.json"`
- Loads raw session from file
- Collects all issues — does not stop on first failure

Run all of the following checks in order:

1. `session["session_id"]` matches `session_id` — if not add issue: `"session_id inside JSON does not match expected session_id"`
2. All required top-level fields exist — call `_validate_session_fields`, catch and record any `ValueError`
3. `song_plan` contains all required subfields — call `_validate_song_plan`, catch and record any `ValueError`
4. `schema_version` matches `SUPPORTED_SCHEMA_VERSION`
5. `status` is in `VALID_STATUSES`
6. `created_at` <= `updated_at` — parse both with `datetime.fromisoformat()` for comparison
7. All list fields are actually of type `list`: `entropy_log`, `checkpoints`, `decisions`, `assets`, `prompt_metadata`, `duration_warnings`
8. `checkpoints` is non-empty
9. No duplicate asset IDs in `assets`
10. All assets with `status="verified"` have non-empty `sha256`

Returns:
```python
{"valid": True, "issues": []}
# or
{"valid": False, "issues": ["issue description 1", "issue description 2"]}
```

Prints `[OK] State reconciliation passed` or `[FAIL] State reconciliation failed — <issues>`

---

## Task 3 — Module 13 CLI Interface

At the bottom of `/backend/module13.py`, add a `__main__` block using `argparse`. This CLI is for testing only — not part of system UX.

**Global CLI rules:**
- Wrap the entire execution of every command in a top-level `try/except`. On any unhandled error, print `[FAIL] <error message>` and exit with code 1.
- Every command that receives `--id` must call `_validate_uuid(args.id, "session_id")` before any file access. If invalid, print `[FAIL]` and exit with code 1.
- For any command that accepts `--data` or `--payload`, wrap JSON parsing explicitly:

```python
try:
    data = json.loads(args.data)
except (json.JSONDecodeError, TypeError):
    print("[FAIL] Invalid JSON for --data. Ensure it is a valid single-line JSON string.")
    sys.exit(1)
```

**`create-session` argument mapping:**

The CLI must build the `song_plan` dict internally from individual arguments before passing to `create_session`. The mapping is explicit:

```
--title       → song_plan["title"]
--mood        → song_plan["mood"]
--energy      → song_plan["energy"]
--tempo       → song_plan["tempo_bpm"]                  (cast to int)
--duration    → song_plan["target_duration_seconds"]     (cast to int)
--harmonic    → song_plan["harmonic_center"]             (optional, default "")
--notes       → song_plan["notes"]                      (optional, default "")
```

Full command set:

```bash
python -m backend.module13 create-session --title "Test Song" --mood dark --energy driving --tempo 120 --duration 180

python -m backend.module13 load-session --id <session_id>

python -m backend.module13 write-checkpoint --id <session_id> --module "test_module" --data '{"key": "value"}'

python -m backend.module13 log-decision --id <session_id> --type candidate_review --payload '{"score": 8}'

python -m backend.module13 update-status --id <session_id> --status ACTIVE

python -m backend.module13 register-asset --id <session_id> --filename "stem_bass.wav" --filepath "/path/to/file"

python -m backend.module13 verify-asset --id <session_id> --asset-id <asset_id> --filepath "/path/to/file"

python -m backend.module13 reconcile-state --id <session_id>
```

---

## Task 4 — Verification Script

Create `/backend/verify_phase1.py`:

```python
from pathlib import Path
import sys
import json

ROOT = Path(__file__).resolve().parents[1]
failed = False

# Check schema file exists and is valid JSON
schema_path = ROOT / "data" / "session_schema.json"
try:
    with open(schema_path) as f:
        schema = json.load(f)
    print("[OK] session_schema.json — valid JSON")
except Exception as e:
    print(f"[FAIL] session_schema.json — {e}")
    failed = True

# Check module13 imports cleanly
try:
    import backend.module13 as m13
    print("[OK] module13 — imported successfully")
except Exception as e:
    print(f"[FAIL] module13 — {e}")
    failed = True
    sys.exit(1)

try:
    # Create session
    session = m13.create_session({
        "title": "Phase1 Test",
        "mood": "dark",
        "energy": "driving",
        "tempo_bpm": 120,
        "target_duration_seconds": 180
    })
    sid = session["session_id"]
    print(f"[OK] create_session — session_id: {sid}")

    # Verify all required top-level fields
    required = [
        "schema_version", "session_id", "status", "created_at", "updated_at",
        "song_plan", "identity_vector", "entropy_log", "checkpoints",
        "decisions", "assets", "prompt_metadata", "duration_warnings"
    ]
    for field in required:
        assert field in session, f"Missing field: {field}"
    print("[OK] all required top-level fields present")

    # Verify identity_vector structure
    iv_keys = ["mood", "energy_trajectory", "density_trajectory",
               "harmonic_center", "energy_slope", "peak_timing", "density_volatility"]
    for key in iv_keys:
        assert key in session["identity_vector"], f"Missing identity_vector key: {key}"
    print("[OK] identity_vector initialized with correct structure")

    # Write checkpoint
    m13.write_checkpoint(sid, "verify_phase1", {"test": True})
    print("[OK] write_checkpoint")

    # Log decision
    m13.log_decision(sid, "candidate_review", {"score": 9, "approved": True})
    print("[OK] log_decision")

    # Valid transition INITIALIZED → ACTIVE
    m13.update_status(sid, "ACTIVE")
    print("[OK] update_status INITIALIZED → ACTIVE")

    # Invalid transition ACTIVE → INITIALIZED must be rejected
    try:
        m13.update_status(sid, "INITIALIZED")
        print("[FAIL] invalid transition was not rejected")
        failed = True
    except ValueError:
        print("[OK] invalid state transition correctly rejected")

    # Load and verify schema version
    loaded = m13.load_session(sid)
    assert loaded["schema_version"] == m13.SUPPORTED_SCHEMA_VERSION
    print("[OK] load_session — schema version match")

    # Reconcile state
    result = m13.reconcile_state(sid)
    assert result["valid"], f"Reconciliation failed: {result['issues']}"
    print("[OK] reconcile_state")

    # session_id inside JSON matches expected session_id
    assert loaded["session_id"] == sid
    print("[OK] session_id inside JSON matches expected session_id")

    # Duration warning — below minimum
    session_short = m13.create_session({
        "title": "Short Song",
        "mood": "bright",
        "energy": "sparse",
        "tempo_bpm": 90,
        "target_duration_seconds": 120
    })
    loaded_short = m13.load_session(session_short["session_id"])
    assert len(loaded_short["duration_warnings"]) > 0
    print("[OK] duration warning logged for short song")

    # Duration warning — above maximum
    session_long = m13.create_session({
        "title": "Long Song",
        "mood": "neutral",
        "energy": "ambient",
        "tempo_bpm": 80,
        "target_duration_seconds": 400
    })
    loaded_long = m13.load_session(session_long["session_id"])
    assert len(loaded_long["duration_warnings"]) > 0
    print("[OK] duration warning logged for long song")

    # Invalid decision type rejection
    try:
        m13.log_decision(sid, "invalid_type", {})
        print("[FAIL] invalid decision type was not rejected")
        failed = True
    except ValueError:
        print("[OK] invalid decision type correctly rejected")

    # Invalid status rejection
    try:
        m13.update_status(sid, "INVALID_STATUS")
        print("[FAIL] invalid status was not rejected")
        failed = True
    except ValueError:
        print("[OK] invalid status correctly rejected")

    # Invalid UUID rejection
    try:
        m13.load_session("not-a-uuid")
        print("[FAIL] invalid UUID was not rejected")
        failed = True
    except (ValueError, FileNotFoundError):
        print("[OK] invalid UUID correctly rejected")

except Exception as e:
    print(f"[FAIL] lifecycle test — {e}")
    failed = True

sys.exit(1 if failed else 0)
```

Run it:

```bash
python3.11 backend/verify_phase1.py
```

All lines must print `[OK]`. Any `[FAIL]` means stop and diagnose before proceeding.

---

## Completion Gate

Phase 1 is complete when ALL of the following are true:

1. `verify_phase1.py` exits code 0 — all `[OK]`
2. A real session JSON file exists in `/data/sessions/` from the lifecycle test
3. That session file contains all required fields with correct structure including fully initialized `identity_vector`
4. All CLI commands run without error on a test session
5. `reconcile_state` returns `valid: True` on the test session
6. `session["session_id"]` inside JSON matches `session_id` argument for all test sessions
7. Invalid state transitions are rejected with a clear error message

---

## What You Must Not Do

- Do not build any other module besides Module 13
- Do not add any audio processing logic
- Do not connect to Google Drive in this phase
- Do not build any frontend
- Do not add dependencies beyond what is already in `requirements.txt`
- Do not continue past any failed task — stop and print diagnostic
- Do not hardcode `SUPPORTED_SCHEMA_VERSION` anywhere except the constants block
- Do not mutate session dicts directly — all mutations must go through Module 13 public functions
- Do not duplicate `_validate_song_plan` logic inline — always call the shared function