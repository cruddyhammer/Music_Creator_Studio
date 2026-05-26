**Music Creator Studio (MCS) — Architecture v2.1**

---

**Core Philosophy:**
A closed-loop adaptive control system. Not a linear pipeline. Constrain → Generate → Evaluate → Re-constrain. AI generates prompts only, never music directly. Stems are the atomic unit of composition. Human perception is the final authoritative override at every decision boundary. No final mix exists until manually triggered. Zero cost, free tier only. The system gets more calibrated to your taste the more you use it — early outputs will be generic, later outputs will reflect your actual sonic identity.

---

**Tech Stack:**
- Backend: Python, GitHub Actions
- Frontend: HTML + JavaScript, GitHub Pages
- Audio analysis: librosa
- Audio assembly: pydub
- Storage: Google Drive API (3 accounts = 45GB), GitHub repo (metadata + state authority), GitHub Releases (final songs archive)
- Notifications: Email (same pattern as MRE)
- Separate GitHub account from MRE (independent 2,000 Actions minutes/month)

---

**Foundational Design Rules:**

**Rule 1 — Single Source of Truth:**
GitHub JSON is the only authoritative state for every song, session, and system decision. Google Drive and GitHub Releases are asset stores only — they never define state. If any conflict exists between Drive contents and the JSON record, the JSON wins. Drive and Releases are always derived from JSON, never parallel to it.

**Rule 2 — Perceptual Score is Final Authority:**
The Perceptual Score P = (vibe, originality, coherence) can override any filter decision in either direction at any review boundary. If P scores high and the diversity filter flagged the session, it passes. If P scores low and all filters cleared the session, it is rejected. Feature space metrics inform decisions — they never make them.

**Rule 3 — Entropy Injection is Mandatory:**
Every generation cycle must include at least one dimension that is either unconstrained or randomly seeded from within an allowed range. This is not optional and not emergent. It is enforced by Module 1 before prompt generation begins. Prevents the system from converging into a safe variance zone over time where everything passes filters but output becomes stylistically uniform.

**Rule 4 — Checkpoint at Every Module:**
Every module writes its completion state to the session JSON before passing control to the next module. If any module fails — Drive upload, stem mismatch, API issue, anything — the system records exactly which module failed and can resume from that point. No work is lost due to mid-pipeline failure.

**Rule 5 — Stems are Never Merged Until Final Export:**
Stems remain independent files at all stages. No permanent mix exists until you manually trigger Module 14. The toggle interface always operates on independent stem files.

---

**System States:**

The system is always in exactly one of two states. Never both simultaneously.

**STATE 1 — Candidate Generation**
Default state. Runs every weekday until at least one candidate is selected at end of week review. If no candidate selected, State 1 continues the following week unchanged.

**STATE 2 — Refinement**
Activated when at least one candidate is approved at end of week review. Candidate generation stops completely. Multiple active projects can exist simultaneously and can be worked on in any order. Returns to State 1 only when you manually trigger it after all active projects are marked finished.

---

**Foundational Vectors:**
*(Defined at system level, referenced by all modules)*

**Song Identity Vector (per session):**
I = (mood, energy_trajectory, density_trajectory, harmonic_center, energy_slope, peak_timing, density_volatility)
Captures how the song moves and evolves over time, not just what it is at a static moment.

**Style Anchor Vector (system-wide, long-term):**
A = (genre_range, tempo_range, energy_profile, rhythmic_signature)
Long-term identity constraint for the entire system. Prevents stylistic drift across weeks and months. Initialized manually by you before first run. Updated only when you explicitly recalibrate it. Never updated automatically.

**Diversity Feature Vector (per song):**
D = (MFCC, tempo, spectral_centroid, chroma, energy_curve_shape, density_curve_shape, harmonic_movement_shape, beat_interval_distribution)
Multi-dimensional. Compares curve shapes not averages. Rhythmic fingerprint included as a separate dimension to catch the most common AI sameness signature.

**Perceptual Score (human input, per session):**
P = (vibe, originality, coherence)
Scored explicitly by you at every review boundary. Final authority over all filter decisions. Feeds the song library as taste profile data. Grows more meaningful as the library grows.

**Entropy Quota (per generation cycle):**
E_quota = at least one dimension from {identity, rhythm, energy_trajectory} must be randomly seeded within an allowed range per cycle. Enforced by Module 1. Logged in session JSON so entropy injection is auditable over time.

---

**STATE 1 — Candidate Generation Pipeline**

---

**Module 1: Pre-Generation Constraint Engine**
First module to run every cycle. Performs three tasks in order:

*Task A — Library scan:* Reads all approved songs from the song library JSON. Computes current feature space coverage across all diversity dimensions. Identifies underrepresented areas — what moods, tempos, rhythmic patterns, energy profiles are missing from your library.

*Task B — Constraint profile generation:* Produces a constraint profile the Prompt Generator must satisfy. Derived from Style Anchor Vector plus library gap analysis. Ensures prompts are guided by diversity requirements before any generation occurs.

*Task C — Entropy injection:* Selects at least one dimension from the entropy quota and seeds it randomly within the Style Anchor's allowed range. Records which dimension was randomized and the seeded value in the session JSON. This is enforced as a hard rule — no cycle runs without entropy injection.

Output: Constraint profile + entropy seed → passed to Module 2 and Module 3.

---

**Module 2: Song Planner**
Two trigger modes, identical pipeline after activation:

*Scheduled:* Runs daily at 12:00 PM. System autonomously selects a theme and feeling from a curated weighted pool constrained by Module 1's output. Generates song blueprint — theme, mood, BPM, key, instrument stem list.

*Manual:* You type a feeling or concept into the HTML page and trigger generation yourself. Same constraint injection applies.

Override available at any time before stems are executed. Override replaces the day's plan entirely and re-runs Module 1 constraint injection for the new concept. Output is a structured session file written to GitHub repo as JSON with checkpoint status: PLANNED.

---

**Module 3: Prompt Generator**
Reads session blueprint and constraint profile from Module 1. Generates one Gemini prompt per stem:

*Algorithmic skeleton (enforced, non-negotiable):* BPM, instrument isolation, duration, key, structural rules — these never vary and are always technically precise.

*AI-generated creative descriptors (deliberately varied):* Feel, texture, dynamics, style — seeded partly by the entropy quota dimension selected in Module 1 to ensure language variation drives output variation in Gemini.

Each prompt is tagged with its stem identity. Prompts appear on the HTML GitHub Pages as a copyable queue, one per stem, clearly labeled. Session JSON checkpoint updated to: PROMPTS_READY.

---

**Module 4: Email Notification**
Sends immediately after prompt queue is ready. Contains song theme summary, blueprint details, BPM, planned stem list, entropy dimension injected this cycle, and direct link to the MCS GitHub Pages prompt queue. Notification only — no action required in the email itself. Session JSON checkpoint updated to: NOTIFIED.

---

**Module 5: Stem Intake**
You download each Gemini output and drop into a locally watched folder. System detects the file and attempts identity matching in order:

1. Filename convention match (primary): filename includes stem tag from prompt generation
2. Feature-based auto-classification (fallback): librosa extracts audio characteristics and matches against expected stem profile for the session

Manual override always available. System runs immediate tolerance-based analysis on each stem:
- BPM: ±3 BPM tolerance
- Key detection: probabilistic confidence threshold
- Energy profile: flagged if outside expected range for stem type

Flags presented to you for manual decision — no auto-rejection. Stems stored locally only at this stage. Session JSON checkpoint updated per stem: STEM_[ID]_INTAKE_COMPLETE.

---

**Module 6: Sample Extender**
Extends each 30-second stem to target duration algorithmically:

1. Detect micro-sections within the stem using librosa segmentation
2. Identify stable regions suitable for extension without phase artifacts
3. Reconstruct using segment-aware reassembly (A-B-A-C pattern logic) not naive looping
4. Apply crossfade at all join points following the rhythmic grid of the stem

AI dependency ends completely at this module. All downstream processing is pure Python. Extended stems stored locally as independent files. Session JSON checkpoint updated: STEM_[ID]_EXTENDED.

---

**Module 7: Song Identity Validator**
Computes the session's Song Identity Vector from the full extended stem set. Checks each stem against the session-level identity vector within tolerance. Flags stems that pull the session incoherently — instruments that contradict the mood trajectory, energy profile, or harmonic center of the session. Presents flags to you with the specific vector dimension that mismatches. You decide whether to replace the stem, adjust the tolerance, or override the flag using Perceptual Score authority. Session JSON checkpoint updated: IDENTITY_VALIDATED.

---

**Module 8: Diversity Filter**
Extracts full Diversity Feature Vector from the current session. Compares against all approved songs in the library using cosine distance on curve shapes. Rhythmic fingerprint checked separately as an independent dimension. Decision output:

- Multiple dimension similarity detected → flagged for your attention
- All dimensions clear → session cleared for candidate review
- Thresholds configurable per dimension, tunable as library grows

Flags are advisory only. Perceptual Score at Module 9 is the final authority. Session JSON checkpoint updated: DIVERSITY_CHECKED.

---

**Module 9: Candidate Review Interface**
HTML page presents all week's draft sessions at end of week. Per session:

- All stems play simultaneously with individual mute toggles
- Muting removes stem from playback in real time
- Core tracks category only at this stage
- Diversity flags and identity flags displayed as context, not as blockers
- You enter Perceptual Score: vibe, originality, coherence

P score is applied as final authority:
- High P overrides any flags → session approved as candidate
- Low P overrides clean filter pass → session rejected

On approval: session metadata, identity vector, diversity vector, perceptual score, and decision log committed to GitHub repo JSON. Approved stems uploaded to Google Drive across the 3-account pool. Session JSON checkpoint updated: CANDIDATE_APPROVED or CANDIDATE_REJECTED.

If no sessions approved → State 1 continues next week.
If at least one approved → State 2 activated.

---

**STATE 2 — Refinement Pipeline**

---

**Module 10: Refinement Planner**
Daily notification repurposed for all active projects. Analyzes each active project's current approved stem set and its identity vector. Suggests new layers based on the project's theme and underrepresented dimensions — vocals, harmonics, counter-melody, percussion additions, texture layers. One suggestion per active project per day. Suggestion only — you decide whether to act, modify, or ignore. No generation pressure, no time constraint. Session JSON checkpoint updated: REFINEMENT_SUGGESTION_SENT.

---

**Module 11: Refinement Prompt Generator**
Same hybrid approach as Module 3. Constraint injection from Module 1 still applies. Entropy injection still mandatory. Prompts tagged explicitly as refinement tracks — always separated from core tracks in all storage and display contexts. Prompt queue appears on the active project's HTML page ready to copy.

---

**Module 12: Track Toggle Interface**
HTML page per active project. Two permanently separated categories:

*Core Tracks:* Approved stems from Phase 1. Locked unless you explicitly choose to replace one.

*Refinement Tracks:* New stems added in Phase 2. Pending verification status until you act.

All toggled tracks play simultaneously. Muting removes from playback in real time. You can audition any combination. Verification flow per refinement track:

- Approved → added to song layer, Drive uploaded, logged in session JSON with reason
- Rejected → discarded locally, rejection reason logged

Perceptual Score applies here too — your toggle decisions and time spent on each stem are logged by Module 13 as implicit perceptual data. Refinement loop repeats until you are satisfied. Session JSON checkpoint updated per decision: STEM_[ID]_APPROVED or STEM_[ID]_REJECTED.

---

**Module 13: Human Decision Logger + State Authority**
Active across both states at every decision point. Dual role:

*Decision logging:* Records context at every approval and rejection — which filters flagged what, perceptual scores given, stems muted during review, time spent per session, entropy dimension injected, override decisions made. Builds your personal taste profile over time. Serves as the authorship record for every song.

*State authority enforcement:* GitHub JSON is the single source of truth. Module 13 is the only writer of final state. All other modules write checkpoint status. Module 13 reconciles checkpoints into authoritative state on every pipeline run. If Drive and JSON conflict, Module 13 flags it and defers to JSON. No other module resolves state conflicts.

---

**Module 14: Final Mix and Export**
Triggered manually when you mark a project as finished. Assembles all approved tracks — core and refinement — into a single mix using pydub:

- Timing alignment across all stems
- Volume balancing per stem category
- Basic EQ separation to prevent frequency masking
- Crossfade at natural transition points

Song naming: manual entry or system generates from session theme and mood. Final mix downloaded locally first. On your explicit confirmation: uploaded to Google Drive as master file, published to GitHub Releases as permanent archive link. Session metadata marked COMPLETE in GitHub repo JSON. Project archived in song library.

---

**Module 15: Song Library + Adaptive Feedback**
All completed songs stored with full metadata: theme, mood, stems used, identity vector, diversity vector, perceptual scores, decision log, entropy injection history, Drive link, Releases link. Two feedback loops active:

*Short-term:* Feeds Module 1 constraint engine for next generation cycle. Library gap analysis updates every cycle.

*Long-term:* Perceptual score history and decision logs gradually inform Style Anchor Vector recalibration. Not automatic — presented to you as a suggested recalibration when enough data has accumulated. You approve or reject the recalibration.

---

**Storage Architecture:**

| Data Type | Location | Authority |
|---|---|---|
| Active stems (unapproved) | Local temp only | None — ephemeral |
| Approved stems + candidates | Google Drive (3 accounts) | JSON reference |
| Final mixed songs | Google Drive + GitHub Releases | JSON reference |
| All session state + vectors + logs | GitHub repo JSON | Authoritative |
| Frontend | GitHub Pages | Derived from JSON |
| System logic + backend | GitHub repo | — |

---

**Failure Recovery:**

Every module writes a checkpoint to session JSON before passing control. On any failure the system:

1. Logs the failure with module ID and error details to session JSON
2. Preserves all work completed up to the failure point
3. On next run, reads checkpoint status and resumes from the failed module
4. Never re-runs a completed module unless you explicitly trigger a reset

This applies to Drive upload failures, stem mismatches, API issues, and Actions runner timeouts.

---

**Weekly Rhythm — State 1:**

12:00 PM — Constraint engine runs, entropy injected, plan generated, email sent, prompt queue live on page

Lunch window — You paste prompts to Gemini, download stems, drop into intake folder

Afternoon — Modules 5 through 8 run automatically

End of week — Candidate review interface ready, you score and select using P as final authority

No candidates selected → State 1 repeats

At least one candidate selected → State 2 activated

---

**Unverified Assumptions Requiring Testing Before Build:**

1. Pydub stem sync reliability across separately generated Gemini sessions at same BPM
2. Sample Extender loop point detection quality on real Gemini output
3. Gemini creative descriptor variation producing meaningfully different audio outputs
4. Google Drive API authentication across 3 accounts in a single Python pipeline
5. GitHub Actions runner stability for audio processing workloads (librosa + pydub memory usage)

---

**What v2.1 Does Not Cover (Implementation Phase):**

- Exact threshold calibration for diversity filter dimensions
- Style Anchor Vector initial values (requires your first approved songs)
- Google Drive folder structure across 3 accounts
- GitHub Actions workflow YAML design
- HTML page layouts in detail
- Entropy quota allowed ranges per dimension

---

**System Classification:**
Closed-loop adaptive control system with perceptual feedback. Not a content generator. Not an automation pipeline. A personal music studio where constraints guide generation, human perception governs decisions, and every approved song teaches the system more about what you actually want.

---