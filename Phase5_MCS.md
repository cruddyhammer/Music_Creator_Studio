# MCS Phase 5 — Google Drive Integration Agent Prompt (v1)

## Context

You are building the Google Drive integration layer for MCS (Music Creator Studio). This is Phase 5. Phases 0–4 are complete and verified.

Phase 5 builds five things:
- **`backend/hash_utils.py`** — SHA256 computation and verification utility
- **`backend/drive_uploader.py`** — quota-aware account selection, upload, shareable link return, round-robin fallback
- **`backend/module9.py`** — backend-only approval execution layer: orchestrates hash → upload → session update
- **Module 13 extension** — hash verification function wired into reconciliation
- **`backend/verify_phase5.py`** — full verification suite including assumption 4 test

**No frontend. No audio processing. No new dependencies beyond `requirements.txt` (google-api-python-client already present from Phase 0).**

**Stop on first failure. Print `[OK]` / `[FAIL]` diagnostics. Do not continue to the next task.**

---

## What Phase 5 Does (Read Before Writing Any Code)

Francisco drops stems into a local intake folder. After Module 8 clears them, he triggers approval via Module 9. For each approved stem:

1. SHA256 hash computed from local file before any upload
2. File uploaded to the Drive account with the most free space
3. If quota query fails for any reason, round-robin fallback selects next account silently
4. Shareable link returned from Drive API
5. Hash + link stored in session JSON via Module 13
6. On any subsequent reconciliation run, Module 13 re-verifies stored hashes against Drive — mismatch triggers a flag, never silent failure

Drive is an asset store only. GitHub JSON is the single source of truth. Drive contents are always derived from JSON references, never the reverse.

---

## Architectural Rules (Read Before Writing Any Code)

1. **Drive is asset store only.** JSON is state authority. If Drive and JSON conflict, JSON wins. Module 13 flags the conflict — it never auto-resolves by trusting Drive.
2. **Every upload hashes before sending.** SHA256 is computed from the local file. Hash is stored in session JSON. Upload without hash is not permitted.
3. **Hash mismatch on reconciliation is a hard flag.** Never silent. Module 13 adds `"hash_mismatch"` to session flags and logs the discrepancy with stem ID, stored hash, and recomputed hash.
4. **Drive credentials come from environment variables only.** `MCS_DRIVE_CREDENTIALS_1`, `MCS_DRIVE_CREDENTIALS_2`, `MCS_DRIVE_CREDENTIALS_3` — each contains the full JSON credentials string for one Google service account. Never hardcoded.
5. **Account selection is quota-first, round-robin fallback.** Query all accounts for free space, pick highest. On any failure (API error, timeout, partial response), fall back to round-robin via persistent pointer in `data/drive_state.json`. Log `[WARN]` but never interrupt pipeline.
6. **Round-robin pointer persists across runs.** Stored in `data/drive_state.json`. Loaded at start of selection, incremented after use, written back immediately.
7. **Module 9 does not implement upload logic or credential management.** It only orchestrates: load session → hash → upload → write checkpoint. All Drive logic lives in `drive_uploader.py`.
8. **Module 9 does not manage Drive credentials.** It calls `drive_uploader.upload_file()` only.
9. **All session mutations go through Module 13 public functions.** Module 9 never writes to session dicts directly.
10. **Standard failure envelope applies to Module 9.** Same shape as all prior modules.
11. **Path rule applies everywhere.** All scripts derive root from `Path(__file__).resolve().parents[N]`. Never hardcode absolute paths.
12. **No new dependencies.** Use only `google-api-python-client`, `google-auth`, and Python stdlib (`hashlib`, `json`, `os`, `pathlib`). Both Google packages are already in `requirements.txt`.
13. **`drive_uploader.py` never raises on account selection failure.** Falls back silently with warning log.
14. **Every upload result is logged as a checkpoint via Module 13.** Including failures — a failed upload is checkpointed as failed, not silently skipped.
15. **Shareable links use Drive API `webViewLink`.** Not `webContentLink`. This gives a browser-openable link usable in the frontend later.

---

## Path Rule (Mandatory)

```python
ROOT            = Path(__file__).resolve().parents[1]
DATA_DIR        = ROOT / "data"
DRIVE_STATE     = ROOT / "data" / "drive_state.json"
SESSIONS_DIR    = ROOT / "data" / "sessions"
```

Never hardcode absolute paths. Never assume a working directory.

---

## Standard Failure Envelope (Same as All Prior Phases)

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

## Module 13 Dependency Reference

Phase 5 uses these existing Module 13 public functions:

| Function | Purpose |
|---|---|
| `load_session(session_id)` | Load session dict from disk |
| `write_checkpoint(session_id, module, data)` | Append checkpoint to session |
| `list_sessions()` | Return all session dicts |
| `reconcile_state(session_id)` | Validate session integrity |

Phase 5 adds one new public function to Module 13 in Task 0.

---

## Preconditions

Confirm Phases 0–4 are complete before starting:

1. Folder structure exists: `/backend`, `/data`, `/data/sessions`, `/data/audio`, `/data/library`, `/temp`
2. `requirements.txt` present with `google-api-python-client` and `google-auth` listed
3. `backend/module13.py` exists and imports cleanly
4. `backend/module13.py` has `list_sessions()` and `write_checkpoint()` public functions
5. `backend/session_manager.py` exists and imports cleanly
6. Test audio file exists at: `data/audio/heavy_gauge.mp3`
7. Environment variables `MCS_DRIVE_CREDENTIALS_1`, `MCS_DRIVE_CREDENTIALS_2`, `MCS_DRIVE_CREDENTIALS_3` are set (for assumption 4 test only — not required for all other tasks)

If any are missing, print `[FAIL] Precondition failed — <what is missing>` and stop.

---

## Task 0 — Module 13 Extension: Hash Verification

Make one surgical addition to `backend/module13.py`. Do not restructure any existing logic.

### Add `verify_asset_hashes(session_id: str) -> dict`

```python
def verify_asset_hashes(session_id: str) -> dict:
    """
    Re-verifies all asset hashes stored in the session against
    a provided recompute function. Flags any mismatch.

    Returns:
    {
        "session_id":   str,
        "checked":      int,
        "mismatches":   int,
        "mismatch_ids": [str, ...],
        "flags_added":  [str, ...]
    }

    Never raises. Returns empty result on load failure.
    """
    try:
        session = load_session(session_id)
    except Exception:
        return {
            "session_id":   session_id,
            "checked":      0,
            "mismatches":   0,
            "mismatch_ids": [],
            "flags_added":  []
        }

    assets     = session.get("assets", [])
    mismatches = []
    flags      = []

    for asset in assets:
        stored_hash   = asset.get("sha256")
        recomputed    = asset.get("sha256_recomputed")  # populated by caller before this runs
        stem_id       = asset.get("stem_id", "unknown")

        if stored_hash and recomputed and stored_hash != recomputed:
            mismatches.append(stem_id)
            flag = f"hash_mismatch:{stem_id}"
            flags.append(flag)
            print(
                f"[FAIL] Hash mismatch — stem: {stem_id} "
                f"stored: {stored_hash[:12]}... recomputed: {recomputed[:12]}..."
            )

    if flags:
        write_checkpoint(session_id, module="module13_hash_verify", data={
            "event":          "hash_mismatch_detected",
            "mismatch_ids":   mismatches,
            "flags":          flags
        })

    return {
        "session_id":   session_id,
        "checked":      len(assets),
        "mismatches":   len(mismatches),
        "mismatch_ids": mismatches,
        "flags_added":  flags
    }
```

**Design note:** The recomputed hash is expected to be populated in the asset dict by the caller (e.g. `drive_uploader` or a reconciliation utility) before calling `verify_asset_hashes`. This keeps Module 13 as state authority without giving it file I/O responsibilities.

### Verification

```bash
python3.11 -c "
import backend.module13 as m
print(hasattr(m, 'verify_asset_hashes'))
"
```

Expected: `True`

---

## Task 1 — Hash Utility

Create `backend/hash_utils.py`.

```python
from pathlib import Path
import hashlib

ROOT = Path(__file__).resolve().parents[1]

CHUNK_SIZE = 65536  # 64KB read chunks — safe for large audio files
```

### `compute_sha256(file_path: str | Path) -> str`

```python
def compute_sha256(file_path: str | Path) -> str:
    """
    Computes SHA256 hash of a file.
    Returns hex digest string.
    Raises FileNotFoundError if file does not exist.
    Raises IOError on read failure.
    """
    path = Path(file_path)
    if not path.exists():
        raise FileNotFoundError(f"File not found: {path}")

    h = hashlib.sha256()
    with open(path, "rb") as f:
        while chunk := f.read(CHUNK_SIZE):
            h.update(chunk)
    return h.hexdigest()
```

### `verify_sha256(file_path: str | Path, expected_hash: str) -> bool`

```python
def verify_sha256(file_path: str | Path, expected_hash: str) -> bool:
    """
    Recomputes SHA256 and compares against expected.
    Returns True if match, False if mismatch.
    Returns False (not raises) if file is missing or unreadable.
    """
    try:
        return compute_sha256(file_path) == expected_hash
    except Exception:
        return False
```

### Verification

```bash
python3.11 -c "
from backend.hash_utils import compute_sha256, verify_sha256
h = compute_sha256('data/audio/heavy_gauge.mp3')
print(f'Hash: {h}')
print(f'Verify match: {verify_sha256(\"data/audio/heavy_gauge.mp3\", h)}')
print(f'Verify mismatch: {verify_sha256(\"data/audio/heavy_gauge.mp3\", \"deadbeef\")}')
"
```

Expected: valid 64-char hex hash, `True`, `False`.

---

## Task 2 — Drive Uploader

Create `backend/drive_uploader.py`.

```python
from pathlib import Path
import os, json
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from google.oauth2 import service_account

ROOT        = Path(__file__).resolve().parents[1]
DRIVE_STATE = ROOT / "data" / "drive_state.json"

SCOPES           = ["https://www.googleapis.com/auth/drive.file"]
ACCOUNT_ENV_VARS = [
    "MCS_DRIVE_CREDENTIALS_1",
    "MCS_DRIVE_CREDENTIALS_2",
    "MCS_DRIVE_CREDENTIALS_3",
]
NUM_ACCOUNTS = len(ACCOUNT_ENV_VARS)
```

### `_load_drive_state() -> dict`

```python
def _load_drive_state() -> dict:
    """
    Loads persistent drive state (round-robin pointer).
    Returns default state if file missing or corrupt.
    Never raises.
    """
    try:
        if DRIVE_STATE.exists():
            return json.loads(DRIVE_STATE.read_text())
    except Exception:
        pass
    return {"last_used_index": -1}
```

### `_save_drive_state(state: dict) -> None`

```python
def _save_drive_state(state: dict) -> None:
    """
    Writes drive state to disk atomically.
    Never raises — prints warn on failure.
    """
    try:
        DRIVE_STATE.parent.mkdir(parents=True, exist_ok=True)
        tmp = DRIVE_STATE.with_suffix(".tmp")
        tmp.write_text(json.dumps(state, indent=2))
        tmp.replace(DRIVE_STATE)
    except Exception as e:
        print(f"[WARN] Could not save drive state — {e}")
```

### `_build_service(account_index: int)`

```python
def _build_service(account_index: int):
    """
    Builds and returns an authenticated Drive API service for the given account index.
    Reads credentials from environment variable MCS_DRIVE_CREDENTIALS_<N>.
    Raises EnvironmentError if credentials env var is missing or empty.
    Raises ValueError if credentials JSON is malformed.
    """
    env_var = ACCOUNT_ENV_VARS[account_index]
    creds_json = os.environ.get(env_var, "").strip()
    if not creds_json:
        raise EnvironmentError(f"Missing credentials env var: {env_var}")
    try:
        creds_dict = json.loads(creds_json)
    except json.JSONDecodeError as e:
        raise ValueError(f"Malformed credentials JSON in {env_var}: {e}")

    creds = service_account.Credentials.from_service_account_info(
        creds_dict, scopes=SCOPES
    )
    return build("drive", "v3", credentials=creds, cache_discovery=False)
```

### `_get_free_space(service) -> int`

```python
def _get_free_space(service) -> int:
    """
    Queries Drive API for remaining storage in bytes.
    Returns 0 on any failure — caller treats as unusable account.
    """
    try:
        about = service.about().get(fields="storageQuota").execute()
        quota = about.get("storageQuota", {})
        limit = int(quota.get("limit", 0))
        usage = int(quota.get("usage", 0))
        return max(0, limit - usage)
    except Exception:
        return 0
```

### `_select_account() -> int`

```python
def _select_account() -> int:
    """
    Selects Drive account index using quota-first strategy.
    Falls back to round-robin if quota query fails for all accounts.

    Returns account index (0, 1, or 2).
    Never raises.
    """
    # Attempt quota-based selection
    try:
        free_spaces = []
        for i in range(NUM_ACCOUNTS):
            try:
                svc   = _build_service(i)
                space = _get_free_space(svc)
                free_spaces.append((space, i))
            except Exception as e:
                print(f"[WARN] Could not query quota for account {i+1} — {e}")
                free_spaces.append((0, i))

        if any(space > 0 for space, _ in free_spaces):
            best_index = max(free_spaces, key=lambda x: x[0])[1]
            return best_index
        else:
            raise RuntimeError("All quota queries returned 0")

    except Exception as e:
        # Round-robin fallback
        print(f"[WARN] Quota-based selection failed — using round-robin fallback: {e}")
        state = _load_drive_state()
        next_index = (state.get("last_used_index", -1) + 1) % NUM_ACCOUNTS
        state["last_used_index"] = next_index
        _save_drive_state(state)
        return next_index
```

### `upload_file(local_path: str | Path, filename: str, mime_type: str = "audio/mpeg") -> dict`

```python
def upload_file(
    local_path: str | Path,
    filename:   str,
    mime_type:  str = "audio/mpeg"
) -> dict:
    """
    Uploads a file to the best available Drive account.
    Returns result dict — never raises.

    Returns:
    {
        "success":       bool,
        "account_index": int | None,
        "file_id":       str | None,
        "web_view_link": str | None,
        "error":         str | None
    }
    """
    local_path = Path(local_path)
    if not local_path.exists():
        return {
            "success":       False,
            "account_index": None,
            "file_id":       None,
            "web_view_link": None,
            "error":         f"File not found: {local_path}"
        }

    account_index = _select_account()

    try:
        svc   = _build_service(account_index)
        meta  = {"name": filename}
        media = MediaFileUpload(str(local_path), mimetype=mime_type, resumable=True)
        result = (
            svc.files()
            .create(body=meta, media_body=media, fields="id,webViewLink")
            .execute()
        )
        # Make file readable via link
        svc.permissions().create(
            fileId=result["id"],
            body={"type": "anyone", "role": "reader"}
        ).execute()

        # Update round-robin pointer on successful upload
        state = _load_drive_state()
        state["last_used_index"] = account_index
        _save_drive_state(state)

        print(f"[OK] Uploaded to Drive account {account_index + 1} — {filename}")
        return {
            "success":       True,
            "account_index": account_index,
            "file_id":       result["id"],
            "web_view_link": result.get("webViewLink"),
            "error":         None
        }

    except Exception as e:
        print(f"[FAIL] Drive upload failed — account {account_index + 1}: {e}")
        return {
            "success":       False,
            "account_index": account_index,
            "file_id":       None,
            "web_view_link": None,
            "error":         str(e)
        }
```

---

## Task 3 — Module 9: Approval Execution Layer

Create `backend/module9.py`.

```python
from pathlib import Path
import backend.module13 as module13
from backend.hash_utils import compute_sha256
from backend.drive_uploader import upload_file

ROOT = Path(__file__).resolve().parents[1]
```

### `run(session_id: str, approved_stem_ids: list, stem_local_paths: dict) -> dict`

`stem_local_paths` is a dict mapping `stem_id -> local file path string`.

**Logic (in order, per stem):**

1. Load session via `module13.load_session(session_id)`
2. For each `stem_id` in `approved_stem_ids`:
   - Resolve local path from `stem_local_paths[stem_id]`
   - If path missing from dict or file does not exist: checkpoint as failed, add to errors, continue
   - Compute SHA256 via `hash_utils.compute_sha256()`
   - Call `drive_uploader.upload_file(local_path, filename=stem_id + ".mp3")`
   - If upload failed: checkpoint as failed, add to errors, continue — never raise
   - If upload succeeded:
     - Write checkpoint via `module13.write_checkpoint()` with full asset record
     - Asset record shape:

```python
{
    "event":        "stem_approved",
    "stem_id":      stem_id,
    "sha256":       hash_str,
    "drive_file_id":   result["file_id"],
    "web_view_link":   result["web_view_link"],
    "account_index":   result["account_index"],
    "status":          "uploaded"
}
```

3. After all stems processed:
   - If all succeeded: return `status: "ok"`
   - If some failed: return `status: "warn"`, list failures in `errors`
   - If all failed: return `status: "fail"`

```python
# Return shape
{
    "module":     "module9",
    "session_id": session_id,
    "status":     "ok" | "warn" | "fail",
    "flags":      [str, ...],
    "errors":     [str, ...],
    "data": {
        "approved_count":  int,   # stems successfully uploaded
        "failed_count":    int,   # stems that failed
        "uploaded_stems":  [str, ...],
        "failed_stems":    [str, ...]
    }
}
```

**Module 9 must never:**
- Manage Drive credentials
- Implement upload logic
- Write to session dicts directly
- Raise under any circumstance

---

## Task 4 — Assumption 4 Verification Script

Create `backend/verify_assumption4.py`.

This script tests that one Python pipeline can authenticate and write to all 3 Drive accounts.

```python
from pathlib import Path
import sys, os

ROOT = Path(__file__).resolve().parents[1]
sys.path.insert(0, str(ROOT))

from backend.drive_uploader import _build_service, _get_free_space, upload_file
from backend.hash_utils import compute_sha256

TEST_FILE = ROOT / "data" / "audio" / "heavy_gauge.mp3"
failed    = False

print("=" * 60)
print("MCS Assumption 4 — Drive Multi-Account Verification")
print("=" * 60)
print()

if not TEST_FILE.exists():
    print(f"[FAIL] Test file not found: {TEST_FILE}")
    sys.exit(1)

# ── Test each account ─────────────────────────────────────────────────────
for i in range(3):
    env_var = f"MCS_DRIVE_CREDENTIALS_{i+1}"
    print(f"--- Account {i+1} ({env_var}) ---")

    if not os.environ.get(env_var, "").strip():
        print(f"[FAIL] {env_var} not set — skipping")
        failed = True
        print()
        continue

    # Auth test
    try:
        svc   = _build_service(i)
        space = _get_free_space(svc)
        print(f"[OK] Authenticated — free space: {space // (1024**3):.1f} GB")
    except Exception as e:
        print(f"[FAIL] Auth failed — {e}")
        failed = True
        print()
        continue

    # Upload test
    result = upload_file(TEST_FILE, filename=f"mcs_assumption4_test_account{i+1}.mp3")
    if result["success"]:
        print(f"[OK] Upload succeeded — file_id: {result['file_id']}")
        print(f"     Link: {result['web_view_link']}")
    else:
        print(f"[FAIL] Upload failed — {result['error']}")
        failed = True
    print()

# ── Hash verification ─────────────────────────────────────────────────────
print("--- SHA256 verification ---")
try:
    h = compute_sha256(TEST_FILE)
    print(f"[OK] SHA256: {h}")
except Exception as e:
    print(f"[FAIL] Hash computation failed — {e}")
    failed = True

print()
if failed:
    print("[FAIL] Assumption 4 — one or more accounts failed")
    sys.exit(1)
else:
    print("[OK] Assumption 4 verified — all 3 accounts authenticated and uploaded successfully")
    sys.exit(0)
```

Run it:

```bash
export MCS_DRIVE_CREDENTIALS_1='<json string>'
export MCS_DRIVE_CREDENTIALS_2='<json string>'
export MCS_DRIVE_CREDENTIALS_3='<json string>'
python3.11 backend/verify_assumption4.py
```

---

## Task 5 — Verification Script

Create `backend/verify_phase5.py`.

```python
from pathlib import Path
import sys, os, json, importlib

ROOT   = Path(__file__).resolve().parents[1]
failed = False

print("=" * 60)
print("MCS Phase 5 — Verification")
print("=" * 60)
print()

# ── Import checks ──────────────────────────────────────────────────────────
for mod_path in [
    "backend.module13",
    "backend.hash_utils",
    "backend.drive_uploader",
    "backend.module9",
]:
    try:
        importlib.import_module(mod_path)
        print(f"[OK] {mod_path} — imported")
    except Exception as e:
        print(f"[FAIL] {mod_path} — {e}")
        failed = True

if failed:
    sys.exit(1)

import backend.module13      as m13
import backend.hash_utils    as hu
import backend.drive_uploader as du
import backend.module9       as m9

# ── Module 13: verify_asset_hashes present ────────────────────────────────
try:
    assert hasattr(m13, "verify_asset_hashes")
    print("[OK] module13.verify_asset_hashes — present")
except AssertionError:
    print("[FAIL] module13.verify_asset_hashes — missing")
    failed = True

# ── verify_asset_hashes on empty session ──────────────────────────────────
try:
    result = m13.verify_asset_hashes("nonexistent_session_id")
    assert result["checked"] == 0
    assert result["mismatches"] == 0
    print("[OK] verify_asset_hashes — returns empty result on missing session")
except Exception as e:
    print(f"[FAIL] verify_asset_hashes empty session — {e}")
    failed = True

# ── hash_utils: compute and verify ────────────────────────────────────────
TEST_FILE = ROOT / "data" / "audio" / "heavy_gauge.mp3"
if not TEST_FILE.exists():
    print(f"[FAIL] Test file missing: {TEST_FILE}")
    failed = True
else:
    try:
        h = hu.compute_sha256(TEST_FILE)
        assert len(h) == 64 and all(c in "0123456789abcdef" for c in h)
        print(f"[OK] compute_sha256 — {h[:16]}...")
    except Exception as e:
        print(f"[FAIL] compute_sha256 — {e}")
        failed = True

    try:
        assert hu.verify_sha256(TEST_FILE, h) is True
        print("[OK] verify_sha256 — match returns True")
    except Exception as e:
        print(f"[FAIL] verify_sha256 match — {e}")
        failed = True

    try:
        assert hu.verify_sha256(TEST_FILE, "deadbeef" * 8) is False
        print("[OK] verify_sha256 — mismatch returns False")
    except Exception as e:
        print(f"[FAIL] verify_sha256 mismatch — {e}")
        failed = True

    try:
        assert hu.verify_sha256("/nonexistent/file.mp3", h) is False
        print("[OK] verify_sha256 — missing file returns False (no raise)")
    except Exception as e:
        print(f"[FAIL] verify_sha256 missing file — {e}")
        failed = True

# ── hash_utils: missing file raises correctly ─────────────────────────────
try:
    hu.compute_sha256("/nonexistent/file.mp3")
    print("[FAIL] compute_sha256 — should raise FileNotFoundError on missing file")
    failed = True
except FileNotFoundError:
    print("[OK] compute_sha256 — raises FileNotFoundError on missing file")
except Exception as e:
    print(f"[FAIL] compute_sha256 raised wrong exception — {e}")
    failed = True

# ── drive_uploader: drive_state defaults correctly ────────────────────────
try:
    state = du._load_drive_state()
    assert isinstance(state, dict)
    assert "last_used_index" in state
    print(f"[OK] _load_drive_state — returned state: {state}")
except Exception as e:
    print(f"[FAIL] _load_drive_state — {e}")
    failed = True

# ── drive_uploader: _save_drive_state round-trips ─────────────────────────
try:
    test_state = {"last_used_index": 1}
    du._save_drive_state(test_state)
    loaded = du._load_drive_state()
    assert loaded["last_used_index"] == 1
    print("[OK] _save_drive_state — round-trip verified")
except Exception as e:
    print(f"[FAIL] _save_drive_state round-trip — {e}")
    failed = True

# ── drive_uploader: missing file returns fail dict (no raise) ─────────────
try:
    result = du.upload_file("/nonexistent/file.mp3", "test.mp3")
    assert result["success"] is False
    assert result["error"] is not None
    print("[OK] upload_file — missing file returns fail dict without raising")
except Exception as e:
    print(f"[FAIL] upload_file missing file — raised unexpectedly: {e}")
    failed = True

# ── drive_uploader: missing credentials returns fail dict (no raise) ──────
try:
    orig = {k: os.environ.pop(k, None) for k in [
        "MCS_DRIVE_CREDENTIALS_1", "MCS_DRIVE_CREDENTIALS_2", "MCS_DRIVE_CREDENTIALS_3"
    ]}
    if TEST_FILE.exists():
        result = du.upload_file(TEST_FILE, "test.mp3")
        assert result["success"] is False
        print("[OK] upload_file — missing credentials returns fail dict without raising")
    for k, v in orig.items():
        if v: os.environ[k] = v
except Exception as e:
    print(f"[FAIL] upload_file missing credentials — raised unexpectedly: {e}")
    failed = True

# ── module9: run returns correct envelope shape ────────────────────────────
try:
    # Create a real test session
    test_sess = m13.create_session({
        "title": "phase5_test", "mood": "dark", "energy": "driving",
        "tempo_bpm": 128, "target_duration_seconds": 180
    })
    sid = test_sess["session_id"]

    # Run with nonexistent stem path — should warn, not fail hard
    result = m9.run(
        session_id=sid,
        approved_stem_ids=["test_stem"],
        stem_local_paths={"test_stem": "/nonexistent/stem.mp3"}
    )
    assert "module"     in result
    assert "session_id" in result
    assert "status"     in result
    assert "data"       in result
    assert result["module"] == "module9"
    assert result["data"]["failed_count"] == 1
    print(f"[OK] module9.run — correct envelope, failed_count=1 for missing file")
except Exception as e:
    print(f"[FAIL] module9.run envelope — {e}")
    failed = True

# ── module9: never raises ─────────────────────────────────────────────────
try:
    result = m9.run("nonexistent_session", [], {})
    assert isinstance(result, dict)
    print("[OK] module9.run — never raises on nonexistent session")
except Exception as e:
    print(f"[FAIL] module9.run raised unexpectedly — {e}")
    failed = True

# ── drive_state.json: exists after save ───────────────────────────────────
try:
    assert (ROOT / "data" / "drive_state.json").exists()
    print("[OK] drive_state.json — exists")
except AssertionError:
    print("[FAIL] drive_state.json — not found after save test")
    failed = True

# ── ACCOUNT_ENV_VARS: correct count ───────────────────────────────────────
try:
    assert du.NUM_ACCOUNTS == 3
    assert len(du.ACCOUNT_ENV_VARS) == 3
    print("[OK] drive_uploader — 3 accounts configured")
except AssertionError as e:
    print(f"[FAIL] drive_uploader account count — {e}")
    failed = True

print()
if failed:
    print("[FAIL] Phase 5 verification failed — fix all failures before proceeding")
    sys.exit(1)
else:
    print("[OK] Phase 5 verification complete — all checks passed")
    sys.exit(0)
```

Run it:

```bash
python3.11 backend/verify_phase5.py
```

---

## Task 6 — Manual Integration Test

After verification passes, run a full approval cycle with a real stem:

```bash
export MCS_DRIVE_CREDENTIALS_1='<json>'
export MCS_DRIVE_CREDENTIALS_2='<json>'
export MCS_DRIVE_CREDENTIALS_3='<json>'

python3.11 -c "
import backend.module13 as m13
import backend.module9  as m9

# Create test session
sess = m13.create_session({
    'title': 'drive_test', 'mood': 'dark', 'energy': 'driving',
    'tempo_bpm': 128, 'target_duration_seconds': 180
})
sid = sess['session_id']
print(f'Session: {sid}')

# Run approval with real file
result = m9.run(
    session_id=sid,
    approved_stem_ids=['heavy_gauge'],
    stem_local_paths={'heavy_gauge': 'data/audio/heavy_gauge.mp3'}
)
print(result)

# Inspect session checkpoint
loaded = m13.load_session(sid)
for cp in loaded['checkpoints']:
    if cp['module'] == 'module9':
        print('Checkpoint:', cp)
"
```

Confirm:
- Upload succeeds to one Drive account
- Checkpoint written with `sha256`, `drive_file_id`, `web_view_link`
- `web_view_link` is a valid `https://drive.google.com/...` URL
- No exceptions raised

---

## Completion Gate

Phase 5 is complete when ALL of the following are true:

1. `verify_phase5.py` exits code 0 — all `[OK]`
2. `verify_assumption4.py` exits code 0 — all 3 accounts authenticated and uploaded
3. `compute_sha256()` produces consistent 64-char hex digest on same file
4. `verify_sha256()` returns `True` on match, `False` on mismatch, `False` (not raises) on missing file
5. `compute_sha256()` raises `FileNotFoundError` on missing file
6. `upload_file()` never raises — returns structured result dict on all code paths
7. Account selection uses quota-first, falls back to round-robin on any failure
8. Round-robin pointer persists in `data/drive_state.json` across runs
9. `[WARN]` printed on quota query failure — pipeline never interrupted
10. Uploaded files have `webViewLink` (browser-openable), not `webContentLink`
11. All uploaded files have `anyone/reader` permission set
12. `module9.run()` never raises — returns standard envelope on all code paths
13. Failed stems logged as failed checkpoints — not silently skipped
14. Successful stems checkpointed with `sha256`, `drive_file_id`, `web_view_link`, `account_index`
15. `verify_asset_hashes()` returns empty result on missing session (never raises)
16. Hash mismatch triggers `write_checkpoint()` with `hash_mismatch_detected` event and `[FAIL]` print
17. Module 9 contains zero credential management or upload implementation logic
18. All session mutations go through Module 13 public functions
19. No new dependencies added beyond `requirements.txt`
20. All paths derived from `Path(__file__).resolve().parents[N]` — no hardcoded paths

---

## What You Must Not Do

- Do not hardcode Drive credentials, account emails, or folder IDs
- Do not use `webContentLink` — use `webViewLink` only
- Do not implement upload logic in Module 9 — it calls `drive_uploader.upload_file()` only
- Do not auto-resolve Drive vs JSON conflicts — flag them, defer to JSON
- Do not skip checkpointing failed uploads — every outcome is logged
- Do not allow `upload_file()` or `module9.run()` to raise
- Do not add new dependencies beyond `requirements.txt`
- Do not hardcode absolute paths anywhere
- Do not allow hash mismatch to pass silently — always print `[FAIL]` and checkpoint
- Do not mutate session dicts directly — all mutations through Module 13
- Do not make quota queries blocking — any failure must fall through to round-robin immediately
- Do not commit audio files to the repository — stems stay local until Drive-uploaded, then Drive link is the reference
