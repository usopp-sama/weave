# 05 — Telltale (Mobile App Architecture)

**App name:** Telltale  
**Talks to:** Whisper (band)  
**v1 apps:** one Flutter codebase, iOS + Android. Bundle sketch: `com.vivica.telltale` (confirm it’s free).
**Native plugins:** BLE (CoreBluetooth / Android `BluetoothGatt`), background sync, secure storage.
**System of record:** on-device SQLite (Drift). No required cloud.

First-run line: *Whisper keeps its mouth shut. Telltale doesn’t.*
**Native plugins:** BLE (CoreBluetooth / Android `BluetoothGatt`), background sync, secure storage.
**System of record:** on-device SQLite (Drift). No required cloud.

Why Flutter: one person can ship UI for meetups. BLE and background limits are the risk; they are isolated behind a `BandClient` interface so we can rewrite the plugin without rewriting insights.

## 1. Process map

```text
┌──────────── UI (Flutter) ─────────────┐
│  Today  Night  Load  Trends  Device   │
└──────────────┬────────────────────────┘
               │ Riverpod / similar
┌──────────────▼────────────────────────┐
│ InsightEngine   ProfileStore          │
│ AlgorithmPipeline (pure Dart + TFLite)│
└──────────────┬────────────────────────┘
               │
┌──────────────▼────────────────────────┐
│ FeatureStore (SQLite)                 │
│  sec_frames / ibi / min_frames / scores│
└──────────────┬────────────────────────┘
               │
┌──────────────▼────────────────────────┐
│ SyncDaemon  ◄── BandClient (BLE)      │
│ TimeSync    DFUClient                 │
└───────────────────────────────────────┘
```

Rules:

- UI never talks GATT.
- Algorithms never talk BLE.
- FeatureStore is the only writer of historical samples.
- InsightEngine is deterministic given store + profile + clock.

## 2. Modules

| Module | Owns |
|---|---|
| `band_client` | Scan, bond, connect, MTU, notify, credit-based pull of logs |
| `sync_daemon` | Resume tokens, backoff, catch-up after airplane mode |
| `feature_store` | Insert frames, idempotent on `(device_id, t_mono, type)` |
| `pipeline` | Clean-already-on-band → HR series interpolate → HRV, activity, sleep, calories, scores |
| `insight_engine` | Maps pipeline outputs to the catalog in [08](08-user-insights-catalog.md) |
| `profile` | Age, sex, height, weight, rest HR seed; in encrypted prefs |
| `secure_store` | Wrapping key, bonding is OS-level |
| `dfu` | Image picker from our signed CDN **or** local file in Lab |
| `export` | User-triggered ZIP of CSV + json; no auto upload |
| `notify` | Local notifications: charge, “Rebound is ready”, fit quality — not HR spam |

## 3. Local data protection

- SQLite in app sandbox.
- SQLCipher or file-level **AES-256-GCM** with key in iOS Keychain / Android Keystore (`setIsInsideSecureHardware` when available).
- Key generated with platform CSPRNG on first launch. Never in source.
- `Cache-Control` N/A for local; screenshots of Night/Rebound: avoid showing in app switcher where OS allows (`FLAG_SECURE` / iOS screenshot hide on those routes).
- Logging: no HR, IBI, names. Connection events only. Strip CR/LF if we ever log user-provided device nickname (allowlist charset, max 32).

No `localStorage` for session tokens because there is no session. If we add accounts later: opaque tokens, HTTPS TLS 1.3 only, never in source.

## 4. BLE UX

1. User taps **Pair**. Telltale scans for name `WSPR-XXXX` (last 2 bytes of static random until bonded).
2. Connect, LE SC Just Works, bond.
3. Exchange `device_uuid`, `fw_version`, `boot_policy`, clock.
4. Pull all unread flash.
5. Stay connected for live Pulse while app foregrounded.
6. Background: iOS restore state + Android Foreground Service when user enables “always sync.” Be honest in the OS permission copy.

Unpair: OS unbond + app wipe option.

Do not use BLE advertisements as a covert HR channel.

## 5. Pipeline scheduling

| Trigger | Work |
|---|---|
| Frame batch ingested | Incremental HR, steps, SQI coverage |
| Workout end event | Activity class + Burn for that bout |
| Local 04:30–11:00 first open | Night + Rebound for previous sleep |
| Local midnight | Close the day’s Load, Burn, Wear |
| Profile change | Recompute Burn and zone bounds; do not rewrite raw frames |

Heavy sleep staging: once per morning, not per second.

## 6. On-device inference runtime

- Default: **pure Dart/Kotlin-port of the DSP-completed algorithms** in [09](09-algorithms-and-inference.md) for v1 (rules + classical stats).
- Optional: TFLite Micro-style models for activity class and sleep 3-class, shipped as app assets, **versioned**. Model files are not secrets.
- No remote model fetch in v1 (supply chain + privacy).

Inference never blocks BLE thread.

## 7. Screens (v1)

1. **Today** — Pulse, Load so far, Rebound (if morning done), Wear, last Night one-liner.
2. **Live** — big Pulse + SQI dots + haptic mark.
3. **Night** — hypnogram-like 3-class bar, duration, awakenings, Glow.
4. **Load** — 0–21 gauge, HR zone minutes, workouts.
5. **Trends** — 7 / 28 day RHR, HRV, sleep, Glow.
6. **Device** — battery, firmware, boot policy, fit tips, DFU, export, privacy text.

No social feed. No leaderboards. No “share to HR.”

## 8. Profile and calories honesty

Burn requires height, weight, age, sex. If missing: show Pulse/Night/Load from HR only; **hide Burn** rather than assume 70 kg male.

Zone bounds: either % of estimated HRmax (`208 − 0.7×age`, Tanaka) or a user-entered HRmax. Label the formula.

## 9. Permissions (minimum)

| Permission | Why |
|---|---|
| Bluetooth | band |
| Notifications | optional |
| Background BLE / location-on-Android-legacy | only if always-sync; explain |
| HealthKit / Health Connect | **v1.1 optional write** of steps/HR; off by default |
| Camera / contacts / mic | never |

Do not read the full health graph. If we write out, we never need to read others’ data.

## 10. Error copy

Pairing / auth failures: **“Couldn’t connect to Whisper. Try again.”** Same text for unknown device vs wrong device. No stack traces in UI.

## 11. Testing

- Fake `BandClient` replaying logged frames from wear studies.
- Golden insight snapshots (JSON) per catalog ID.
- BLE integration on one iPhone + one Android before any meetup demo.

## 12. Later (explicitly not v1)

- Accounts, family, coaches
- Wear OS / watch app
- Cloud backup
- Employer packs
- Web dashboard

Changelog: 2026-09-21 — Telltale named; scans `WSPR-XXXX`.
