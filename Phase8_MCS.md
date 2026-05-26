# MCS Phase 8 — Song Library and Adaptive Feedback Agent Prompt (v5)

## Context

You are building the song library and adaptive feedback layer for MCS (Music Creator Studio). This is Phase 8. Phases 0–7 are complete and verified.

Phase 8 builds one new module and one surgical patch:
- **Module 15** — Song Library + Adaptive Feedback (archive writer, gap analyzer, recalibration suggester)
- **Module 1 patch** — Wire gap analysis output into constraint profile generation
- **`backend/verify_phase8.py`** — Verification script
- **`backend/integration_test_phase8.py`** — Full feedback loop integration test

**No new dependencies beyond `requirements.txt`. No frontend changes. No changes to any Phase 0–7 module beyond the surgical Module 1 patch listed explicitly.**

**Stop on first failure. Print `[OK]` / `[FAIL]` diagnostics. Do not continue to the next task.**

---

## What Phase 8 Does (Read Before Writing Any Code)

Module 15 has two roles:

**Role 1 — Archive writer:** Called by Module 14 after export. Writes the completed song's full metadata to `song_library.json`. In Phase 7, `_append_to_library()` inside Module 14 already handles this write. Module 15 does not duplicate that write — instead it exposes a `read_library()` helper and owns all library analysis logic.

**Role 2 — Adaptive feedback engine:** Runs as part of the daily pipeline (both states). Reads all completed songs from `song_library.json`, computes gap analysis across diversity vectors, and returns a constraint profile delta that Module 1 injects into its constraint profile. Also monitors perceptual score history and surfaces a style anchor recalibration suggestion when enough data has accumulated.

Module 1 patch: `run()` calls `module15.get_gap_analysis()` after its own library scan, merges the result into the constraint profile it returns, and logs which gaps were detected.

---

## Architectural Rules (Read Before Writing Any Code)

1. **Module 15 never writes to `song_library.json` directly.** Module 14's `_append_to_library()` is the only writer. Module 15 reads only.
2. **Module 15 never writes session state.** It has no session_id. It reads the library and returns analysis dicts. No checkpoints, no session mutations.
3. **Gap analysis operates on curve shapes, not averages.** Curve dimensions (`energy_curve_shape`, `density_curve_shape`, `harmonic_movement_shape`, `beat_interval_distribution`, `MFCC`, `chroma`) are compared using average pairwise cosine similarity — a gap is flagged when average similarity exceeds `GAP_COSINE_SIMILARITY_FLOOR`. Scalar dimensions (`tempo`, `spectral_centroid`) use symmetric range ratio — a gap is flagged when range ratio is below `GAP_SCALAR_RANGE_THRESHOLD`.
4. **Recalibration suggestion is read-only output.** Module 15 never updates the Style Anchor Vector. It surfaces a suggestion dict that Francisco reads and approves manually. The Style Anchor is only ever updated by explicit user action.
5. **All thresholds from `backend/config.py`.** No module defines threshold constants locally. This includes the originality margin.
6. **No new dependencies.** Use only `requirements.txt`: numpy, scipy, librosa. All already installed.
7. **Standard failure envelope on Module 1's `run()`.** The patch must not change Module 1's return shape.
8. **Path rule applies everywhere.** All scripts derive root from `Path(__file__).resolve().parents[N]`.
9. **Module 15 functions never raise.** Return empty analysis dicts with a `"warning"` key on any error. Never propagate exceptions to callers.
10. **Gap analysis result is advisory.** Module 1 merges it into the constraint profile but never hard-blocks generation based on it.

---

## Path Rule (Mandatory)

```python
ROOT         = Path(__file__).resolve().parents[1]
LIBRARY_FILE = ROOT / "data" / "library" / "song_library.json"
ANCHOR_FILE  = ROOT / "data" / "library" / "style_anchor.json"
```

---

## New Config Constants

Add to `backend/config.py` **before writing any other Phase 8 file**:

```python
# ── Adaptive feedback / Module 15 ──────────────────────────────────────────
RECALIBRATION_MIN_SONGS              = 5     # minimum approved songs before recalibration suggested
RECALIBRATION_PERCEPTUAL_FLOOR       = 6.0   # average P score below this triggers suggestion
RECALIBRATION_ORIGINALITY_MARGIN     = 1.5   # originality must be this far below next-lowest to trigger
GAP_SCALAR_RANGE_THRESHOLD           = 0.25  # scalar range ratio below this = gap detected
GAP_COSINE_SIMILARITY_FLOOR          = 0.85  # avg pairwise similarity above this = dimension too uniform
GAP_SCALAR_DENOMINATOR_CLAMP        = 1e-6   # denominator below this → ratio treated as 0.0 (no variation)
```

Verify:

```bash
python3.11 -c "from backend import config; print(config.RECALIBRATION_MIN_SONGS, config.RECALIBRATION_ORIGINALITY_MARGIN)"
```

Expected: `5 1.5`

---

## Preconditions

Confirm Phases 0–7 are complete:

1. `backend/module1.py` exists, `run()` returns a constraint profile dict with at least `{"entropy": {...}, "constraint_profile": {...}}`
2. `backend/module13.py` with `update_asset_field()` present
3. `backend/module14.py` with `_append_to_library()` present
4. `backend/config.py` exists
5. `data/library/` directory exists
6. `data/audio/heavy_gauge.mp3` present for testing
7. `data/library/style_anchor.json` present (initialized manually in Phase 1)

If any are missing: print `[FAIL] Precondition failed — <what is missing>` and stop.

---

## Task 0 — Config Extension

Add the six constants above to `backend/config.py`. Verify with the one-liner. Do not proceed until verification passes.

---

## Task 1 — Module 15: Song Library + Adaptive Feedback

Create `backend/module15.py`.

```python
from pathlib import Path
import json
import numpy as np
from backend import config

ROOT         = Path(__file__).resolve().parents[1]
LIBRARY_FILE = ROOT / "data" / "library" / "song_library.json"
ANCHOR_FILE  = ROOT / "data" / "library" / "style_anchor.json"

# Scalar dimensions: compared by symmetric range ratio
SCALAR_DIMENSIONS = ["tempo", "spectral_centroid"]

# Curve/vector dimensions: compared by average pairwise cosine similarity.
# MFCC and chroma are intentionally included here — both are multi-dimensional
# arrays and cosine similarity is the correct comparison method for them.
CURVE_DIMENSIONS = [
    "energy_curve_shape",
    "density_curve_shape",
    "harmonic_movement_shape",
    "beat_interval_distribution",
    "MFCC",
    "chroma",
]

ALL_DIMENSIONS = SCALAR_DIMENSIONS + CURVE_DIMENSIONS
```

---

### `read_library() -> list`

```python
def read_library() -> list:
    """
    Reads song_library.json and returns list of song dicts.
    Returns empty list if file missing or unreadable — never raises.
    """
    try:
        if not LIBRARY_FILE.exists():
            return []
        return json.loads(LIBRARY_FILE.read_text())
    except Exception as e:
        print(f"[WARN] Module 15 — could not read library: {e}")
        return []
```

---

### `_cosine_similarity(a: list, b: list) -> float`

```python
def _cosine_similarity(a: list, b: list) -> float:
    """
    Returns cosine similarity between two vectors.
    Returns 0.0 if vectors have different lengths, either is zero,
    or inputs are incompatible — never raises.

    Vectors must be the same length. Mismatched lengths return 0.0.
    Callers must run _is_valid_vector() on both inputs before calling this.
    """
    try:
        va = np.array(a, dtype=np.float32)
        vb = np.array(b, dtype=np.float32)
        if len(va) != len(vb):
            return 0.0
        if len(va) == 0:
            return 0.0
        norm_a = np.linalg.norm(va)
        norm_b = np.linalg.norm(vb)
        if norm_a == 0.0 or norm_b == 0.0:
            return 0.0
        return float(np.dot(va, vb) / (norm_a * norm_b))
    except Exception:
        return 0.0
```

---

### `_is_valid_vector(v) -> bool`

```python
def _is_valid_vector(v) -> bool:
    """
    Returns True if v is a list or tuple of at least 2 elements where every
    element is numeric (castable to float). Rejects None, short lists,
    non-sequences, and sequences containing non-numeric values.
    """
    if not isinstance(v, (list, tuple)) or len(v) < 2:
        return False
    try:
        for x in v:
            float(x)
        return True
    except Exception:
        return False
```

---

### `get_gap_analysis(library: list = None) -> dict`

Reads diversity vectors from all completed songs and identifies which dimensions are underrepresented or too uniform.

**Logic:**

1. If `library` is None: call `read_library()`
2. If fewer than 2 songs: return `{"gaps": [], "coverage": {}, "gap_reasons": {}, "gap_scores": {}, "skipped_counts": {}, "song_count": len(library), "warning": "insufficient_library"}`
3. Collect all `diversity_vector` dicts from library entries — skip entries where `diversity_vector` is missing or empty
4. Initialise `skipped_counts: dict` — tracks per-dimension count of skipped malformed or non-numeric values, for both scalar and curve dimensions
5. For each dimension in `ALL_DIMENSIONS`:

   **Scalar dimensions** (`tempo`, `spectral_centroid`):
   - Collect values for this dimension across all songs, casting each to `float`. Skip and increment `skipped_counts[dimension]` for any value that is None, missing, or not castable to float (including strings)
   - If zero valid values collected: mark coverage as `"missing_dimension"`, add to gaps list, set `gap_reasons[dimension] = "missing_dimension"`, skip ratio computation
   - If exactly one valid value collected: mark coverage as `"insufficient_data"`, skip ratio computation
   - If two or more valid values collected:
     - `max_val`, `min_val` = max and min of collected values
     - `den = abs(max_val) + abs(min_val)`
     - If `den < config.GAP_SCALAR_DENOMINATOR_CLAMP`: `ratio = 0.0` (values clustered at zero — no variation)
     - Otherwise: `ratio = (max_val - min_val) / den` — no `+ 1e-8`, use only the clamp
     - Store ratio in `gap_scores[dimension]`
     - If ratio below `config.GAP_SCALAR_RANGE_THRESHOLD`: mark as gap with reason `"narrow_scalar_range"`
     - Otherwise: mark as covered

   **Curve dimensions** (all entries in `CURVE_DIMENSIONS`):
   - Collect all values for this dimension, filtering with `_is_valid_vector()` — silently skip any value that fails validation (None, single-element list, non-list, non-numeric elements), increment `skipped_counts[dimension]` for each skip
   - If zero valid vectors collected: mark coverage as `"missing_dimension"`, add to gaps list, set `gap_reasons[dimension] = "missing_dimension"`, skip similarity computation
   - If exactly one valid vector collected: mark coverage as `"insufficient_data"`, skip similarity computation
   - If two or more valid vectors collected: build contrast-maximizing deterministic pairs:
     1. Sort vectors by L2 norm ascending — compute raw L2 norm, do not normalize before sorting
     2. Use two pointers `i = 0`, `j = n - 1`. While `i < j` and `len(pairs) < 10`: append `(vectors[i], vectors[j])`, increment `i`, decrement `j`
     3. If vector count is odd and `len(pairs) < 10`: append one additional pair of the middle vector (`index n//2`) with its nearest neighbor (`index n//2 - 1`)
     4. For each pair: both vectors must pass `_is_valid_vector()` (already guaranteed by collection step). If lengths differ, skip that pair silently — `_cosine_similarity` will return 0.0 but skipping is cleaner
     5. Compute cosine similarity for each pair via `_cosine_similarity()`
     6. Compute average pairwise similarity across all pairs
     7. Store average similarity in `gap_scores[dimension]`
     8. If average similarity exceeds `config.GAP_COSINE_SIMILARITY_FLOOR`: mark as gap with reason `"uniform_curve"`
     9. Otherwise: mark as covered

6. Return:

```python
{
    "gaps":          [str, ...],    # dimension names flagged as gaps
    "coverage":      {              # per-dimension status string
        "tempo":              "covered" | "gap:narrow_scalar_range" | "gap:missing_dimension" | "insufficient_data",
        "energy_curve_shape": "covered" | "gap:uniform_curve"       | "gap:missingdimension" | "insufficient_data",
        ...
    },
    "gap_reasons":   {              # per-dimension reason string or None
        "tempo":              "narrow_scalar_range" | "missing_dimension" | None,
        "energy_curve_shape": "uniform_curve"       | "missing_dimension" | None,
        ...
    },
    "gap_scores":    {              # raw metric per dimension (only present when computable)
        "tempo":              float,    # range ratio — lower = narrower
        "energy_curve_shape": float,    # avg pairwise similarity — higher = more uniform
        ...
    },
    "skipped_counts": {             # malformed or non-numeric values skipped per dimension
        "tempo":              int,
        "energy_curve_shape": int,
        ...
    },
    "song_count":    int,
    "warning":       str | None
}
```

Never raises. On any exception: return `{"gaps": [], "coverage": {}, "gap_reasons": {}, "gap_scores": {}, "skipped_counts": {}, "song_count": 0, "warning": str(e)}`.

---

### `get_recalibration_suggestion(library: list = None) -> dict`

Reads perceptual score history from the library and determines whether a style anchor recalibration should be suggested.

**Logic:**

1. If `library` is None: call `read_library()`
2. If fewer than `config.RECALIBRATION_MIN_SONGS` songs: return `{"suggest": False, "reason": "insufficient_songs", "song_count": len(library), "average_scores": {}}`
3. Collect all `perceptual_score` dicts — skip entries with missing or incomplete scores (must have all three of `vibe`, `originality`, `coherence`)
4. Track `count_valid_scores` — number of entries with complete scores
5. If `count_valid_scores < config.RECALIBRATION_MIN_SONGS`: return `{"suggest": False, "reason": "insufficient_valid_scores", "song_count": len(library), "average_scores": {}}`
6. Compute average for each P dimension across valid scores: `vibe`, `originality`, `coherence`
7. Compute overall average across all three
8. Determine suggestion:
   - If overall average below `config.RECALIBRATION_PERCEPTUAL_FLOOR`: `suggest=True`, `reason="low_perceptual_average"`
   - Else if `originality` is the lowest of the three averages **and** the gap between `originality` and the next-lowest average is `>= config.RECALIBRATION_ORIGINALITY_MARGIN`: `suggest=True`, `reason="originality_consistently_low"`
   - Otherwise: `suggest=False`, `reason="scores_within_range"`
9. Return:

```python
{
    "suggest":        bool,
    "reason":         str,
    "song_count":     int,
    "average_scores": {
        "vibe":        float,
        "originality": float,
        "coherence":   float,
        "overall":     float
    }
}
```

Never raises. On any exception: return `{"suggest": False, "reason": "error", "song_count": 0, "average_scores": {}, "warning": str(e)}`.

---

### `run() -> dict`

Top-level function called from the daily pipeline for reporting purposes. Does not write anything.

```python
def run() -> dict:
    """
    Reads library, runs gap analysis and recalibration check.
    Returns a summary dict. Never raises.
    Used for daily pipeline reporting — Module 1 calls get_gap_analysis() directly.
    """
    try:
        library       = read_library()
        gap_analysis  = get_gap_analysis(library)
        recalibration = get_recalibration_suggestion(library)

        if recalibration["suggest"]:
            print(f"[WARN] Module 15 — recalibration suggested: {recalibration['reason']}")
            print(f"       Average scores: {recalibration['average_scores']}")
        else:
            print(f"[OK] Module 15 — no recalibration needed ({recalibration['song_count']} songs)")

        if gap_analysis["gaps"]:
            print(f"[OK] Module 15 — gaps detected: {gap_analysis['gaps']}")
        else:
            print(f"[OK] Module 15 — no gaps detected ({gap_analysis['song_count']} songs)")

        if gap_analysis.get("skipped_counts"):
            print(f"[WARN] Module 15 — skipped malformed values: {gap_analysis['skipped_counts']}")

        return {
            "module":  "module15",
            "status":  "ok",
            "flags":   (["recalibration_suggested"] if recalibration["suggest"] else []),
            "errors":  [],
            "data": {
                "song_count":     gap_analysis["song_count"],
                "gaps":           gap_analysis["gaps"],
                "gap_scores":     gap_analysis["gap_scores"],
                "skipped_counts": gap_analysis.get("skipped_counts", {}),
                "coverage":       gap_analysis["coverage"],
                "recalibration":  recalibration
            }
        }
    except Exception as e:
        print(f"[FAIL] Module 15 — {e}")
        return {
            "module": "module15", "status": "warn",
            "flags": [], "errors": [str(e)], "data": {}
        }
```

---

## Task 2 — Module 1 Patch: Wire Gap Analysis into Constraint Profile

Surgical addition to `backend/module1.py`. Do not restructure any existing logic.

### Where to insert

Find the section in `module1.run()` where the constraint profile dict is assembled and returned. It will look something like:

```python
constraint_profile = {
    "entropy":            entropy,
    "constraint_profile": { ... },
    ...
}
return constraint_profile
```

Before the `return`, add:

```python
# ── Phase 8: guarantee keys present before patch ───────────────────────────
constraint_profile.setdefault("gap_analysis", {})
constraint_profile.setdefault("diversity_targets", [])

# ── Phase 8: merge gap analysis into constraint profile ────────────────────
try:
    import backend.module15 as module15
    gap_analysis = module15.get_gap_analysis()
    constraint_profile["gap_analysis"]      = gap_analysis
    constraint_profile["diversity_targets"] = gap_analysis.get("gaps", [])
    if gap_analysis.get("gaps"):
        print(f"[OK] Module 1 — gap analysis merged: {gap_analysis['gaps']}")
    else:
        print(f"[OK] Module 1 — gap analysis merged: no gaps detected")
except Exception as e:
    print(f"[WARN] Module 1 — gap analysis unavailable: {e}")
    constraint_profile["gap_analysis"]      = {"gaps": [], "coverage": {}, "gap_reasons": {}, "gap_scores": {}, "skipped_counts": {}, "song_count": 0, "warning": str(e)}
    constraint_profile["diversity_targets"] = []
```

The `setdefault` calls before the try block guarantee that both keys are always present in the returned dict regardless of where future edits move code. The try/except ensures a Module 15 failure never blocks Module 1.

### Verification

```bash
python3.11 -c "
import backend.module1 as m1
import inspect
src = inspect.getsource(m1.run)
print('gap_analysis' in src)
print('diversity_targets' in src)
print('module15' in src)
print('setdefault' in src)
"
```

Expected: `True`, `True`, `True`, `True`

---

## Task 3 — Wire Module 15 into `run_daily.py`

Surgical addition to `backend/run_daily.py`. Add Module 15 reporting to **both** the State 1 and State 2 branches.

### In the State 2 branch

After `_write_session_index()` and `_mirror_library()`, before `print("[OK] State 2 daily run complete")`, add:

```python
import backend.module15 as m15
print("--- Module 15: Library Feedback ---")
try:
    result_m15 = m15.run()
    if result_m15["data"].get("recalibration", {}).get("suggest"):
        print(f"  [ACTION REQUIRED] Style anchor recalibration suggested — check Module 15 output")
except Exception as e:
    print(f"[WARN] Module 15 — {e}")
print()
```

### In the State 1 branch

After the existing Step 8 (email sent), before `sys.exit(0)`, add the same block:

```python
import backend.module15 as m15
print("--- Module 15: Library Feedback ---")
try:
    result_m15 = m15.run()
    if result_m15["data"].get("recalibration", {}).get("suggest"):
        print(f"  [ACTION REQUIRED] Style anchor recalibration suggested — check Module 15 output")
except Exception as e:
    print(f"[WARN] Module 15 — {e}")
print()
```

### Verification

```bash
python3.11 -c "
import backend.run_daily as rd
import inspect
src = inspect.getsource(rd._run_pipeline)
print('module15' in src)
"
```

Expected: `True`

---

## Task 4 — Verification Script

Create `backend/verify_phase8.py`:

```python
from pathlib import Path
import sys, importlib, inspect

ROOT       = Path(__file__).resolve().parents[1]
AUDIO_FILE = ROOT / "data" / "audio" / "heavy_gauge.mp3"
failed     = False

print("=" * 60)
print("MCS Phase 8 — Verification")
print("=" * 60)
print()

# ── Import checks ──────────────────────────────────────────────────────────
for mod_path in ["backend.config", "backend.module1", "backend.module15"]:
    try:
        importlib.import_module(mod_path)
        print(f"[OK] {mod_path} — imported")
    except Exception as e:
        print(f"[FAIL] {mod_path} — {e}")
        failed = True

if failed:
    sys.exit(1)

import backend.config   as cfg
import backend.module1  as m1
import backend.module15 as m15

# ── Config: Phase 8 constants ─────────────────────────────────────────────
try:
    for attr in ["RECALIBRATION_MIN_SONGS", "RECALIBRATION_PERCEPTUAL_FLOOR",
                 "RECALIBRATION_ORIGINALITY_MARGIN", "GAP_SCALAR_RANGE_THRESHOLD",
                 "GAP_COSINE_SIMILARITY_FLOOR", "GAP_SCALAR_DENOMINATOR_CLAMP"]:
        assert hasattr(cfg, attr), f"Missing: {attr}"
    print("[OK] config.py — all Phase 8 constants present")
except AssertionError as e:
    print(f"[FAIL] config.py — {e}")
    failed = True

# ── Module 15: read_library on missing file ───────────────────────────────
try:
    lib = m15.read_library()
    assert isinstance(lib, list)
    print("[OK] module15.read_library — returns list, never raises on missing file")
except Exception as e:
    print(f"[FAIL] module15.read_library — {e}")
    failed = True

# ── Module 15: _cosine_similarity sanity checks ───────────────────────────
try:
    sim_identical   = m15._cosine_similarity([1.0, 0.5, 0.2], [1.0, 0.5, 0.2])
    sim_orthogonal  = m15._cosine_similarity([1.0, 0.0], [0.0, 1.0])
    sim_zero        = m15._cosine_similarity([0.0, 0.0], [1.0, 0.5])
    sim_mismatch    = m15._cosine_similarity([1.0, 0.5], [1.0, 0.5, 0.2])
    assert abs(sim_identical  - 1.0) < 1e-5, f"Expected ~1.0, got {sim_identical}"
    assert abs(sim_orthogonal)        < 1e-5, f"Expected ~0.0, got {sim_orthogonal}"
    assert sim_zero     == 0.0,               f"Expected 0.0 for zero vector, got {sim_zero}"
    assert sim_mismatch == 0.0,               f"Expected 0.0 for mismatched lengths, got {sim_mismatch}"
    print("[OK] module15._cosine_similarity — identical=1.0, orthogonal=0.0, zero=0.0, mismatch=0.0")
except Exception as e:
    print(f"[FAIL] module15._cosine_similarity — {e}")
    failed = True

# ── Module 15: _is_valid_vector ───────────────────────────────────────────
try:
    assert m15._is_valid_vector([1.0, 2.0])        is True,  "valid list failed"
    assert m15._is_valid_vector([1.0])              is False, "single-element passed"
    assert m15._is_valid_vector(None)               is False, "None passed"
    assert m15._is_valid_vector("bad")              is False, "string passed"
    assert m15._is_valid_vector([])                 is False, "empty list passed"
    assert m15._is_valid_vector(["a", "b"])         is False, "non-numeric passed"
    assert m15._is_valid_vector([None, 1.0])        is False, "None element passed"
    print("[OK] module15._is_valid_vector — all cases correct")
except AssertionError as e:
    print(f"[FAIL] module15._is_valid_vector — {e}")
    failed = True

# ── Module 15: get_gap_analysis — empty library ───────────────────────────
try:
    result = m15.get_gap_analysis([])
    for key in ["gaps", "coverage", "gap_scores", "skipped_counts", "song_count"]:
        assert key in result, f"Missing key: {key}"
    assert result["song_count"] == 0
    print("[OK] module15.get_gap_analysis — empty library returns valid structure with all keys")
except Exception as e:
    print(f"[FAIL] module15.get_gap_analysis empty — {e}")
    failed = True

# ── Module 15: get_gap_analysis — 1 song insufficient ────────────────────
try:
    result = m15.get_gap_analysis([{"diversity_vector": {"tempo": 128}}])
    assert result.get("warning") == "insufficient_library"
    print("[OK] module15.get_gap_analysis — 1 song returns insufficient_library warning")
except Exception as e:
    print(f"[FAIL] module15.get_gap_analysis 1 song — {e}")
    failed = True

# ── Module 15: missing_dimension detected as gap ──────────────────────────
try:
    missing_dim = [
        {"diversity_vector": {"tempo": 128, "spectral_centroid": 2000.0}},
        {"diversity_vector": {"tempo": 130, "spectral_centroid": 2002.0}},
    ]
    result = m15.get_gap_analysis(missing_dim)
    # energy_curve_shape and all curve dims are absent — should be flagged missing_dimension
    assert "energy_curve_shape" in result["gaps"], \
        f"Expected energy_curve_shape missing_dimension gap, got gaps: {result['gaps']}"
    assert result["gap_reasons"].get("energy_curve_shape") == "missing_dimension", \
        f"Expected reason missing_dimension, got: {result['gap_reasons'].get('energy_curve_shape')}"
    assert result["coverage"].get("energy_curve_shape") == "gap:missing_dimension", \
        f"Expected coverage gap:missing_dimension, got: {result['coverage'].get('energy_curve_shape')}"
    print(f"[OK] module15.get_gap_analysis — missing dimension detected as gap: {[g for g in result['gaps'] if 'missing' in (result['gap_reasons'].get(g) or '')]}")
except Exception as e:
    print(f"[FAIL] module15.get_gap_analysis missing_dimension — {e}")
    failed = True

# ── Module 15: scalar denominator clamp near zero ─────────────────────────
try:
    near_zero = [
        {"diversity_vector": {"tempo": 0.0000001, "spectral_centroid": 2000.0}},
        {"diversity_vector": {"tempo": 0.0000002, "spectral_centroid": 2001.0}},
    ]
    result = m15.get_gap_analysis(near_zero)
    score  = result["gap_scores"].get("tempo", None)
    assert score is not None
    assert score == 0.0 or score < cfg.GAP_SCALAR_RANGE_THRESHOLD, \
        f"Expected 0.0 or below threshold for near-zero tempo, got {score}"
    print(f"[OK] module15.get_gap_analysis — scalar denominator clamp handles near-zero values: score={score}")
except Exception as e:
    print(f"[FAIL] module15.get_gap_analysis scalar clamp — {e}")
    failed = True

# ── Module 15: scalar ratio uses clamp only — no +1e-8 ───────────────────
try:
    src_m15 = inspect.getsource(m15.get_gap_analysis)
    assert "1e-8" not in src_m15, \
        "Found +1e-8 in get_gap_analysis — scalar ratio must use clamp only, no +1e-8"
    print("[OK] module15.get_gap_analysis — scalar ratio uses clamp only, no +1e-8")
except AssertionError as e:
    print(f"[FAIL] module15.get_gap_analysis scalar formula — {e}")
    failed = True

# ── Module 15: scalar non-numeric values skipped ─────────────────────────
try:
    bad_scalars = [
        {"diversity_vector": {"tempo": "128",  "spectral_centroid": 2000.0}},
        {"diversity_vector": {"tempo": None,   "spectral_centroid": 2001.0}},
        {"diversity_vector": {"tempo": 130,    "spectral_centroid": 2002.0}},
        {"diversity_vector": {"tempo": 131,    "spectral_centroid": 2003.0}},
    ]
    result = m15.get_gap_analysis(bad_scalars)
    skipped = result["skipped_counts"].get("tempo", 0)
    assert skipped >= 2, f"Expected at least 2 skipped for tempo, got {skipped}"
    print(f"[OK] module15.get_gap_analysis — non-numeric scalars skipped, skipped_counts: {result['skipped_counts']}")
except Exception as e:
    print(f"[FAIL] module15.get_gap_analysis scalar non-numeric — {e}")
    failed = True

# ── Module 15: scalar gap detection — narrow range ────────────────────────
try:
    narrow = [
        {"diversity_vector": {"tempo": 128, "spectral_centroid": 2000.0}},
        {"diversity_vector": {"tempo": 129, "spectral_centroid": 2001.0}},
        {"diversity_vector": {"tempo": 130, "spectral_centroid": 2002.0}},
    ]
    result = m15.get_gap_analysis(narrow)
    assert "tempo" in result["gaps"], f"Expected tempo gap, got: {result['gaps']}"
    assert "tempo" in result["gap_scores"]
    print(f"[OK] module15.get_gap_analysis — scalar gap on narrow tempo range, score={result['gap_scores']['tempo']:.4f}")
except Exception as e:
    print(f"[FAIL] module15.get_gap_analysis scalar gap — {e}")
    failed = True

# ── Module 15: curve gap detection — uniform library ─────────────────────
try:
    uniform = [
        {"diversity_vector": {"tempo": 128, "spectral_centroid": 2000.0,
                              "energy_curve_shape": [0.1, 0.5, 0.9],
                              "density_curve_shape": [0.2, 0.4, 0.8],
                              "harmonic_movement_shape": [0.3, 0.6, 0.9],
                              "beat_interval_distribution": [0.1, 0.2, 0.3],
                              "MFCC": [1.0, 2.0, 3.0], "chroma": [0.5, 0.5, 0.5]}},
        {"diversity_vector": {"tempo": 129, "spectral_centroid": 2001.0,
                              "energy_curve_shape": [0.1, 0.5, 0.9],
                              "density_curve_shape": [0.2, 0.4, 0.8],
                              "harmonic_movement_shape": [0.3, 0.6, 0.9],
                              "beat_interval_distribution": [0.1, 0.2, 0.3],
                              "MFCC": [1.0, 2.0, 3.0], "chroma": [0.5, 0.5, 0.5]}},
    ]
    result = m15.get_gap_analysis(uniform)
    assert len(result["gaps"]) > 0, f"Expected curve gaps on uniform library, got none: {result}"
    print(f"[OK] module15.get_gap_analysis — curve gaps detected: {result['gaps']}")
except Exception as e:
    print(f"[FAIL] module15.get_gap_analysis curve gap — {e}")
    failed = True

# ── Module 15: malformed vectors skipped, skipped_counts populated ─────────
try:
    malformed = [
        {"diversity_vector": {"tempo": 100, "energy_curve_shape": None}},
        {"diversity_vector": {"tempo": 160, "energy_curve_shape": [1]}},          # too short
        {"diversity_vector": {"tempo": 140, "energy_curve_shape": ["a", "b"]}},   # non-numeric
        {"diversity_vector": {"tempo":  80, "energy_curve_shape": [0.5, 0.9]}},   # valid
        {"diversity_vector": {"tempo": 120, "energy_curve_shape": [0.1, 0.3]}},   # valid
    ]
    result = m15.get_gap_analysis(malformed)
    assert isinstance(result["gaps"], list),           "gaps not a list"
    assert "skipped_counts" in result,                 "skipped_counts missing"
    skipped = result["skipped_counts"].get("energy_curve_shape", 0)
    assert skipped >= 3, f"Expected at least 3 skipped for energy_curve_shape, got {skipped}"
    print(f"[OK] module15.get_gap_analysis — malformed skipped silently, skipped_counts: {result['skipped_counts']}")
except Exception as e:
    print(f"[FAIL] module15.get_gap_analysis malformed — {e}")
    failed = True

# ── Module 15: pair sampling — two-pointer + odd middle ──────────────────
try:
    src = inspect.getsource(m15.get_gap_analysis)
    assert "n//2" in src or "// 2" in src, "Middle-vector pair logic not present in source"
    # Verify two-pointer pattern present
    assert "i = 0" in src or "i=0" in src, "Two-pointer i=0 not found in source"
    assert "j = n - 1" in src or "j=n-1" in src or "j = len" in src, \
        "Two-pointer j=n-1 not found in source"
    print("[OK] module15.get_gap_analysis — two-pointer pairing and middle-vector logic present")
except AssertionError as e:
    print(f"[FAIL] module15.get_gap_analysis pairing — {e}")
    failed = True

# ── Module 15: pair sampling capped at 10 ────────────────────────────────
try:
    src = inspect.getsource(m15.get_gap_analysis)
    assert "len(pairs) < 10" in src, "Pair cap of 10 not found in source"
    print("[OK] module15.get_gap_analysis — pair count capped at 10 in source")
except AssertionError as e:
    print(f"[FAIL] module15.get_gap_analysis pair cap — {e}")
    failed = True

# ── Module 15: diverse library runs without error ────────────────────────
try:
    diverse = [
        {"diversity_vector": {"tempo": 80,  "spectral_centroid": 1000.0,
                              "energy_curve_shape": [0.1, 0.2, 0.3],
                              "density_curve_shape": [0.1, 0.2, 0.3],
                              "harmonic_movement_shape": [0.1, 0.2, 0.3],
                              "beat_interval_distribution": [0.1, 0.2, 0.3],
                              "MFCC": [1.0, 0.0, 0.0], "chroma": [1.0, 0.0, 0.0]}},
        {"diversity_vector": {"tempo": 160, "spectral_centroid": 4000.0,
                              "energy_curve_shape": [0.9, 0.5, 0.1],
                              "density_curve_shape": [0.9, 0.5, 0.1],
                              "harmonic_movement_shape": [0.9, 0.5, 0.1],
                              "beat_interval_distribution": [0.9, 0.5, 0.1],
                              "MFCC": [0.0, 0.0, 1.0], "chroma": [0.0, 0.0, 1.0]}},
    ]
    result = m15.get_gap_analysis(diverse)
    assert isinstance(result["gaps"], list)
    print(f"[OK] module15.get_gap_analysis — diverse library: gaps={result['gaps']}")
except Exception as e:
    print(f"[FAIL] module15.get_gap_analysis diverse — {e}")
    failed = True

# ── Module 15: get_recalibration_suggestion — insufficient songs ──────────
try:
    result = m15.get_recalibration_suggestion([])
    assert result["suggest"] is False
    assert result["reason"]  == "insufficient_songs"
    print("[OK] module15.get_recalibration_suggestion — empty → suggest=False")
except Exception as e:
    print(f"[FAIL] module15.get_recalibration_suggestion empty — {e}")
    failed = True

# ── Module 15: recalibration — low scores trigger ─────────────────────────
try:
    low = [{"perceptual_score": {"vibe": 4, "originality": 3, "coherence": 4}}] * 5
    result = m15.get_recalibration_suggestion(low)
    assert result["suggest"] is True
    assert "average_scores" in result
    print(f"[OK] module15.get_recalibration_suggestion — low scores → suggest=True: {result['reason']}")
except Exception as e:
    print(f"[FAIL] module15.get_recalibration_suggestion low — {e}")
    failed = True

# ── Module 15: recalibration — healthy scores no trigger ──────────────────
try:
    good = [{"perceptual_score": {"vibe": 8, "originality": 7, "coherence": 8}}] * 5
    result = m15.get_recalibration_suggestion(good)
    assert result["suggest"] is False
    print("[OK] module15.get_recalibration_suggestion — healthy → suggest=False")
except Exception as e:
    print(f"[FAIL] module15.get_recalibration_suggestion healthy — {e}")
    failed = True

# ── Module 15: recalibration margin from config ───────────────────────────
try:
    src_m15 = inspect.getsource(m15.get_recalibration_suggestion)
    assert "RECALIBRATION_ORIGINALITY_MARGIN" in src_m15, \
        "Hardcoded margin — must use config.RECALIBRATION_ORIGINALITY_MARGIN"
    print("[OK] module15.get_recalibration_suggestion — originality margin from config")
except AssertionError as e:
    print(f"[FAIL] module15 recalibration margin — {e}")
    failed = True

# ── Module 15: run() — envelope shape ─────────────────────────────────────
try:
    result = m15.run()
    assert result["module"] == "module15"
    assert result["status"] in {"ok", "warn"}
    for key in ["gaps", "gap_scores", "skipped_counts", "song_count", "recalibration"]:
        assert key in result["data"], f"Missing from data: {key}"
    print("[OK] module15.run — correct envelope including gap_scores and skipped_counts")
except Exception as e:
    print(f"[FAIL] module15.run shape — {e}")
    failed = True

# ── Module 1 patch: keys in source ───────────────────────────────────────
try:
    src_m1 = inspect.getsource(m1.run)
    assert "gap_analysis"      in src_m1
    assert "diversity_targets" in src_m1
    assert "module15"          in src_m1
    assert "setdefault"        in src_m1
    print("[OK] module1.run — gap_analysis, diversity_targets, module15, setdefault in source")
except AssertionError as e:
    print(f"[FAIL] module1 patch source — {e}")
    failed = True

# ── Module 1 patch: keys in returned profile ─────────────────────────────
try:
    result_m1 = m1.run()
    assert "gap_analysis"      in result_m1
    assert "diversity_targets" in result_m1
    ga = result_m1["gap_analysis"]
    for key in ["gaps", "coverage", "gap_scores", "skipped_counts", "song_count"]:
        assert key in ga, f"Missing from gap_analysis: {key}"
    assert isinstance(result_m1["diversity_targets"], list)
    print(f"[OK] module1.run — gap_analysis and diversity_targets present (songs: {ga['song_count']})")
except Exception as e:
    print(f"[FAIL] module1 patch output — {e}")
    failed = True

# ── Module 1 patch: Module 15 failure does not block Module 1 ─────────────
try:
    original_read    = m15.read_library
    m15.read_library = lambda: (_ for _ in ()).throw(RuntimeError("simulated failure"))
    result_m1        = m1.run()
    assert "gap_analysis"      in result_m1
    assert "diversity_targets" in result_m1
    assert result_m1["gap_analysis"].get("warning") is not None
    assert result_m1["diversity_targets"] == []
    print("[OK] module1 patch — Module 15 failure does not block Module 1")
    m15.read_library = original_read
except Exception as e:
    m15.read_library = original_read
    print(f"[FAIL] module1 patch resilience — {e}")
    failed = True

# ── run_daily.py: module15 wired in ───────────────────────────────────────
try:
    import backend.run_daily as rd
    src_rd = inspect.getsource(rd._run_pipeline)
    assert "module15" in src_rd
    print("[OK] run_daily._run_pipeline — module15 present")
except Exception as e:
    print(f"[FAIL] run_daily module15 — {e}")
    failed = True

print()
if failed:
    print("[FAIL] Phase 8 verification failed — fix all failures before proceeding")
    sys.exit(1)
else:
    print("[OK] Phase 8 verification complete — all checks passed")
    sys.exit(0)
```

Run it:

```bash
python3.11 backend/verify_phase8.py
```

---

## Task 5 — Integration Test

Create `backend/integration_test_phase8.py`:

```python
"""
MCS Phase 8 — Integration Test: Full Feedback Loop

Does not write to the real song_library.json. Uses a temporary patch
on module15.read_library for the duration of the test.
"""

from pathlib import Path
import sys

ROOT   = Path(__file__).resolve().parents[1]
failed = False

print("=" * 60)
print("MCS Phase 8 — Integration Test: Feedback Loop")
print("=" * 60)
print()

import backend.module1  as m1
import backend.module15 as m15

def check(condition, label, detail=""):
    global failed
    if condition:
        print(f"[OK] {label}")
    else:
        print(f"[FAIL] {label}" + (f" — {detail}" if detail else ""))
        failed = True

FAKE_LIBRARY = [
    {
        "session_id": "aaa111", "title": "dark_trap_128bpm",
        "mood": "dark", "genre": "trap", "tempo_bpm": 128,
        "perceptual_score": {"vibe": 8, "originality": 7, "coherence": 8},
        "diversity_vector": {
            "tempo": 128, "spectral_centroid": 2500.0,
            "energy_curve_shape":         [0.2, 0.5, 0.9, 0.8],
            "density_curve_shape":        [0.3, 0.6, 0.8, 0.7],
            "harmonic_movement_shape":    [0.1, 0.4, 0.7, 0.9],
            "beat_interval_distribution": [0.2, 0.3, 0.4, 0.1],
            "MFCC": [1.0, 0.5, 0.2, 0.1], "chroma": [0.8, 0.1, 0.1, 0.0],
        },
    },
    {
        "session_id": "bbb222", "title": "bright_house_140bpm",
        "mood": "bright", "genre": "house", "tempo_bpm": 140,
        "perceptual_score": {"vibe": 9, "originality": 8, "coherence": 9},
        "diversity_vector": {
            "tempo": 140, "spectral_centroid": 4000.0,
            "energy_curve_shape":         [0.9, 0.8, 0.7, 0.6],
            "density_curve_shape":        [0.8, 0.7, 0.6, 0.5],
            "harmonic_movement_shape":    [0.9, 0.7, 0.5, 0.3],
            "beat_interval_distribution": [0.4, 0.3, 0.2, 0.1],
            "MFCC": [0.1, 0.5, 0.8, 1.0], "chroma": [0.1, 0.2, 0.8, 0.9],
        },
    },
    {
        "session_id": "ccc333", "title": "tense_dnb_170bpm",
        "mood": "tense", "genre": "drum_and_bass", "tempo_bpm": 170,
        "perceptual_score": {"vibe": 7, "originality": 9, "coherence": 7},
        "diversity_vector": {
            "tempo": 170, "spectral_centroid": 3200.0,
            "energy_curve_shape":         [0.5, 0.9, 0.4, 0.8],
            "density_curve_shape":        [0.4, 0.8, 0.3, 0.9],
            "harmonic_movement_shape":    [0.6, 0.3, 0.9, 0.2],
            "beat_interval_distribution": [0.1, 0.4, 0.1, 0.4],
            "MFCC": [0.5, 0.9, 0.3, 0.7], "chroma": [0.3, 0.6, 0.4, 0.7],
        },
    },
]

# ── Stage 1: Gap analysis — empty library ────────────────────────────────
print("--- Stage 1: Gap analysis — empty library ---")
try:
    r = m15.get_gap_analysis([])
    check(r["song_count"] == 0,            "song_count=0")
    check(isinstance(r["gaps"], list),     "gaps is list")
    check("skipped_counts" in r,           "skipped_counts key present")
    print()
except Exception as e:
    print(f"[FAIL] Stage 1 — {e}"); failed = True

# ── Stage 2: Gap analysis — 3 diverse songs ──────────────────────────────
print("--- Stage 2: Gap analysis — 3 diverse songs ---")
try:
    r3 = m15.get_gap_analysis(FAKE_LIBRARY)
    check(r3["song_count"] == 3,                  "song_count=3")
    check(isinstance(r3["gaps"], list),            "gaps is list")
    check(isinstance(r3["gap_scores"], dict),      "gap_scores is dict")
    check(isinstance(r3["skipped_counts"], dict),  "skipped_counts is dict")
    check(len(r3["coverage"]) > 0,                 "coverage has entries")
    print(f"  Gaps: {r3['gaps']}")
    print(f"  Gap scores: {r3['gap_scores']}")
    print()
except Exception as e:
    print(f"[FAIL] Stage 2 — {e}"); failed = True

# ── Stage 3: Recalibration — healthy scores ───────────────────────────────
print("--- Stage 3: Recalibration — healthy scores ---")
try:
    rc = m15.get_recalibration_suggestion(FAKE_LIBRARY)
    check("suggest"        in rc, "suggest key present")
    check("average_scores" in rc, "average_scores present")
    check(rc["song_count"] == 3,  "song_count=3")
    print(f"  suggest={rc['suggest']}, reason={rc['reason']}")
    print(f"  scores={rc['average_scores']}")
    print()
except Exception as e:
    print(f"[FAIL] Stage 3 — {e}"); failed = True

# ── Stage 4: Module 1 — populated library ────────────────────────────────
print("--- Stage 4: Module 1 — patched populated library ---")
ga_populated = {}
try:
    orig             = m15.read_library
    m15.read_library = lambda: FAKE_LIBRARY
    r_pop            = m1.run()
    ga_populated     = r_pop.get("gap_analysis", {})
    check("gap_analysis"      in r_pop,          "gap_analysis key present")
    check("diversity_targets" in r_pop,          "diversity_targets key present")
    check(ga_populated.get("song_count") == 3,   "gap_analysis sees 3 songs")
    check("gap_scores"    in ga_populated,       "gap_scores present")
    check("skipped_counts" in ga_populated,      "skipped_counts present")
    check(isinstance(r_pop["diversity_targets"], list), "diversity_targets is list")
    m15.read_library = orig
    print(f"  gap_analysis: {ga_populated}")
    print()
except Exception as e:
    m15.read_library = orig
    print(f"[FAIL] Stage 4 — {e}"); failed = True

# ── Stage 5: Module 1 — empty library ────────────────────────────────────
print("--- Stage 5: Module 1 — patched empty library ---")
ga_empty = {}
try:
    orig             = m15.read_library
    m15.read_library = lambda: []
    r_emp            = m1.run()
    ga_empty         = r_emp.get("gap_analysis", {})
    check("gap_analysis"      in r_emp,         "gap_analysis key present")
    check("diversity_targets" in r_emp,         "diversity_targets key present")
    check(ga_empty.get("song_count", -1) == 0,  "gap_analysis sees 0 songs")
    m15.read_library = orig
    print(f"  gap_analysis (empty): {ga_empty}")
    print()
except Exception as e:
    m15.read_library = orig
    print(f"[FAIL] Stage 5 — {e}"); failed = True

# ── Stage 6: Behavioral difference — empty vs populated ──────────────────
print("--- Stage 6: Constraint profile differs — empty vs populated ---")
try:
    check(
        ga_populated.get("song_count") != ga_empty.get("song_count"),
        "song_count differs",
        f"populated={ga_populated.get('song_count')}, empty={ga_empty.get('song_count')}"
    )
    populated_gaps = set(ga_populated.get("gaps", []))
    empty_gaps     = set(ga_empty.get("gaps", []))
    check(
        populated_gaps != empty_gaps,
        "gaps list differs between empty and populated library",
        f"populated={populated_gaps}, empty={empty_gaps}"
    )
    print()
except Exception as e:
    print(f"[FAIL] Stage 6 — {e}"); failed = True

# ── Stage 7: Missing dimension detected in partial library ────────────────
print("--- Stage 7: Missing dimension — partial diversity vectors ---")
try:
    partial = [
        {"diversity_vector": {"tempo": 128, "spectral_centroid": 2000.0}},
        {"diversity_vector": {"tempo": 140, "spectral_centroid": 2500.0}},
    ]
    r_partial = m15.get_gap_analysis(partial)
    check(
        "energy_curve_shape" in r_partial["gaps"],
        "energy_curve_shape flagged as missing_dimension gap",
        f"gaps: {r_partial['gaps']}"
    )
    check(
        r_partial["gap_reasons"].get("energy_curve_shape") == "missing_dimension",
        "gap_reason is missing_dimension",
        f"reason: {r_partial['gap_reasons'].get('energy_curve_shape')}"
    )
    print()
except Exception as e:
    print(f"[FAIL] Stage 7 — {e}"); failed = True

# ── Stage 8: Module 15 run() full reporting ───────────────────────────────
print("--- Stage 8: Module 15 run() ---")
try:
    r15 = m15.run()
    check(r15["module"] == "module15",                  "module key correct")
    check(r15["status"] in {"ok", "warn"},              "status ok or warn")
    for key in ["gaps", "gap_scores", "skipped_counts", "recalibration", "song_count"]:
        check(key in r15["data"], f"{key} in data")
    print()
except Exception as e:
    print(f"[FAIL] Stage 8 — {e}"); failed = True

print("=" * 60)
if failed:
    print("[FAIL] Integration test failed — fix before proceeding to Phase 9")
    sys.exit(1)
else:
    print("[OK] Integration test passed — feedback loop ready")
    sys.exit(0)
```

Run it:

```bash
python3.11 backend/integration_test_phase8.py
```

---

## Completion Gate

Phase 8 is complete when ALL of the following are true:

1. `verify_phase8.py` exits code 0 — all `[OK]`
2. `integration_test_phase8.py` exits code 0
3. `module15.read_library()` returns empty list on missing file — never raises
4. `module15._is_valid_vector()` returns False for None, single-element, empty, non-list, non-numeric elements
5. `module15._cosine_similarity()` returns 1.0 for identical, 0.0 for orthogonal, 0.0 for zero vectors, 0.0 for mismatched lengths
6. `module15.get_gap_analysis([])` returns valid structure with all keys including `skipped_counts`
7. `module15.get_gap_analysis()` with 1 song returns `warning: "insufficient_library"`
8. `module15.get_gap_analysis()` detects `missing_dimension` as a gap when a dimension is entirely absent across all songs — distinct from `insufficient_data`
9. `module15.get_gap_analysis()` detects scalar gap on narrow tempo range (128/129/130)
10. `module15.get_gap_analysis()` detects curve gaps on near-identical library
11. `module15.get_gap_analysis()` skips non-numeric scalar values silently, populates `skipped_counts` per dimension for both scalars and curves
12. `module15.get_gap_analysis()` skips malformed curve vectors silently, populates `skipped_counts` per dimension
13. `module15.get_gap_analysis()` runs without error on diverse library
14. Scalar range: symmetric formula `(max - min) / (abs(max) + abs(min))` with denominator clamp from `config.GAP_SCALAR_DENOMINATOR_CLAMP` — no `+ 1e-8` anywhere in scalar ratio computation
15. Curve gap: average pairwise similarity compared directly to `config.GAP_COSINE_SIMILARITY_FLOOR`
16. Curve pair sampling: sort by raw L2 norm ascending, two-pointer `i=0` / `j=n-1` while `i < j` and `len(pairs) < 10`; when vector count is odd include middle vector paired with `n//2 - 1`; stop at 10 pairs
17. `_cosine_similarity` returns 0.0 on mismatched vector lengths — no truncation
18. `gap_scores` dict present in output — raw metric per analyzed dimension (only for dimensions where computation was possible)
19. `skipped_counts` dict present in output — per-dimension count of skipped malformed or non-numeric values, initialized for all dimensions with skips
20. `module15.get_recalibration_suggestion([])` returns `suggest: False`, `reason: "insufficient_songs"`
21. `module15.get_recalibration_suggestion()` checks `count_valid_scores >= RECALIBRATION_MIN_SONGS` before originality rule fires
22. Originality margin uses `config.RECALIBRATION_ORIGINALITY_MARGIN` — not a hardcoded literal
23. `module15.get_recalibration_suggestion()` triggers on low average P score
24. `module15.get_recalibration_suggestion()` returns `suggest: False` on healthy scores
25. `module15.run()` envelope includes `gap_scores` and `skipped_counts` in `data`
26. `module1.run()` returns both `gap_analysis` and `diversity_targets` after patch
27. `module1.run()` contains `setdefault` guard for both keys before the try block
28. `diversity_targets` is a flat list of gap dimension name strings
29. Module 15 failure does not block Module 1 — both keys returned with fallback empty values
30. Module 15 wired into both State 1 and State 2 branches of `run_daily.py`
31. Module 15 never writes to `song_library.json`
32. Module 15 never writes session state or checkpoints
33. Recalibration suggestion is output-only — Style Anchor Vector never modified automatically
34. All six config constants present: `RECALIBRATION_MIN_SONGS`, `RECALIBRATION_PERCEPTUAL_FLOOR`, `RECALIBRATION_ORIGINALITY_MARGIN`, `GAP_SCALAR_RANGE_THRESHOLD`, `GAP_COSINE_SIMILARITY_FLOOR`, `GAP_SCALAR_DENOMINATOR_CLAMP`
35. No new dependencies added

---

## What You Must Not Do

- Do not build Phase 9 hardening tests
- Do not make any frontend changes
- Do not add dependencies beyond `requirements.txt`
- Do not write to `song_library.json` inside Module 15 — read only
- Do not write session state or checkpoints inside Module 15
- Do not auto-update the Style Anchor Vector — recalibration output is suggestion only
- Do not define threshold constants locally — import from `backend.config`
- Do not hardcode the originality margin — use `config.RECALIBRATION_ORIGINALITY_MARGIN`
- Do not hardcode absolute paths
- Do not let any Module 15 function raise — return warning dict on all error paths
- Do not let Module 15 failure block Module 1 — the patch is wrapped in try/except
- Do not average curve-shaped dimensions — use pairwise cosine similarity
- Do not compare curve similarity to `1.0 - GAP_COSINE_SIMILARITY_FLOOR` — compare directly to `GAP_COSINE_SIMILARITY_FLOOR`
- Do not use `+ 1e-8` in scalar ratio computation — use the denominator clamp exclusively
- Do not use `(max - min) / max` or asymmetric normalization for scalars — use `(max - min) / (abs(max) + abs(min))` with the clamp
- Do not treat a completely absent dimension as `insufficient_data` — it must be flagged as `missing_dimension` and added to gaps
- Do not accept non-numeric scalar values silently — cast to float, skip and count failures in `skipped_counts`
- Do not truncate mismatched vectors in `_cosine_similarity` — return 0.0 immediately on length mismatch
- Do not use sequential pairs for curve sampling — use two-pointer contrast-maximizing strategy
- Do not normalize vectors before norm-sorting — sort on raw L2 norm
- Do not omit `setdefault` guard in Module 1 patch — it must appear before the try block
- Do not omit `gap_scores` or `skipped_counts` from any `get_gap_analysis()` return path
- Do not omit `diversity_targets` from Module 1's returned constraint profile
- Do not continue past any failed task — stop and print diagnostic