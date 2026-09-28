# Mobile / Browser Test Matrix

Exact device models depend on the local market; the list is a recommended coverage profile (A-13). Replace models with available equivalents.

| Tier | Class | Example devices | Browsers | Viewport | Priority |
|---|---|---|---|---|---|
| A | Low-end Android | 2–3 GB RAM, Android 10–12 (e.g., Samsung Galaxy A0x/A1x class) | Chrome | 360×640 / 360×800 | Highest (performance floor) |
| A | Mid Android | Galaxy A3x/A5x class, Xiaomi Redmi Note class, Android 12–14 | Chrome, Samsung Internet | 393×873, 412×915 | Highest |
| A | iPhone older | iPhone 8/SE (iOS 16) | Safari | 375×667 | High |
| A | iPhone recent | iPhone 12–15 (iOS 17–18) | Safari, Chrome (WebKit) | 390×844, 430×932 | Highest |
| B | High-refresh | 120 Hz Android / ProMotion iPhone | Chrome/Safari | — | High (timing fairness) |
| B | Tablet Android | 10–11" | Chrome | 800×1280 | High |
| B | iPad | iPad 9th gen+ | Safari | 768×1024, 820×1180 | High |
| C | Desktop | Windows/macOS | Chrome, Firefox, Edge, Safari | 1920×1080 | Admin/display |
| C | Booth display | Large TV/monitor + mini PC | Chrome kiosk | 1920×1080, 3840×2160 | Display |
| C | In-app browsers | Telegram/Instagram/WhatsApp in-app webviews (QR scanners often open these) | — | — | Medium (verify QR flows) |

## Per-device checks

1. QR scan → landing (camera app / in-app scanner).
2. OTP autofill (Android WebOTP, iOS one-time-code suggestion).
3. Keyboard overlap on phone/OTP/name screens.
4. Each game: FPS (target 60, floor 45 sustained), input responsiveness, no scroll/zoom/pull-to-refresh during play, safe areas.
5. Background/foreground during gameplay; lock screen; incoming call simulation.
6. Orientation change mid-game.
7. Audio with silent switch/mute; sound toggle.
8. Refresh at each phase; back gesture.
9. Memory: play all 3 games twice in sequence without reload; no crash, memory stable.
10. Standalone PWA (installed) flow on Android and iOS.
11. Persian rendering: font, digits policy, truncation at 360 px.
12. Spin Perfect timing consistency: same scripted input via a tap robot or high-speed video on 60 vs 120 Hz devices (Phase 7).
