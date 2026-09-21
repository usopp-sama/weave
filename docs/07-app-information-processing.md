# 07 — Application-Side Information Processing

The phone consumes cleaned frames and produces **information**: series, nights, scores, language. This is the only place user-facing numbers are born.

## 1. Ingest

1. Deduplicate frames by `(device_uuid, t_mono, type)`.
2. Map `t_mono` → UTC using the sync envelope’s offset (piecewise if the band rebooted; `BOOT` events split segments).
3. Drop SecFrames with clock inversion.
4. Build contiguous **wear segments** (off-wrist gaps break them).

## 2. Derived series (always)

| Series | Cadence | How |
|---|---|---|
| `hr_bpm` | 1 s, then resampled 5 s | band HR if SQI ok; else interpolate ≤ 10 s; else NaN |
| `sqi` | 1 s | as sent |
| `ibi_ms` | irregular | filter physiologically, then HRV |
| `steps` | 1 s cumulative | band delta + app re-peak if needed |
| `mev` motion energy | 1 s | as sent |
| `temp_c` | 1 s / 1 min | as sent |
| `off_wrist` | 1 s | as sent |

## 3. Personal baselines

Stored in SQLite, recomputed nightly.

| Baseline | Window | Method |
|---|---|---|
| RHR | 28 days | median of nightly 10th percentile HR during sleep-classified minutes |
| HRV_RMSSD night | 28 days | median of last 5 min pre-wake RMSSD (or full-night mean if short) |
| Sleep duration | 28 days | median Night total sleep |
| Glow temp | 28 nights | median of 10-min pre-wake skin temp |
| Load typical | 28 days | median daily Load |

Need ≥ 5 qualifying nights before Rebound uses “vs baseline”; until then, show absolute values + “still learning.”

## 4. Processing graph (nightly / daily)

```text
ibi + sqi ──► HRV windows (5 min)
hr + mev ──► activity class + workouts
hr + class + profile ──► Burn (MET)
hr + zones ──► Load
mev + hr + temp + clock ──► Night (sleep)
Night + HRV + RHR ──► Rebound
temp vs baseline ──► Glow
sqi + off_wrist ──► Wear
all ──► InsightEngine
```

Order: **activity and sleep before scores**. Rebound must not use the same minutes it classified as “gym.”

## 5. HRV windows (phone)

On valid IBI (SQI≥120, off-wrist false):

- 5-minute sliding, 50% overlap when still
- Metrics: RMSSD, SDNN, pNN50, mean HR
- Night: also Poincaré SD1/SD2 if ≥ 300 IBIs
- Frequency domain (LF/HF) only if ≥ 4 min still and we want it in v1.1 — **not required for Rebound v1**

Artifact: discard window if > 5% IBI corrections or RMSSD jumps 3× vs neighbors.

## 6. Activity classification (phone)

Input per 12 s epoch: mev mean/var, step rate, gyro energy, HR if valid, time of day.

v1 model: **gradient-boosted trees or logistic ensemble** trained later; until then, **rules**:

| Class | Rule sketch |
|---|---|
| still | low mev, no steps |
| walk | step rate 80–130 spm, moderate mev |
| run | step rate > 140 or high mev + HR |
| cycle-like | high gyro/mev, low step (wrist) — low confidence |
| other_active | residual elevated mev |
| off | off_wrist |

Workouts: merge epochs ≥ 8 min (walk/run/other) with user marks. Conservative: better miss a workout than invent one.

## 7. Sleep segmentation (phone)

See [09](09-algorithms-and-inference.md) for math. Output:

- `sleep_onset`, `sleep_offset`
- epochs 30 s: wake / light / deep  (3-class; REM is v1.1 experimental and hidden if confidence low)
- awakenings ≥ 90 s wake after onset
- efficiency, time in bed (first still-in-evening to rise)

Use local timezone from the phone, not the band.

## 8. Scoring

**Load (0–21):** Banister/TRIMP-like sum of zone-minutes, plus a small movement term. Caps at 21. Not Whoop Strain.

**Rebound (0–100):** weighted z-scores of (night HRV vs baseline, RHR inverted vs baseline, sleep duration vs baseline, sleep efficiency, overnight temp not spiking). Clamp. If Wear night < 70%: show “low coverage” and **do not** pretend Rebound is gospel — gray the score.

**Burn:** BMR (Mifflin-St Jeor) / 24 * hours + MET * hours * weight. See [09](09-algorithms-and-inference.md).

**Glow:** ΔT vs 28-night baseline, nightly, in °C. Not ovulation diagnosis. If the user is a cycling person they may *notice* patterns; we do not name fertile windows in v1.

## 9. Confidence object

Every insight payload includes:

```json
{
  "id": "rebound.morning",
  "value": 64,
  "unit": "score",
  "confidence": 0.0-1.0,
  "reasons": ["low_night_sqi"],
  "as_of": "ISO-8601"
}
```

UI: numeric only if `confidence ≥ 0.4`; else qualitative (“we didn’t see enough overnight data”).

## 10. Recompute policy

Raw frames are immutable. Scores are disposable. App version bump can recompute 28 days on first launch after update (progress spinner, once).

## 11. What the app must never do

- Fill NaN HR with a pretty spline across 20 minutes of off-wrist
- Upload frames
- Infer identity, race, disease
- Combine with calendar/email
- Run JavaScript `eval` on a downloaded recipe

Changelog: 2026-09-20 — initial freeze.
