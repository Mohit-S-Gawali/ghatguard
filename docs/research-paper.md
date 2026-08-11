# GhatGuard: A Store-Carry-Forward BLE Emergency Relay System for Crash Detection on India's Signal-Deficient Ghat Roads

**Mohit Gawali**
Department of Computer Science and Engineering
Sipna College of Engineering and Technology, Amravati, Maharashtra, India
mohitsharadraogawali@gmail.com

**Date:** August 2026

---

## Abstract

India's ghat (mountain pass) roads exhibit a dangerous convergence of high accident rates and near-complete cellular network absence. The National Highways Authority of India (NHAI) has identified 424 mobile network shadow zones spanning approximately 1,750 km of national highways, concentrated heavily in mountainous terrain. Existing crash detection and emergency notification systems — including Apple's Crash Detection (iPhone 14+) and Google's Personal Safety (Pixel) — depend entirely on cellular connectivity to contact emergency services, rendering them non-functional in precisely the locations where they are most needed. Apple's satellite SOS fallback requires a dedicated satellite radio chip absent from all Android devices and budget iPhones, excluding the vast majority of Indian road users.

This paper proposes GhatGuard, a two-layer emergency alert system designed for commodity Android smartphones requiring no specialized hardware. Layer 1 implements vehicle crash detection through multi-sensor fusion of accelerometer, gyroscope, and GPS data. Layer 2 introduces a store-carry-forward relay mechanism using Bluetooth Low Energy (BLE) connectionless advertising: upon detecting a crash, the victim's phone broadcasts a compact 24-byte SOS packet containing GPS coordinates and a crash timestamp. Passing vehicles' phones receive, store, and physically carry this packet as they drive. When any relay phone reaches cellular coverage, it automatically uploads the SOS to emergency services. This paper presents the system architecture, the BLE packet specification, an analysis of the contact-window physics governing vehicle-to-vehicle BLE transmission, and a discussion of the technical constraints and limitations of the approach.

**Keywords:** crash detection, Bluetooth Low Energy, store-carry-forward, delay-tolerant networking, ghat roads, emergency SOS, vehicular relay, India

---

## 1. Introduction

### 1.1 The Ghat Road Problem

India records approximately 4.61 lakh road accidents and 1.68 lakh fatalities annually [1]. The Ministry of Road Transport and Highways (MoRTH) data reveals that curved road alignment and steep gradients — the defining geometric characteristics of ghat roads — account for approximately 10% of total road accidents and 13% of total road fatalities nationally [1]. Highways account for over 55% of all road fatalities despite comprising only 5% of the total road network [1].

Ghat roads — mountain passes traversing the Western Ghats, Eastern Ghats, and Himalayan foothills — represent an acute concentration of this risk. Notorious corridors include Kashedi Ghat (NH-66, Mumbai–Goa), Tamhini Ghat (SH-78, Maharashtra), Shiradi Ghat (NH-75, Bengaluru–Mangaluru), Thamarassery Ghat (NH-766, Wayanad, Kerala), and the Chintur–Maredumilli corridor (Andhra Pradesh). These routes combine hairpin bends, steep gradients, monsoon-induced low visibility, heavy freight traffic, and — critically — absent or severely degraded mobile network coverage.

### 1.2 The Coverage Gap

The Telecom Regulatory Authority of India (TRAI) conducts Independent Drive Tests (IDTs) across Licensed Service Areas. Tests on hilly corridors consistently record weak signal zones with RSSI/RSRP values dropping below acceptable thresholds, elevated call drop rates, and severely degraded data throughput due to non-line-of-sight (NLOS) propagation conditions and microwave-dependent backhaul [2].

NHAI conducted a comprehensive nationwide study identifying **424 mobile network shadow zones** spanning approximately **1,750 kilometres** of national highways and expressways, with heavy concentration in greenfield corridors, forest passes, and mountainous terrain [3]. These findings were shared with the Department of Telecommunications (DoT) and TRAI to mandate telecom service provider (TSP) tower installation.

This coverage gap exists across all major Indian carriers:

- **Reliance Jio**: Strong coverage near entry/exit towns and broad ridge roads; frequent zero-signal dead zones in deep gorges and dense forest ghat sections. High-frequency bands (2300 MHz, 5G sub-6 GHz) struggle with foliage and mountain penetration.
- **Bharti Airtel**: Good performance on primary mountain national highways; patchy to absent on interior state highway ghats. Better low-band (900 MHz) penetration.
- **Vodafone Idea**: Lower tower density in ghat passes; higher frequency of zero-signal stretches compared to Jio and Airtel.
- **BSNL**: Often the only operator with residual signal in remote passes, but largely limited to legacy 2G/3G infrastructure with delayed 4G rollout [4].

Critically, India's 112 emergency number initiates Emergency Call Roaming — forcing the phone to connect to any available tower regardless of the SIM provider. However, when no operator has a tower within radio range in a deep ghat section, 112 calls fail entirely [5].

### 1.3 Documented Consequences

The intersection of high accident rates and absent coverage has documented, fatal consequences:

- **Chintur–Maredumilli Ghat Road, Andhra Pradesh (December 2025)**: A major tourist bus accident resulted in 8 fatalities. News reports explicitly cited complete lack of mobile network coverage in the agency ghat stretch as delaying emergency information from reaching police and medical teams by several hours, directly impacting the critical "golden hour" for medical intervention [6].
- **Ambenali Ghat, Poladpur–Mahabaleshwar, Maharashtra**: A bus plunged 800 feet into a gorge, killing 30 passengers. Survivors faced zero network signal in the ravine floor, forcing them to climb steep cliffs to the main road to find signal or flag down passing vehicles [7].
- **Shiradi and Charmadi Ghats, Karnataka**: Local rescue groups report that stranded travellers frequently cannot dial emergency numbers, having to walk miles uphill to find a single bar of BSNL or Airtel signal [7].

### 1.4 Limitations of Existing Solutions

**Apple Crash Detection (iPhone 14+, 2022)**: Uses a high-g accelerometer (up to 256g), gyroscope, barometer, GPS, and microphone to detect severe crashes. Upon detection, the device auto-dials emergency services via cellular. The satellite SOS fallback (iPhone 14+) requires a dedicated Qualcomm-Globalstar satellite radio chip and line-of-sight to orbiting satellites — hardware absent from all Android devices and all iPhones prior to the 14 series [8].

**Google Personal Safety (Pixel 3+, 2019)**: Uses the Sensor Hub / Context Hub Runtime Environment (CHRE) for low-power continuous monitoring. Detects crashes via accelerometer spikes, gyroscope rotation, microphone audio signatures, and GPS velocity collapse. Notification is exclusively cellular — no offline fallback exists [9].

Both solutions fail completely in NHAI's 424 identified shadow zones. Neither offers any mechanism for deferred or relayed notification.

### 1.5 Contribution and Scope

This paper does not claim novelty in crash detection algorithms, Bluetooth mesh networking, or delay-tolerant networking (DTN) — all are established research areas with substantial prior work (Section 2). The specific contribution is the application of store-carry-forward relay to a **single, narrowly-scoped payload type** (crash-triggered SOS packets of 24 bytes) with a design explicitly shaped by the constraints of Indian ghat roads: commodity Android hardware, BLE as the only available radio, connectionless transmission within sub-3-second vehicle contact windows, and automated cellular upload upon signal restoration.

---

## 2. Related Work

### 2.1 Smartphone-Based Crash Detection

White et al. (2011) presented WreckWatch, one of the earliest systems for automatic traffic accident detection using smartphone accelerometers, demonstrating feasibility with consumer-grade MEMS sensors and establishing empirical g-force thresholds for crash events [10]. Subsequent work refined these thresholds: severe vehicle crashes typically produce accelerometer readings exceeding 4g on consumer MEMS sensors, with angular velocities above 250°/s indicating rollover events [10][11].

The Activity Recognition API, available on Android since 2013, enables low-power detection of user transportation mode (IN_VEHICLE, ON_FOOT, etc.), allowing crash detection algorithms to activate high-frequency sensor monitoring only when relevant, dramatically reducing battery consumption [12].

### 2.2 Delay-Tolerant Networking and Store-Carry-Forward

Delay-Tolerant Networking (DTN), originally developed for interplanetary communication, addresses networks with intermittent connectivity and high latency through a store-carry-forward paradigm: nodes physically carry data bundles until encountering another node or connectivity point [13]. The Bundle Protocol (RFC 5050 and its successor RFC 9171) formalises this approach [14].

Vehicular DTN (VDTN) adapts these principles to road networks. Fan et al. (2019) analysed store-carry-forward routing strategies for vehicular delay-tolerant networks, demonstrating that vehicles naturally moving along road corridors serve as effective data mules, with delivery probability increasing as a function of traffic density and corridor length [15].

Zhang et al. (2024) specifically addressed DTN routing for high mountain roads, tunnels, and bridges — terrain conditions directly analogous to Indian ghat roads — proposing routing optimisations for sparse, linear vehicular topologies [16].

### 2.3 BLE for Emergency and Disaster Communication

Álvarez et al. (2019) proposed Bluemergency, a system using Bluetooth Low Energy for emergency communication in disaster scenarios where cellular infrastructure is destroyed. Their work demonstrated the feasibility of BLE-based peer-to-peer message relay for emergency contexts, though focused on general message exchange rather than a specific payload type or road-safety application [17].

BLE mesh networking (Bluetooth SIG Mesh Profile, 2017) enables multi-hop relay across BLE nodes, but requires connection-oriented managed flooding — unsuitable for the sub-3-second contact windows between vehicles at highway speeds. GhatGuard's connectionless advertising approach differs fundamentally from mesh topology assumptions.

### 2.4 Positioning of This Work

GhatGuard is not a general-purpose DTN system, a BLE mesh network, or a novel crash detection algorithm. It combines established techniques from each domain into a single-purpose system: crash-triggered SOS packets, connectionless BLE broadcast, vehicular store-carry-forward relay, and automated cellular upload — scoped specifically to the ghat road coverage-gap problem. The compact 24-byte packet design (Section 4.2) reflects this narrow scope: the system carries exactly the information emergency services need (location, time, crash identifier) and nothing more.

---

## 3. System Architecture

GhatGuard comprises two functional layers operating on commodity Android smartphones:

### 3.1 Layer 1: Crash Detection

The crash detection module runs as an Android Foreground Service, continuously monitoring the device's inertial sensors when the Activity Recognition API reports an `IN_VEHICLE` state.

**Sensor Pipeline:**
1. Accelerometer and gyroscope data is sampled at 50 Hz (`SENSOR_DELAY_GAME`).
2. A sliding window buffer retains the most recent 2 seconds of samples (100 data points per sensor).
3. The crash detection algorithm applies multi-sensor fusion, requiring agreement from at least two of three independent signals:
   - **Accelerometer resultant force** exceeding a configurable threshold (default: 4.0g)
   - **Gyroscope angular velocity** exceeding 250°/s
   - **GPS-derived deceleration** exceeding 30 km/h velocity drop within a 150 ms window

**False Positive Mitigation:**
- Detection is active only during `IN_VEHICLE` state, eliminating phone drops, vigorous walking, and similar daily accelerometer spikes.
- Multi-sensor agreement (≥2 of 3) reduces single-sensor false positives.
- Upon triggering, a mandatory 20-second confirmation countdown activates a full-screen, high-volume, vibrating alert. The user can cancel the alert at any point. Only upon timeout or explicit confirmation does the system generate a CrashEvent and activate Layer 2.

### 3.2 Layer 2: BLE SOS Relay (Store-Carry-Forward)

Upon confirmed crash detection, the victim phone transitions to SOS broadcast mode:

**Victim Phone (Broadcaster):**
1. Constructs a 24-byte SOS packet (Section 4.2) containing GPS coordinates, Unix timestamp, a unique crash event identifier, a TTL hop counter, and a truncated HMAC authentication signature.
2. Begins BLE connectionless advertising using `BluetoothLeAdvertiser` in `ADVERTISE_MODE_LOW_LATENCY` (100 ms advertising interval) at maximum transmission power (`ADVERTISE_TX_POWER_HIGH`).
3. Broadcasting continues until battery depletion or manual cancellation.

**Relay Phone (Scanner and Forwarder):**
1. All GhatGuard-equipped phones continuously scan for SOS advertisements using `BluetoothLeScanner` with a `ScanFilter` matching the GhatGuard service UUID.
2. Scan intensity is dynamically adjusted: `SCAN_MODE_LOW_LATENCY` (continuous) when `IN_VEHICLE`, `SCAN_MODE_LOW_POWER` otherwise.
3. Upon receiving an SOS packet, the relay phone:
   - Stores the packet in a local Room database, deduplicating by crash event ID.
   - Checks cellular connectivity immediately. If available, uploads the SOS packet to the backend server via HTTPS.
   - If no connectivity, re-broadcasts the packet via BLE advertising with the TTL decremented by one. Packets with TTL = 0 are stored but not re-broadcast, preventing infinite relay loops.
4. A `ConnectivityManager.NetworkCallback` monitors network state. The instant cellular or Wi-Fi connectivity is restored, all stored undelivered SOS packets are uploaded.

**Backend Server:**
1. Receives SOS packets via HTTPS POST.
2. Deduplicates by crash event ID (multiple relay phones may upload the same crash).
3. Stores crash events with GPS coordinates, timestamp, and relay metadata.
4. Presents a real-time dashboard displaying crash alerts on a map.
5. (Future scope) Forwards verified alerts to India's Emergency Response Support System (ERSS 112).

---

## 4. Technical Design

### 4.1 Why Connectionless BLE Advertising

The choice of connectionless BLE advertising — rather than connection-oriented GATT, Bluetooth mesh, or Google's Nearby Connections API — is driven by the physics of vehicle-to-vehicle contact windows.

**Contact Window Calculation:**

Given:
- Effective BLE range between vehicles (accounting for chassis and glass attenuation): R ≈ 30 m
- Victim vehicle: stationary (post-crash)
- Passing vehicle speed: v = 100 km/h = 27.8 m/s
- Total coverage path length: D = 2R = 60 m

Contact window:

$$t = \frac{D}{v} = \frac{60}{27.8} \approx 2.16 \text{ seconds}$$

BLE connection establishment (advertising discovery + connection request + service discovery + data exchange) requires 1.5–4.0 seconds under ideal conditions. At highway speeds, this exceeds the available contact window.

Connectionless BLE advertising requires no handshake. A scanning device reads the advertising PDU's manufacturer data field in a single radio event (<50 ms). At a 100 ms advertising interval, the passing scanner has approximately 20 opportunities to receive the packet within the 2.16-second window — sufficient for reliable delivery even with packet loss.

Google's Nearby Connections API was specifically evaluated and rejected: it requires a bidirectional pairing handshake, is heavily throttled by Google Play Services in background/screen-off states, and depends on Play Services availability (absent on some budget devices).

### 4.2 SOS Packet Specification

The SOS packet is designed to fit within the 24-byte manufacturer-specific data payload of a standard BLE 4.0 legacy advertising frame, ensuring compatibility with any Android device manufactured in the last decade:

| Byte Offset | Size | Field | Encoding |
|-------------|------|-------|----------|
| 0–1 | 2 bytes | Magic Identifier | `0x534F` (ASCII "SO") |
| 2–5 | 4 bytes | Crash Timestamp | `uint32`, Unix epoch seconds |
| 6–9 | 4 bytes | Latitude | `int32`, degrees × 10⁷ |
| 10–13 | 4 bytes | Longitude | `int32`, degrees × 10⁷ |
| 14–17 | 4 bytes | Crash Event ID | `uint32`, hash of device ID + timestamp |
| 18 | 1 byte | TTL (Hop Counter) | `uint8`, initial value 5, decremented per hop |
| 19–23 | 5 bytes | HMAC Signature | Truncated HMAC-SHA256 (anti-spoofing) |
| **Total** | **24 bytes** | | |

**Design rationale:**
- **24 bytes** fits within the legacy advertising manufacturer data limit (31-byte frame minus 3-byte flags and 4-byte manufacturer data header).
- **Latitude/Longitude scaled by 10⁷** provides approximately 1.1 cm precision — more than sufficient for crash localisation.
- **4-byte Crash Event ID** enables deduplication across multiple relay chains converging on the backend.
- **TTL of 5** limits relay depth, preventing packet storms in dense traffic. On sparse ghat roads, 5 hops with average inter-vehicle spacing of 500 m covers approximately 2.5 km — sufficient to bridge most shadow zones.
- **5-byte truncated HMAC** provides basic anti-spoofing for the proof-of-concept. Production deployment would require a more robust authentication mechanism.

Devices supporting BLE 5.0 Extended Advertising (up to 255 bytes) can accommodate richer payloads in future versions, but the 24-byte legacy format is maintained as the mandatory baseline.

### 4.3 Android Background Execution

Reliable background BLE operation on Android requires navigating several platform constraints:

**Foreground Service:** Continuous BLE scanning and advertising requires an Android Foreground Service with a persistent notification. On Android 14+, the service must declare `android:foregroundServiceType="connectedDevice"` in the manifest [18].

**Permissions (Android 12+):** The application requires `BLUETOOTH_SCAN`, `BLUETOOTH_ADVERTISE`, `BLUETOOTH_CONNECT`, `ACCESS_FINE_LOCATION`, `ACCESS_BACKGROUND_LOCATION`, and `FOREGROUND_SERVICE_CONNECTED_DEVICE` [18].

**OEM Battery Management:** Manufacturers including Xiaomi (MIUI), Samsung (One UI), Oppo (ColorOS), and Vivo (FuntouchOS) implement aggressive background process killers beyond stock Android's Doze mode. The application must detect the device manufacturer and guide users to grant unrestricted battery usage [19].

**Scan Mode and Battery Impact:**

| Scan Mode | Duty Cycle | Battery Drain | Use Case |
|-----------|------------|---------------|----------|
| `SCAN_MODE_LOW_POWER` | ~10% | ~0.5–1%/hr | Parked / walking |
| `SCAN_MODE_BALANCED` | ~25% | ~2–3%/hr | — |
| `SCAN_MODE_LOW_LATENCY` | 100% | ~8–15%/hr | Driving (`IN_VEHICLE`) |

The application uses Activity Recognition to dynamically switch between `LOW_POWER` (default) and `LOW_LATENCY` (when driving), balancing battery preservation against the need for reliable packet reception during the narrow contact window.

### 4.4 BLE Range Under Real-World Conditions

Theoretical BLE range and effective vehicle-to-vehicle range diverge significantly:

| Condition | BLE 4.2 Legacy | BLE 5.0 (1M PHY) | BLE 5.0 (Coded PHY S=8) |
|-----------|---------------|-------------------|------------------------|
| Outdoor line-of-sight | 50–100 m | 100–200 m | 300–500 m |
| Vehicle chassis attenuation (15–20 dB) | 20–40 m | 40–80 m | 100–200 m |
| + Phone in pocket (10–25 dB additional) | 10–15 m | 15–30 m | 40–80 m |

The system is designed for the worst case: BLE 4.2 legacy, phone in pocket inside vehicle, yielding approximately 10–15 m effective range and a contact window of approximately 1 second at highway speed. Even at this extreme, the connectionless advertising approach provides approximately 10 reception opportunities per second at a 100 ms advertising interval.

---

## 5. Limitations and Future Work

### 5.1 Adoption and Critical Mass

The relay mechanism requires that passing vehicles are also running GhatGuard. On sparsely trafficked ghat roads at night, inter-vehicle gaps of 10–15 minutes are common. If no passing vehicle carries the application, the SOS packet cannot be relayed.

**Mitigation paths for future work:**
- Partnership with state road transport corporations (MSRTC, KSRTC, APSRTC) to pre-install on inter-city bus fleet devices — each bus becomes a guaranteed relay node.
- Integration with commercial fleet management systems (trucking companies routinely traverse ghat corridors).
- Exploration of Wi-Fi Direct as a complementary transport for longer-range relay between stationary vehicles (e.g., vehicles stopped at hairpin queues).

### 5.2 False Positive Calibration

Crash detection thresholds (>4g, >250°/s) are derived from published research on consumer MEMS sensors [10][11]. Real-world calibration with Indian road conditions (severe potholes, speed breakers, unpaved ghat sections) has not been performed. False positive rates on Indian roads may differ from published benchmarks. Field validation with controlled test drives is required before deployment.

### 5.3 Security

The 5-byte truncated HMAC provides minimal anti-spoofing for the proof-of-concept. A malicious actor could broadcast fabricated SOS packets to trigger false emergency responses. Production deployment would require a device attestation mechanism and a longer cryptographic signature, potentially requiring the BLE 5.0 Extended Advertising payload.

### 5.4 Legal and Regulatory

Automated emergency alerting to India's ERSS 112 system has regulatory implications. Integration with official emergency response infrastructure would require coordination with the Ministry of Home Affairs and compliance with ERSS protocols. This is outside the scope of the current proof-of-concept.

### 5.5 Multi-Platform Support

The current design targets Android exclusively, reflecting the platform's dominant market share in India (~95%). An iOS implementation would face additional constraints: iOS restricts background BLE advertising to a single, non-customisable advertising packet and throttles background scanning aggressively, potentially making reliable relay impractical without significant platform-specific workarounds.

---

## 6. Conclusion

GhatGuard addresses a documented gap at the intersection of India's ghat road accident rates and cellular coverage absence. By combining established crash detection techniques with a store-carry-forward BLE relay mechanism, the system provides an emergency notification pathway that requires no cellular connectivity, no satellite hardware, and no specialised equipment — only the accelerometer, gyroscope, GPS, and Bluetooth radio present in every modern smartphone.

The system's viability depends on the contact-window physics of connectionless BLE advertising (demonstrated to be feasible within the 2-second highway-speed window), the willingness of users to install and run the application, and field validation of crash detection thresholds on Indian roads. The 24-byte SOS packet design ensures compatibility with all BLE-capable Android devices, and the store-carry-forward relay requires no infrastructure deployment — the existing flow of traffic on ghat roads serves as the data transport layer.

---

## References

[1] Ministry of Road Transport and Highways (MoRTH), "Road Accidents in India — 2023," Government of India, 2024. Available: https://morth.nic.in

[2] Telecom Regulatory Authority of India (TRAI), "Independent Drive Test Reports and Quality of Service Monitoring," 2023–2025. Available: https://trai.gov.in

[3] National Highways Authority of India (NHAI), "Identification of Telecom Black Spots on National Highways," shared with DoT/TRAI, 2024. Referenced via Press Information Bureau (PIB) releases. Available: https://pib.gov.in

[4] Department of Telecommunications (DoT), "4G Saturation Project under Universal Service Obligation Fund (Digital Bharat Nidhi)," Cabinet approval July 2022, budget ₹26,316 crore for 24,680 uncovered villages. Available: https://dot.gov.in

[5] TRAI, "Recommendations on Emergency Communication Services," noting 112 Emergency Call Roaming limitations in zero-coverage areas.

[6] "Chintur–Maredumilli Ghat Road Bus Accident," news reports, December 2025. The News Minute, The Statesman.

[7] News reports on Ambenali Ghat and Shiradi/Charmadi Ghat rescue operations citing signal absence. Multiple regional news sources.

[8] Apple Inc., "Use Crash Detection on iPhone and Apple Watch," 2022. Available: https://support.apple.com

[9] Google, "Personal Safety app — Car crash detection," Pixel Support, 2019. Available: https://support.google.com/pixelphone

[10] J. White, C. Thompson, H. Turner, B. Dougherty, and D. C. Schmidt, "WreckWatch: Automatic Traffic Accident Detection and Notification with Smartphones," *Mobile Networks and Applications*, vol. 16, no. 3, pp. 285–303, 2011.

[11] Automotive crash detection threshold ranges are referenced from WreckWatch [10] and subsequent telemetry studies. Severe crash accelerometer readings: >4g on consumer MEMS sensors; gyroscope angular velocity: >250°/s for rollover events.

[12] Google, "Activity Recognition API," Android Developers Documentation. Available: https://developer.android.com

[13] K. Fall, "A Delay-Tolerant Network Architecture for Challenged Internets," in *Proc. ACM SIGCOMM*, 2003, pp. 27–34.

[14] S. Burleigh, K. Fall, and E. Birrane, "Bundle Protocol Version 7," RFC 9171, IETF, January 2022.

[15] L. Fan, W. Yu, X. Lin, and M. Li, "Store-Carry-Forward Routing in Vehicular Delay Tolerant Networks," in *Proc. IEEE International Conference on Communications (ICC)*, 2019.

[16] Y. Zhang et al., "DTN Routing for High Mountain Roads, Tunnels, and Bridges," arXiv:2404.03033, 2024.

[17] F. Álvarez et al., "Bluemergency: Emergency Communication via Bluetooth Low Energy," arXiv:1909.08094, 2019.

[18] Android Developers, "Bluetooth Low Energy Overview," "Foreground Service Types (Android 14)," and "Bluetooth Permissions (Android 12)." Available: https://developer.android.com

[19] "Don't Kill My App — Manufacturer-specific background execution restrictions," community-maintained database. Available: https://dontkillmyapp.com
