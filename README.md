# Writing Process Signal

A self-contained, browser-based tool that captures the keystroke-level **process** of a
learner answering an open-ended / reflection question, replays that process, and computes
an integrity signal that distinguishes **authentic composition** from **pasted or
AI-paste-then-edited** text.

This is the writing-process analogue of the attention/eye-gaze tracker: a raw behavioural
signal turned into a defensible, privacy-respecting instrument.

## Why this exists

Conventional "AI content detectors" judge the *style* of finished text. They falsely flag
the majority of non-native-English writing as AI-generated, which is unfair to our
learners (placement-track B.Tech students in India writing English as a second language).

We deliberately do **not** do that. Instead we observe *how the text came to exist* — the
process, not the product. Authentic composition has a recognizable rhythm: planning
pauses, insertions, deletions, self-correction. Pasted or transcribed text arrives in
large clean blocks with no such rhythm. Measuring **behaviour instead of prose style** is
fairer to ESL writers — good or imperfect English is treated identically; only *how* the
text was produced matters.

## Privacy posture (non-negotiable)

The tool captures only **event type and timing** — never the characters typed:

- keydown timestamps (a keystroke happened, when)
- paste events with **character COUNT only**, never pasted content
- deletion counts
- mid-text insertion flag (caret was not at the end — a revision behaviour)

There is **no keylogging of content.** The replay reconstructs the *rhythm and structure*
of writing, not its words — and that is the point. The privacy posture is what keeps the
tool defensible, and it is stated visibly in the UI.

## How to run

No build, no backend. Open the file in a browser:

- Double-click `index.html`, **or**
- Serve it: `python -m http.server` in this folder, then visit `http://localhost:8000`.

Then:

1. Type an answer to the reflection prompt in the text area.
2. Click **Analyze & replay**.
3. **Learner view** shows a friendly reflection of your writing rhythm.
4. **Reviewer view** (top-right toggle) shows the process features, the 0–100 score, the
   threshold control, and the JSON export. This is the back-end/reviewer view; the learner
   is shown only the reflection framing.

## The features (and why each separates authentic from pasted)

Computed in `computeFeatures()` from the event stream alone:

| Feature | Definition | Why it separates |
|---|---|---|
| **planning_pauses** | Mean length of pauses that precede a typing burst (gap ≥ `pause_burst_ms`). | Authentic writing pauses to plan a sentence/word. Pasted text has no planning pauses. |
| **revision_rate** | (deletions + mid-text insertions) per character produced. | Authentic writers self-correct mid-composition. Pasted text arrives finished. |
| **paste_share** | Fraction of final length that arrived via paste events. | Direct measure of how much of the answer was pasted vs. typed. |
| **longest_paste** | Largest single paste, in characters. | One big block is the strongest single transcription signal. |
| **burstiness** | Coefficient of variation of inter-event gaps (clumpiness of rhythm). | Paste-heavy sessions are extremely bursty — huge instant chunks between long idle gaps. |

Each raw feature is mapped to an authenticity sub-score in `[0,1]` (`subScores()`), then
combined into a single **0–100 composition-authenticity score** as a weighted average
(`compositeScore()`).

## Thresholds & weights are uncalibrated placeholders

**The numbers in `CONFIG` are placeholders.** They have not been tuned on real learner
data. The score, the per-feature good/watch/flag bands, and the review threshold all need
calibration against a labelled NxtWave dataset before any operational use. Treat the
current output as a demonstration of the *mechanism*, not a validated cut-off.

## How to adjust the feature weights

All calibration knobs live in one clearly-labelled `CONFIG` object at the top of the
`<script>` in `index.html`. Nothing is hardcoded inline.

```js
const CONFIG = {
  weights: {            // relative weight of each feature (ratios matter; auto-normalized)
    planning_pauses: 0.25,
    revision_rate:   0.25,
    paste_share:     0.20,
    longest_paste:   0.20,
    burstiness:      0.10,
  },
  norm: {               // where each feature's authenticity sub-score saturates
    pause_burst_ms:      1200,  // gap >= this = a planning pause before a burst
    pause_target_ms:     900,   // mean pause that maps to "fully authentic"
    revision_target:     0.08,  // revision rate that maps to authentic
    longest_paste_cap:   120,   // single paste this big = least authentic
    burstiness_baseline: 1.0,   // CV of natural typing
    burstiness_range:    4.0,   // CV span above baseline = fully bursty/pasted
  },
  bands: { good: 0.66, watch: 0.40 },  // sub-score cut-offs for the reviewer indicator
  defaultThreshold: 50,                // review threshold on the 0-100 score
};
```

- **Re-weight a feature:** change its number in `weights`. Only ratios matter — they are
  normalized, so you don't need them to sum to 1.
- **Make a feature stricter/looser:** change its target/cap in `norm`. Lowering
  `pause_target_ms`, for example, makes it easier to reach "fully authentic" on pauses.
- **Move the good/watch/flag bands:** edit `bands`.
- **Change the default review threshold:** edit `defaultThreshold` (the slider also
  overrides it live in the UI).

Save the file and reload — no rebuild.

## Export for calibration

In Reviewer view, **Export session JSON** downloads the full event stream plus all
computed features, sub-scores, the final score, the threshold used, and the `CONFIG` that
produced them (schema `nxtwave.writing-process-signal/v1`). These files are the inputs for
later model calibration on real data.

## Scope & policy limits (stated in the UI too)

- **Open-ended / reflection answers only.** Not for coding answers — normal coding
  involves heavy legitimate pasting, which this signal would misread.
- **No auto-penalty.** A below-threshold score is a **human-review trigger**, never an
  automatic punishment or accusation. Flagged answers go to a person, full stop.
- **Behaviour, not style.** The tool never judges whether the English is "good." It only
  measures *how* the text was produced.
