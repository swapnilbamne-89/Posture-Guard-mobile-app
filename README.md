<div align="center">

<img src="https://img.shields.io/badge/Platform-iOS%2017%2B-1B2E3C?style=for-the-badge&logo=apple&logoColor=white" />
<img src="https://img.shields.io/badge/Swift-5.9-F05138?style=for-the-badge&logo=swift&logoColor=white" />
<img src="https://img.shields.io/badge/SwiftUI-Framework-2D6A4F?style=for-the-badge&logo=apple&logoColor=white" />
<img src="https://img.shields.io/badge/Claude%20AI-claude--sonnet--4--6-9B7CF4?style=for-the-badge" />
<img src="https://img.shields.io/badge/Status-Concept%20%2F%20Portfolio-E9A825?style=for-the-badge" />

<br /><br />

# 🧍 PostureGuard

### Posture Drift + Micro-Fatigue Alert for iOS

*A native SwiftUI iOS app that detects sustained postural drift using the iPhone's motion sensor and delivers a 3-stage escalating haptic alert — no camera, no wearable, works entirely from your pocket or desk.*

<br />

> **Portfolio project by [Swapnil Aware](https://github.com/swapnilaware)**  
> Senior AI Product Manager · Aeron Systems · Pune, India  
> Closing the product gap between IMU posture research and a shippable consumer iOS app.

</div>

---

## 📌 The Problem This Solves

Posture correction apps exist — but they rely on cameras, wearables, or manual check-ins. **No mainstream phone app uses `CMDeviceMotion` drift detection combined with `CHHapticEngine` escalation to alert you without ever looking at a screen.**

Most back and neck strain from seated work accumulates silently over 20–30 minutes. PostureGuard detects it while it's happening and tells your body — not your eyes.

---

## ✨ Features

| Feature | Detail |
|---|---|
| **Personalised baseline** | 5-second calibration captures your ideal pitch/roll via `CMDeviceMotion` |
| **Placement modes** | Pocket · Desk flat · Desk stand · Chest pocket |
| **Low-pass filtered drift** | IIR α=0.15 removes micro-jitter; Euclidean angle distance gives a single clean scalar |
| **3-stage haptic escalation** | Stage 1 (1 min): soft nudge · Stage 2 (10 min): double pulse · Stage 3 (20 min): repeating triple burst |
| **Grace period logic** | Timer decays at 2× rate on correction — prevents false resets from brief posture fixes |
| **Live stick-figure** | Canvas-rendered silhouette leans with real-time pitch and roll — the signature visual |
| **DriftArc gauge** | Sweeping arc shows drift vs 8° threshold at a glance |
| **Session reporting** | Good-posture donut, drift timeline, alert breakdown, AI coaching note |
| **AI coaching** | Claude claude-sonnet-4-6 generates a personalised workplace wellness note after each session |
| **Battery-conscious** | `CMDeviceMotion` at 50 Hz — ~60% lower CPU than 200 Hz, adequate for slow postural drift |
| **On-device only** | All session data stored locally via `UserDefaults`; no accounts, no cloud |

---

## 🏗️ Architecture

```
PostureGuard/
│
├── PostureGuardApp.swift          App entry · SessionStore injection
├── ContentView.swift              Screen router: Onboard → Calibrate → Main
│
├── Models/
│   └── Models.swift               AlertStage · PhonePlacement · PostureEvent
│                                  PostureBaseline · PostureSession · SessionStore
│
├── Managers/
│   ├── PostureGyroManager.swift   CMDeviceMotion @ 50 Hz · IIR low-pass (α=0.15)
│   │                              Euclidean drift · sustained timer · stage promotion
│   ├── HapticAlertEngine.swift    CHHapticEngine · 3-stage patterns · Stage 3 repeat timer
│   ├── PostureAnalyser.swift      Combine pipeline · good/poor time tracking
│   │                              session controller · haptic gate · report builder
│   └── AIService.swift            Claude API call → wellness coaching note
│
└── Views/
    ├── DesignSystem.swift          Tokens · PostureSilhouette (Canvas) · DriftArc
    │                               StageBadge · PGCard · StatTile · PGButton
    ├── OnboardingView.swift        4-step product walkthrough
    ├── CalibrationView.swift       Placement picker · 5s baseline · live silhouette
    ├── MonitorView.swift           Hero: silhouette + DriftArc · stage banner
    │                               · drift timer bar · alert log · AI note
    └── AppViews.swift              ReportView · SettingsView · HistoryView · MainTabView
```

---

## 🔬 Detection Pipeline

```
CMDeviceMotion (50 Hz)
  — sensor fusion: gyro + accelerometer + magnetometer
  — gravity-compensated absolute attitude reference
        │
        ▼
  IIR Low-Pass Filter  (α = 0.15)
  ┌────────────────────────────────────────┐
  │  lpPitch += 0.15 × (rawPitch − lp)    │  ← removes 1–3 Hz micro-jitter
  │  lpRoll  += 0.15 × (rawRoll  − lp)    │    without perceptible lag
  └────────────────────────────────────────┘
        │
        ▼
  Euclidean Drift Angle
  ┌────────────────────────────────────────────────────────────────────────┐
  │  drift = √( (lpPitch − baseline.pitch)² + (lpRoll − baseline.roll)² ) │
  └────────────────────────────────────────────────────────────────────────┘
        │
        ▼
  Threshold Check  (drift > 8°  ← configurable 4°–20°)
        │
        ▼
  Sustained Drift Timer  (1 Hz tick on main thread)
  ├── timer++   when drifting
  └── timer−=2  when corrected  ← grace period: decay 2× faster than accumulation
        │
        ▼
  Stage Promotion
  ├──  60s → Stage 1: single soft pulse     (intensity 0.45, sharpness 0.30)
  ├── 600s → Stage 2: double medium pulse   (intensity 0.70 + echo at 180ms)
  └── 1200s→ Stage 3: repeating triple burst every 8s (intensity 0.90 × 3 taps)
                       ← cancelled immediately when posture corrected
```

**Why `CMDeviceMotion` not `CMGyroData`?**  
Raw gyro data integrates angular velocity — it drifts significantly over a 20-minute monitoring window. `CMDeviceMotion` uses sensor fusion (gyro + accelerometer + magnetometer) to maintain a stable absolute attitude reference, which is essential for long-session posture tracking.

**Why 50 Hz?**  
Postural drift is slow — it unfolds over minutes. 50 Hz provides ample temporal resolution while drawing approximately 60% less CPU than the 200 Hz required for tremor detection (a completely separate use case).

---

## 📱 Screen Flow

```
┌──────────────┐    ┌──────────────────────────┐    ┌────────────────────────────────────┐
│  Onboarding  │───▶│       Calibration         │───▶│         Main App (Tab Bar)          │
│  (4 steps)   │    │  Placement picker         │    │  ┌──────────┬─────────┬──────────┐  │
│              │    │  5s baseline capture      │    │  │ Monitor  │ History │ Settings  │  │
│              │    │  Live silhouette preview  │    │  │ Silhouette│ Donut  │ Sliders   │  │
│              │    │  Axis readout (pitch/roll)│    │  │ DriftArc │Timeline │ Timings   │  │
└──────────────┘    └──────────────────────────┘    │  │ AI note  │ Export │ Haptic     │  │
                                                     │  └──────────┴─────────┴──────────┘  │
                                                     └────────────────────────────────────┘
```

---

## 📳 Haptic Escalation Design

| Stage | Trigger | Haptic Pattern | `CHHapticEvent` Details |
|---|---|---|---|
| **Stage 1** | 1 min continuous drift | Single soft tap | `hapticTransient`, intensity 0.45, sharpness 0.30, duration 100ms |
| **Stage 2** | 10 min continuous drift | Double medium pulse | Two events: t=0 (intensity 0.70) + t=180ms (intensity 0.65) |
| **Stage 3** | 20 min continuous drift | Triple burst, repeats every 8s | 3× events at 0ms / 140ms / 280ms, intensity 0.90, sharpness 0.75 |
| **Reset** | Posture corrected | Stage 3 repeat timer cancelled | `stopStage3()` called immediately on stage transition to `.none` |

Stage 3 uses a dedicated `Timer` (not a `CHHapticPattern` loop) so it can be cancelled cleanly mid-cycle the moment posture is corrected.

---

## 🧠 AI Integration

After each session, PostureGuard calls **Claude claude-sonnet-4-6** with structured session data and returns a personalised coaching note in 3–4 warm sentences.

**Prompt engineering highlights:**
- Persona: *workplace wellness coach embedded in the app* — not a medical assistant
- Structured input: good-posture %, drift statistics, alert counts by stage
- Three explicit output requirements: honest reflection, one ergonomic tip, encouragement
- Hard constraints: no bullet points, plain prose, under 85 words

```swift
// AIService.swift — prompt excerpt
"You are a workplace wellness coach embedded in PostureGuard...
Session data:
- Good posture: \(s.goodPosturePercent)% (\(s.timeInGoodPosture)s)
- Avg drift: \(s.avgDriftDeg)° (threshold: 8°)
- Peak drift: \(s.peakDriftDeg)°
- Total alerts: \(s.totalAlerts) (\(sig) significant, \(mod) moderate)"
```

---

## 🚀 Getting Started

### Requirements

- Xcode 15.4+
- iOS 17.0+ on a **physical iPhone** (`CMDeviceMotion` unavailable in Simulator)
- Apple Developer account (free tier works for device testing)
- Anthropic API key (optional — app works without it, AI note card is hidden)

### Setup

```bash
# 1. Clone
git clone https://github.com/swapnilaware/posture-guard-ios.git
cd posture-guard-ios

# 2. Open project
open PostureGuard.xcodeproj

# 3. Set your team
# Xcode → PostureGuard target → Signing & Capabilities → Team

# 4. Add API key (optional)
# Xcode → Build Settings → User-Defined
# Add:  ANTHROPIC_API_KEY = sk-ant-...

# 5. Select your iPhone → ▶ Run
```

### API Key Security

The key is read from `Info.plist` via `$(ANTHROPIC_API_KEY)` build variable — it never touches source control. For production, load from iOS Keychain or proxy through a server endpoint.

---

## 📐 Technical Reference

| Parameter | Value | Rationale |
|---|---|---|
| Sensor API | `CMDeviceMotion` (.xMagneticNorthZVertical) | Gravity-compensated absolute attitude; raw gyro drifts over 20-min windows |
| Sample rate | 50 Hz | Sufficient for slow drift; ~60% lower CPU vs 200 Hz |
| Low-pass filter | IIR, α = 0.15 | Removes 1–3 Hz micro-jitter without perceptible smoothing lag |
| Drift metric | Euclidean: √(ΔPitch² + ΔRoll²) | Single scalar captures combined forward + lateral lean |
| Alert threshold | 8° (configurable 4°–20°) | Consistent with ergonomics literature for "significant" seated drift |
| Stage 1 | 60 seconds | Early enough to prevent strain onset |
| Stage 2 | 600 seconds | Consistent with 10-min workplace break guidance |
| Stage 3 | 1200 seconds | 20-min clinical threshold for musculoskeletal fatigue risk |
| Grace period | Decay at 2× accumulation rate | Prevents timer reset from brief posture corrections |
| Haptic cool-down | 300 seconds for same-stage re-fire | Prevents alert fatigue |
| Bundle ID | com.postureguard.app | — |
| Deployment target | iOS 17.0 | Required for latest `CHHapticEngine` APIs |

---

## ⚠️ Disclaimer

**PostureGuard is a portfolio concept and is not a medical device.** It has not been reviewed or approved by any regulatory authority. Posture alerts are for wellness awareness only — not clinical diagnosis or treatment. All data stays on device. No telemetry is collected.

---

## 🗺️ Roadmap

- [ ] ML-based placement auto-detection (removes manual placement picker)
- [ ] HealthKit write: standing/sitting minutes as `HKCategoryType`
- [ ] Apple Watch companion — wrist-based independent detection
- [ ] Daily posture score widget (WidgetKit)
- [ ] PDF report export via PDFKit
- [ ] Background monitoring via `CMBatchedSensorManager` (iOS 17)
- [ ] Localisation: Hindi, Marathi (India-first market)

---

## 📚 Research Basis

| Source | Finding |
|---|---|
| IMU posture classification literature (2022–2025) | Wearable IMU sensors reliably detect postural deviation — establishes sensor feasibility |
| Ergonomics research: trunk deviation | 8° lateral trunk deviation in seated workers correlates with elevated musculoskeletal risk |
| Workplace wellness guidelines | 10-minute and 20-minute seated intervals are standard intervention thresholds |
| Product gap identified | No consumer iOS app closes the CMDeviceMotion → CHHapticEngine feedback loop for posture |

---

## 👨‍💻 About the Author

**Swapnil Aware** — Senior AI Product Manager & AI Documentation Lead at Aeron Systems, Pune, India.

13+ years across Defense & Aerospace, IoT, Cybersecurity, and SaaS. CSPO certified (2025). B.Tech Aeronautical Engineering + Diploma in Technical Communication.

This is one of a series of GitHub portfolio projects exploring undeveloped mobile sensor fusion concepts — each grounded in published research, each delivered as a complete Xcode project with React prototype.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0077B5?style=flat&logo=linkedin)](https://linkedin.com/in/swapnilaware)
[![GitHub](https://img.shields.io/badge/GitHub-Portfolio-181717?style=flat&logo=github)](https://github.com/swapnilaware)

---

## 🔗 Related Portfolio Projects

| Project | Core Concept | Sensor Stack |
|---|---|---|
| [TremorGuard](https://github.com/swapnilaware/tremor-guard-ios) | Counter-phase haptic for Parkinson's tremor | `CMGyroData` 200 Hz + `CHHapticEngine` |
| **PostureGuard** ← *this repo* | 3-stage drift alert for seated workers | `CMDeviceMotion` 50 Hz + `CHHapticEngine` |
| Spatial Emotion Transfer *(planned)* | Body-language mirroring over network | Gyro + WebRTC + Haptic |
| Breath-Sync Biofeedback *(planned)* | Chest-held breath detection + haptic guide | `CMDeviceMotion` + `CHHapticEngine` |

---

<div align="center">

Made with SwiftUI · Core Motion · Core Haptics · Claude AI

*From ergonomics research gap to Xcode project — the AI PM way.*

</div>
