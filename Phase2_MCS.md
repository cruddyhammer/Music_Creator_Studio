# MCS Phase 2 — Audio Analysis Foundation Agent Prompt (v2 Final)

## Context

You are building the audio analysis foundation for MCS (Music Creator Studio). This is Phase 2. Phase 0 (environment setup) and Phase 1 (data layer, Module 13, session JSON schema) are already complete and verified.

Phase 2 builds four things:
- Audio analysis utility — BPM, key, energy profile, MFCC, spectral centroid, chroma
- Diversity Feature Vector extractor — all 8 dimensions
- Song Identity Vector extractor — all 7 dimensions
- Cosine distance and rhythmic fingerprint comparators

Plus two unverified assumption tests:
- Test 1 — pydub stem layering sync check
- Test 2 — librosa segmentation loop point quality check

**No new system modules beyond audio analysis utilities and comparators. Integration with Module 13 is allowed strictly via its public functions.**

**Stop on first failure. Print `[OK]` / `[FAIL]` diagnostics. Do not continue to the next task.**

---

## Architectural Rules (Read Before Writing Any Code)

1. All session mutations must occur only through Module 13 public functions. Import and call `backend.module13` — never write to session dicts directly.
2. Every task boundary must write a checkpoint via `module13.write_checkpoint()` before passing control.
3. No new dependencies may be added. Use only what is already in `requirements.txt`: librosa, pydub, numpy, scipy.
4. All vector extractors must be deterministic — identical input must always produce identical output. Enforce this by using module-level constants `SAMPLE_RATE` and `HOP_LENGTH` in every librosa call. Never pass numeric literals to librosa functions. Use `center=False` explicitly in all STFT-based operations where the parameter is available.
5. Curve-shape cosine distance only — never compare scalar averages. All similarity comparisons operate on full feature curves.

---

## Path Rule (Mandatory — Read Before Writing Any Code)

Every script and module in Phase 2 that needs the repository root or the audio directory must define these two lines at the top, derived from its own file location:

```python
ROOT = Path(__file__).resolve().parents[1]
AUDIO_DIR = ROOT / "data" / "audio"
```

This is the single source of truth for all path resolution. Never hardcode absolute paths. Never assume a working directory. All references to `heavy_gauge.mp3` resolve to:

```
AUDIO_DIR / "heavy_gauge.mp3"
```

This rule applies to: `audio_analysis.py`, `diversity_vector.py`, `identity_vector.py`, `similarity.py`, `verify_phase2.py`, `test_pydub_sync.py`, `test_segmentation.py`.

---

## Preconditions

Confirm Phase 0 and Phase 1 are complete before starting:

1. Folder structure exists: `/backend`, `/data`, `/data/sessions`, `/data/audio`, `/temp`
2. `requirements.txt` is present and all packages installed
3. `backend/__init__.py` exists
4. `backend/module13.py` exists and imports cleanly
5. Test audio file exists at exactly: `AUDIO_DIR / "heavy_gauge.mp3"`

If any are missing, print `[FAIL] Precondition failed — <what is missing>` and stop.

For item 5 specifically: if `heavy_gauge.mp3` is not found, print:
```
[FAIL] heavy_gauge.mp3 not found at data/audio/heavy_gauge.mp3 — place the file there before running Phase 2
```
And stop. Do not search for it elsewhere. Do not accept any other path.

---

## Task 1 — Audio Analysis Utility

Create `/backend/audio_analysis.py`.

This module provides all low-level audio feature extraction. All future modules that need audio features import from here. It must never be bypassed.

### Constants

Define these once at the top of the file. Never hardcode these values anywhere else in the module:

```python
from pathlib import Path
import numpy as np
import librosa

ROOT = Path(__file__).resolve().parents[1]
AUDIO_DIR = ROOT / "data" / "audio"

SAMPLE_RATE = 22050
HOP_LENGTH = 512
N_MFCC = 13
BPM_TOLERANCE = 3.0
```

All librosa calls throughout this module must reference `SAMPLE_RATE` and `HOP_LENGTH` by name. Never pass numeric literals to librosa functions.

---

### `load_audio(filepath: str) -> tuple[np.ndarray, int]`

- Loads audio file using `librosa.load()` with `sr=SAMPLE_RATE`, `mono=True`
- Raises `FileNotFoundError` with clear message if file does not exist
- Returns `(y, sr)` where `y` is the audio time series and `sr` is the sample rate
- Never modifies the audio signal — load only

---

### `detect_bpm(y: np.ndarray, sr: int) -> float`

- Uses `librosa.beat.beat_track(y=y, sr=sr, hop_length=HOP_LENGTH)` to detect tempo
- Returns BPM as a float rounded to 2 decimal places
- Never raises — if detection fails for any reason, returns `0.0` and prints `[WARN] BPM detection failed — returning 0.0`

---

### `detect_key(y: np.ndarray, sr: int) -> str`

- Computes chromagram using `librosa.feature.chroma_cqt(y=y, sr=sr, hop_length=HOP_LENGTH)`
- Sums chroma energy across time to get per-pitch-class totals
- Returns the pitch class with highest energy as a string: one of `{C, C#, D, D#, E, F, F#, G, G#, A, A#, B}`
- This is a simplified key estimate — advisory only

---

### `extract_energy_profile(y: np.ndarray, sr: int) -> np.ndarray`

- Computes RMS energy using `librosa.feature.rms(y=y, hop_length=HOP_LENGTH)`
- Returns the RMS array flattened to 1D
- Normalizes to range [0.0, 1.0]: divide by max value. If max is 0, return array of zeros.

---

### `extract_mfcc(y: np.ndarray, sr: int) -> np.ndarray`

- Computes MFCC using `librosa.feature.mfcc(y=y, sr=sr, n_mfcc=N_MFCC, hop_length=HOP_LENGTH)`
- Returns mean across time axis: shape `(N_MFCC,)` — one value per coefficient
- This produces a fixed-length feature vector regardless of audio duration

---

### `extract_spectral_centroid(y: np.ndarray, sr: int) -> np.ndarray`

- Computes spectral centroid using `librosa.feature.spectral_centroid(y=y, sr=sr, hop_length=HOP_LENGTH)`
- Returns the centroid array flattened to 1D
- Normalizes to range [0.0, 1.0]: divide by Nyquist frequency `(sr / 2)`

---

### `extract_chroma(y: np.ndarray, sr: int) -> np.ndarray`

- Computes chromagram using `librosa.feature.chroma_cqt(y=y, sr=sr, hop_length=HOP_LENGTH)`
- Returns mean across time axis: shape `(12,)` — one value per pitch class
- Normalizes to range [0.0, 1.0]: divide by max value. If max is 0, return array of zeros.

---

### `extract_beat_intervals(y: np.ndarray, sr: int) -> np.ndarray`

- Uses `librosa.beat.beat_track(y=y, sr=sr, hop_length=HOP_LENGTH)` to get beat frame positions
- Converts beat frames to time in seconds using `librosa.frames_to_time(beats, sr=sr, hop_length=HOP_LENGTH)`
- Computes intervals as `np.diff()` on the beat times array
- Returns interval array as 1D numpy array
- If fewer than 2 beats detected, returns `np.array([])` and prints `[WARN] Fewer than 2 beats detected — beat interval array empty`

---

## Task 2 — Diversity Feature Vector Extractor

Create `/backend/diversity_vector.py`.

The Diversity Feature Vector is defined as:
`D = (MFCC, tempo, spectral_centroid, chroma, energy_curve_shape, density_curve_shape, harmonic_movement_shape, beat_interval_distribution)`

All 8 dimensions must be present.

### Constants

```python
from pathlib import Path
ROOT = Path(__file__).resolve().parents[1]
AUDIO_DIR = ROOT / "data" / "audio"
```

### `extract_diversity_vector(filepath: str) -> dict`

- Loads audio via `audio_analysis.load_audio(filepath)`
- Extracts all 8 dimensions by calling functions from `audio_analysis.py`:

```python
{
    "mfcc": extract_mfcc(y, sr).tolist(),
    "tempo": detect_bpm(y, sr),
    "spectral_centroid": extract_spectral_centroid(y, sr).tolist(),
    "chroma": extract_chroma(y, sr).tolist(),
    "energy_curve_shape": extract_energy_profile(y, sr).tolist(),
    "density_curve_shape": extract_energy_profile(y, sr).tolist(),        # placeholder — reuses energy profile until Module 6 provides stem density data
    "harmonic_movement_shape": extract_chroma(y, sr).tolist(),             # placeholder — reuses chroma shape until harmonic tracker built
    "beat_interval_distribution": extract_beat_intervals(y, sr).tolist()
}
```

- Returns the dict. All numpy arrays converted to plain Python lists for JSON serialization.
- Prints `[OK] Diversity vector extracted — <filename>`

**Note:** `density_curve_shape` and `harmonic_movement_shape` are explicitly marked as placeholders in code comments. They will be replaced in later phases. Do not attempt to build those now.

---

## Task 3 — Song Identity Vector Extractor

Create `/backend/identity_vector.py`.

The Song Identity Vector is defined as:
`I = (mood, energy_trajectory, density_trajectory, harmonic_center, energy_slope, peak_timing, density_volatility)`

All 7 dimensions must be present.

### Constants

```python
from pathlib import Path
ROOT = Path(__file__).resolve().parents[1]
AUDIO_DIR = ROOT / "data" / "audio"
```

### `extract_identity_vector(filepath: str, mood: str) -> dict`

- `mood` is passed in from the session `song_plan` — it is not derived from audio
- Loads audio via `audio_analysis.load_audio(filepath)`
- Computes all derived fields:

**`energy_trajectory`** — direction of energy over the track:
- Split energy profile into first half and second half
- If second half mean > first half mean: `"rising"`
- If second half mean < first half mean: `"falling"`
- Otherwise: `"flat"`

**`density_trajectory`** — placeholder, set to `"unknown"` until Module 6 provides stem density data. Mark with comment.

**`harmonic_center`** — call `audio_analysis.detect_key(y, sr)`

**`energy_slope`** — linear regression slope of the energy profile curve:
- Use `np.polyfit(x, energy_profile, 1)[0]` where `x = np.arange(len(energy_profile))`
- Returns float rounded to 6 decimal places

**`peak_timing`** — normalized position of peak energy:
- `peak_index / len(energy_profile)`
- Returns float in range [0.0, 1.0] rounded to 4 decimal places

**`density_volatility`** — placeholder, set to `0.0` until Module 6 provides stem density data. Mark with comment.

Returns:
```python
{
    "mood": mood,
    "energy_trajectory": "rising" | "falling" | "flat",
    "density_trajectory": "unknown",      # placeholder
    "harmonic_center": "C" | "C#" | ...,
    "energy_slope": float,
    "peak_timing": float,
    "density_volatility": 0.0             # placeholder
}
```

Prints `[OK] Identity vector extracted — <filename>`

---

### `write_identity_vector_to_session(session_id: str, filepath: str, mood: str) -> None`

- Calls `extract_identity_vector(filepath, mood)`
- Loads session via `module13.load_session(session_id)`
- Updates `session["identity_vector"]` with extracted values
- Saves via `module13._save_session(session)` — explicitly permitted here as a temporary bridge until Module 13 receives a dedicated `update_identity_vector` public function in a future phase
- Writes checkpoint: `module="identity_vector"`, `data={"event": "identity_vector_written", "filepath": filepath}`
- Prints `[OK] Identity vector written to session — <session_id>`

**Note to agent:** `module13._save_session` is an internal function. Its use is permitted only in this function and nowhere else outside `module13.py`. Do not call `_save_session` anywhere else in Phase 2.

---

## Task 4 — Similarity Comparators

Create `/backend/similarity.py`.

### Constants

```python
from pathlib import Path
ROOT = Path(__file__).resolve().parents[1]
AUDIO_DIR = ROOT / "data" / "audio"
```

### `cosine_similarity_curves(curve_a: list, curve_b: list) -> float`

- Accepts two 1D lists or numpy arrays of any length
- If lengths differ: resample the shorter one to match the longer using `np.interp`
- Converts both to numpy arrays
- Computes cosine similarity: `dot(a, b) / (norm(a) * norm(b))`
- If either norm is zero: returns `0.0`
- Returns float rounded to 6 decimal places

### `compare_beat_intervals(intervals_a: list, intervals_b: list) -> float`

- Accepts two beat interval arrays
- If either is empty: returns `0.0` and prints `[WARN] Beat interval comparison skipped — one or both arrays empty`
- Resamples shorter to match longer using `np.interp`
- Returns cosine similarity via `cosine_similarity_curves()`

### `compare_diversity_vectors(vec_a: dict, vec_b: dict) -> dict`

- Computes per-dimension similarity for all comparable dimensions:
  - `mfcc` — cosine similarity on the mean vector (shape 13, fixed length — no resampling needed)
  - `spectral_centroid` — `cosine_similarity_curves()`
  - `chroma` — cosine similarity on the mean vector (shape 12, fixed length — no resampling needed)
  - `energy_curve_shape` — `cosine_similarity_curves()`
  - `beat_interval_distribution` — `compare_beat_intervals()`
- Skips `tempo` (scalar), `density_curve_shape` (placeholder), `harmonic_movement_shape` (placeholder)
- Returns:
```python
{
    "mfcc": float,
    "spectral_centroid": float,
    "chroma": float,
    "energy_curve_shape": float,
    "beat_interval_distribution": float,
    "overall": float   # mean of all computed dimension scores
}
```

---

## Task 5 — Unverified Assumption Tests

These are standalone test scripts. They do not write to sessions. They produce documented results only.

### Constants (both scripts)

Both scripts must define at the top:

```python
from pathlib import Path
ROOT = Path(__file__).resolve().parents[1]
AUDIO_DIR = ROOT / "data" / "audio"
AUDIO_FILE = AUDIO_DIR / "heavy_gauge.mp3"
```

---

### Test 1 — Pydub Stem Layering Sync

Create `/backend/test_pydub_sync.py`:

- Loads `AUDIO_FILE` using pydub `AudioSegment.from_file()`
- Creates a second instance by reloading the same file from disk
- Overlays them using pydub's `overlay()` method
- Exports the result to `ROOT / "temp" / "sync_test_output.wav"`
- Measures duration of original vs layered output
- 50ms threshold — delta below this is sync-safe
- Prints result:

```
[OK] Pydub sync test complete
Original duration: X.XXs
Layered duration: X.XXs
Duration delta: X.XXs
[RESULT] SYNC CONFIRMED | [RESULT] DRIFT DETECTED
```

- Does not assert pass/fail — documents result for Phase 3 planning

---

### Test 2 — Librosa Segmentation Loop Point Quality

Create `/backend/test_segmentation.py`:

- Loads `AUDIO_FILE` using `audio_analysis.load_audio()`
- Detects beat frames using `librosa.beat.beat_track(y=y, sr=sr, hop_length=HOP_LENGTH)` — must use module constants from `audio_analysis`
- Selects a segment: beats 4 through 12 (or fewer if file is short)
- Computes RMS energy at the start sample and end sample of the segment
- 0.05 RMS delta threshold
- Prints result:

```
[OK] Segmentation test complete
Segment: beat 4 → beat 12
Start RMS: X.XXXXXX
End RMS: X.XXXXXX
RMS delta: X.XXXXXX
[RESULT] CLEAN LOOP POINT | [RESULT] ROUGH LOOP POINT
```

- Does not assert pass/fail — documents result for Module 6 planning

---

## Task 6 — Verification Script

Create `/backend/verify_phase2.py`:

```python
from pathlib import Path
import sys
import numpy as np

ROOT = Path(__file__).resolve().parents[1]
AUDIO_DIR = ROOT / "data" / "audio"
AUDIO_FILE = AUDIO_DIR / "heavy_gauge.mp3"

failed = False

# --- Precondition: test audio file ---
if not AUDIO_FILE.exists():
    print("[FAIL] heavy_gauge.mp3 not found at data/audio/heavy_gauge.mp3 — place the file there before running Phase 2")
    sys.exit(1)
else:
    print("[OK] heavy_gauge.mp3 found")

# --- Import checks ---
try:
    import backend.audio_analysis as aa
    print("[OK] audio_analysis — imported successfully")
except Exception as e:
    print(f"[FAIL] audio_analysis — {e}")
    sys.exit(1)

try:
    import backend.diversity_vector as dv
    print("[OK] diversity_vector — imported successfully")
except Exception as e:
    print(f"[FAIL] diversity_vector — {e}")
    sys.exit(1)

try:
    import backend.identity_vector as iv
    print("[OK] identity_vector — imported successfully")
except Exception as e:
    print(f"[FAIL] identity_vector — {e}")
    sys.exit(1)

try:
    import backend.similarity as sim
    print("[OK] similarity — imported successfully")
except Exception as e:
    print(f"[FAIL] similarity — {e}")
    sys.exit(1)

try:
    import backend.module13 as m13
    print("[OK] module13 — imported successfully")
except Exception as e:
    print(f"[FAIL] module13 — {e}")
    sys.exit(1)

# --- Audio loading ---
try:
    y, sr = aa.load_audio(str(AUDIO_FILE))
    assert isinstance(y, np.ndarray), "y must be numpy array"
    assert sr == aa.SAMPLE_RATE, f"sr must be {aa.SAMPLE_RATE}"
    print("[OK] load_audio")
except Exception as e:
    print(f"[FAIL] load_audio — {e}")
    failed = True

# --- BPM detection ---
try:
    bpm = aa.detect_bpm(y, sr)
    assert isinstance(bpm, float), "BPM must be float"
    assert bpm > 0, "BPM must be positive"
    print(f"[OK] detect_bpm — {bpm} BPM")
except Exception as e:
    print(f"[FAIL] detect_bpm — {e}")
    failed = True

# --- Key detection ---
try:
    key = aa.detect_key(y, sr)
    valid_keys = {"C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"}
    assert key in valid_keys, f"Key {key} not in valid set"
    print(f"[OK] detect_key — {key}")
except Exception as e:
    print(f"[FAIL] detect_key — {e}")
    failed = True

# --- Energy profile ---
try:
    energy = aa.extract_energy_profile(y, sr)
    assert isinstance(energy, np.ndarray), "Energy must be numpy array"
    assert energy.min() >= 0.0 and energy.max() <= 1.0, "Energy must be normalized [0, 1]"
    print(f"[OK] extract_energy_profile — shape {energy.shape}")
except Exception as e:
    print(f"[FAIL] extract_energy_profile — {e}")
    failed = True

# --- MFCC ---
try:
    mfcc = aa.extract_mfcc(y, sr)
    assert mfcc.shape == (aa.N_MFCC,), f"MFCC must have shape ({aa.N_MFCC},)"
    print(f"[OK] extract_mfcc — shape {mfcc.shape}")
except Exception as e:
    print(f"[FAIL] extract_mfcc — {e}")
    failed = True

# --- Spectral centroid ---
try:
    centroid = aa.extract_spectral_centroid(y, sr)
    assert isinstance(centroid, np.ndarray)
    assert centroid.min() >= 0.0 and centroid.max() <= 1.0, "Centroid must be normalized [0, 1]"
    print(f"[OK] extract_spectral_centroid — shape {centroid.shape}")
except Exception as e:
    print(f"[FAIL] extract_spectral_centroid — {e}")
    failed = True

# --- Chroma ---
try:
    chroma = aa.extract_chroma(y, sr)
    assert chroma.shape == (12,), "Chroma must have shape (12,)"
    assert chroma.min() >= 0.0 and chroma.max() <= 1.0, "Chroma must be normalized [0, 1]"
    print(f"[OK] extract_chroma — shape {chroma.shape}")
except Exception as e:
    print(f"[FAIL] extract_chroma — {e}")
    failed = True

# --- Beat intervals ---
try:
    intervals = aa.extract_beat_intervals(y, sr)
    assert isinstance(intervals, np.ndarray)
    print(f"[OK] extract_beat_intervals — {len(intervals)} intervals")
except Exception as e:
    print(f"[FAIL] extract_beat_intervals — {e}")
    failed = True

# --- Diversity vector ---
try:
    vec = dv.extract_diversity_vector(str(AUDIO_FILE))
    required_keys = ["mfcc", "tempo", "spectral_centroid", "chroma",
                     "energy_curve_shape", "density_curve_shape",
                     "harmonic_movement_shape", "beat_interval_distribution"]
    for key in required_keys:
        assert key in vec, f"Missing diversity vector key: {key}"
    print("[OK] extract_diversity_vector — all 8 dimensions present")
except Exception as e:
    print(f"[FAIL] extract_diversity_vector — {e}")
    failed = True

# --- Determinism check: extract twice, compare ---
try:
    vec_a = dv.extract_diversity_vector(str(AUDIO_FILE))
    vec_b = dv.extract_diversity_vector(str(AUDIO_FILE))
    for key in ["mfcc", "chroma"]:
        a = np.array(vec_a[key])
        b = np.array(vec_b[key])
        assert np.allclose(a, b), f"Non-deterministic output on dimension: {key}"
    print("[OK] diversity vector determinism — identical output on two extractions")
except Exception as e:
    print(f"[FAIL] diversity vector determinism — {e}")
    failed = True

# --- Identity vector ---
try:
    ivec = iv.extract_identity_vector(str(AUDIO_FILE), mood="dark")
    required_keys = ["mood", "energy_trajectory", "density_trajectory",
                     "harmonic_center", "energy_slope", "peak_timing", "density_volatility"]
    for key in required_keys:
        assert key in ivec, f"Missing identity vector key: {key}"
    assert ivec["mood"] == "dark"
    assert ivec["energy_trajectory"] in {"rising", "falling", "flat"}
    assert ivec["harmonic_center"] in {"C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"}
    assert 0.0 <= ivec["peak_timing"] <= 1.0
    print(f"[OK] extract_identity_vector — trajectory: {ivec['energy_trajectory']}, key: {ivec['harmonic_center']}")
except Exception as e:
    print(f"[FAIL] extract_identity_vector — {e}")
    failed = True

# --- Cosine similarity — identical vectors must return 1.0 ---
try:
    curve = [0.1, 0.5, 0.9, 0.4, 0.2]
    result = sim.cosine_similarity_curves(curve, curve)
    assert abs(result - 1.0) < 1e-5, f"Expected 1.0, got {result}"
    print("[OK] cosine_similarity_curves — identical vectors return 1.0")
except Exception as e:
    print(f"[FAIL] cosine_similarity_curves — {e}")
    failed = True

# --- Cosine similarity — different length curves ---
try:
    a = [0.1, 0.5, 0.9, 0.4, 0.2]
    b = [0.1, 0.5, 0.9]
    result = sim.cosine_similarity_curves(a, b)
    assert 0.0 <= result <= 1.0, f"Result out of range: {result}"
    print(f"[OK] cosine_similarity_curves — different lengths handled, result: {result}")
except Exception as e:
    print(f"[FAIL] cosine_similarity_curves — different lengths — {e}")
    failed = True

# --- Cosine similarity — zero vector returns 0.0 ---
try:
    result = sim.cosine_similarity_curves([0.0, 0.0, 0.0], [0.1, 0.2, 0.3])
    assert result == 0.0, f"Expected 0.0 for zero vector, got {result}"
    print("[OK] cosine_similarity_curves — zero vector returns 0.0")
except Exception as e:
    print(f"[FAIL] cosine_similarity_curves — zero vector — {e}")
    failed = True

# --- Beat interval empty array edge case ---
try:
    result = sim.compare_beat_intervals([], [])
    assert result == 0.0, f"Expected 0.0 for empty intervals, got {result}"
    print("[OK] compare_beat_intervals — empty arrays return 0.0")
except Exception as e:
    print(f"[FAIL] compare_beat_intervals — empty array edge case — {e}")
    failed = True

# --- Full diversity vector comparison on same file (expect high similarity) ---
try:
    vec_a = dv.extract_diversity_vector(str(AUDIO_FILE))
    vec_b = dv.extract_diversity_vector(str(AUDIO_FILE))
    comparison = sim.compare_diversity_vectors(vec_a, vec_b)
    assert comparison["overall"] > 0.99, f"Same file overall similarity expected > 0.99, got {comparison['overall']}"
    print(f"[OK] compare_diversity_vectors — same file overall similarity: {comparison['overall']}")
except Exception as e:
    print(f"[FAIL] compare_diversity_vectors — {e}")
    failed = True

# --- Module 13 integration: write identity vector to a real session ---
try:
    session = m13.create_session({
        "title": "Phase2 Test",
        "mood": "dark",
        "energy": "driving",
        "tempo_bpm": 120,
        "target_duration_seconds": 180
    })
    sid = session["session_id"]
    iv.write_identity_vector_to_session(sid, str(AUDIO_FILE), mood="dark")
    loaded = m13.load_session(sid)
    assert loaded["identity_vector"]["mood"] == "dark"
    assert loaded["identity_vector"]["energy_trajectory"] in {"rising", "falling", "flat"}
    assert len(loaded["checkpoints"]) >= 2   # session_created + identity_vector_written
    print(f"[OK] write_identity_vector_to_session — identity vector in session, checkpoints: {len(loaded['checkpoints'])}")
except Exception as e:
    print(f"[FAIL] write_identity_vector_to_session — {e}")
    failed = True

print()
if failed:
    print("[FAIL] Phase 2 verification failed — fix all failures before proceeding")
    sys.exit(1)
else:
    print("[OK] Phase 2 verification complete — all checks passed")
    sys.exit(0)
```

Run it:

```bash
python3.11 backend/verify_phase2.py
```

Then run the assumption tests and document results:

```bash
python3.11 backend/test_pydub_sync.py
python3.11 backend/test_segmentation.py
```

Both assumption tests print their results to the console. Save the output — it informs Phase 3 planning.

---

## Completion Gate

Phase 2 is complete when ALL of the following are true:

1. `verify_phase2.py` exits code 0 — all `[OK]`
2. Full vector extraction runs on `heavy_gauge.mp3` without error
3. Cosine similarity of identical vectors returns > 0.99
4. Diversity vector determinism confirmed — two extractions of same file produce identical output
5. Identity vector written to a real session JSON with correct structure and checkpoint logged
6. Empty beat interval arrays handled cleanly — `compare_beat_intervals([], [])` returns `0.0`
7. `test_pydub_sync.py` result documented — `[RESULT] SYNC CONFIRMED` or `[RESULT] DRIFT DETECTED` recorded
8. `test_segmentation.py` result documented — `[RESULT] CLEAN LOOP POINT` or `[RESULT] ROUGH LOOP POINT` recorded
9. No new dependencies added beyond `requirements.txt`

---

## What You Must Not Do

- Do not build any pipeline modules (Module 1, 2, 3, 5, 6, 7, 8 — none of these)
- Do not add any audio generation or prompt logic
- Do not connect to Google Drive
- Do not build any frontend
- Do not add dependencies beyond what is already in `requirements.txt`
- Do not continue past any failed task — stop and print diagnostic
- Do not use scalar averages in cosine distance comparisons — curve shape only
- Do not mutate session dicts directly — all session mutations through Module 13 public functions
- Do not call `module13._save_session` anywhere except `identity_vector.py` where it is explicitly permitted
- Do not hardcode absolute paths — always derive from `ROOT = Path(__file__).resolve().parents[1]`
- Do not inline numeric values in librosa calls — always use `SAMPLE_RATE` and `HOP_LENGTH` constants