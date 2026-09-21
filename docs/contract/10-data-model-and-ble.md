# 10 — Data Model and BLE

## 1. Endianness and packing

Little-endian, packed structs, CRC32 (Castagnoli or IEEE — pick IEEE in firmware and app tests). No Protobuf on the band (code size). Phone may wrap the same fields in Dart classes.

Magic per record: `0x57 0x53` (`WS` — Whisper).

## 2. Band monotonic time

`t_mono_ms`: uint64 uptime milliseconds since boot.  
`boot_id`: uint32 random at boot (CSPRNG).

Phone maps `(boot_id, t_mono_ms)` → UTC.

## 3. Flash TLV

```text
uint16 magic
uint8  type
uint8  flags
uint16 length
uint8  payload[length]
uint32 crc32  # over everything except crc
```

Erase 4096. Wear-level sequential. `flags.bit0` = encrypted payload (Secure/One optional).

### Type 0x01 — SecFrame (1 Hz, RAM-heavy, usually rolled up)

Not all SecFrames hit flash. RAM ring 60. Flash only if Lab flag or extreme debug.

Fields (32 bytes typical):

| Field | Type | Notes |
|---|---|---|
| t_mono_ms | u32 | 49 days wrap — use u64 if we can spare |
| hr_q8 | u16 | 0xFFFF invalid |
| sqi | u8 | |
| steps_d | u8 | delta |
| mev_q8 | u16 | |
| temp_mC | i16 | |
| flags | u16 | off_wrist, sat, workout_hint, charger |
| led_cur | u8 | |
| pd_dc_hi | u8 | compressed DC |

Prefer **u64 t_mono** in v1 even if 8 bytes. Don’t be cute.

### Type 0x02 — MinFrame (1 minute) — primary flash citizen

| Field | Type |
|---|---|
| boot_id | u32 |
| t_mono_ms | u64 |
| hr_med | u16 |
| hr_p10 | u16 |
| hr_p90 | u16 |
| sqi_med | u8 |
| sqi_pct_ge80 | u8 |
| steps | u16 |
| mev_mean | u16 |
| still_s | u8 |
| off_s | u8 |
| temp_mC_med | i16 |
| flags | u16 |
| ibi_count | u8 |
| reserved |  |

### Type 0x03 — IbiBatch

`n` (u8), then `n` × `u16 ibi_ms` + `u8 sqi_at_peak`. t_mono of first peak + implied chain.

### Type 0x04 — Event

`code` u8, `t_mono_ms` u64, `arg` u32.

Codes: BOOT=1, CHARGE_ON=2, CHARGE_OFF=3, OFF_WRIST=4, ON_WRIST=5, MARK=6, FIFO_DROP=7, DFU=8, TIME_SET=9, BROWN=10, WORKOUT_HINT=11.

### Type 0x05 — TimeSync (also sent live)

Phone UTC unix_s, tz offset min, monotonic at receipt.

## 4. BLE GAP

- Appearance: wrist worn
- Name: `WSPR-XXXX` until bonded; then empty or generic `Whisper`
- Adv: flags + name + service UUID. **No HR.**
- Bonding required for data service

## 5. GATT — custom service

UUID base (example, generate a real v4 UUID at implementation, put in firmware config **not a secret**):

`Service 8e8d0000-6a3b-4c9e-9c11-whisper000001` — placeholder; generate a real RFC4122 UUID at implementation. Not a secret.

| Char | UUID +1 | Props | Payload |
|---|---|---|---|
| DeviceInfo | 01 | R | uuid, hw_rev, fw_semver, boot_policy (0 lab, 1 one, 2 secure), serial |
| Battery | 02 | R, N | pct, charging, mv |
| Time | 03 | R, W | unix, tz |
| LivePulse | 04 | N | hr, sqi, flags (only if encrypted + CCCD) |
| LogCtrl | 05 | R, W | read_ptr, write_ptr, credit |
| LogData | 06 | N, R | TLV chunks ≤ MTU-3 |
| Config | 07 | R, W | dsp_params blob + CRC |
| EventLive | 08 | N | Event TLV |
| Haptic | 09 | W | pattern id |
| BootPolicy | 0A | R | duplicate of policy |

Standard Device Information Service + Battery Service may coexist. Heart Rate Service **not used** (we don’t want every random app reading HR without our pairing UX). LivePulse is ours.

DFU: SMP service as in NCS, gated.

## 6. Sync protocol

1. Phone writes `credit = N` (number of notifications allowed).
2. Band notifies LogData until credit 0 or catch-up done.
3. Phone ACKs by writing `read_ptr` (flash address or record seq).
4. Seq number uint32 monotonic in each chunk header; replay-safe.

Chunk header: `seq, boot_id, n_records, bytes`.

LivePulse does not consume log credits.

## 7. Phone SQLite (simplified)

```sql
CREATE TABLE frames_min (
  device TEXT NOT NULL,
  boot_id INTEGER NOT NULL,
  t_mono INTEGER NOT NULL,
  utc_ms INTEGER,
  payload BLOB NOT NULL,
  PRIMARY KEY (device, boot_id, t_mono)
);

CREATE TABLE ibi (
  device TEXT NOT NULL,
  t_utc INTEGER NOT NULL,
  ibi_ms INTEGER NOT NULL,
  sqi INTEGER NOT NULL
);

CREATE TABLE scores (
  id TEXT NOT NULL,
  local_date TEXT NOT NULL,
  json TEXT NOT NULL,
  app_ver TEXT NOT NULL,
  PRIMARY KEY (id, local_date)
);

CREATE TABLE profile (
  k TEXT PRIMARY KEY,
  v TEXT NOT NULL
);
```

Indexes on `utc_ms`. No SQL string-built from BLE bytes: bound parameters only.

## 8. Config blob

Fixed struct, version byte first. Unknown version → reject. Length check. CRC. Allowlisted fields only (see firmware). No JSON parse on-band.

## 9. Factory identity

`device_uuid` v4 in UICR / OTP at provision. Serial human `WSPR-A-#####`.  
Not a secret; still don’t put other people’s UUIDs in analytics (we have no analytics v1).

Manufacturer string: `Vivica`. Model: `Whisper`.

Changelog: 2026-09-21 — Whisper / Telltale identifiers.
