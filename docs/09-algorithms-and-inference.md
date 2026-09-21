# 09 — Algorithms and Inference

This file is the scientific contract for [07](07-app-information-processing.md) and [08](08-user-insights-catalog.md). Firmware counterparts live in [06](06-on-device-signal-processing.md).

**Accuracy stance:** we will validate against a chest strap (Polar H10) and a consumer watch, plus sleep diaries — not against PSG or a metabolic cart in v1. Publish error bars internally; do not print “medical-grade.”

All “models” v1 are **classical + rules**. Learned models are v1.1 once we have our own labeled wear dataset. Do not download random GitHub weights.

---

## 1. Heart rate (already partly on-band)

Phone may **recompute HR from IBI**:

\[
HR_{5} = 60000 / \mathrm{median}(IBI_{ms}\ \mathrm{in\ 5s})
\]

If IBI stream is sparse, trust band `hr_bpm` when SQI ≥ 80.

**Smoothing (display):** 3-median then 5-sample mean. Charts can show raw 5 s.

**Validation target (internal):** MAE ≤ 5 bpm vs H10 on walk; ≤ 8 bpm easy jog; no target for HIIT until motion-cancel v1.5.

---

## 2. IBI cleaning (phone, for HRV only)

Let \( RR_i \) be IBI in ms.

1. Keep 333–2000 ms (workout: 272–2000).
2. If \( |RR_i - RR_{i-1}| / RR_{i-1} > 0.2 \), mark ectopic-ish; interpolate from neighbors **or** drop (drop if two in a row).
3. Require SQI ≥ 120 at both bounding seconds.

HRV uses cleaned RR only.

---

## 3. HRV metrics

For a window of cleaned RR (ms):

**RMSSD**

\[
\mathrm{RMSSD} = \sqrt{ \frac{1}{N-1} \sum_{i=1}^{N-1} (RR_{i+1}-RR_i)^2 }
\]

**SDNN** — standard deviation of RR.

**pNN50** — fraction of successive diffs > 50 ms.

**SD1 (Poincaré)** ≈ RMSSD / √2.

Night value for Rebound: RMSSD over **all valid night windows** with still+sleep, median of 5-min RMSSDs (robust to one noisy hour).

**Do not** convert RMSSD to “parasympathetic %.”

---

## 4. Resting HR

During `night.stages` in {light, deep} (not wake):

- Collect 1 s HR with SQI ≥ 100
- RHR_night = **10th percentile** (not min — min is artifact)
- `pulse.rhr_today` = that value
- Baseline = median of last 28 RHR_night with Night Wear ≥ 70%

---

## 5. HR max and zones

Default HRmax (Tanaka):

\[
HR_{max} = 208 - 0.7 \cdot age
\]

If user sets HRmax, use that.

HRrest = latest baseline RHR (or 60 if learning).

Karvonen optional v1.1:

\[
HR_{reserve} = HR_{max} - HR_{rest},\quad
HR_z = HR_{rest} + p \cdot HR_{reserve}
\]

v1 uses % HRmax as in catalog (simpler copy).

---

## 6. Load (0–21)

**Zone weighted minutes** (Banister-style TRIMP, simplified):

Let \( m_k \) be minutes in zone k=1..5 with weights \( w = [1, 2, 3, 4, 5] \).

\[
T = \sum_{k=1}^{5} w_k m_k
\]

Motion add-on (small):

\[
M = \mathrm{clip}( \mathrm{active\_minutes} / 20,\ 0,\ 4)
\]

\[
\mathrm{Load}_{raw} = 0.85 T + 0.15 \cdot (5M)
\]

Map to 0–21 with a saturating curve so a brutal day is 21, an office day ~4–8:

\[
\mathrm{Load} = 21 \cdot (1 - e^{-T_{mix}/\tau})
\]

with \( T_{mix} = T + 0.3 M_{minutes} \), \(\tau \approx 45\) (tune on wear).

Round to 1 decimal internally, display integer.

**Not** a training-injury model. No ACWR in v1 scoring; v1.1 chart only.

---

## 7. Activity class

Until a trained model exists:

```text
epoch = 12 s
if off_wrist: OFF
elif mev_rms < M1 and steps < 2: STILL
elif step_spm in [70, 135] and mev in [M1, M3]: WALK
elif step_spm >= 136 or mev > M3 and HR > RHR+25: RUN
elif mev > M2 and step_spm < 40: OTHER_ACTIVE (cycle/row/weights guess)
else: STILL or OTHER by mev
```

Thresholds M* from 10-person pilot, not from a blog.

**v1.1:** sklearn/XGBoost on our epochs, export to TFLite. Features: mev mean/std, step, gyro rms, HR, HR slope, TOD sin/cos. Labels from user-corrected workouts + phone pockets for walk/run tests.

---

## 8. Steps

Primary: BMI270 step delta.

Secondary (phone): peak count on vertical-ish linear accel when class is walk/run, 0.8–2.2 Hz band.

Display: band steps; if phone and band diverge > 20% over a walk test, prefer phone for that bout.

---

## 9. Calories

**BMR (Mifflin-St Jeor), kcal/day**

Male: \( 10W + 6.25H - 5A + 5 \)  
Female: \( 10W + 6.25H - 5A - 161 \)

W kg, H cm, A years. Other / unspecified: average of both **or** hide Burn until they pick a formula in settings (prefer hide vs misgendering silently — let them choose “use formula X”).

**RMR per second:** BMR / 86400.

**Activity MET table (v1, conservative):**

| Class | MET |
|---|---|
| still (awake) | 1.3 |
| still (sleep) | 0.95 |
| walk | 3.0 (adjust by cadence later) |
| run | 8.0 (adjust v1.1 by HR) |
| other_active | 4.5 |

\[
kcal_{epoch} = MET \times W_{kg} \times \Delta t_{hours}
\]

Active calories = total − BMR_pro-rata for that time.

**HR refinement (v1.1, Keytel-like, optional):** only during workouts with valid HR, as an alternate estimate shown as “HR-based (experimental).”

Never show kJ and kcal mixed.

---

## 10. Sleep

### 10.1 In-bed / out-of-bed

Evening: first sustained STILL ≥ 20 min after user typical bedtime window (default 21:00–02:00 local, adapts).

Morning: sustained motion + HR rise, or off-wrist, or 10 min walk class.

User edit overrides; store both inferred and user.

### 10.2 Sleep vs wake (actigraphy + physiology)

30 s epochs.

Cole-Kripke-ish score on wrist energy, then **override**:

- If HR < RHR_baseline + 5 **and** still **and** after in-bed → sleep
- If energy high → wake
- If SQI bad → don’t guess stage; mark `unknown` (chart gray)

### 10.3 3-class (light / deep / wake)

v1 heuristic (not PSG):

- Wake: as above
- Deep *candidate:* sleep + lowest tertile of HR for that night + lowest tertile of mev + not first 20 min
- Else light

Cap deep at 30% of TST so we do not paint the whole night blue.

**REM v1.1:** higher HRV variability + small motion + after 90 min cycles — **experimental flag**. Default off.

### 10.4 Metrics

- TST = sum sleep epochs  
- TIB = out − in  
- Efficiency = TST / TIB  
- WASO = wake after onset before final rise  
- Latency = onset − in-bed  

---

## 11. Rebound (0–100)

Compute z-scores vs 28-day baseline (need 5 nights):

\[
z_{hrv} = (RMSSD - \mu_{hrv}) / \sigma_{hrv}
\]
\[
z_{rhr} = -(RHR - \mu_{rhr}) / \sigma_{rhr}
\]
\[
z_{sleep} = (TST - \mu_{tst}) / \sigma_{tst}
\]
\[
z_{eff} = (eff - \mu_{eff}) / \sigma_{eff}
\]

Glow penalty: if |ΔT| > 0.4 °C, \( p_{temp} = 0.15 \); else 0. (Direction: both unusual warm and cold can mean bad contact — only apply if Night Wear ≥ 80% so it is likely physiology.)

\[
S = 0.35 z_{hrv} + 0.25 z_{rhr} + 0.25 z_{sleep} + 0.15 z_{eff} - p_{temp}
\]

Logistic-ish map:

\[
\mathrm{Rebound} = 100 / (1 + e^{-S})
\]

Clip 1–99. If σ of a component is tiny (boring life), use a floor σ so we don’t explode z.

If coverage < 70%: **suppress score** (catalog C13).

---

## 12. Glow

Night skin temp = median of last 30 min before rise (more stable than whole night).

Baseline = median of last 28 such values with Wear ≥ 70%.

\[
\Delta T = T_{night} - T_{base}
\]

Display to 0.1 °C. Wrist ≠ core. No fever claim.

---

## 13. Wear %

\[
\mathrm{Wear}_{day} = \frac{\#\ seconds\ (not\ off\_wrist)\ \cap\ (SQI \ge 80\ \mathrm{or\ IMU\ only})}{86400}
\]

Night coverage uses the sleep window denominator, SQI ≥ 80.

---

## 14. Workout detection

Start: 8 min rolling OTHER/WALK/RUN with density ≥ 70% and (HR ≥ RHR+15 or mev high).

End: 5 min still/off.

Hysteresis to avoid flicker. User can discard.

Load contribution = Load formula on that interval only, displayed on the card.

---

## 15. SpO2 (lab only)

Classic ratio-of-ratios on red/IR AC/DC. Uncalibrated. **No user insight.** If we ever ship, it is a new charter + regulatory review.

---

## 16. Training and data

We will collect:

- Time-synced H10 + Whisper traces (HR/HRV)
- Sleep diary + optional phone-in-bed time
- Labeled walks/runs

Store wear-study data **off phones**, anonymized IDs, consent form. Not in the public app database.

Model update path: ship new app with new weights. No silent cloud models in v1.

---

## 17. Numeric hygiene

- No NaN in JSON; use null + confidence 0
- Timezones: store UTC, display local
- Don’t average HR across off-wrist
- Seeds: none required; if TFLite later, pin versions

Changelog: 2026-09-20 — initial freeze.
