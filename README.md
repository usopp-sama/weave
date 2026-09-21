# Telltale

The phone app for the **[Whisper](https://github.com/usopp-sama/Whisper)** band, by **Vivica**.

Whisper wears the sensors and says nothing. Telltale is the part that tells you everything it saw — sleep, resting pulse, load, recovery, calories, skin temperature — and tells nobody else.

> *Whisper keeps its mouth shut. Telltale doesn't.*

## Status

Design freeze **v0.1** — documentation only. No Flutter project committed yet. First milestone is a live heart rate screen reading BLE notifications from an nRF54L15 development kit.

## Not a medical device

Telltale shows wellness estimates from optical and motion sensors. It is not a medical device and does not diagnose, treat, or prevent disease. Scores are compared against your own recent baseline, never a clinical threshold.

## Planned stack

| Layer | Choice |
|---|---|
| UI | Flutter, iOS + Android, one codebase |
| BLE | native plugins (CoreBluetooth, Android `BluetoothGatt`) behind a `BandClient` interface |
| Storage | on-device SQLite (Drift), encrypted at rest |
| Keys | iOS Keychain / Android Keystore |
| Inference | pure Dart rules and statistics for v1; optional TFLite assets later |
| Cloud | none |

Bundle identifier sketch: `com.vivica.telltale`.

## Architecture rules

1. The UI never talks to BLE directly.
2. Algorithms never talk to BLE.
3. The feature store is the only writer of historical samples.
4. The insight engine is deterministic given the store, the profile, and the clock.
5. Every insight carries a confidence value; low confidence shows words, not a fabricated number.

## Docs

| Doc | Settles |
|---|---|
| [05-mobile-app-architecture](docs/05-mobile-app-architecture.md) | Modules, sync, screens, permissions, local data protection |
| [07-app-information-processing](docs/07-app-information-processing.md) | Ingest, baselines, processing graph, recompute policy |
| [08-user-insights-catalog](docs/08-user-insights-catalog.md) | Every insight, cadence, confidence gate, v1 vs later |
| [09-algorithms-and-inference](docs/09-algorithms-and-inference.md) | Heart rate variability, sleep staging, calories, scoring |
| [contract/10-data-model-and-ble](docs/contract/10-data-model-and-ble.md) | Frame formats and GATT services (mirror) |

The charter, BLE contract, privacy rules, and roadmap live in the [Whisper](https://github.com/usopp-sama/Whisper) repository. Files under `docs/contract/` are mirrors — edit them there.

## Insight vocabulary

| Name | What it is |
|---|---|
| Pulse | Heart rate now and at rest |
| Load | How hard the day hit the heart, 0–21 |
| Rebound | Morning recovery estimate vs your own baseline, 0–100 |
| Night | Sleep duration, awakenings, three-class estimate |
| Burn | Estimated calories |
| Glow | Wrist skin temperature vs your 28-night baseline |
| Wear | Coverage, fit quality, battery |

Deliberately absent: SpO2, ECG, blood pressure, atrial fibrillation detection, illness prediction, VO2 max as a hard number, social feeds, and any employer or coach dashboard.

## Privacy

All biometrics stay on the phone. No account, no analytics, no background upload. Export is explicit. Telltale will never grow a fleet view for employers.

## License

Not chosen yet.
