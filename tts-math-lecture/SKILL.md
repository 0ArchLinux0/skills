---
name: tts-math-lecture
description: Generate stable, well-paced ElevenLabs narration for math and technical lectures with equations. Use when converting formula-heavy scripts to speech, fixing rushed/robotic equation audio, splitting long TTS generations into segments, or adding pauses and pronunciation handling for math terms.
license: MIT
compatibility: Requires an ElevenLabs API key (ELEVENLABS_API_KEY) and the `elevenlabs` Python SDK.
metadata: {"openclaw": {"requires": {"env": ["ELEVENLABS_API_KEY"]}, "primaryEnv": "ELEVENLABS_API_KEY"}}
---

# Math lecture narration with ElevenLabs

Equation-dense narration breaks normal TTS: formulas get rushed while prose drags,
too many `<break>` tags cause instability artifacts, and symbols like λ or A* get
mispronounced. This skill encodes a production-tested approach: split the script
into **segments** (one API call per screen-visible step), assign **per-segment
speeds**, budget **breaks per call**, and normalize spoken math.

> **Setup:** See [Installation Guide](../text-to-speech/references/installation.md).
> API basics: [text-to-speech](../text-to-speech/SKILL.md).

## Core rules (violating these causes audible failures)

1. **Break budget:** ≤ 4 `<break>` tags per API call, ≤ 8 per slide/chapter. Exceeding this produces rushing, noise, or robotic artifacts within that clip.
2. **Speed gap:** formula segments must be ≥ 0.07–0.10 slower than prose on the same section (prose ~0.92 → formula ~0.82). Global speed alone cannot fix equation pacing — it makes intros drag without slowing chains of equalities.
3. **Never start a generation with a long break.** A `<break time="≥0.8s" />` as the first token renders as room tone / hallway reverb on short clips. Put the pause at the END of the previous segment; start the next one with words.
4. **One generation = one screen-visible step** or one logical derivation sub-block — not an entire multi-minute proof. Shorter calls are more stable and cheaper to regenerate.
5. **Prefer fewer, longer pauses** over many micro-pauses; tune by ear in 0.1 s steps before adding another break.
6. **Validate for free first** — dry-run character counts and mock synthesis before any paid call. Billing is by character count, so segmenting does not increase cost, only the number of calls.

## Segmenting strategy

Split each section into segments at semantic boundaries:

```text
Section script (source)
    │
    ▼  split into segments          kind: prose | formula | transition
    ▼  per-segment TTS call         own speed + optional neighbor context
    ▼  concat: segments → section → full   (ffmpeg)
    ▼  merge word timestamps        offset each segment's times by cumulative duration
```

### Explicit segments (preferred for derivations)

```python
segments = [
    {
        "kind": "prose",
        "speed": 0.94,
        "text": "Let us execute the proof. We verify that lambda is strictly real.",
    },
    {
        "kind": "formula",
        "speed": 0.82,
        "text": (
            'We assume M x equals lambda x where x is non-zero. <break time="0.8s" /> '
            "Left-multiplying by x star yields x star M x equals lambda times x star x."
        ),
    },
]
```

Classify with one heuristic: **if the viewer would glance at the on-screen equation
while hearing the line, it is `formula`.**

### Auto-split fallback

When no explicit segmentation exists, split when any hard limit would be exceeded:
break count over budget, prepared text over ~450 chars (~30–40 s of speech), or
estimated duration over ~45 s. Split at, in order of preference: authored
sub-section breaks after transition phrases ("Now, for…", "Finally,", "Step N."),
paragraph boundaries, conclusion sentences, entry points into dense symbol runs.

## Break placement

| Pause type | Duration | Use |
|------------|----------|-----|
| Micro | 0.3–0.4 s | Rare — two equalities still blurring after slowing speed |
| Sentence / list item | 0.5–0.7 s | Default workhorse |
| Sub-section shift | 0.7–0.9 s | "Now, for orthogonality." |
| Formula block entry | 0.8–1.0 s | Before a multi-step derivation |
| Section transition | 1.0–1.5 s | Before "On the next slide…" |

Place breaks at semantic boundaries, never between every symbol. One break per
*screen-visible step*, not per token.

## Spoken math conventions

Write math as consistent spoken tokens in source text, then expand mechanically:

| Written | Spoken treatment |
|---------|------------------|
| `lambda_1`, `sigma_i` | "lambda-one", "sigma-i" (subscript index appended; hyphenate so it is not read as separate words) |
| `A star`, `x star y` | "A star", "x star y" — adjoint/inner-product notation stays literal |
| `M_n-1` | "M-n-minus-one" |
| Long chains | chunk products, insert a small break at equals signs |

Pronunciation pitfalls observed with PVC voices:

- **"conjugate"** is unstable — prefer rewording to **"adjoint transpose"** for $A^*$, and "lambda bar" for $\bar\lambda$.
- **"Hermitian"** gets mis-split ("hermit-ian") — use a pronunciation dictionary lexicon (PLS, CMU arpabet: `HH ER1 M IH SH AH N`) rather than inline respellings where possible.
- Keep alias count low per generation; alias-heavy text correlates with glitchy output.

## Continuity across calls

Pass `previous_text` / `next_text` (prepared neighbor text, truncated to
~200–400 chars) so tone matches at segment boundaries. For regenerating one clip
inside a sequence, prefer `previous_request_ids` / `next_request_ids`.

## Timestamp merging

Each response returns word-level alignment starting at 0 for that clip. To build
a global timeline:

```text
offset_i      = sum(durations of segments 0..i-1)
word.global   = offset_i + word.local
```

Store local sidecars per segment; keep global times only in the merged timeline.
Concat with the ffmpeg demuxer and re-encode once at final assembly.

## Debugging quality problems

Independent API calls mean a bad later section is **not** model fatigue. Check in order:

1. Break budget exceeded on that section (most common)
2. Alias / symbol-token density stacking up
3. Artifacts localized to the second half of one long clip — split further
4. Concatenation loss — compare the isolated segment MP3 against the merged file

Fix the specific section's breaks/aliases; do not change global settings.

## References

- [`text-to-speech`](../text-to-speech/SKILL.md) — API usage, models, voice settings
- [ElevenLabs docs: pausing](https://elevenlabs.io/docs/capabilities/text-to-speech/pausing)
- [ElevenLabs docs: pronunciation dictionaries](https://elevenlabs.io/docs/capabilities/text-to-speech/pronunciation-dictionaries)
