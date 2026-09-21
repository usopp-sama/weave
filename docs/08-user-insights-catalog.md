# 08 — User Insights Catalog

Shown in **Telltale**. Whisper does not display any of this.

This is the product. If it is not in this file, Telltale does not show it.

**Legend**

- **Ship:** `v1` first drop, `v1.1` after wear data, `v2` hardware or heavy ML, `lab` never customer-facing
- **Conf:** minimum confidence to show a number (else copy-only)
- **Cadence:** how often it updates

All scores are **wellness estimates**. Copy must remain non-diagnostic.

---

## A. Pulse (heart)

### A1. Live heart rate
- **ID:** `pulse.live`
- **User sees:** large BPM
- **Inputs:** SecFrame HR + SQI
- **Cadence:** 1 s while connected
- **Ship:** v1 | **Conf:** 0.45
- **Notes:** Hold last value ≤ 5 s on dip; then “—” 

### A2. Live quality
- **ID:** `pulse.sqi_live`
- **User sees:** 3-dot fit meter (poor / ok / good)
- **Inputs:** SQI, sat flags
- **Cadence:** 1 s
- **Ship:** v1

### A3. Resting heart rate (today)
- **ID:** `pulse.rhr_today`
- **User sees:** “Resting 54 bpm”
- **Inputs:** night 10th percentile HR
- **Cadence:** morning
- **Ship:** v1 | **Conf:** 0.5
- **Copy if low wear:** “Need a fuller night for resting pulse”

### A4. Resting heart rate trend
- **ID:** `pulse.rhr_trend_28d`
- **User sees:** sparkline + Δ vs 28-day median
- **Ship:** v1

### A5. Average HR (day, wear-only)
- **ID:** `pulse.hr_avg_day`
- **Inputs:** valid HR seconds
- **Ship:** v1
- **Notes:** exclude off-wrist

### A6. Min / max HR (day)
- **ID:** `pulse.hr_min_day` / `pulse.hr_max_day`
- **Ship:** v1
- **Notes:** max ignores SQI<80 spikes

### A7. HR zone minutes
- **ID:** `pulse.zone_minutes`
- **User sees:** stacked bar Z1–Z5
- **Inputs:** Tanaka or user HRmax, valid HR
- **Zones:** Z1 50–60%, Z2 60–70%, Z3 70–80%, Z4 80–90%, Z5 90%+
- **Ship:** v1
- **Copy:** “Zones use estimated max HR unless you set it”

### A8. Heart rate during last workout
- **ID:** `pulse.workout_avg` / `pulse.workout_max`
- **Ship:** v1

### A9. HRV RMSSD (night)
- **ID:** `pulse.hrv_rmssd_night`
- **User sees:** ms, “higher often looks more recovered for *you*”
- **Inputs:** valid night IBIs
- **Ship:** v1 | **Conf:** 0.55

### A10. HRV RMSSD trend
- **ID:** `pulse.hrv_rmssd_28d`
- **Ship:** v1

### A11. HRV SDNN (night)
- **ID:** `pulse.hrv_sdnn_night`
- **Ship:** v1.1 (show in detail sheet in v1 if easy)

### A12. rMSSD vs last night
- **ID:** `pulse.hrv_delta_1d`
- **Ship:** v1

### A13. Daytime HRV (still windows)
- **ID:** `pulse.hrv_day_still`
- **Ship:** v1.1 — too easy to misread as “stress”

### A14. Pulse context line
- **ID:** `pulse.context`
- **User sees:** “Lower than your usual morning” / “A bit elevated”
- **Inputs:** vs baseline
- **Ship:** v1
- **Never:** “You may have a fever / illness”

### A15. Lab: raw PPG snippet
- **ID:** `lab.ppg_trace`
- **Ship:** lab only

---

## B. Load (daily cardiovascular load)

### B1. Load score
- **ID:** `load.score`
- **User sees:** 0–21, named bands: Easy 0–7, Work 8–14, Heavy 15–21
- **Inputs:** zone-minutes TRIMP + small motion term
- **Cadence:** live through the day, freeze 00:00 local
- **Ship:** v1 | **Conf:** 0.4 if Wear day ≥ 60%

### B2. Load vs your typical
- **ID:** `load.vs_baseline`
- **User sees:** “+3 vs your usual Thursday”
- **Ship:** v1 (after 7 days)

### B3. Load 7-day sum
- **ID:** `load.week_sum`
- **User sees:** week bars
- **Ship:** v1

### B4. Load momentum
- **ID:** `load.acute_chronic`
- **User sees:** “this week vs last 4 weeks”
- **Ship:** v1.1 (ACWR-like; label carefully, not injury prediction)

### B5. Time in elevated HR
- **ID:** `load.minutes_z3plus`
- **Ship:** v1

### B6. Workout list
- **ID:** `load.workouts`
- **User sees:** start/stop, class, duration, avg HR, Load contribution
- **Ship:** v1
- **Actions:** merge/split/delete (user is source of truth)

### B7. User mark (“this mattered”)
- **ID:** `load.marks`
- **Inputs:** double-tap / button
- **Ship:** v1
- **Use:** annotate, not auto-physiology

### B8. Cardio vs moving
- **ID:** `load.split_hr_vs_motion`
- **User sees:** “mostly from heart / mostly from movement”
- **Ship:** v1.1

---

## C. Rebound (morning readiness analog)

### C9. Rebound score
- **ID:** `rebound.score`
- **User sees:** 0–100
- **Inputs:** night HRV z, RHR z (inverted), sleep duration z, efficiency, Glow penalty if |ΔT| large
- **Cadence:** once per morning
- **Ship:** v1 | **Conf:** 0.45 and night Wear ≥ 70%
- **Copy:** “A sketch of how recovered you look vs *your* recent nights — not a medical clearance”

### C10. Rebound drivers
- **ID:** `rebound.drivers`
- **User sees:** 4 bars: Sleep, HRV, Resting pulse, Temperature
- **Ship:** v1
- **Each driver:** helping / neutral / dragging

### C11. Rebound trend
- **ID:** `rebound.trend_14d`
- **Ship:** v1

### C12. “Still learning”
- **ID:** `rebound.learning_state`
- **User sees:** nights needed N/5
- **Ship:** v1

### C13. Rebound ignored (low wear)
- **ID:** `rebound.suppressed`
- **User sees:** no number, “Charged / worn too little last night”
- **Ship:** v1 — **important: do not emit a fake 50**

---

## D. Night (sleep)

### D1. Time asleep
- **ID:** `night.total_sleep`
- **User sees:** 7h 12m
- **Ship:** v1

### D2. Time in bed
- **ID:** `night.tib`
- **Ship:** v1

### D3. Sleep efficiency
- **ID:** `night.efficiency`
- **User sees:** %
- **Ship:** v1

### D4. Sleep window
- **ID:** `night.onset_offset`
- **User sees:** 00:21 → 07:05
- **Ship:** v1
- **Edit:** user can fix start/end

### D5. 3-class hypnogram
- **ID:** `night.stages_3`
- **User sees:** wake / light / deep bars
- **Ship:** v1
- **Copy:** “Estimated from movement and pulse, not a lab sleep study”

### D6. Deep estimate minutes
- **ID:** `night.deep_minutes`
- **Ship:** v1 | **Conf:** 0.5

### D7. Light estimate minutes
- **ID:** `night.light_minutes`
- **Ship:** v1

### D8. REM estimate
- **ID:** `night.rem_minutes`
- **Ship:** v1.1 experimental, behind “show experiments”
- **Default:** hidden — optical REM is weakly identified

### D9. Awakenings
- **ID:** `night.awakenings`
- **User sees:** count + timestamps
- **Ship:** v1
- **Definition:** ≥ 90 s wake after onset

### D10. Sleep latency
- **ID:** `night.latency`
- **User sees:** minutes from in-bed to asleep
- **Ship:** v1
- **Caveat:** in-bed is inferred (still + clock); user edit

### D11. Mid-sleep awakenings duration
- **ID:** `night.waso`
- **Ship:** v1 (WASO)

### D12. Sleep consistency (social jetlag-ish)
- **ID:** `night.midpoint_consistency_7d`
- **User sees:** “mid-sleep drifting later”
- **Ship:** v1.1

### D13. Sleep debt vs 28-day median
- **ID:** `night.debt`
- **User sees:** minutes behind typical
- **Ship:** v1
- **Not:** “you need 8 hours” universal

### D14. Overnight HR drop
- **ID:** `night.hr_drop`
- **User sees:** evening vs nadir BPM
- **Ship:** v1

### D15. Overnight HRV rise
- **ID:** `night.hrv_rise`
- **Ship:** v1

### D16. Night Wear
- **ID:** `night.coverage`
- **User sees:** % of sleep window with good SQI
- **Ship:** v1

### D17. Nap
- **ID:** `night.nap`
- **User sees:** daytime still + HR drop 15–90 min
- **Ship:** v1.1 (easy to false-positive at a desk)

---

## E. Burn (calories) — always labeled estimate

### E1. Active calories today
- **ID:** `burn.active`
- **Ship:** v1 if profile complete else hidden

### E2. Total calories today
- **ID:** `burn.total`
- **Inputs:** BMR + active
- **Ship:** v1 if profile complete

### E3. Workout calories
- **ID:** `burn.workout`
- **Ship:** v1

### E4. BMR shown in settings
- **ID:** `burn.bmr`
- **User sees:** formula name Mifflin-St Jeor
- **Ship:** v1

### E5. Burn vs yesterday
- **ID:** `burn.delta_1d`
- **Ship:** v1

### E6. Honesty footer
- **ID:** `burn.disclaimer`
- **Always visible** under Burn
- **Copy:** “Estimated from pulse, movement, and the body stats you entered. Not a lab measurement.”

---

## F. Movement

### F1. Steps
- **ID:** `move.steps`
- **Ship:** v1
- **Notes:** wrist steps are imperfect; do not market 99% accuracy

### F2. Steps 7-day
- **ID:** `move.steps_7d`
- **Ship:** v1

### F3. Hours inactive
- **ID:** `move.sedentary_hours`
- **Definition:** still epochs, excluding Night
- **Ship:** v1
- **Not:** hourly nag by default (opt-in later)

### F4. Walk minutes
- **ID:** `move.walk_minutes`
- **Ship:** v1

### F5. Run minutes
- **ID:** `move.run_minutes`
- **Ship:** v1

### F6. Other active minutes
- **ID:** `move.other_active`
- **Ship:** v1

### F7. Auto workout card
- **ID:** `move.auto_workout`
- **Ship:** v1 conservative

### F8. Cadence (spm) last walk/run
- **ID:** `move.cadence`
- **Ship:** v1.1

### F9. Wrist-off events
- **ID:** `move.off_wrist_log`
- **Ship:** v1 (device page)

---

## G. Glow (skin temperature)

### G1. Last night deviation
- **ID:** `glow.delta_night`
- **User sees:** +0.3 °C vs your nights
- **Ship:** v1 | **Conf:** 0.5
- **Copy:** “Skin temperature at the wrist, not core temperature”

### G2. Glow 28-day strip
- **ID:** `glow.strip_28d`
- **Ship:** v1

### G3. Elevated vs your baseline (qualitative)
- **ID:** `glow.state`
- **User sees:** typical / warmer / cooler
- **Ship:** v1
- **Never:** “you’re getting sick”

### G4. Cycle-aware (opt-in)
- **ID:** `glow.cycle_optin`
- **Ship:** v2
- **Only** if user opts in; still not a contraceptive

---

## H. Wear (coverage and device)

### H1. Wear % today
- **ID:** `wear.pct_day`
- **Ship:** v1

### H2. Charge remaining
- **ID:** `wear.battery`
- **Ship:** v1
- **Estimate days left from recent mA, not marketing 14**

### H3. Charge reminder
- **ID:** `wear.charge_nudge`
- **Local notify < 20%
- **Ship:** v1 opt-in

### H4. Fit coaching
- **ID:** `wear.fit_tip`
- **User sees:** “Higher on the wrist / tighter / cleaner window”
- **Inputs:** SQI patterns, sat, DC
- **Ship:** v1

### H5. Firmware + boot policy
- **ID:** `wear.fw`
- **User sees:** version, Lab / One / Secure
- **Ship:** v1

### H6. Last sync
- **ID:** `wear.last_sync`
- **Ship:** v1

### H7. Data gap map
- **ID:** `wear.gaps`
- **User sees:** gray on charts
- **Ship:** v1

---

## I. Trends and stories (composite)

### I1. 7-day review
- **ID:** `story.week`
- **User sees:** 4 sentences max: sleep, load, rebound, one anomaly
- **Ship:** v1.1
- **Generated from templates**, not an LLM in v1 (no surprise medical language)

### I2. Personal records (opt-in, playful)
- **ID:** `story.records`
- **User sees:** longest Night, lowest RHR week
- **Ship:** v1.1
- **Not:** calorie shame

### I3. Correlation toys
- **ID:** `story.corr_sleep_load`
- **User sees:** “Heavy Load days vs next Night”
- **Ship:** v2
- **Need N≥20; show scatter, no p-hacking copy**

---

## J. Explicitly out of product (do not implement as insights)

| Missing on purpose | Why |
|---|---|
| SpO2 %, AFib, ECG, BP | Regulatory + electrodes + honesty |
| VO2max number | Optical-only VO2 is a story we will not tell in v1 |
| Stress % as a single scary number | HRV daytime is noisy; Rebound covers overnight |
| Illness detection | Harmful false positives |
| Whoop Age / healthspan | Not our science |
| Employer / team strain board | Charter |
| Food logging, GPS maps | Scope |
| Social feed | Charter |

Lab may record red/IR for internal SpO2 experiments (`lab.spo2_ratio`) with **no UI**.

---

## K. Notification catalog (local)

| ID | When | Default |
|---|---|---|
| `n.rebound_ready` | morning score computed | on |
| `n.charge` | <20% | on |
| `n.fit` | live SQI poor for 10 min while “should be wearing” | off |
| `n.workout_detected` | auto workout | off |
| `n.sync_failed_3d` | no sync 3 days | on |

No HR alerts. No “your heart is irregular.”

---

## L. Copy constraints (all insights)

- Prefer “for you / vs your baseline” over population norms.
- Prefer “estimate, sketch, looks like” over “is.”
- Equal empty states: if we hide a number, say **why** (wear, profile, learning).
- Same error tone for all failures (no extra detail that leaks device state to a shoulder-surfer beyond what’s on screen).

Changelog: 2026-09-21 — Whisper / Telltale.
