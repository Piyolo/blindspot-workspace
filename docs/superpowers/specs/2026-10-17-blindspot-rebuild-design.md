# BlindSpot Rebuild — Design Specification

**Date:** 2026-10-17
**Status:** Approved design (user-approved 2026-10-17); pending spec review
**Author:** MAS team (lead: orchestrator; mas-architect, mas-designer, mas-researcher)
**Origin:** Flutter rebuild + upgrade of the award-winning ASU capstone "BlindSpot: A Mobile Application that Detects Household Objects for Visually Impaired People" (see `BlindSpot_FinalManuscript.pdf`, parent archive).

---

## 1. Overview

BlindSpot is an Android-first Flutter app that helps blind / visually impaired users navigate indoor spaces. It:

- Detects household objects on-device with the **existing YOLOv8n TFLite model** (14 classes, reused asset), announces object + approximate distance + direction via TTS, draws bounding boxes, and warns about close objects (proximity alarm).
- Listens **always-on** for **bilingual (English + Filipino) voice commands**. Everything a user says is treated as noise unless it matches a known command. No push-to-talk.
- Reserves the **volume buttons exclusively for emergency calling** (single press → 5-second cancelable countdown; double press → immediate call).
- Authenticates against a **new FastAPI backend on Render** with **Neon Postgres**; signup requires no email verification; login is fully validated (argon2 + JWT). Core features work offline; the session is cached locally after first login.

### Goals (success criteria)

1. All manuscript features preserved: detection + audio feedback + approximate distance + bounding boxes + proximity alarm + emergency call + tutorial/command reference + auth.
2. Hardware volume keys are ONLY for emergency and never change system volume.
3. Voice recognition triggers ONLY on commands; zero emergency false-triggers from ambient speech.
4. Installable APK verified on the user's phone; backend live on Render and reachable.
5. Usability: voice-first, haptic feedback, ≥48dp targets, TalkBack-safe semantics, 200% text scaling without clipping.

### Non-goals (locked YAGNI cuts)

Voice-triggered emergency calls · iOS support · cloud/server-side detection fallback · multiple emergency contacts / contact priority · email (SMTP) anything · SMS · push notifications · account deletion · refresh-token rotation (single long-lived JWT for now) · Alembic migrations (create_all at startup) · phone-state plugin (call end detected via app lifecycle) · OCR / face recognition / new model training · Play Store release.

---

## 2. System Architecture

```
┌──────────────────── Android (Flutter app, offline-first) ───────────────────┐
│ Screens (voice-first UI)                                                     │
│ Providers (ChangeNotifier): Auth · Profile · Settings · Detection ·          │
│                             Command · Sos · Tts                              │
│ Services:                                                                    │
│  ┌────────────┐ ┌─────────────┐ ┌───────────┐ ┌──────────┐ ┌─────────────┐  │
│  │ Detection  │ │ SttService  │ │ TtsService│ │ SosService│ │ ApiClient   │  │
│  │ camera →   │ │ restart-loop│ │ priority  │ │ vol keys │ │ (http, /v1) │  │
│  │ isolate →  │ │ → matcher   │ │ queue     │ │ → call   │ │             │  │
│  │ TFLite     │ │ (pure Dart) │ │           │ │          │ │             │  │
│  └────────────┘ └─────────────┘ └───────────┘ └──────────┘ └──────┬──────┘  │
│  Kotlin MethodChannel: volume keys, ACTION_CALL, (call flow)        │        │
│  ── Everything above works fully OFFLINE ──                          │       │
└──────────────────────────────────────────────────────────────────────┼──────┘
                                    HTTPS (auth/profile se only)        │
                          ┌────────────────────────────────────────────┘
                          ▼
            Render (free, Singapore) — FastAPI /api/v1
            ├── /health, /health/db
            ├── /auth/signup  (no verification)
            ├── /auth/login   (argon2 → JWT 30 days)
            ├── /me (GET/PUT), /me/password (PUT)
            └── SQLAlchemy 2.x async + asyncpg (NullPool, statement cache off)
                          ▼
                Neon Postgres — project "blindspot", region Singapore
                (pooled connection string; channel_binding stripped in code)
```

### Offline / online matrix

| Capability | Offline | Online required |
|---|---|---|
| Detection, announcements, proximity alarm | ✅ | — |
| Always-on commands, TTS, settings | ✅ | — |
| Emergency SOS call (contact cached locally) | ✅ | — |
| Signup / login | ❌ | ✅ |
| Profile edit, change password | ❌ (spoken notice) | ✅ |
| App usable after first login (cached session) | ✅ | — |

---

## 3. Project / Repository Layout

Workspace root: `C:\Users\Piolo Mangilog\Dev\BlindSpot\` (short path; outside OneDrive).

```
BlindSpot/                        # ← parent repo "blindspot-workspace" (GitHub: Piyolo)
├── README.md                     # workspace overview + repo links
├── .gitignore                    # ignores /app/, /backend/ (separate repos)
├── docs/
│   └── superpowers/specs/        # this spec + implementation plan + copy deck
├── app/                          # ← repo "blindspot-app" (Flutter)
└── backend/                      # ← repo "blindspot-backend" (FastAPI)
```

- Two code repos plus this workspace/docs repo, all pushed to GitHub (`Piyolo`); no secrets in code (secrets live only in Render env vars). Repo visibility confirmed with the user in Phase 0 (default: public).
- Git init happens in Phase 0; assets copied from the old archive at
  `C:\Users\Piolo Mangilog\OneDrive\Documents\Academic Documents\BlindSpot\BlindSpot\`.

### Flutter project layout (`app/`) — follows workspace conventions

```
app/
├── pubspec.yaml                          # pinned deps (§12)
├── assets/
│   ├── ml/yolov8nV5.tflite               # reused (old: assets/yolov8nV5.tflite)
│   ├── ml/labelsV5.txt                   # reused (14 classes)
│   ├── audio/alarm.mp3                   # reused (old: res/raw/alarm.mp3)
│   └── images/…                          # reused icons (ic_eye, ic_brand, ic_camera,
│                                         #   ic_flashlight, ic_commandlist, ic_bulb,
│                                         #   ic_emergency3, ic_user, ic_person,
│                                         #   ic_call, ic_settings, ic_logout, ic_wifi_off)
├── lib/
│   ├── main.dart                         # MultiProvider + routes; plain Navigator
│   ├── constants/                        # thresholds, API paths, asset paths
│   ├── models/                           # user, auth_result, api_exception, detection,
│   │                                     #   speech_command, app_settings, contact
│   ├── services/
│   │   ├── api_client.dart               # single http wrapper (conventions: /api/v1,
│   │   │                                 #   25s timeout, Bearer, ApiException)
│   │   ├── token_store.dart              # flutter_secure_storage
│   │   ├── settings_store.dart           # shared_preferences
│   │   ├── tts_service.dart              # priority queue, language, rate
│   │   ├── stt_service.dart              # speech_to_text restart loop
│   │   ├── command_matcher.dart          # PURE: transcript → command (unit-tested)
│   │   ├── command_actions.dart          # command → action dispatch
│   │   ├── hardware_channel.dart         # MethodChannel "blindspot/system"
│   │   ├── sos_service.dart              # countdown state machine + call
│   │   ├── proximity_alarm.dart          # port: vibrate + alarm.mp3 + TTS
│   │   ├── audio_focus.dart              # claim/release; STT/TTS arbitration
│   │   └── detection/
│   │       ├── detection_engine.dart     # camera, frame pump, backpressure
│   │       ├── detection_isolate.dart    # long-lived isolate: convert + infer + post
│   │       ├── frame_converter.dart      # YUV420 → RGB letterbox 640 (de-strided)
│   │       ├── yolo_postprocess.dart     # PURE port of LocalYoloDetector.kt
│   │       ├── depth_estimator.dart      # PURE port of DepthEstimator.kt
│   │       └── detection_announcer.dart  # cadence, direction, FIL labels
│   ├── repositories/
│   │   ├── auth_repository.dart          # abstract + api_/mock_ impls (threes rule)
│   │   ├── profile_repository.dart       # abstract + api_/mock_ impls
│   │   └── session_repository.dart       # abstract + secure-storage impl
│   ├── providers/                        # auth, profile, settings, detection,
│   │                                     #   command, sos, tts
│   ├── screens/                          # splash, welcome, login, signup, permissions,
│   │                                     #   home, detection, contacts, profile,
│   │                                     #   settings, help(commands), tutorial
│   ├── widgets/                          # big_action_tile, status_strip, voice_field,
│   │                                     #   permission_card, setting_row, confirm_dialog,
│   │                                     #   countdown_overlay, object_row, bbox_canvas
│   └── theme/                            # tokens (§5.3), dark + cream-high-contrast
└── android/app/src/main/kotlin/com/blindspot/app/
    ├── MainActivity.kt                   # volume key interception → MethodChannel
    └── CallHelper.kt                     # ACTION_CALL with CALL_PHONE; ACTION_DIAL fallback
```

- **applicationId:** `com.blindspot.app` · **app name:** "BlindSpot" · **minSdk 24**, compile/target = Flutter defaults (≥35).
- AndroidManifest permissions (exact): `INTERNET, ACCESS_NETWORK_STATE, CAMERA, RECORD_AUDIO, CALL_PHONE, VIBRATE, WAKE_LOCK`. Plus `<queries>` entries for `TTS_SERVICE` and speech recognition (Android 11+ package visibility).
- **Single-writer rule:** mas-developer implements phases serially; only mas-tester/mas-reviewer read. No two agents edit the same file.

---

## 4. Screens & Navigation

Flow: Splash (session check + voice boot line) → Welcome (no cached session) → Login / Signup → Permissions (if missing) → **Home**. Cached session skips straight to Home. Plain `Navigator` push/pop.

| # | Screen | Purpose / key elements |
|---|--------|------------------------|
| 1 | Splash | Logo; voice "BlindSpot is starting."; routes by session + permissions state |
| 2 | Welcome | Log in / Sign up tiles; "ask a trusted person for setup help" note; fully spoken |
| 3 | Signup | VoiceField: display name, email, password, confirm, **emergency contact number** (required, one field, no verification step); validation spoken |
| 4 | Login | VoiceField email + password; spoken errors; cold-start "waking server" messaging |
| 5 | Permissions | Cards: microphone, camera, phone — each explained aloud; request buttons; limited-mode continue with spoken warning |
| 6 | Home | Status strip; 6 tiles: **Start Detection**, **Emergency SOS**, Contacts, Profile, Settings, Help & Commands; SOS hint line (volume key) |
| 7 | Detection | Camera (dimmed), bbox overlay, status strip (Detecting/Paused/Offline), last announcement text, live transcript, big STOP, flashlight, pause/resume, object-list bottom sheet (sighted helpers) |
| 8 | Contacts | Emergency contact row (red badge) + other contacts; tap-to-call; add/edit (online for edits; call always works) |
| 9 | Profile | Name/email readout, edit contact number, **change password** (current required), logout (spoken confirm) |
| 10 | Settings | TTS language EN/FIL, speech rate, detection sensitivity (Conservative/Balanced/Aggressive), announcement cadence, proximity alarm toggle, haptics toggle, keep-awake toggle, high-contrast theme, listening mode (always / detection-only), about |
| 11 | Help & Commands | Spoken command reference, replayable; practice-mode walkthrough (stretch, Phase 5) |

All screens: every state (loading/error/empty/offline) has a spoken message and a recovery path. No color-only signals. Destructive actions require spoken confirmation.

---

## 5. Design System (summary; full tokens in §14 copy/token tables)

### 5.1 Principles
Voice-first, screen-second. One primary action per screen. Honest about uncertainty ("approximately"). Dark by default (battery + privacy + contrast) with a cream high-contrast mode. Text scales to 200% without clipping. Haptics reinforce, never replace, speech. Blind-first, guardian-friendly.

### 5.2 Haptic vocabulary

| Event | Pattern |
|---|---|
| Listening started | 30 ms light |
| Command recognized | 60 ms medium |
| Command not recognized | 25 ms light ×3, 40 ms gaps |
| Object warning (proximity) | 100 ms medium, repeating every 600 ms while < threshold |
| SOS countdown tick | 40 ms medium per second (last 2 s: 40+40 ms) |
| SOS cancelled | 120 ms soft long pulse |
| Error | 100 ms heavy ×2 |
| Success / saved | 80 ms soft |

### 5.3 Key tokens (rest in `theme/`)
- Colors: surface `#0A101C`, elevated `#121A2C`, container `#162033`, border `#334056`, on-surface `#FFFFFF`, variant `#B9C2D6`; brand `#2C5A86` (fills/icons only), brand-light `#7FB3D9` (brand text on dark), cream `#F2DEC1` / on-cream `#0E2A4A` (high-contrast mode); error `#D9534F` / error-text `#F5A9A3` / error-surface `#3A1A18`; warning `#F0B429`; success `#4CAF50`. Body text ≥4.5:1, large ≥3:1, components ≥3:1.
- Type: brand 30/700 · screen title 24/700 · section 20/600 · body-large **18**/400 (default) · body 16/400 · button 18/600 · caption 14/400 · status 16/600 · countdown 64/700. Title case only (no ALL CAPS — screen readers spell letters).
- Layout: 4px grid; screen padding 20; tiles/pills **56dp** high; minimum touch target **48dp**; STOP button 64dp; bottom bar 88dp + safe area. Radii: pill (h/2), card 16, dialog 20.

---

## 6. Core Feature Designs

### 6.1 Detection pipeline (faithful port + Flutter architecture)

- Camera: `camera 0.12.0+2`, `ResolutionPreset.medium`, `enableAudio: false`, `startImageStream` with `ImageFormatGroup.yuv420`.
- Frame pump on main isolate: drop frames while isolate busy (backpressure); minimum interval **600 ms** (≈1.6 detections/s), adaptive thermal backoff **600 → 1200 → 2000 ms** when inference latency or frame time rises; return to 600 ms after 10 s of fast frames, rising immediately again if slow. Motion gate (tiny-gray MAD ≥ 18) skips static scenes — port of `CameraFragment.kt` gate.
- Long-lived **isolate**: raw YUV planes + metadata → de-stride by `bytesPerRow` → rotate by `sensorOrientation` → nearest-neighbor resize/letterbox to 640×640 → `Float32List [1,640,640,3]` ÷255.
- Inference: **`tflite_flutter 0.12.1`** with `IsolateInterpreter`, CPU threads 4, **no GPU/NNAPI delegates** initially (stability).
- Postprocess — **pure-Dart port of `LocalYoloDetector.kt`** (must not be simplified): runtime output-shape discovery (`C-4`/`C-5` vs label count, channels-first vs anchors-first with smaller-dim heuristic), normalized-coordinate heuristic (×640 when all box values ≤2), sigmoid on logits, objness multiply, conf ≥ 0.60, per-class NMS IoU 0.50. Sensitivity setting maps conf threshold: Conservative 0.75 / Balanced 0.60 / Aggressive 0.45.
- Distance — port of `DepthEstimator.kt`: `d = realWidthCm × 980 / boxWidthPx`; same real-width table + aliases (`tv→television`, `wc→toilet`); always spoken as "approximately", rounded to whole meters (≥1 m) or tens of cm (<1 m).
- Direction from bbox center thirds: left / ahead / right.
- Overlay: boxes drawn to view coordinates with aspect-fit letterbox math; per-class colors; screen dimmed (40% scrim) but kept awake (`wakelock_plus 1.7.0`).
- Announcer (port of `DetectionSpeaker.kt`): speaks the nearest object with a distance estimate; label changed → ~2 s cooldown, same label → random 4–5 s cooldown; suppressed while higher-priority TTS is speaking; "approximately" phrasing mandatory; one-time accuracy disclaimer on first detection ("Distances are approximate. Use your own judgment.").
- Proximity alarm (port of `ProximityAlarm.kt`): distance < **0.7 m** → vibrate 300 ms → `alarm.mp3` → "X is too close" → 3 s cooldown; suppresses normal detection speech during alarm.
- Camera failure/degraded path: spoken state + retry button; never silent.

### 6.2 Always-on voice command system

**Engine:** `speech_to_text 7.4.0`, auto-restart loop. `onDevice: true` (Android on-device recognizer when present; network fallback otherwise). Sessions run long (`listenFor` max, `pauseFor` platform max ≈5 s); on `done`/`error`/`notListening` → restart with 200 ms debounce, exponential backoff on repeated errors (1.5 s → cap 10 s).

**Honest limits (documented, communicated in-app):** Android cannot listen 100% continuously — brief gaps (~0.5–1 s) between sessions; **STT pauses while TTS speaks** (prevents the app hearing itself); on devices without an offline recognizer, commands need internet. A future upgrade path (sherpa_onnx streaming KWS) is noted in §16.

**Echo prevention:** before any TTS utterance: `stop()` STT; on TTS completion handler: restart STT. Transcripts received while TTS is active are discarded.

**Matching (pure `command_matcher.dart`, unit-tested):**
1. Normalize: lowercase, strip punctuation, collapse whitespace, drop fillers (`please`, `could you`, `pakiusap`, `kung pwede`).
2. Consider the final ≤6-token window of the utterance (recognizer appends; utterance end is high-signal).
3. Command matches if the window **equals** a phrase (or matches with ≤1 edit per token for tokens ≥6 chars). **Never substring matching** (`contains("stop")` is forbidden).
4. Find-object pattern: `(find|where is|hanapin ang|nasaan ang|saan ang) + <label>` where label resolves fuzzily against the 14 EN labels + FIL map.
5. Confidence gate: if recognizer confidence available and < 0.4 → discard.
6. Anti-double-fire: per-command cooldown 2 s; identical transcript within 3 s ignored.

**Locked grammar (~18 actions).** EN | FIL:

| Action | English anchors | Filipino anchors |
|---|---|---|
| Describe scene | "what do you see", "describe", "what's around me" | "ano ang nakikita mo", "ilarawan mo", "ano ang nasa paligid" |
| Repeat last | "repeat", "say it again" | "ulitin mo", "ulitin" |
| Find object | "find X", "where is X", "where's X" | "hanapin ang X", "nasaan ang X", "saan ang X" |
| Battery | "battery", "battery level" | "baterya", "ilan ang baterya" |
| Time | "time", "what time is it" | "oras", "anong oras na" |
| Status | "status" | "status" |
| Flashlight on | "flashlight on", "turn on the light" | "buksan ang ilaw", "ilawan mo" |
| Flashlight off | "flashlight off", "turn off the light" | "patayin ang ilaw" |
| Silence announcements | "stop talking", "be quiet", "silence" | "tumahimik ka", "tumahimik" |
| Resume announcements | "start talking", "speak again" | "magsalita ka", "magsalita ka ulit" |
| Pause detection | "stop detecting", "stop detection", "pause" | "itigil ang detection", "tigil" |
| Resume detection | "resume detection", "continue" | "ituloy ang detection", "ituloy" |
| Help | "help", "commands" | "tulong", "mga utos" |
| Faster speech | "speak faster", "faster" | "bilisan mo", "mabilis" |
| Slower speech | "speak slower", "slower" | "bagalan mo", "dahan-dahan" |
| Louder | "louder", "volume up" | "lakasan mo" |
| Softer | "softer", "volume down" | "hinaan mo" |
| Navigation | "settings", "contacts", "home", "profile" | "buksan ang settings", "buksan ang kontak", "home", "profile" |

**Emergency is deliberately NOT in the grammar.** "help"/"tulong" respond with the command list + a reminder that emergencies use the volume key. This prevents ambient speech ("help!" on TV) from placing calls.

**Command acknowledgments** follow the TTS language setting; every recognized command → haptic "recognized"; unrecognized speech → silence (no nagging).

**Battery policy:** listening mode setting (always listen / detection screen only); automatic STT pause below 10% battery with spoken notice; STT suspended during calls and when app backgrounded.

### 6.3 TTS architecture

- `flutter_tts 4.2.5`, single queue with priorities: **SOS_COUNTDOWN (10) > PROXIMITY_ALARM (9) > COMMAND_RESPONSE (7) > DETECTION_ANNOUNCEMENT (3)**. Higher priority preempts (stop + speak). Same priority: newest replaces stale detection announcements; command responses queue ≤1.
- Language per settings: `en-US` default; `fil-PH` when selected **and** `isLanguageAvailable("fil-PH")` — otherwise spoken notice + English fallback ("Filipino voice is not installed on this phone…").
- Speech rate 0.5–2.0 (settings, voice-controllable). Completion handler drives STT resume + optional ducking.
- All utterances go through TtsService; screens never call flutter_tts directly.

### 6.4 Emergency SOS (volume keys only)

**Channel contract** `blindspot/system` (MethodChannel):
- Android → Dart: `volumeKey {key: "up"|"down", action: "down"|"up", eventTime: int}`.
- Dart → Android: `configure {}` · `placeCall {number: string}` → `ACTION_CALL` (CALL_PHONE) with `ACTION_DIAL` fallback + distinctive haptic/beep when fallback used.

**Kotlin:** `MainActivity.onKeyDown/onKeyUp` intercept `KEYCODE_VOLUME_UP/DOWN`, **consume both (`return true`) so system volume never changes**, forward events; ignore repeats (`repeatCount > 0`). Double-press detection in Dart from `eventTime` deltas (≤500 ms, same or either key).

**Constraint:** interception applies only while BlindSpot is in the foreground (Activity focused) — no foreground service in v1 (matches the original app's behavior; documented limitation).

**Flow:**
1. Single press (either key) → press-ack haptic → **5 s countdown**: TTS "Emergency call in 5 seconds. Press any volume key to cancel." + per-second spoken digit + tick haptic; cancel = any volume press ("Emergency call cancelled." + soft long haptic).
2. Double-press → skip countdown → call immediately.
3. Countdown reaching 0 → `placeCall` to the cached emergency contact; spoken "Calling [name]."; screen shows calling state.
4. Offline/airplane → attempt fails gracefully: spoken "Could not place the call. Check your signal." + error haptic; app returns to previous state.
5. Post-call: `AppLifecycleState.resumed` (app returns to foreground) → re-init camera stream + resume STT after 1 s settle delay.
6. Contact source: local cache (secure storage), updated at signup/login/profile-edit. Never fetched at call time.
7. Works from any screen, including during detection.

### 6.5 Auth & session

- **Signup:** name (1–80), email (format validated, normalized lowercase), password (≥8, letter + digit), **emergency contact number (required; 7–15 chars of digits/+/spaces, normalized)**. No verification step. Success → token + user returned; contact cached locally.
- **Login:** email + password; argon2 verify; failures → 401 with generic "Invalid email or password." (no user enumeration on login; signup 409 reveals existing email by necessity, phrased helpfully).
- **JWT:** HS256, single access token, **30-day expiry**, `sub` = user id, `iat`/`exp`; secret from env. Logout = clear local token (server is stateless). Password change requires current password; existing tokens remain valid until expiry (documented).
- **Offline session:** token + user + contact cached in `flutter_secure_storage` at login/signup; app fully usable offline; profile edit/password change require online (spoken notice "You need internet to change this.").
- **Cold start UX:** auth calls use the 25 s ApiClient timeout with **one automatic retry**; while retrying, spoken + visual "The server is waking up. One moment." (Render free tier spins down after 15 min idle, ~1 min cold start).
- **Error mapping (bilingual):** invalid credentials · email exists · rate-limited · network unreachable · server waking · generic.

### 6.6 Settings (persisted via `shared_preferences`, non-sensitive)

TTS language (EN/FIL) · speech rate (0.5–2.0) · detection sensitivity (Conservative/Balanced/Aggressive → conf 0.75/0.60/0.45) · announcement cadence (Immediate / Every few seconds / On request only) · proximity alarm on/off (+ threshold fixed 0.7 m in v1) · haptics on/off · keep screen awake on/off · high-contrast cream theme · listening mode (always / detection-only) · replay tutorial. Every change: spoken confirmation + persisted immediately.

---

## 7. Backend Design (`backend/`)

```
backend/
├── app/
│   ├── main.py            # FastAPI app, routers, CORS (open, harmless for mobile)
│   ├── config.py          # env: DATABASE_URL, JWT_SECRET, ACCESS_TTL=2592000
│   ├── database.py        # create_async_engine: NullPool +
│   │                      #   connect_args={"statement_cache_size": 0,
│   │                      #                 "prepared_statement_cache_size": 0}
│   │                      #   + sanitize URL: strip channel_binding param
│   ├── models.py          # User (portable column types for SQLite-based tests)
│   ├── schemas.py         # pydantic v2 + EmailStr
│   ├── security.py        # argon2-cffi hash/verify; PyJWT encode/decode
│   ├── deps.py            # get_db, get_current_user (Bearer)
│   └── routers/           # health.py, auth.py, me.py
├── tests/                 # pytest + httpx AsyncClient + aiosqlite (dependency override)
├── requirements.txt       # pinned (§12)
├── .env.example           # DATABASE_URL=..., JWT_SECRET=...
└── .gitignore             # .env, .venv, __pycache__
```

On startup: `create_all` (two-table schema; no Alembic). CORS enabled permissively (dev convenience only).

### Endpoints (all under `/api/v1`)

| Method | Path | Auth | Request | Success | Errors |
|---|---|---|---|---|---|
| GET | `/health` | — | — | `{ok:true, version}` (no DB) | — |
| GET | `/health/db` | — | — | `{db:"ok"}` (SELECT 1) | 503 |
| POST | `/auth/signup` | — | `{name, email, password, contact_number}` | 201 `{access_token, token_type, user}` | 400 validation · 409 email exists · 429 |
| POST | `/auth/login` | — | `{email, password}` | 200 `{access_token, token_type, user}` | 401 generic · 429 |
| GET | `/me` | Bearer | — | `{user}` | 401 |
| PUT | `/me` | Bearer | `{name?, contact_number?}` | `{user}` | 400 · 401 |
| PUT | `/me/password` | Bearer | `{current_password, new_password}` | `{ok:true}` | 400 rules · 401 wrong current |

`user` = `{id, name, email, contact_number, created_at}`. Error body always `{"detail": "..."}` (ApiClient convention).

- **Validation rules:** email ≤254 + EmailStr; password ≥8 with letter+digit (both signup & change); name 1–80 stripped; contact_number normalized to digits/`+` (7–15).
- **Hashing:** argon2id via `argon2-cffi` (default params; memory 64 MB is fine on Render free 512MB).
- **Rate limiting (in-memory, per-process):** login: 10 failures / 10 min per email → 429; signup: 5 / hour per IP → 429. Reset on success. (Resets on redeploy — acceptable.)
- **Health check path for Render:** `/api/v1/health`.

### Neon specifics
- Use the **pooled** connection string; **strip `channel_binding=require`** (asyncpg rejects it) — done in `database.py` so pasted URLs "just work"; keep `sslmode=require`.
- `NullPool` (PgBouncer pools), statement caches disabled.
- Project `blindspot` (Singapore) already provisioned; branch `main`, db `neondb`, role `neondb_owner`. Connection string stored ONLY in Render env vars + local `.env` (gitignored). Never in repo, never in logs.

### Render deployment
- Service: `blindspot-api` (free plan, region **singapore**), runtime python, repo `Piyolo/blindspot-backend` (root = repo root), autoDeploy on.
- Build: `pip install -r requirements.txt` · Start: `uvicorn app.main:app --host 0.0.0.0 --port $PORT`.
- Env vars: `DATABASE_URL`, `JWT_SECRET` (64-hex random, generated at provisioning), `PYTHON_VERSION=3.13.5`.
- Managed via Render MCP where available; fallback = dashboard steps documented for the user. Health verified by `GET /api/v1/health` returning 200.

---

## 8. Integration Contract (Flutter ↔ backend)

- `ApiClient` per workspace conventions: single `http.Client`; base URL from `--dart-define=API_BASE_URL` (default = the deployed Render URL once known); paths relative to `/api/v1`; **25 s timeout**; `Authorization: Bearer <token>` from AuthProvider; all errors normalized to `ApiException(message, statusCode?)` (FastAPI `detail` → message).
- Auth repository sets/clears `ApiClient.authToken` on login/logout; session repository persists token/user/contact.
- First request after idle may hit a cold backend → the auth screens show "waking up" UX; one retry before surfacing an error.
- No other endpoints are called at runtime (no telemetry, no analytics).

---

## 9. Assets Reuse Inventory

| Source (old archive) | Destination | Notes |
|---|---|---|
| `app/src/main/assets/yolov8nV5.tflite` | `app/assets/ml/yolov8nV5.tflite` | Model used by the shipped code |
| `app/src/main/assets/labelsV5.txt` | `app/assets/ml/labelsV5.txt` | 14 classes |
| `app/src/main/res/raw/alarm.mp3` | `app/assets/audio/alarm.mp3` | Proximity alarm sound |
| `res/drawable/ic_eye.png`, `ic_brand.png`, `ic_camera.png`, `ic_flashlight.png`, `ic_commandlist.png`, `ic_bulb.png`, `ic_emergency3.png`, `ic_user.png`, `ic_person.png`, `ic_call.png`, `ic_settings.png`, `ic_logout.png`, `ic_wifi_off.png` | `app/assets/images/` | Reused as-is; padded to 48dp targets |
| `res/mipmap-*/ic_launcher*.webp` | regenerate for `com.blindspot.app` | Use `ic_notext`/brand as base |

Not reused: old Kotlin code (reference only), old layouts/animations, other TFLite variants (yolov11n etc.), Depth-Anything model.

---

## 10. Constants (single source of truth → `constants/`)

| Constant | Value |
|---|---|
| Detection input | 640 × 640, ÷255, NHWC |
| Confidence threshold | 0.60 (Balanced) / 0.75 (Conservative) / 0.45 (Aggressive) |
| NMS IoU | 0.50, per-class |
| Frame min-interval | 600 ms; thermal backoff 1200 / 2000 ms |
| Motion gate MAD | ≥18 |
| Focal length (px) | 980 |
| Proximity alarm | < 0.7 m; alarm cooldown 3 s; vibration 300 ms |
| Announce cooldown | 2 s (new label) / 4–5 s random (same label) |
| Distance rounding | ≥1 m → whole meters; <1 m → tens of cm; spoken "approximately" |
| SOS countdown | 5 s, tick 1 s; double-press window 500 ms |
| STT restart debounce | 200 ms; error backoff 1.5 s → 10 s cap |
| Command cooldown | 2 s per command; 3 s identical transcript |
| Battery STT cutoff | 10% |
| JWT expiry | 30 days |
| ApiClient timeout | 25 s + 1 retry on auth calls |

---

## 11. Copy Deck (core EN/FIL strings — full deck reviewed with user in Phase 5)

| Context | English | Filipino |
|---|---|---|
| Boot | BlindSpot is starting. | Nagsisimula ang BlindSpot. |
| Welcome | Welcome to BlindSpot. You can log in, sign up, or ask a trusted person for setup help. | Maligayang pagdating sa BlindSpot. Pwede kang mag-log in, mag-sign up, o humingi ng tulong sa pinagkakatiwalaan mong tao para sa setup. |
| Login field | Email field. Speak your email or tap to type. | Email field. Sabihin ang iyong email o pindutin para mag-type. |
| Password field | Password field. Characters are hidden. | Password field. Nakatago ang mga karakter. |
| Readback | You said [value]. Say "yes" to confirm, or "no" to try again. | Sinabi mo ang [value]. Sabihin ang "oo" para ituloy, o "hindi" para ulitin. |
| Password rule | Password must be at least 8 characters with a letter and a number. | Ang password ay dapat hindi bababa sa 8 karakter, at may titik at numero. |
| Emergency number field | Speak the mobile number, starting with zero nine, or plus six three. | Sabihin ang mobile number, magsisimula sa zero nine, o plus six three. |
| Mic permission | BlindSpot needs the microphone to hear your voice commands. | Kailangan ng BlindSpot ang mikropono para marinig ang iyong mga utos. |
| Camera permission | BlindSpot needs the camera to detect objects around you. | Kailangan ng BlindSpot ang kamera para matukoy ang mga bagay sa paligid mo. |
| Phone permission | BlindSpot needs phone access to call your emergency contact. | Kailangan ng BlindSpot ang akses sa telepono para matawagan ang emergency contact mo. |
| Home | Home. Say "start detection", or choose from the menu. | Home. Sabihin ang "simulan ang detection", o pumili sa menu. |
| SOS hint | For emergencies, press a volume key. It calls after five seconds. | Para sa emergency, pindutin ang volume key. Tatawag ito pagkatapos ng limang segundo. |
| SOS countdown | Emergency call in [n] seconds. Press any volume key to cancel. | Tatawag ng emergency sa [n] segundo. Pindutin ang volume key para kanselahin. |
| SOS cancelled | Emergency call cancelled. | Kinansela ang emergency call. |
| SOS calling | Calling [name]. | Tumatawag na kay [name]. |
| Detection started | Detection started. Point your camera around you. | Nagsimula na ang detection. Itutok ang kamera sa paligid mo. |
| Detection stopped | Detection stopped. | Huminto na ang detection. |
| Announcement (EN mode) | [Object], approximately [distance] [direction]. | [Object], mga [distance], sa [direction]. |
| Hazard | Warning: [object] is very close, about [distance] [direction]. | Babala: napakalapit ang [object], mga [distance], sa [direction]. |
| Nothing found | No objects detected. | Walang natukoy na bagay. |
| Disclaimer (once) | Distances are approximate. Use your own judgment. | Tantya lamang ang mga distansya. Gamitin pa rin ang iyong paghusga. |
| Camera unavailable | Camera unavailable. Allow camera permission in Settings. | Hindi available ang kamera. Payagan ang permiso ng kamera sa Settings. |
| Login error | Invalid email or password. | Mali ang email o password. |
| Email exists | This email is already registered. Try logging in. | Nakarehistro na ang email na ito. Subukang mag-log in. |
| Offline | No internet connection. | Walang koneksyon sa internet. |
| Server waking | The server is waking up. One moment. | Nagigising pa ang server. Sandali lang. |
| Logout confirm | Log out? You will need to sign in again. | Mag-log out? Kailangan mong mag-log in muli. |
| Password changed | Password changed. | Nabago na ang password. |
| Wrong current password | Current password is incorrect. | Mali ang kasalukuyang password. |
| Contacts empty | No contacts yet. Add an emergency contact so BlindSpot can call for help. | Wala pang kontak. Magdagdag ng emergency contact para makatawag ng tulong ang BlindSpot. |
| Call failed | Could not place the call. Check your signal. | Hindi tumuloy ang tawag. Tingnan ang signal mo. |
| Setting changed | [Setting] set to [value]. | Naitakda ang [Setting] sa [value]. |
| FIL voice missing | Filipino voice is not installed on this phone. I will speak English. | Walang Filipino na boses sa teleponong ito. Magsasalita ako sa English. |

**Object label map (announcement when TTS = Filipino):** Bed→Kama · Cabinet→Kabinet · Chair→Silya · Door→Pinto · Electric Fan→Bentilador · Laptop→Laptop · Monitor→Monitor · Person→Tao · Refrigerator→Pridyider · Sink→Lababo · Sofa→Sofa · Table→Mesa · Television→TV · Toilet→Kubeta. Directions: ahead→sa harap · left→sa kaliwa · right→sa kanan.

Filipino copy is a first pass — final deck gets a native-speaker review with the user in Phase 5.

---

## 12. Pinned Dependencies

### Flutter (`app/pubspec.yaml`) — compatible with Flutter 3.41.5 / Dart 3.11.3

| Package | Pin | Note |
|---|---|---|
| tflite_flutter | 0.12.1 | `IsolateInterpreter`; fork `flutter_litert 3.9.4` as fallback if issues |
| camera | 0.12.0+2 | 0.12.1 needs Flutter ≥3.44 — do not upgrade |
| speech_to_text | 7.4.0 | 7.5.0 needs Flutter ≥3.44 — do not upgrade |
| flutter_tts | ^4.2.5 | |
| permission_handler | ^13.0.2 | |
| flutter_secure_storage | ^11.2.0 | |
| shared_preferences | ^2.5.6 | prefer `SharedPreferencesAsync` APIs |
| provider | ^6.1.5+1 | only state-management lib allowed (conventions) |
| wakelock_plus | 1.7.0 | 1.8.x needs Flutter ≥3.44 |

Volume keys: **custom Kotlin MethodChannel** (no third-party package).

### Backend (`backend/requirements.txt`)

fastapi==0.143.3 · uvicorn[standard]==0.54.0 · SQLAlchemy==2.0.55 (2.1 line requires Python ≥3.11; local dev Python is 3.10 — revisit later) · asyncpg==0.32.0 · pydantic==2.14.1 · email-validator==2.3.0 · argon2-cffi==25.1.0 · PyJWT==2.15.1 · pytest + pytest-asyncio + httpx + aiosqlite (dev). Render pins `PYTHON_VERSION=3.13.5`; code stays 3.10-compatible. Exact resolutions confirmed at install time in Phase 0 (adjust pins if a wheel is unavailable for 3.10).

Avoided: passlib (unmaintained), python-jose (CVEs), phone_state (dropped plugin), go_router, Riverpod/Bloc/GetX.

---

## 13. Risk Register

| # | Risk | Sev | Mitigation |
|---|---|---|---|
| 1 | Always-on STT battery drain | High | Pause during TTS/calls/background; listening-mode setting; <10% auto-pause; debounced restarts |
| 2 | False command triggers from ambient speech/TV | High | Whole-phrase anchored matching (no substring), confidence gate, cooldowns, "silence" escape command |
| 3 | TTS ↔ STT feedback loop | High | STT stop before speak; discard transcripts while TTS active; resume on completion |
| 4 | Android volume-key interception leaking to system UI | High | onKeyDown/onKeyUp consume both keys (old MainActivity pattern, proven); device test on Android 13+/OEMs |
| 5 | Camera thermal throttling / battery on long sessions | Med-High | Adaptive interval 600→2000 ms, motion gate, sensor released on background, dim screen |
| 6 | Render cold start ~1 min UX | Med | Offline-first core; 25 s timeout + 1 retry with "waking up" spoken UX |
| 7 | Neon pooler prepared-statement failures | Med | NullPool + statement caches off; `channel_binding` stripped in code |
| 8 | TFLite/plugin version skew on Flutter 3.41.5 | Med | Exact pins (§12); no NNAPI/GPU delegates; smoke-test in Phase 0/2 |
| 9 | fil-PH TTS voice missing on device | Med | `isLanguageAvailable` gate + English fallback + spoken notice |
| 10 | Offline recognizer missing on some devices | Med | Graceful: online recognizer when available; UI states honest; documented |
| 11 | Render/GitHub integration friction (MCP availability, private-repo access) | Med | Public repos; fallback = dashboard setup instructions for user |
| 12 | Emergency call silently blocked (airplane/DND/OEM) | Med | ACTION_CALL → ACTION_DIAL fallback + distinct haptic/beep + spoken failure |
| 13 | Windows long-path/OneDrive build issues | Low | Projects at short path `Dev\BlindSpot` outside OneDrive |
| 14 | Mic/camera permission denied permanently | Low | Spoken guidance + intent to app settings; limited-mode operation |

---

## 14. Verification Strategy (per phase)

For every phase: **mas-developer implements → mas-tester verifies → mas-reviewer audits the diff → fixes (developer) → re-verify.** Evidence before claims: build logs, `flutter analyze`, `dart test`, backend `pytest`, curl outputs, on-device checklists.

- **Static:** `flutter analyze` zero errors; `python -m compileall` / pytest green.
- **Unit tests (pure Dart):** command_matcher (≥40 cases incl. near-miss noise, both languages), yolo_postprocess (golden output tensors for both shape layouts), depth_estimator, copy formatting, message templates.
- **Widget tests:** screens render + semantics labels exist (smoke, mocked providers).
- **Device checklist (user's phone, sideloaded APK):** §6.1 detection behaviors, §6.2 command recognition (≥90% of 18 phrases, ≤2 false triggers in 30-min ambient test), §6.4 gestures + system-volume-never-changes, offline scenarios (airplane), 10-min soak, thermal log.
- **Backend:** pytest suite against SQLite; live smoke via curl on Render (health, signup, login, me, password change, 401/409/429 paths).
- **Final:** release APK build; ISO-25010-flavored checklist rerun; known-issues documented.

---

## 15. Implementation Phases (order locked)

**Phase 0 — Scaffold + assets + deployment pipeline.** Repos (`blindspot-app`, `blindspot-backend`, `blindspot-workspace`) created & pushed; Flutter project compiles with theme + navigation skeleton + ApiClient; assets copied; backend skeleton with `/health` + `/health/db` deployed to Render (`blindspot-api`, Neon wired); debug APK builds.
Acceptance: APK installs/runs; live health endpoints 200; repos auto-deploy wired.

**Phase 1 — Auth + offline session.** Backend: models/schemas/security/routers/tests/rate-limit. App: Auth/Profile/Session repos (trios), TokenStore, providers, Welcome/Login/Signup/Permissions/Profile screens, error mapping, cold-start UX, offline cache.
Acceptance: signup→login→edit→password change works on live backend; airplane relaunch stays signed in; pytest green; 429/409/401 paths verified.

**Phase 2 — Detection core.** Services per §6.1 + TtsService queue + detection screen + overlay + object list + DetectionProvider; camera permission flow.
Acceptance: on-device announcement ≤2 s of object appearing, sane distance/direction, overlay aligned, proximity alarm <0.7 m, 10-min soak stable, thermal backoff logged, unit tests green, works in airplane mode.

**Phase 3 — SOS volume-key flow.** MainActivity interception + channel + CallHelper; SosService state machine + overlay + provider; post-call resume.
Acceptance: gestures per §6.4 on device; system volume never changes; cancel works; double-press immediate; graceful failure on airplane; haptics verified.

**Phase 4 — Always-on command engine.** SttService loop, matcher + tests, actions, echo prevention, transcript UI, provider wiring.
Acceptance: ≥90% recognition of the 18 phrases (EN+FIL) on device; ≤2 false triggers in a 30-min ambient test AND no call-capable misfire; TTS pause/resume cycle leak-free; battery sanity check.

**Phase 5 — Settings, onboarding, polish.** Settings screen + persistence, spoken onboarding/tutorial, help/command reference, practice mode (stretch), high-contrast theme, a11y sweep (48dp/semantics/200%), copy deck finalization **with user review**, motion polish.
Acceptance: all settings persist; full blind-user walkthrough (SOS cancel included) passes; copy deck approved; a11y checklist complete.

**Phase 6 — Hardening, device QA, release.** Risk-register fixes, full-diff mas-reviewer audit, regression rerun, release APK (shrunk), README + demo script, known-issues list.
Acceptance: release APK installed on user's phone; all phase criteria rerun; documentation complete.

---

## 16. Future Work (explicitly deferred)

sherpa_onnx streaming offline KWS for gap-free listening · refresh-token rotation + revocation · multiple emergency contacts / SMS fallback on call failure (needs an SMS gateway) · iOS port investigation · cloud detection fallback for low-light · OCR/scene captioning · account deletion (Data Privacy Act compliance) · Play Store release (icon/signing/listing) · Alembic migrations · per-command language matching.

---

## 17. Provisioning Status (as of spec date)

- Neon project `blindspot` created (region Singapore; branch `main`; db `neondb`). Pooled connection string stored outside the repo (temp secrets + will live in Render env vars only). Neon API key used for provisioning — **recommended to rotate it in the Neon console once deployment is complete**.
- Render workspace: `My Workspace` (free). Service will be created in Phase 0.
- GitHub: `Piyolo` (authenticated via `gh` on this machine).
- Local toolchain: Flutter 3.41.5 / Dart 3.11.3 · Python 3.10.11 · Windows 11. Do **not** run `pub upgrade --major-versions`.
