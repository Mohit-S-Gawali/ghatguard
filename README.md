# GhatGuard 🛡️

Crash detection and emergency SOS relay for Indian ghat roads, where cellular coverage is absent.

## Problem

India's ghat (mountain pass) roads have the highest accident rates AND the worst cellular coverage. NHAI has identified **424 telecom black spots** spanning ~1,750 km of national highways. Existing solutions (iPhone Crash Detection, Pixel Personal Safety) fail because they depend on cellular connectivity to notify emergency services.

Even Apple's satellite SOS requires a specific satellite radio chip that no Android phone — and no budget iPhone — has. The people driving India's deadliest roads can't afford the hardware that has workarounds.

## Solution

A two-layer system that works on **any Android phone** (no special hardware required):

### Layer 1 — Crash Detection
Uses accelerometer + gyroscope (sensors every smartphone already has) to detect severe vehicle crashes in real-time using multi-sensor fusion:
- Accelerometer impact threshold: > 4g
- Gyroscope angular velocity: > 250°/s
- GPS-based rapid deceleration detection
- Requires ≥2 of 3 signals to trigger (minimizes false positives)
- 20-second confirmation countdown before SOS broadcast

### Layer 2 — BLE SOS Relay (Store-Carry-Forward)
Broadcasts a compact 24-byte SOS packet (GPS coordinates + timestamp + crash ID) via Bluetooth Low Energy to passing vehicles. Each passing phone **stores and carries** the packet as it drives, and when it reaches cellular coverage, **auto-uploads** to emergency services — without the carrying driver doing anything.

```
Victim Phone (crash detected, no signal)
    ↓ BLE broadcast (connectionless, no pairing needed)
Passing Vehicle A (stores SOS packet, still no signal)
    ↓ BLE re-broadcast (TTL decremented)
Passing Vehicle B (stores SOS packet, reaches signal)
    ↓ HTTPS upload
Emergency Services Dashboard (map + alert + GPS coordinates)
```

## SOS Packet Format (24 bytes)

Designed to fit in a standard BLE 4.0 legacy advertising frame — compatible with any Android phone from the last 10 years:

| Bytes | Field | Description |
|-------|-------|-------------|
| 0–1 | Magic | `0x534F` — identifies as GhatGuard SOS |
| 2–5 | Timestamp | Unix epoch (crash time) |
| 6–9 | Latitude | Scaled ×10⁷ |
| 10–13 | Longitude | Scaled ×10⁷ |
| 14–17 | Crash ID | Unique hash for deduplication |
| 18 | TTL | Hop counter (starts at 5) |
| 19–23 | HMAC | Truncated authentication signature |

## Key Technical Decisions

- **Connectionless BLE advertising** (not Nearby Connections) — at 100 km/h, a passing vehicle is within BLE range (~30m) for only ~2 seconds. Connection-based protocols need 1.5–4s for handshake and will fail. Connectionless broadcast takes <50ms.
- **Activity Recognition API** — BLE scanning elevates to high-frequency mode only when the phone detects `IN_VEHICLE`, preserving battery otherwise.
- **Android Native (Kotlin)** — BLE timing constraints and background service requirements demand low-level API access that cross-platform frameworks can't reliably provide.

## Evidence Base

- **NHAI** identified 424 mobile network shadow zones on national highways (~1,750 km)
- **TRAI** drive tests confirm severe signal degradation on hill road corridors
- **Documented incidents**: Maredumilli Ghat (2025), Ambenali Ghat — zero signal delayed rescue by hours
- **MoRTH data**: curved roads and steep gradients account for ~10% of crashes and ~13% of road fatalities in India

## Status

🔬 Research phase complete — entering implementation phase.

## Tech Stack

- **Platform**: Android (Kotlin), minSdk 29 (Android 10)
- **Crash Detection**: SensorManager (accelerometer + gyroscope) + Activity Recognition API
- **Relay**: Bluetooth Low Energy (BluetoothLeAdvertiser / BluetoothLeScanner)
- **Networking**: Store-carry-forward (delay-tolerant networking)
- **Local Storage**: Room database
- **Backend**: TBD (Firebase / FastAPI)

## Prior Art & What's New

This project builds on established research areas — **delay-tolerant networks (DTN)**, **vehicular DTN**, and **BLE mesh for disaster communications** are not new. Key references include Bluemergency (Álvarez et al., 2019), vehicular DTN store-carry-forward (Fan et al., 2019), and DTN routing for mountain roads (arXiv:2404.03033).

**What this project specifically contributes**: applying store-carry-forward to a single, narrowly-scoped payload type (crash-triggered SOS packets), designed for the specific intersection of India's ghat-road accident rates, multi-carrier coverage gaps, and smartphone affordability constraints.

## Documentation

- [Research Paper](docs/research-paper.pdf)

## Author

**Mohit Gawali** — 2nd Year CSE, Sipna College of Engineering and Technology, Maharashtra

## License

MIT License — see [LICENSE](LICENSE)
