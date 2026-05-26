# MCS Phase 9 — Hardening and Production Readiness Agent Prompt (v2)

## Context

You are hardening the Music Creator Studio (MCS) for production use. This is Phase 9. Phases 0–8 are complete and verified.

Phase 9 does not build new modules. It builds test infrastructure that proves the complete system is stable, recoverable, and ready for real weekly use.

Phase 9 delivers:
- **`backend/test_failure_recovery.py`** — Automated failure injection, checkpoint state, and resume behavior tests across all modules
- **`backend/test_drive_desync.py`** — Drive asset hash corruption detection test
- **`backend/test_entropy_audit.py`** — Entropy injection logging audit across 5 simulated cycles
- **`backend/test_temp_purge.py`** — Unapproved stem temp folder purge verification
- **`backend/verify_phase9.py`** — Master verification script that runs all automated tests and reports results
- **`docs/manual_checklist.md`** — Manual procedure checklist for state transition and secrets review

**No new modules. No new dependencies beyond `requirements.txt`. No changes to any Phase 0–8 module unless a bug is discovered during testing — in which case stop, report the bug with full diagnostic, and wait for instruction before patching.**

**Stop on first failure within each test script. Print `[OK]` / `[FAIL]` diagnostics. Do not continue to the next task.**

---

## What Phase 9 Does (Read Before Writing Any Code)

Phase 9 has two categories:

**Category A — Automated tests** (built and run by the agent):
All scripts are non-destructive. They use temporary state, patched functions, and isolated test directories. They never write to real session JSON, real Drive, or real GitHub. They clean up after themselves in `finally` blocks regardless of pass or fail.

**Category B — Manual checklist** (executed by Francisco after Category A passes):
A documented procedure file covering the full State 1 → State 2 → State 1 cycle and the GitHub Actions secrets review. The agent writes the file. Francisco runs through it manually after all automated tests pass.

Phase 9 is complete only when both categories are done.

---

## Architectural Rules (Read Before Writing Any Code)

1. **Test scripts never modify production data.** All tests use isolated temp directories, in-memory patches, or explicitly scoped test fixtures. Real `data/` files are never written or deleted by test scripts.
2. **Test scripts clean up after themselves.** Any temp directory or file created during a test must be removed in a `finally` block regardless of pass or fail.
3. **Stop on first failure within each script.** Each test script exits with code 1 on first `[FAIL]`. The master verify script runs all scripts and collects results — it does not stop on first script failure, but reports all failures at the end.
4. **No new modules.** If a test reveals a bug in a Phase 0–8 module, stop, print the full diagnostic, and wait for instruction. Do not patch inline.
5. **Path rule applies everywhere.** All scripts derive root from `Path(__file__).resolve().parents[N]`.
6. **All thresholds and constants from `backend/config.py`.** Test scripts import config — they never define threshold literals locally.
7. **Failure injection must be surgical.** Patch only the specific function being tested. Restore the original immediately after in a `finally` block. Never leave a module in a patched state.
8. **Test resume behavior, not just checkpoint state.** Verifying that a checkpoint key reads `"complete"` is not sufficient. Resume tests must call `m13.reconcile_state()` and verify the returned resume point is the failed module — not any earlier module.
9. **Temp purge correctness is verified by set membership, not filename pattern.** All purge test cases must confirm that deletion is driven exclusively by the `approved_paths` set passed to `purge_temp_stems`.

---

## Path Rule (Mandatory)

```python
ROOT = Path(__file__).resolve().parents[1]
```

---

## Preconditions

Confirm Phases 0–8 are complete before starting any task:

1. `backend/module1.py` through `backend/module15.py` all exist and import cleanly
2. `backend/module13.py` — `write_checkpoint()`, `reconcile_state()`, `verify_hash()`, `update_asset_field()`, `purge_temp_stems()` all present
3. `backend/run_daily.py` — `_run_pipeline()` present
4. `backend/config.py` — all Phase 0–8 constants present
5. `data/library/song_library.json` exists (may be empty list `[]`)
6. `data/library/style_anchor.json` exists
7. `data/audio/heavy_gauge.mp3` exists

If any precondition fails: print `[FAIL] Precondition failed — <what is missing>` and stop. Do not proceed to any task.

---

## Task 1 — Failure Recovery Test

Create `backend/test_failure_recovery.py`.

This script tests two guarantees for every module that writes a checkpoint:
1. **State guarantee** — the checkpoint correctly records failure state and preserves all prior completed checkpoints
2. **Resume guarantee** — `m13.reconcile_state()` returns the failed module as the resume point, never any earlier module

### Modules to cover (exhaustive — do not skip any)

```
module1, module2, module3, module3_5, module4, module5, module6,
module7, module8, module9, module10, module11, module12,
module13, module14, module15
```

### Implementation

```python
from pathlib import Path
import sys, json, tempfile, shutil

ROOT   = Path(__file__).resolve().parents[1]
failed = False

print("=" * 60)
print("MCS Phase 9 — Failure Recovery Test")
print("=" * 60)
print()

import backend.module13 as m13

def check(condition, label, detail=""):
    global failed
    if condition:
        print(f"[OK] {label}")
    else:
        print(f"[FAIL] {label}" + (f" — {detail}" if detail else ""))
        failed = True
        sys.exit(1)
```

### Test function per module

For each module ID in the coverage list, define and call a test function following this exact pattern. The example below is for `module5` — replicate it for all 16 module IDs, adjusting `TARGET_MODULE` and the prior completed modules list each time:

```python
def test_recovery(tmp_dir, target_module, prior_complete):
    """
    Tests failure recovery for target_module.

    prior_complete: list of module IDs that were complete before target_module failed.
    These must all remain 'complete' after failure injection.
    reconcile_state must return target_module as the resume point.
    """
    # 1. Build session JSON with all prior modules marked complete
    session = {
        "session_id": f"test_recovery_{target_module}",
        "checkpoints": {mod: "complete" for mod in prior_complete},
        "failure_log": []
    }
    session[target_module] = "pending"

    session_file = tmp_dir / f"session_{target_module}.json"
    session_file.write_text(json.dumps(session))

    # 2. Inject failure at target module via module13
    m13.write_checkpoint(
        str(session_file),
        target_module,
        "failed",
        error=f"simulated failure at {target_module}"
    )

    # 3. Read back session and verify state guarantee
    result = json.loads(session_file.read_text())

    check(
        result["checkpoints"].get(target_module) == "failed",
        f"{target_module} — checkpoint records failed state"
    )
    check(
        len(result.get("failure_log", [])) > 0,
        f"{target_module} — failure log populated"
    )
    check(
        result["failure_log"][-1].get("module") == target_module,
        f"{target_module} — failure log identifies correct module",
        f"got: {result['failure_log'][-1]}"
    )

    # 4. Verify all prior completed modules are still marked complete (state preserved)
    for mod in prior_complete:
        check(
            result["checkpoints"].get(mod) == "complete",
            f"{target_module} — prior checkpoint preserved: {mod}"
        )

    # 5. Verify resume behavior — reconcile_state returns target_module as resume point
    resume = m13.reconcile_state(str(session_file))

    check(
        resume.get("resume_from") == target_module,
        f"{target_module} — reconcile_state resume point is {target_module}",
        f"got resume_from: {resume.get('resume_from')}"
    )

    # 6. Verify no prior completed module is scheduled to re-run
    scheduled = resume.get("scheduled_modules", [])
    for mod in prior_complete:
        check(
            mod not in scheduled,
            f"{target_module} — completed module not scheduled for re-run: {mod}",
            f"scheduled: {scheduled}"
        )
```

### Ordered module sequence and prior_complete lists

Call `test_recovery()` once per module in pipeline order. Each call's `prior_complete` is all modules before it in the sequence:

```python
MODULE_ORDER = [
    "module1", "module2", "module3", "module3_5", "module4",
    "module5", "module6", "module7", "module8", "module9",
    "module10", "module11", "module12", "module13", "module14", "module15"
]

tmp_dir = Path(tempfile.mkdtemp(prefix="mcs_test_recovery_"))
try:
    for i, target in enumerate(MODULE_ORDER):
        prior = MODULE_ORDER[:i]  # all modules before target
        test_recovery(tmp_dir, target, prior)

    print()
    print("[OK] Failure recovery — all modules passed (state + resume)")

finally:
    shutil.rmtree(tmp_dir, ignore_errors=True)
    print("[OK] Temp directory cleaned up")

if failed:
    sys.exit(1)
sys.exit(0)
```

### Verification

```bash
python3.11 backend/test_failure_recovery.py
```

Expected: all `[OK]`, exit code 0.

---

## Task 2 — Drive Desync Test

Create `backend/test_drive_desync.py`.

This script tests that Module 13's hash verification catches a corrupted or mismatched Drive asset reference and flags it correctly without modifying the session JSON.

### What to test

1. A session JSON with a correct SHA256 hash for an asset — verification passes with status `"ok"`
2. A session JSON with a deliberately corrupted hash (64 zero characters) — verification flags status `"mismatch"`
3. A session JSON referencing an asset path that does not exist locally — verification flags status `"missing"`
4. After all three cases — session JSON is byte-identical to its state before the checks (verification is strictly read-only)

### Implementation

```python
from pathlib import Path
import sys, json, tempfile, shutil, hashlib

ROOT   = Path(__file__).resolve().parents[1]
failed = False

print("=" * 60)
print("MCS Phase 9 — Drive Desync Test")
print("=" * 60)
print()

import backend.module13 as m13

def check(condition, label, detail=""):
    global failed
    if condition:
        print(f"[OK] {label}")
    else:
        print(f"[FAIL] {label}" + (f" — {detail}" if detail else ""))
        failed = True
        sys.exit(1)

tmp_dir = Path(tempfile.mkdtemp(prefix="mcs_test_desync_"))
try:
    # Create a real test asset with known content
    test_asset = tmp_dir / "test_stem.mp3"
    test_asset.write_bytes(b"fake audio content for hash test")
    correct_hash = hashlib.sha256(test_asset.read_bytes()).hexdigest()

    session = {
        "session_id": "test_desync",
        "assets": [
            {"filename": str(test_asset), "sha256": correct_hash}
        ]
    }
    session_file = tmp_dir / "session_desync.json"
    session_file.write_text(json.dumps(session))

    # Snapshot session content before any verification calls
    session_snapshot = session_file.read_bytes()

    # ── Case 1: Correct hash — verification passes ────────────────────────
    result = m13.verify_hash(str(session_file), str(test_asset), correct_hash)
    check(
        result.get("status") == "ok",
        "Drive desync — correct hash passes verification",
        f"got: {result}"
    )

    # ── Case 2: Corrupted hash — mismatch flagged ─────────────────────────
    bad_hash = "0" * 64
    result_bad = m13.verify_hash(str(session_file), str(test_asset), bad_hash)
    check(
        result_bad.get("status") == "mismatch",
        "Drive desync — corrupted hash flagged as mismatch",
        f"got: {result_bad}"
    )

    # ── Case 3: Missing asset — flagged ───────────────────────────────────
    missing_path = str(tmp_dir / "nonexistent_stem.mp3")
    result_missing = m13.verify_hash(str(session_file), missing_path, correct_hash)
    check(
        result_missing.get("status") == "missing",
        "Drive desync — missing asset flagged correctly",
        f"got: {result_missing}"
    )

    # ── Case 4: Session JSON byte-identical after all checks ──────────────
    session_after = session_file.read_bytes()
    check(
        session_after == session_snapshot,
        "Drive desync — session JSON byte-identical after all verification calls"
    )

    print()
    print("[OK] Drive desync — all cases passed")

finally:
    shutil.rmtree(tmp_dir, ignore_errors=True)
    print("[OK] Temp directory cleaned up")

if failed:
    sys.exit(1)
sys.exit(0)
```

### Verification

```bash
python3.11 backend/test_drive_desync.py
```

Expected: all `[OK]`, exit code 0.

---

## Task 3 — Entropy Audit Test

Create `backend/test_entropy_audit.py`.

This script tests that entropy injection is correctly logged in session JSON across 5 consecutive simulated generation cycles, and that no cycle is missing an entropy record.

### What to test

1. Each of 5 cycles calling `m1.run()` produces a result with an `entropy` key containing `dimension`, `intensity`, and `seed`
2. `dimension` is always one of the five allowed values: `identity`, `rhythm`, `energy_trajectory`, `timbre`, `harmonic`
3. `intensity` is always a float between 0.0 and 1.0 inclusive
4. `seed` is always an integer
5. All 5 entropy records are written to and recoverable from individual session JSON files — the audit trail is complete
6. Across the 5 cycles, at least 2 distinct dimensions appear — this check is a signal, not a hard gate (see note below)

### Implementation

```python
from pathlib import Path
import sys, json, tempfile, shutil

ROOT   = Path(__file__).resolve().parents[1]
failed = False

print("=" * 60)
print("MCS Phase 9 — Entropy Audit Test")
print("=" * 60)
print()

import backend.module1 as m1
from backend import config

ALLOWED_DIMENSIONS = {"identity", "rhythm", "energy_trajectory", "timbre", "harmonic"}

def check(condition, label, detail=""):
    global failed
    if condition:
        print(f"[OK] {label}")
    else:
        print(f"[FAIL] {label}" + (f" — {detail}" if detail else ""))
        failed = True
        sys.exit(1)

tmp_dir = Path(tempfile.mkdtemp(prefix="mcs_test_entropy_"))
try:
    entropy_records = []

    for cycle in range(1, 6):
        result  = m1.run()
        entropy = result.get("entropy", {})

        # Structure checks
        check("dimension" in entropy,
              f"Cycle {cycle} — entropy has dimension key",   f"entropy: {entropy}")
        check("intensity" in entropy,
              f"Cycle {cycle} — entropy has intensity key",   f"entropy: {entropy}")
        check("seed"      in entropy,
              f"Cycle {cycle} — entropy has seed key",        f"entropy: {entropy}")

        # Value checks
        check(
            entropy.get("dimension") in ALLOWED_DIMENSIONS,
            f"Cycle {cycle} — dimension is valid: {entropy.get('dimension')}",
            f"allowed: {ALLOWED_DIMENSIONS}"
        )
        check(
            isinstance(entropy.get("intensity"), float)
            and 0.0 <= entropy["intensity"] <= 1.0,
            f"Cycle {cycle} — intensity in range: {entropy.get('intensity')}"
        )
        check(
            isinstance(entropy.get("seed"), int),
            f"Cycle {cycle} — seed is integer: {entropy.get('seed')}"
        )

        # Write to temp session file simulating checkpoint
        cycle_file = tmp_dir / f"session_cycle_{cycle}.json"
        cycle_file.write_text(json.dumps({
            "session_id": f"test_entropy_cycle_{cycle}",
            "entropy":    entropy
        }))

        entropy_records.append(entropy)
        print(f"  Cycle {cycle}: {entropy}")

    # Audit trail completeness — all 5 records recoverable from JSON
    for cycle in range(1, 6):
        cycle_file = tmp_dir / f"session_cycle_{cycle}.json"
        recovered  = json.loads(cycle_file.read_text())
        check(
            "entropy" in recovered,
            f"Cycle {cycle} — entropy recoverable from session JSON"
        )

    # Dimension diversity — signal check, not hard gate
    dimensions_seen = {r["dimension"] for r in entropy_records}
    if len(dimensions_seen) >= 2:
        print(f"[OK] Entropy audit — dimension diversity confirmed: {dimensions_seen}")
    else:
        print(f"[WARN] Entropy audit — only 1 distinct dimension across 5 cycles: {dimensions_seen}")
        print(f"       This may indicate Module 1 entropy selection is not varying.")
        print(f"       Report this finding. Do not patch Module 1 inline.")
        # Not a hard failure — entropy structure and values are correct.
        # Diversity is advisory at this library size.

    print()
    print("[OK] Entropy audit — all 5 cycles structurally valid and recoverable")

finally:
    shutil.rmtree(tmp_dir, ignore_errors=True)
    print("[OK] Temp directory cleaned up")

if failed:
    sys.exit(1)
sys.exit(0)
```

### Verification

```bash
python3.11 backend/test_entropy_audit.py
```

Expected: all `[OK]`, exit code 0. A `[WARN]` on diversity is not a failure — report it and continue.

---

## Task 4 — Temp Purge Test

Create `backend/test_temp_purge.py`.

This script tests that `m13.purge_temp_stems()` deletes files based exclusively on set membership in `approved_paths`, never on filename pattern. All test cases are designed to verify the set-membership contract specifically.

### What to test

1. A file in `approved_paths` survives regardless of its filename — even if the name looks unapproved
2. A file not in `approved_paths` is deleted regardless of its filename — even if the name looks approved
3. A non-audio file not in `approved_paths` is deleted (purge is not limited to `.mp3` files unless the implementation explicitly scopes it — test the actual behavior)
4. Session JSON asset references for approved stems are intact after purge
5. Empty temp folder — purge completes without error

### Implementation

```python
from pathlib import Path
import sys, json, tempfile, shutil

ROOT   = Path(__file__).resolve().parents[1]
failed = False

print("=" * 60)
print("MCS Phase 9 — Temp Purge Test")
print("=" * 60)
print()

import backend.module13 as m13

def check(condition, label, detail=""):
    global failed
    if condition:
        print(f"[OK] {label}")
    else:
        print(f"[FAIL] {label}" + (f" — {detail}" if detail else ""))
        failed = True
        sys.exit(1)

# Precondition: purge_temp_stems must exist
if not hasattr(m13, "purge_temp_stems"):
    print("[FAIL] Precondition — module13.purge_temp_stems not found.")
    print("       This function must exist in module13 per Phase 1 architecture.")
    print("       Do not add it inline. Report to Francisco before proceeding.")
    sys.exit(1)

tmp_dir = Path(tempfile.mkdtemp(prefix="mcs_test_purge_"))
try:
    temp_stems = tmp_dir / "temp"
    temp_stems.mkdir()

    # File A: in approved_paths, name looks unapproved — must SURVIVE
    file_a = temp_stems / "stem_guitar_UNAPPROVED_name.mp3"
    file_a.write_bytes(b"approved by path membership")

    # File B: not in approved_paths, name looks approved — must be DELETED
    file_b = temp_stems / "stem_drums_APPROVED_name.mp3"
    file_b.write_bytes(b"not in approved set")

    # File C: in approved_paths, neutral name — must SURVIVE
    file_c = temp_stems / "stem_bass_001.mp3"
    file_c.write_bytes(b"approved neutral name")

    # File D: not in approved_paths, neutral name — must be DELETED
    file_d = temp_stems / "stem_pad_002.mp3"
    file_d.write_bytes(b"unapproved neutral name")

    # approved_paths contains only file_a and file_c — by path, not by name
    approved_paths = {str(file_a), str(file_c)}

    # Build session JSON referencing approved stems
    session = {
        "session_id": "test_purge",
        "assets": [
            {"filename": str(file_a), "status": "approved"},
            {"filename": str(file_c), "status": "approved"},
        ]
    }
    session_file = tmp_dir / "session_purge.json"
    session_file.write_text(json.dumps(session))
    session_snapshot = session_file.read_bytes()

    # Run purge
    m13.purge_temp_stems(str(temp_stems), approved_paths)

    # File A: approved by path, survives despite unapproved-looking name
    check(file_a.exists(),
          "Temp purge — approved-by-path file survives (unapproved-looking name)")

    # File B: not approved, deleted despite approved-looking name
    check(not file_b.exists(),
          "Temp purge — unapproved-by-path file deleted (approved-looking name)")

    # File C: approved by path, neutral name, survives
    check(file_c.exists(),
          "Temp purge — approved-by-path file survives (neutral name)")

    # File D: not approved, neutral name, deleted
    check(not file_d.exists(),
          "Temp purge — unapproved-by-path file deleted (neutral name)")

    # Session JSON byte-identical after purge
    check(
        session_file.read_bytes() == session_snapshot,
        "Temp purge — session JSON byte-identical after purge"
    )

    # ── Empty folder — no error ───────────────────────────────────────────
    empty_dir = tmp_dir / "empty_temp"
    empty_dir.mkdir()
    try:
        m13.purge_temp_stems(str(empty_dir), set())
        check(True, "Temp purge — empty folder purge completes without error")
    except Exception as e:
        check(False, "Temp purge — empty folder purge raised exception", str(e))

    print()
    print("[OK] Temp purge — all cases passed")

finally:
    shutil.rmtree(tmp_dir, ignore_errors=True)
    print("[OK] Temp directory cleaned up")

if failed:
    sys.exit(1)
sys.exit(0)
```

### Verification

```bash
python3.11 backend/test_temp_purge.py
```

Expected: all `[OK]`, exit code 0.

---

## Task 5 — Master Verification Script

Create `backend/verify_phase9.py`.

Runs all four automated test scripts as subprocesses. Does not stop on first script failure — runs all four and reports final summary.

```python
from pathlib import Path
import sys, subprocess

ROOT = Path(__file__).resolve().parents[1]

SCRIPTS = [
    "backend/test_failure_recovery.py",
    "backend/test_drive_desync.py",
    "backend/test_entropy_audit.py",
    "backend/test_temp_purge.py",
]

print("=" * 60)
print("MCS Phase 9 — Master Verification")
print("=" * 60)
print()

results = {}

for script in SCRIPTS:
    script_path = ROOT / script
    print(f"--- Running {script} ---")
    try:
        proc = subprocess.run(
            ["python3.11", str(script_path)],
            capture_output=False,
            text=True
        )
        results[script] = proc.returncode == 0
    except Exception as e:
        print(f"[FAIL] Could not run {script} — {e}")
        results[script] = False
    print()

print("=" * 60)
print("Phase 9 — Automated Test Summary")
print("=" * 60)
all_passed = True
for script, passed in results.items():
    status = "[OK]  " if passed else "[FAIL]"
    print(f"{status} {script}")
    if not passed:
        all_passed = False

print()
if all_passed:
    print("[OK] All automated tests passed.")
    print("     Proceed to manual checklist: docs/manual_checklist.md")
    sys.exit(0)
else:
    print("[FAIL] One or more automated tests failed.")
    print("       Fix all failures before running the manual checklist.")
    sys.exit(1)
```

Run it:

```bash
python3.11 backend/verify_phase9.py
```

---

## Task 6 — Manual Checklist

Create `docs/manual_checklist.md` with exactly this content:

```markdown
# MCS Phase 9 — Manual Checklist

Complete this checklist after `verify_phase9.py` exits with code 0.
Check each item as you complete it. Do not proceed to production use until all items are checked.

---

## Section A — GitHub Actions Secrets Review

[ ] Open the MCS GitHub repository → Settings → Secrets and Variables → Actions
[ ] Confirm the following secrets are present (names only — do not open values):
    - DRIVE_ACCOUNT_1_CREDENTIALS
    - DRIVE_ACCOUNT_2_CREDENTIALS
    - DRIVE_ACCOUNT_3_CREDENTIALS
    - EMAIL_SENDER_ADDRESS
    - EMAIL_SENDER_PASSWORD
    - EMAIL_RECIPIENT_ADDRESS
[ ] Confirm no secrets from MRE have leaked into this repository
[ ] Confirm the Actions workflow YAML references secrets by name only — no hardcoded values

---

## Section B — Full State 1 → State 2 → State 1 Cycle Test

This is the real system running with your actual accounts.
Follow each step in order. Do not skip steps.

### State 1 — Candidate Generation

[ ] Trigger a manual run of the daily pipeline (or wait for the scheduled 12:00 PM run)
[ ] Confirm email arrives with song blueprint, BPM, stem list, entropy dimension, and GitHub Pages link
[ ] Open the GitHub Pages prompt queue — confirm all stem prompts are visible and individually copyable
[ ] Confirm prompts are displayed in execution priority order (foundational first)
[ ] Paste each stem prompt into Gemini App manually and download the audio outputs
[ ] Drop downloaded stems into the local intake folder
[ ] Confirm Modules 5–8 run automatically and any flags surface correctly without auto-rejection
[ ] Open the candidate review interface — confirm all stems play simultaneously and mute toggles work
[ ] Enter a Perceptual Score (vibe, originality, coherence) for the session
[ ] Approve at least one candidate
[ ] Confirm session metadata committed to GitHub repo JSON
[ ] Confirm approved stems uploaded to Google Drive — check all 3 accounts for correct routing
[ ] Confirm SHA256 hash stored in session JSON for each uploaded stem
[ ] Confirm system transitions to State 2

### State 2 — Refinement

[ ] Confirm daily notification now shows active project refinement suggestion instead of new song plan
[ ] Open the track toggle interface — confirm core stems display under Core Tracks category
[ ] Trigger at least one refinement stem: paste prompt into Gemini, download, drop into intake
[ ] Confirm refinement stem appears under Refinement Tracks (not Core Tracks)
[ ] Approve or reject the refinement stem — confirm decision logged in session JSON with reason
[ ] Confirm Drive upload for approved refinement stem completes with hash stored in JSON
[ ] Trigger Module 14 Final Mix and Export manually
[ ] Confirm final mix downloads locally before any upload begins
[ ] Confirm final mix uploads to Google Drive as master file
[ ] Confirm final mix published to GitHub Releases with permanent archive link
[ ] Confirm session marked COMPLETE in GitHub repo JSON
[ ] Confirm song appears in song library (song_library.json) with full metadata including all vectors

### Return to State 1

[ ] Manually trigger return to State 1
[ ] Confirm next scheduled run generates a new song plan — not a refinement suggestion
[ ] Confirm Module 15 gap analysis runs and the constraint profile reflects the completed song
[ ] Confirm Module 1 diversity_targets list updates based on the new library entry

---

## Section C — Entropy Injection Audit (Manual Spot Check)

[ ] Open GitHub repo → data/library/ → inspect the last 3 session JSON files
[ ] Confirm each session JSON contains an `entropy` key with `dimension`, `intensity`, `seed`
[ ] Confirm `dimension` values are not identical across all 3 sessions (at least 2 distinct values)
[ ] If all 3 sessions have the same dimension, note it as a finding and review Module 1 entropy logic

---

## Section D — Failure Recovery Spot Check

[ ] Simulate a network interruption during a Drive upload (disconnect WiFi mid-upload)
[ ] Reconnect and trigger a pipeline resume
[ ] Confirm the system resumes from the failed module — not from the beginning
[ ] Confirm completed modules are not re-run
[ ] Confirm the failure is logged in session JSON with module ID and error detail
[ ] Confirm no duplicate data was written by the re-run

---

## Completion Gate

Phase 9 is complete when:
- `verify_phase9.py` exits code 0
- All items in Sections A, B, C, and D are checked
- At least one full song has been published to GitHub Releases

MCS is production ready.

Phase 10 (video generation + YouTube publish) may be scoped after:
- At least one real song is published to GitHub Releases
- The full pipeline has completed at least one additional weekly cycle without intervention
```

---

## Completion Gate

Phase 9 is complete when ALL of the following are true:

1. `verify_phase9.py` exits code 0 — all four automated scripts pass
2. `test_failure_recovery.py` — all 16 modules tested; checkpoint records failed state; prior completed modules preserved; `m13.reconcile_state()` returns target module as resume point; no prior completed module appears in `scheduled_modules`
3. `test_drive_desync.py` — correct hash returns `"ok"`; corrupted hash returns `"mismatch"`; missing asset returns `"missing"`; session JSON byte-identical after all checks
4. `test_entropy_audit.py` — all 5 cycles have valid entropy records with correct structure and value ranges; all records recoverable from session JSON; diversity warning printed if only 1 dimension seen but script still exits code 0
5. `test_temp_purge.py` — approved-by-path files survive regardless of filename; unapproved-by-path files deleted regardless of filename; session JSON byte-identical after purge; empty folder handled without error
6. All temp directories removed in `finally` blocks after each script
7. No production data files modified by any automated test script
8. `docs/manual_checklist.md` present in the repository
9. All items in manual checklist Sections A, B, C, and D completed and checked by Francisco
10. At least one full song published to GitHub Releases during the Section B cycle test

---

## What You Must Not Do

- Do not build Phase 10 or any v3.0 features
- Do not add new modules or modify any Phase 0–8 module unless a bug is found during testing — if found, stop and report with full diagnostic, do not patch inline
- Do not write to real session JSON, real Drive, or real GitHub during automated tests
- Do not leave any module in a patched state — always restore in `finally` blocks
- Do not define threshold or config constants locally — import from `backend.config`
- Do not hardcode absolute paths
- Do not let any test script silently swallow a failure — every `[FAIL]` must print the specific condition that failed and exit code 1
- Do not make the entropy dimension diversity check a hard failure — it is a `[WARN]` signal only
- Do not patch Module 1 if entropy diversity check warns — report the finding and wait for instruction
- Do not call `purge_temp_stems` if it is missing from `module13` — print the precondition failure and stop
- Do not verify purge correctness by filename pattern — verify by set membership only
- Do not verify resume behavior by checkpoint state alone — call `m13.reconcile_state()` and check `resume_from` and `scheduled_modules`
- Do not run the manual checklist before `verify_phase9.py` exits code 0
- Do not mark Phase 9 complete without at least one real song published to GitHub Releases