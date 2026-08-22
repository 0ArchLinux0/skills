# Math lecture narration — worked example

A complete walkthrough: turning a proof slide into stable, well-paced narration.

## Source script (raw prose, no markup)

```text
Let us execute the proof. To verify that the eigenvalue lambda is strictly real,
we assume Mx equals lambda x where x is a non-zero vector. Left-multiplying by
x star on both sides yields x star M x equals lambda times x star x. When we take
the conjugate transpose of this entire scalar quantity, the Hermitian property
forces x star M x starred to equal x star M x. This equates lambda times x star x
to lambda bar times x star x. Because the vector is non-zero, its squared norm
x star x is strictly positive, forcing lambda to equal its own complex conjugate,
lambda bar. Therefore, lambda must be a real number. Now, for orthogonality...
```

Problems with one single call for this text:

- ~1,100 prepared chars and 9+ natural pause points → over both budgets
- One global speed: either the derivation rushes or the setup drags
- "conjugate" appears twice — a known PVC pronunciation failure

## Step 1 — Segment

| Seg | Kind | Content | Breaks | Speed |
|-----|------|---------|--------|-------|
| real-part | prose | Setup through "...strictly real," | ≤4 | 0.94 |
| real-derivation | formula | Assumption through "...a real number." | ≤4 | 0.82 |
| orthogonality-setup | prose | "Now, for orthogonality." | 0–1 | 0.94 |
| orthogonality-derivation | formula | Inner-product argument to Q.E.D. | ≤4 | 0.82 |

Split points chosen at authored sub-section boundaries ("Now, for…") and between
setup and equation runs.

## Step 2 — Prepare spoken math per segment

```text
lambda_1        → lambda-one
x star y        → x star y          (already token-consistent)
sigma_{k+1}     → sigma-kay-plus-one
A^*             → A star            (avoid "conjugate transpose" where possible)
```

Rewording pass: "its own complex conjugate, lambda bar" → "lambda bar" directly;
"the conjugate transpose of this scalar" → "the adjoint transpose of this scalar."

## Step 3 — Add breaks within budget

```python
segments = [
    {
        "kind": "prose", "speed": 0.94,
        "text": (
            "Let us execute the proof. To verify that the eigenvalue lambda is strictly "
            'real, <break time="0.6s" /> we assume M x equals lambda x where x is a '
            "non-zero vector."
        ),
    },
    {
        "kind": "formula", "speed": 0.82,
        "text": (
            "Left-multiplying by x star on both sides yields x star M x equals lambda "
            'times x star x. <break time="0.6s" /> Taking the adjoint transpose forces '
            "x star M x starred to equal x star M x. <break time=\"0.5s\" /> This equates "
            "lambda times x star x to lambda bar times x star x. <break time=\"0.6s\" /> "
            "Because x star x is strictly positive, lambda equals lambda bar, so lambda "
            'is a real number. <break time="0.8s" />'
        ),
    },
    # note: the trailing 0.8 s break lives at the END of this segment, not the
    # start of the next one — long breaks as first tokens render as room tone
]
```

Each call now has ≤ 3 breaks and under ~450 prepared chars.

## Step 4 — Generate with continuity

```python
from elevenlabs import ElevenLabs

client = ElevenLabs()
prepared = [seg["text"] for seg in segments]

for i, seg in enumerate(segments):
    kwargs = {}
    if i > 0:
        kwargs["previous_text"] = prepared[i - 1][-300:]
    if i + 1 < len(segments):
        kwargs["next_text"] = prepared[i + 1][:300]

    audio, alignment = client.text_to_speech.convert_with_timestamps(
        voice_id="<voice_id>",
        text=seg["text"],
        model_id="eleven_multilingual_v2",
        voice_settings={"speed": seg["speed"], "stability": 0.65},
        **kwargs,
    )
    # decode audio_base64 → mp3; save alignment words as local sidecar JSON
```

## Step 5 — Concatenate and merge timestamps

```bash
# concat_list.txt lists slide_04_seg_01.mp3 ... in order
ffmpeg -y -f concat -safe 0 -i concat_list.txt -c:a libmp3lame -q:a 4 slide_04.mp3
```

```python
offset = 0.0
global_words = []
for seg in segments:
    for w in load_sidecar_words(seg):
        global_words.append({**w, "start": w["start"] + offset, "end": w["end"] + offset})
    offset += probe_duration(seg_mp3_path(seg))
```

The merged timeline drives subtitle highlighting or cursor sync; segment sidecars
stay local (0-based) so any single segment can be regenerated without touching others.

## Step 6 — Review loop

1. Listen to each segment MP3 in isolation before accepting the concat.
2. Sentences still running together → add/lengthen a break at that boundary.
3. Equation steps too fast → lower that formula segment's speed by 0.03, add an entry break.
4. Pauses robotic → reduce by 0.1 s rather than adding another break.
5. Artifacts/rushing in one clip → break budget exceeded; split it again.
