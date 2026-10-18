# BlindSpot Phase 0 — Scaffold & Deployment Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. In this project, execution runs through the **MAS team**: `mas-developer` implements each task (receives that task's text verbatim), `mas-tester` independently re-runs the verification commands before the next task starts, and `mas-reviewer` audits the full Phase 0 diff across all repos at the end. One task = one commit minimum.

**Goal:** Stand up the BlindSpot rebuild's foundations: a buildable Flutter Android app (all dependencies pinned and resolved, original model/assets imported) and a live FastAPI backend on Render backed by Neon Postgres — with all three repos on GitHub.

**Architecture:** Two code repos (`app/` Flutter, `backend/` FastAPI) inside a workspace repo (this one, docs only). Phase 0 delivers **scaffolding + the deployment pipeline only** — no detection, TTS/STT, SOS, or auth logic (those are Phases 1–6, each with its own plan written at phase start). The backend runs as Render runtime `python` (free plan, Singapore) and reads `DATABASE_URL` (Neon pooled URL, sanitized in code for asyncpg/PgBouncer) and `JWT_SECRET` from environment variables.

**Tech Stack:** Flutter 3.41.5 / Dart 3.11.3 · FastAPI 0.143.3 · SQLAlchemy 2.0.55 (async) · asyncpg 0.32.0 · Neon Postgres · Render (Singapore) · Windows 11 + PowerShell · GitHub CLI (`Piyolo`) · Render MCP tools · filesystem MCP (workspace-scoped).

**Spec:** `docs/superpowers/specs/2026-10-17-blindspot-rebuild-design.md` (§9 assets, §10 constants, §12 pins, §15 Phase 0, §17 provisioning status). Executors should read the spec alongside this plan.

## Starting State (verified 2026-10-17)

- Workspace `C:\Users\Piolo Mangilog\Dev\BlindSpot` is a git repo (commit `fc786c7`, docs only). `app/` and `backend/` **do not exist yet**.
- Old archive (read-only donor): `C:\Users\Piolo Mangilog\OneDrive\Documents\Academic Documents\BlindSpot\BlindSpot\`. Verified: `app\src\main\assets\yolov8nV5.tflite` (12,290,965 B), `assets\labelsV5.txt` (117 B, exactly: Bed, Cabinet, Chair, Door, Electric Fan, Laptop, Monitor, Person, Refrigerator, Sink, Sofa, Table, Television, Toilet), `res\raw\alarm.mp3` (50,105 B), all needed `res\drawable\*.png` icons, `res\mipmap-*\ic_launcher*.webp`.
- GitHub `Piyolo` authenticated via `gh`. **Existing repos to never touch:** `blindspot-backend`, `blindspot-site`, `blindspot-roboflow-proxy`. Names verified free: `blindspot-app`, `blindspot-api`, `blindspot-workspace`.
- Render workspace id `tea-d35dt5ogjchc73dvvo50` ("My Workspace"). **Existing services never to modify:** `blindspot` (legacy, repo blindspot-backend), `blindspot-roboflow-proxy`, and all others. New service name: `blindspot-api` (no collision).
- Render MCP tools confirmed available: `create_web_service` (accepts `envVars` inline, `autoDeploy`, `plan`, `region`, `runtime`, `buildCommand`, `startCommand`), `update_environment_variables`, `trigger_deploy`, `get_deploy`, `list_deploys`, `get_service`, `list_services`.
- Filesystem MCP can read **only** the workspace directory → secrets are staged into a gitignored `.secrets/` folder for the deploy step.
- Secrets live in `C:\Users\Piolo Mangilog\AppData\Local\Temp\opencode\blindspot_secrets\` (`neon_blindspot.env` contains a `DATABASE_URL=` line — verified present; value must never be printed).
- Local toolchain: Flutter 3.41.5, Python 3.10.11, `gh` authed, JDK/Android SDK via Flutter (verify in Task 1). No Android device on adb → on-device install is user-verified at the end (see Phase Exit Criteria).

## Global Constraints

Copied from the spec — every task implicitly includes these:

- **Flutter 3.41.5 / Dart 3.11.3.** Never run `pub upgrade --major-versions`. Exact pins (§12): `tflite_flutter: 0.12.1`, `camera: 0.12.0+2`, `speech_to_text: 7.4.0`, `wakelock_plus: 1.7.0` (plain pins), `flutter_tts ^4.2.5`, `permission_handler ^13.0.2`, `flutter_secure_storage ^11.2.0`, `shared_preferences ^2.5.6`, `provider ^6.1.5+1`.
- **Python compatibility:** backend code must run on local Python **3.10.11** AND Render's **3.13.5** → write 3.10-compatible syntax. Backend pins: `fastapi==0.143.3`, `uvicorn[standard]==0.54.0`, `SQLAlchemy==2.0.55`, `asyncpg==0.32.0`, `pydantic==2.14.1`, `email-validator==2.3.0`, `argon2-cffi==25.1.0`, `PyJWT==2.15.1`.
- **Android identity:** applicationId & namespace = `com.blindspot.app`; app label = `BlindSpot`; `minSdk = 24`; compile/target = Flutter defaults. Permissions exactly: `INTERNET`, `ACCESS_NETWORK_STATE`, `CAMERA`, `RECORD_AUDIO`, `CALL_PHONE`, `VIBRATE`, `WAKE_LOCK` + `<queries>` for `android.intent.action.TTS_SERVICE` and `android.speech.RecognitionService`. No other permissions in Phase 0.
- **Secrets:** never commit, never echo, never put in tool-call results. `DATABASE_URL`/`JWT_SECRET` live only in `.secrets/` (gitignored), OS env during local runs, and Render env vars. `.env`/`.env.*` gitignored in every repo. Test fixtures must use fake credentials.
- **Old archive is a read-only donor** — copy out of it, never write into it. Old repos and Render services are untouched.
- **PowerShell on Windows.** Run every command from the stated CWD (repo dirs: `app/`, `backend/`, workspace root).
- **Commits:** conventional prefixes (`chore:`, `feat:`, `test:`, `docs:`); one commit per task minimum; push happens in Tasks 7–8.
- **Phase 0 scope only:** no detection/TTS/STT/SOS/auth implementation yet.

## Review Focus

The five failure modes most likely to bite, each pinned to a verifying test in its owning task:

1. **Secrets leaking into commits, logs, or tool output.** → Task 6: `.gitignore` written before any venv/env work; live ping prints only `neon ping ok`; tests use fake URLs. Task 8: `.secrets/` staged + `git check-ignore` proven before use; MCP reads route values without echoing; results returned contain no secret values.
2. **Dependency resolution on Flutter 3.41.5 drifting or failing.** → Task 1: `flutter pub get` + full `flutter build apk --debug` (all plugins compile) with exact pins; contingency = adjust only the failing pin to the nearest compatible one, record it in the commit message — never upgrade Flutter or other pins.
3. **Android identity drift** (old `com.example`/`blindspot_app` package left behind, wrong applicationId, missing label/permissions). → Task 1 edits + grep for `com.blindspot.blindspot_app` in `android/` (expect none); Task 3: merged-manifest check for all 7 permissions + 2 queries.
4. **Neon pooler breakage via asyncpg** (stale prepared statements, `channel_binding` rejection). → Task 6: 5 sanitizer unit tests + live `ping_db()` through the pooled host printing `neon ping ok`. Task 8: live `/health/db` returns `{"db":"ok"}`.
5. **Render creation friction** (repo access, slug, cold start making checks flaky). → Task 8: service name `blindspot-api` (verified free), URL read from the API response (never guessed), health polls with generous timeouts + retries; contingency = dashboard fallback steps; never touch legacy services.

## Execution Protocol (MAS)

Tasks run **in order 1 → 8**. For each task: mas-developer executes the steps and commits; mas-tester independently re-runs the task's verification commands and the relevant Review Focus check before the next task starts. If a task fails twice on the same issue, stop and report to the lead with evidence. Record any deviation (e.g., a changed pin) in the commit message. Do not edit this plan or the spec; report needed changes to the lead.

---

### Task 1: Flutter project scaffold with pinned dependencies

**Files:**
- Create: `app/` (via `flutter create`), `app/pubspec.yaml` (edit), `app/android/app/build.gradle.kts` (edit), `app/android/app/src/main/AndroidManifest.xml` (label edit), `app/android/app/src/main/kotlin/com/blindspot/app/MainActivity.kt` (new), delete old `.../com/blindspot/blindspot_app/MainActivity.kt`
- Test: build + analyze (no unit tests yet)

**Interfaces:**
- Consumes: nothing (first task).
- Produces: Flutter project named `blindspot_app` at `app/`, applicationId/namespace `com.blindspot.app`, minSdk 24, all §12 dependencies resolved and compiling. Every later app task builds on this.

- [ ] **Step 1: Preflight toolchain**

Run (workspace root):
```powershell
flutter doctor
```
Expected: "Android toolchain - develop for Android devices" shows `[√]`. If it shows `[X]`, STOP and report (needs SDK/JDK install; do not proceed).

- [ ] **Step 2: Create the project**

Run (workspace root):
```powershell
flutter create --empty --platforms android --org com.blindspot --project-name blindspot_app app
```
Expected: "All done!" and `app\lib\main.dart` exists.

- [ ] **Step 3: Edit `app/pubspec.yaml`**

Set `description: "BlindSpot — household object detection for the visually impaired (Flutter rebuild)."` and make the dependencies block exactly:

```yaml
dependencies:
  flutter:
    sdk: flutter
  provider: ^6.1.5+1
  http: ^1.2.0
  tflite_flutter: 0.12.1
  camera: 0.12.0+2
  speech_to_text: 7.4.0
  flutter_tts: ^4.2.5
  permission_handler: ^13.0.2
  flutter_secure_storage: ^11.2.0
  shared_preferences: ^2.5.6
  wakelock_plus: 1.7.0
```
(Remove any generated `cupertino_icons` dependency. Leave `dev_dependencies` defaults — `flutter_test`, `flutter_lints`.)

- [ ] **Step 4: Resolve dependencies**

Run (in `app/`): `flutter pub get`
Expected: "Got dependencies!" — **if resolution fails**, adjust ONLY the failing package to the nearest version compatible with Flutter 3.41.5 (record the change + reason in the Task 1 commit message); never run major upgrades.

- [ ] **Step 5: Set Android identity**

In `app/android/app/build.gradle.kts`: `namespace = "com.blindspot.app"`, `applicationId = "com.blindspot.app"`, `minSdk = 24` (leave compileSdk/targetSdk on `flutter.*`).

Create `app/android/app/src/main/kotlin/com/blindspot/app/MainActivity.kt`:

```kotlin
package com.blindspot.app

import io.flutter.embedding.android.FlutterActivity

class MainActivity : FlutterActivity()
```

Delete the old `app/android/app/src/main/kotlin/com/blindspot/blindspot_app/MainActivity.kt` (and its now-empty directory).

In `app/android/app/src/main/AndroidManifest.xml` set `android:label="BlindSpot"` on `<application>`.

- [ ] **Step 6: Verify identity + analyze**

Run (in `app/`):
```powershell
Select-String -Path "android\app\build.gradle.kts" -Pattern 'com\.blindspot\.app', 'minSdk'
(Test-Path "android\app\src\main\kotlin\com\blindspot\blindspot_app")
flutter analyze
```
Expected: two matching lines; `False`; "No issues found!".

- [ ] **Step 7: Build the debug APK**

Run (in `app/`): `flutter build apk --debug` (first run downloads Gradle — allow up to 15 min).
Expected: "√ Built build\app\outputs\flutter-apk\app-debug.apk". If it fails on an NDK version mismatch, set `ndkVersion` in `build.gradle.kts` to the version named in the error and rebuild.

- [ ] **Step 8: Init repo and commit**

Run (in `app/`):
```powershell
git init -b main
git add -A
git commit -m "chore: scaffold Flutter app with pinned dependencies (Phase 0)"
```

---

### Task 2: Import reuse assets + integrity tests

**Files:**
- Create: `app/assets/ml/yolov8nV5.tflite`, `app/assets/ml/labelsV5.txt`, `app/assets/audio/alarm.mp3`, `app/assets/images/*.png` (16 files), launcher icons overwritten in `app/android/app/src/main/res/mipmap-{mdpi,hdpi,xhdpi,xxhdpi,xxxhdpi}/`
- Modify: `app/pubspec.yaml` (assets section)
- Test: `app/test/assets_test.dart`

**Interfaces:**
- Consumes: `app/` project from Task 1.
- Produces: bundled asset paths — `assets/ml/yolov8nV5.tflite`, `assets/ml/labelsV5.txt`, `assets/audio/alarm.mp3`, `assets/images/ic_back.png|ic_backwhite.png|ic_brand.png|ic_bulb.png|ic_call.png|ic_camera.png|ic_commandlist.png|ic_emergency3.png|ic_eye.png|ic_flashlight.png|ic_logout.png|ic_notext.png|ic_person.png|ic_settings.png|ic_user.png|ic_wifi_off.png`. Phase 2 (detection) and Phase 5 (polish) consume these.

- [ ] **Step 1: Copy binaries and images**

Run (workspace root):
```powershell
$src = "C:\Users\Piolo Mangilog\OneDrive\Documents\Academic Documents\BlindSpot\BlindSpot\app\src\main"
New-Item -ItemType Directory -Force -Path "app\assets\ml","app\assets\audio","app\assets\images" | Out-Null
Copy-Item "$src\assets\yolov8nV5.tflite" "app\assets\ml\"
Copy-Item "$src\assets\labelsV5.txt" "app\assets\ml\"
Copy-Item "$src\res\raw\alarm.mp3" "app\assets\audio\"
"ic_back.png","ic_backwhite.png","ic_brand.png","ic_bulb.png","ic_call.png","ic_camera.png","ic_commandlist.png","ic_emergency3.png","ic_eye.png","ic_flashlight.png","ic_logout.png","ic_notext.png","ic_person.png","ic_settings.png","ic_user.png","ic_wifi_off.png" | ForEach-Object { Copy-Item "$src\res\drawable\$_" "app\assets\images\" }
foreach ($d in "mdpi","hdpi","xhdpi","xxhdpi","xxxhdpi") {
  Copy-Item "$src\res\mipmap-$d\ic_launcher.webp" "app\android\app\src\main\res\mipmap-$d\" -Force
  Copy-Item "$src\res\mipmap-$d\ic_launcher_round.webp" "app\android\app\src\main\res\mipmap-$d\" -Force
}
```

- [ ] **Step 2: Verify copies by hash (source vs destination)**

Run (workspace root):
```powershell
$src = "C:\Users\Piolo Mangilog\OneDrive\Documents\Academic Documents\BlindSpot\BlindSpot\app\src\main"
(Get-FileHash "app\assets\ml\yolov8nV5.tflite").Hash -eq (Get-FileHash "$src\assets\yolov8nV5.tflite").Hash
(Get-FileHash "app\assets\ml\labelsV5.txt").Hash -eq (Get-FileHash "$src\assets\labelsV5.txt").Hash
(Get-FileHash "app\assets\audio\alarm.mp3").Hash -eq (Get-FileHash "$src\res\raw\alarm.mp3").Hash
```
Expected: `True` three times.

- [ ] **Step 3: Register assets in `app/pubspec.yaml`**

```yaml
flutter:
  uses-material-design: true
  assets:
    - assets/ml/
    - assets/audio/
    - assets/images/
```

- [ ] **Step 4: Write `app/test/assets_test.dart`**

```dart
import 'package:flutter/services.dart' show rootBundle;
import 'package:flutter_test/flutter_test.dart';

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();

  test('labelsV5.txt has exactly the 14 expected class names', () async {
    final text = await rootBundle.loadString('assets/ml/labelsV5.txt');
    final labels = text
        .trim()
        .split('\n')
        .map((l) => l.trim())
        .where((l) => l.isNotEmpty)
        .toList();
    expect(labels.length, 14);
    expect(labels.first, 'Bed');
    expect(labels.last, 'Toilet');
  });

  test('yolov8nV5.tflite loads and matches the frozen model size', () async {
    final data = await rootBundle.load('assets/ml/yolov8nV5.tflite');
    expect(data.lengthInBytes, 12290965);
  });

  test('alarm.mp3 loads', () async {
    final data = await rootBundle.load('assets/audio/alarm.mp3');
    expect(data.lengthInBytes, 50105);
  });
}
```

- [ ] **Step 5: Run tests**

Run (in `app/`): `flutter test test/assets_test.dart`
Expected: "All tests passed!"

- [ ] **Step 6: Commit**

Run (in `app/`):
```powershell
git add -A
git commit -m "feat: import reuse assets (model, labels, alarm, icons, launcher) with integrity tests"
```

---

### Task 3: Android manifest — permissions + queries

**Files:**
- Modify: `app/android/app/src/main/AndroidManifest.xml`
- Test: merged-manifest grep after build

**Interfaces:**
- Consumes: `app/` project from Task 1.
- Produces: manifest declaring exactly the Phase 0 permission set and the TTS/Speech `<queries>`. Later phases rely on these being present (camera, mic, phone call).

- [ ] **Step 1: Add permissions and queries**

In `app/android/app/src/main/AndroidManifest.xml`, before `<application>` add:

```xml
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
    <uses-permission android:name="android.permission.CAMERA"/>
    <uses-permission android:name="android.permission.RECORD_AUDIO"/>
    <uses-permission android:name="android.permission.CALL_PHONE"/>
    <uses-permission android:name="android.permission.VIBRATE"/>
    <uses-permission android:name="android.permission.WAKE_LOCK"/>
```

After `</application>` add:

```xml
    <queries>
        <intent>
            <action android:name="android.intent.action.TTS_SERVICE"/>
        </intent>
        <intent>
            <action android:name="android.speech.RecognitionService"/>
        </intent>
    </queries>
```

- [ ] **Step 2: Build**

Run (in `app/`): `flutter build apk --debug`
Expected: "√ Built build\app\outputs\flutter-apk\app-debug.apk".

- [ ] **Step 3: Verify the merged manifest**

Run (in `app/`):
```powershell
$m = Get-ChildItem "build" -Recurse -Filter "AndroidManifest.xml" | Where-Object { $_.FullName -match "merged_manifest" } | Select-Object -First 1
if (-not $m) { throw "merged manifest not found" }
$content = Get-Content $m.FullName -Raw
@("android.permission.INTERNET","android.permission.ACCESS_NETWORK_STATE","android.permission.CAMERA","android.permission.RECORD_AUDIO","android.permission.CALL_PHONE","android.permission.VIBRATE","android.permission.WAKE_LOCK","android.intent.action.TTS_SERVICE","android.speech.RecognitionService") | ForEach-Object { if ($content -notmatch [regex]::Escape($_)) { throw "MISSING: $_" } }
"merged manifest OK"
```
Expected: `merged manifest OK`.

- [ ] **Step 4: Commit**

Run (in `app/`):
```powershell
git add -A
git commit -m "feat: declare camera/mic/call permissions and TTS/speech queries"
```

---

### Task 4: App shell — theme tokens, splash, placeholder home, smoke tests

**Files:**
- Create: `app/lib/theme/tokens.dart`, `app/lib/theme/app_theme.dart`, `app/lib/screens/splash_screen.dart`, `app/lib/screens/home_screen.dart`
- Rewrite: `app/lib/main.dart`
- Test: `app/test/app_smoke_test.dart`

**Interfaces:**
- Consumes: Task 1 (deps), Task 2 (`assets/images/ic_eye.png`).
- Produces: `BlindSpotApp` (root widget, used by every later test); `AppColors` with fields `surface`, `elevated`, `container`, `border`, `onSurface`, `variant`, `brand`, `brandLight`, `cream`, `onCream`, `error`, `errorText`, `errorSurface`, `warning`, `success`; `AppTextStyles` with fields `display`, `screenTitle`, `section`, `bodyLarge`, `body`, `button`, `caption`, `status`, `countdown`; `AppTheme.dark()` → `ThemeData`; `SplashScreen`; `HomeScreen`.

- [ ] **Step 1: Write `app/lib/theme/tokens.dart`**

`AppColors` — use these exact values: `surface 0xFF0A101C`, `elevated 0xFF121A2C`, `container 0xFF162033`, `border 0xFF334056`, `onSurface 0xFFFFFFFF`, `variant 0xFFB9C2D6`, `brand 0xFF2C5A86`, `brandLight 0xFF7FB3D9`, `cream 0xFFF2DEC1`, `onCream 0xFF0E2A4A`, `error 0xFFD9534F`, `errorText 0xFFF5A9A3`, `errorSurface 0xFF3A1A18`, `warning 0xFFF0B429`, `success 0xFF4CAF50`.

`AppTextStyles` (color = `AppColors.onSurface` unless noted): `display` 30/w700 · `screenTitle` 24/w700 · `section` 20/w600 · `bodyLarge` 18/w400 (the app default body size) · `body` 16/w400 · `button` 18/w600 · `caption` 14/w400 (color `AppColors.variant`) · `status` 16/w600 · `countdown` 64/w700.

- [ ] **Step 2: Write `app/lib/theme/app_theme.dart`**

`AppTheme.dark()` → `ThemeData(brightness: Brightness.dark, scaffoldBackgroundColor: AppColors.surface, colorScheme: ColorScheme.dark(primary: AppColors.brandLight, surface: AppColors.surface, error: AppColors.error), appBarTheme: AppBarTheme(backgroundColor: AppColors.elevated, foregroundColor: AppColors.onSurface, elevation: 0, centerTitle: false), textTheme: ...)` built from `AppTextStyles`.

- [ ] **Step 3: Write `app/lib/main.dart`, `splash_screen.dart`, `home_screen.dart`**

- `BlindSpotApp` (StatelessWidget, `const` constructor): `MaterialApp(title: 'BlindSpot', theme: AppTheme.dark(), home: const SplashScreen())`.
- `SplashScreen` (Stateful): centered column — `Image.asset('assets/images/ic_eye.png', width: 96)`, `Text('BlindSpot', style: AppTextStyles.display)`. In `initState`, `_timer = Timer(const Duration(milliseconds: 1500), ...)` → `Navigator.of(context).pushReplacement(MaterialPageRoute(builder: (_) => const HomeScreen()))` (guard with `mounted`); cancel the timer in `dispose`.
- `HomeScreen` (Stateless): `AppBar(title: Text('BlindSpot'))`, body `Center(child: Padding(..., child: Text('Detection and voice features arrive in the next phases.', style: AppTextStyles.bodyLarge, textAlign: TextAlign.center)))`.

- [ ] **Step 4: Write `app/test/app_smoke_test.dart`**

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:blindspot_app/main.dart';
import 'package:blindspot_app/screens/home_screen.dart';

void main() {
  testWidgets('app boots to splash then navigates to home', (tester) async {
    await tester.pumpWidget(const BlindSpotApp());
    expect(find.text('BlindSpot'), findsWidgets);
    await tester.pump(const Duration(seconds: 2));
    await tester.pumpAndSettle();
    expect(find.byType(HomeScreen), findsOneWidget);
  });
}
```

- [ ] **Step 5: Analyze + test**

Run (in `app/`): `flutter analyze` → "No issues found!"; `flutter test` → "All tests passed!".

- [ ] **Step 6: Commit**

Run (in `app/`):
```powershell
git add -A
git commit -m "feat: dark theme tokens, splash screen, placeholder home, smoke test"
```

---

### Task 5: ApiClient + ApiException + unit tests

**Files:**
- Create: `app/lib/models/api_exception.dart`, `app/lib/services/api_client.dart`
- Test: `app/test/api_client_test.dart`

**Interfaces:**
- Consumes: Task 1 (`http` dependency).
- Produces: `ApiException` (`final String message; final int? statusCode;` implements `Exception`) and `ApiClient` with: constructor `ApiClient({http.Client? client, String? baseUrl, Duration timeout = const Duration(seconds: 25)})`; `static const String defaultBaseUrl = String.fromEnvironment('API_BASE_URL', defaultValue: 'https://blindspot-api.onrender.com')`; mutable `String? authToken`; `Future<Map<String, dynamic>> get(String path)`, `post(String path, {Map<String, dynamic>? body})`, `put(String path, {Map<String, dynamic>? body})`. Phase 1's auth repository consumes all of this. Task 8 updates `defaultBaseUrl` if the real URL differs.

Behavior (pins): requests go to `'$_baseUrl/api/v1$path'`; headers `Content-Type: application/json`, `Accept: application/json`, plus `Authorization: Bearer $authToken` when `authToken != null`; each request wrapped in `.timeout(_timeout)`; success = 2xx → `jsonDecode(utf8.decode(response.bodyBytes))` as `Map<String, dynamic>` (empty body → `{}`); non-2xx → parse body's `detail` string → `ApiException(detail, statusCode: status)`; unparsable detail → `ApiException('Request failed', statusCode: status)`; `TimeoutException`, `http.ClientException`, `SocketException` → `ApiException('Network error')` (`statusCode: null`).

- [ ] **Step 1: Write the failing tests** — `app/test/api_client_test.dart`

Copy this (imports: `package:flutter_test/flutter_test.dart`, `package:http/http.dart` as http, `package:http/testing.dart`, `package:blindspot_app/models/api_exception.dart`, `package:blindspot_app/services/api_client.dart`; wrap the four tests in `void main() { ... }`):

```dart
test('GET sends bearer token, hits /api/v1, and parses JSON', () async {
  late http.Request captured;
  final client = ApiClient(
    baseUrl: 'https://example.test',
    client: MockClient((req) async {
      captured = req;
      return http.Response('{"ok": true}', 200);
    }),
  );
  client.authToken = 't123';
  final data = await client.get('/health');
  expect(data['ok'], true);
  expect(captured.url.toString(), 'https://example.test/api/v1/health');
  expect(captured.headers['Authorization'], 'Bearer t123');
});

test('no token means no Authorization header', () async {
  late http.Request captured;
  final client = ApiClient(
    baseUrl: 'https://example.test',
    client: MockClient((req) async {
      captured = req;
      return http.Response('{}', 200);
    }),
  );
  await client.get('/health');
  expect(captured.headers.containsKey('Authorization'), false);
});

test('non-2xx maps FastAPI detail to ApiException', () async {
  final client = ApiClient(
    baseUrl: 'https://example.test',
    client: MockClient((req) async =>
        http.Response('{"detail": "Invalid email or password."}', 401)),
  );
  await expectLater(
    client.post('/auth/login', body: {}),
    throwsA(isA<ApiException>()
        .having((e) => e.statusCode, 'statusCode', 401)
        .having((e) => e.message, 'message', 'Invalid email or password.')),
  );
});

test('timeout maps to network ApiException', () async {
  final client = ApiClient(
    baseUrl: 'https://example.test',
    timeout: const Duration(milliseconds: 50),
    client: MockClient((req) async {
      await Future<void>.delayed(const Duration(milliseconds: 300));
      return http.Response('{}', 200);
    }),
  );
  await expectLater(
    client.get('/health'),
    throwsA(isA<ApiException>()
        .having((e) => e.message, 'message', 'Network error')
        .having((e) => e.statusCode, 'statusCode', isNull)),
  );
});
```

- [ ] **Step 2: Run to verify failure**

Run (in `app/`): `flutter test test/api_client_test.dart`
Expected: FAIL — "Error: Couldn't resolve the package 'blindspot_app'" or ApiClient undefined.

- [ ] **Step 3: Implement `api_exception.dart` and `api_client.dart`**

Per the interface block above. Use `utf8.decode(response.bodyBytes)` (not `response.body`) so Filipino text decodes correctly.

- [ ] **Step 4: Run to verify pass**

Run (in `app/`): `flutter test test/api_client_test.dart`
Expected: "All tests passed!"

- [ ] **Step 5: Commit**

Run (in `app/`):
```powershell
git add -A
git commit -m "feat: ApiClient with bearer auth, timeouts, and ApiException mapping"
```

---

### Task 6: Backend skeleton — health endpoints, Neon-safe database layer, tests, live ping

**Files:**
- Create: `backend/app/__init__.py`, `backend/app/config.py`, `backend/app/database.py`, `backend/app/main.py`, `backend/app/routers/__init__.py`, `backend/app/routers/health.py`, `backend/requirements.txt`, `backend/requirements-dev.txt`, `backend/pytest.ini`, `backend/.env.example`, `backend/.gitignore`, `backend/README.md`, `backend/tests/test_health.py`, `backend/tests/test_database.py`

**Interfaces:**
- Consumes: local Python 3.10.11; the temp secrets file for the live ping step only.
- Produces: `app.config.get_database_url() -> str`, `app.config.get_jwt_secret() -> str`, `app.config.APP_VERSION = "0.1.0"`, `app.config.ACCESS_TTL_SECONDS = 2592000`; `app.database.sanitize_database_url(url: str) -> str`, `app.database.get_engine() -> AsyncEngine`, `app.database.ping_db() -> None`; FastAPI app `app.main:app` with routes `GET /api/v1/health` and `GET /api/v1/health/db`. Phase 1 builds models/routers on top of config/database.

- [ ] **Step 1: Create the package files**

`backend/app/__init__.py` and `backend/app/routers/__init__.py`: empty.

`backend/app/config.py`:

```python
import os

APP_VERSION = "0.1.0"
ACCESS_TTL_SECONDS = 2592000  # 30 days


def get_database_url() -> str:
    url = os.getenv("DATABASE_URL")
    if not url:
        raise RuntimeError("DATABASE_URL is not set")
    return url


def get_jwt_secret() -> str:
    secret = os.getenv("JWT_SECRET")
    if not secret:
        raise RuntimeError("JWT_SECRET is not set")
    return secret
```

`backend/app/database.py` — implement `sanitize_database_url` using `urllib.parse` (`urlsplit`/`parse_qsl`/`urlencode`/`urlunsplit`): strip whitespace; drop any `channel_binding` query param; force `prepared_statement_cache_size=0` (replace if present); rewrite scheme `postgresql://` or `postgres://` → `postgresql+asyncpg://`; keep all other params; must be idempotent. Then:

```python
from urllib.parse import parse_qsl, urlencode, urlsplit, urlunsplit

from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncEngine, create_async_engine
from sqlalchemy.pool import NullPool

from app.config import get_database_url

_engine: AsyncEngine | None = None


def get_engine() -> AsyncEngine:
    global _engine
    if _engine is None:
        _engine = create_async_engine(
            sanitize_database_url(get_database_url()),
            poolclass=NullPool,
            connect_args={"statement_cache_size": 0},
        )
    return _engine


async def ping_db() -> None:
    async with get_engine().connect() as conn:
        await conn.execute(text("SELECT 1"))
```
(The `NullPool` + `prepared_statement_cache_size=0` + `statement_cache_size=0` combination is the documented PgBouncer-safe configuration for the asyncpg dialect. `prepared_statement_cache_size=0` is forced inside `sanitize_database_url` as a URL parameter.)

`backend/app/routers/health.py`:

```python
from fastapi import APIRouter
from fastapi.responses import JSONResponse

from app import config
from app.database import ping_db

router = APIRouter(tags=["health"])


@router.get("/health")
async def health() -> dict:
    return {"ok": True, "version": config.APP_VERSION}


@router.get("/health/db")
async def health_db():
    try:
        await ping_db()
    except Exception:
        return JSONResponse(status_code=503, content={"detail": "database unavailable"})
    return {"db": "ok"}
```
(The `from app.database import ping_db` import style is required so tests can monkeypatch `app.routers.health.ping_db`.)

`backend/app/main.py`:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app import config
from app.routers import health

app = FastAPI(title="BlindSpot API", version=config.APP_VERSION)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(health.router, prefix="/api/v1")
```

- [ ] **Step 2: Create requirements, gitignore, env example, README**

`backend/requirements.txt` (exact):
```
fastapi==0.143.3
uvicorn[standard]==0.54.0
SQLAlchemy==2.0.55
asyncpg==0.32.0
pydantic==2.14.1
email-validator==2.3.0
argon2-cffi==25.1.0
PyJWT==2.15.1
```

`backend/requirements-dev.txt`:
```
-r requirements.txt
pytest>=8.0
pytest-asyncio>=0.24
httpx>=0.27
aiosqlite>=0.20
```

`backend/pytest.ini`:
```
[pytest]
asyncio_mode = auto
testpaths = tests
```

`backend/.gitignore`:
```
.venv/
__pycache__/
*.pyc
.pytest_cache/
.env
.env.*
```

`backend/.env.example`:
```
DATABASE_URL=postgresql://user:password@ep-xxxx-pooler.aws.neon.tech/neondb?sslmode=require
JWT_SECRET=replace-me
```

`backend/README.md`: one short page — what this is, local setup (`python -m venv .venv`, `.venv\Scripts\python -m pip install -r requirements-dev.txt`, `.venv\Scripts\python -m pytest`), local run (`uvicorn app.main:app --reload`), deploy notes (Render service `blindspot-api`, env var NAMES only, never values).

- [ ] **Step 3: Write the tests**

`backend/tests/test_health.py`:

```python
from fastapi.testclient import TestClient

from app.main import app

client = TestClient(app)


def test_health_ok():
    r = client.get("/api/v1/health")
    assert r.status_code == 200
    assert r.json()["ok"] is True


def test_health_db_returns_503_when_database_unavailable(monkeypatch):
    async def boom():
        raise RuntimeError("db down")

    monkeypatch.setattr("app.routers.health.ping_db", boom)
    r = client.get("/api/v1/health/db")
    assert r.status_code == 503
```

`backend/tests/test_database.py`:

```python
from app.database import sanitize_database_url

POOLED = (
    "postgresql://neondb_owner:pw@ep-xxx-pooler.aws.neon.tech/neondb"
    "?sslmode=require&channel_binding=require"
)


def test_sanitize_strips_channel_binding():
    assert "channel_binding" not in sanitize_database_url(POOLED)


def test_sanitize_converts_scheme_to_asyncpg():
    assert sanitize_database_url(POOLED).startswith("postgresql+asyncpg://")


def test_sanitize_preserves_sslmode():
    assert "sslmode=require" in sanitize_database_url(POOLED)


def test_sanitize_forces_prepared_statement_cache_off():
    assert "prepared_statement_cache_size=0" in sanitize_database_url(POOLED)


def test_sanitize_idempotent():
    once = sanitize_database_url(POOLED)
    assert sanitize_database_url(once) == once
```

- [ ] **Step 4: Create venv, install, run tests**

Run (in `backend/`):
```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install --disable-pip-version-check -r requirements-dev.txt
.\.venv\Scripts\python.exe -m pytest -q
```
Expected: `7 passed` (2 + 5). **If a pin fails to install on Python 3.10.11**, adjust only that package to the nearest 3.10-compatible version, record it in the commit message, keep the Render-side floor (3.13.5) in mind.

- [ ] **Step 5: Live Neon ping (the Review Focus #4 test)**

Run (in `backend/`) — this reads the secret from the temp file into an env var of the shell process only; it must never be echoed:
```powershell
$env:DATABASE_URL = ((Get-Content "C:\Users\Piolo Mangilog\AppData\Local\Temp\opencode\blindspot_secrets\neon_blindspot.env") | Where-Object { $_ -match '^\s*DATABASE_URL\s*=' } | Select-Object -First 1) -replace '^\s*DATABASE_URL\s*=\s*',''
$env:DATABASE_URL = $env:DATABASE_URL.Trim().Trim('"')
.\.venv\Scripts\python.exe -c "import asyncio; from app.database import ping_db; asyncio.run(ping_db()); print('neon ping ok')"
```
Expected: exactly `neon ping ok` — **no URL in the output**. If it fails with a prepared-statement/channel_binding error, fix `sanitize_database_url` (it should already force both cache settings), re-run, and document.

- [ ] **Step 6: Init repo and commit**

Run (in `backend/`):
```powershell
git init -b main
git add -A
git commit -m "feat: FastAPI skeleton with health endpoints and PgBouncer-safe Neon database layer"
```

---

### Task 7: Create GitHub repos and push all three

**Files:** none (repo operations).
**Interfaces:**
- Consumes: commits from Tasks 1–6 (`app/`, `backend/`) and the workspace repo's history.
- Produces: public GitHub repos `Piyolo/blindspot-app`, `Piyolo/blindspot-api`, `Piyolo/blindspot-workspace` — `blindspot-api` is what Task 8 deploys; all later phases push here.

- [ ] **Step 1: Preflight — confirm names are still free**

Run (anywhere):
```powershell
gh repo list Piyolo --limit 100 | Select-String -Pattern "blindspot-app|blindspot-api|blindspot-workspace"
```
Expected: no matches (only the legacy `blindspot-backend` / `blindspot-site` / `blindspot-roboflow-proxy` exist). If any name now exists, STOP and report to the lead.

- [ ] **Step 2: Create and push the three repos**

Run (anywhere):
```powershell
gh repo create Piyolo/blindspot-app --public --source "C:\Users\Piolo Mangilog\Dev\BlindSpot\app" --push --description "BlindSpot — Flutter Android app: on-device YOLOv8n object detection, bilingual voice commands, emergency SOS"
gh repo create Piyolo/blindspot-api --public --source "C:\Users\Piolo Mangilog\Dev\BlindSpot\backend" --push --description "BlindSpot — FastAPI backend (auth, profile) for the BlindSpot Android app"
gh repo create Piyolo/blindspot-workspace --public --source "C:\Users\Piolo Mangilog\Dev\BlindSpot" --push --description "BlindSpot workspace — design spec and implementation plans"
```
Expected: each prints the new repo URL.

- [ ] **Step 3: Verify**

Run:
```powershell
gh repo view Piyolo/blindspot-app --json "name,visibility,defaultBranchRef"
gh repo view Piyolo/blindspot-api --json "name,visibility,defaultBranchRef"
gh repo view Piyolo/blindspot-workspace --json "name,visibility,defaultBranchRef"
git -C "C:\Users\Piolo Mangilog\Dev\BlindSpot\app" status --short
git -C "C:\Users\Piolo Mangilog\Dev\BlindSpot\backend" status --short
```
Expected: three repos, `"visibility":"public"`, default branch `main`; both `status` outputs empty (clean trees).

---

### Task 8: Deploy `blindspot-api` to Render + verify live + wire the URL

**Files:**
- Create: `.secrets/neon_blindspot.env`, `.secrets/jwt_secret.txt` (both gitignored, never committed)
- Modify: workspace `.gitignore` (+`.secrets/`), `app/lib/services/api_client.dart` (only if the live URL differs from `defaultBaseUrl`), workspace `README.md`, spec §17
- Test: live health endpoints; re-run app tests after URL wiring

**Interfaces:**
- Consumes: Task 6 backend, Task 7 repo `Piyolo/blindspot-api`, staged secrets, Render MCP (`create_web_service`, `get_service`), filesystem MCP (workspace-scoped read).
- Produces: live Render web service `blindspot-api` (URL captured, e.g. `https://blindspot-api.onrender.com/`), `DATABASE_URL`/`JWT_SECRET`/`PYTHON_VERSION` env vars set, `GET /api/v1/health` → 200 and `GET /api/v1/health/db` → `{"db":"ok"}`. Phase 1's app work points at this URL.

- [ ] **Step 1: Gitignore the secrets staging dir FIRST**

Add a `.secrets/` line to the workspace `.gitignore`, then commit (workspace root):
```powershell
git add .gitignore
git commit -m "chore: ignore staged secrets directory"
```

- [ ] **Step 2: Stage secrets and prove they are ignored**

Run (workspace root) — the JWT secret is generated and written without echoing:
```powershell
New-Item -ItemType Directory -Force -Path ".secrets" | Out-Null
Copy-Item "C:\Users\Piolo Mangilog\AppData\Local\Temp\opencode\blindspot_secrets\neon_blindspot.env" ".secrets\neon_blindspot.env" -Force
python -c "import secrets,pathlib; pathlib.Path(r'C:\Users\Piolo Mangilog\Dev\BlindSpot\.secrets\jwt_secret.txt').write_text(secrets.token_hex(32), encoding='ascii')"
git check-ignore .secrets
git status --short
```
Expected: `.secrets` printed and `git status` **empty** — both staged files are ignored; this is the proof they can never be committed.

- [ ] **Step 3: Create the Render service (MCP)**

Run via the `execute` tool. **Never return or print env values** — adapt the response-field access to the actual shape, but return only ids/urls/name:

```js
const envFile = await tools.filesystem.read_text_file({ path: "C:\\Users\\Piolo Mangilog\\Dev\\BlindSpot\\.secrets\\neon_blindspot.env" });
const jwtFile = await tools.filesystem.read_text_file({ path: "C:\\Users\\Piolo Mangilog\\Dev\\BlindSpot\\.secrets\\jwt_secret.txt" });

const textOf = (res) => {
  const c = res && res.content;
  if (typeof c === "string") return c;
  if (Array.isArray(c)) return c.map((x) => (x && x.text) || "").join("\n");
  return String(c ?? "");
};

const dbLine = textOf(envFile).split(/\r?\n/).find((l) => /^\s*DATABASE_URL\s*=/.test(l));
if (!dbLine) throw new Error("DATABASE_URL line not found; aborting (no values echoed)");
const dbUrl = dbLine.replace(/^\s*DATABASE_URL\s*=\s*/, "").trim().replace(/^["']|["']$/g, "");
const jwtSecret = textOf(jwtFile).trim();
if (!dbUrl || !jwtSecret) throw new Error("empty secret parsed; aborting (no values echoed)");

const svc = await tools.render.create_web_service({
  name: "blindspot-api",
  runtime: "python",
  repo: "https://github.com/Piyolo/blindspot-api",
  branch: "main",
  buildCommand: "pip install -r requirements.txt",
  startCommand: "uvicorn app.main:app --host 0.0.0.0 --port $PORT",
  plan: "free",
  region: "singapore",
  autoDeploy: "yes",
  envVars: [
    { key: "DATABASE_URL", value: dbUrl },
    { key: "JWT_SECRET", value: jwtSecret },
    { key: "PYTHON_VERSION", value: "3.13.5" },
  ],
  workspaceId: "tea-d35dt5ogjchc73dvvo50",
});

const id = svc && (svc.id ?? (svc.service && svc.service.id));
const url = (svc && (svc.serviceDetails?.url ?? svc.url ?? (svc.service && (svc.service.serviceDetails?.url ?? svc.service.url)))) ?? null;
return { serviceId: id, url, name: svc?.name ?? "blindspot-api" };
```

Contingencies: if creation fails with a repo-access error, ask the user to grant Render access to `Piyolo/blindspot-api` (dashboard → connect repo), then re-run. If MCP creation is unavailable, fall back to documenting exact dashboard values (name/region/plan/build/start/env var names) and ask the user to create it, then continue from Step 4. **Never touch the legacy `blindspot` service.**

Known deviation: the MCP tool does not expose Render's `healthCheckPath` setting (spec §7 targets `/api/v1/health`). Leave it unset — the service is still reachable and auto-deploy works; record this as a Phase 0 deviation in the final report (it can be set later in the dashboard if ever needed).

- [ ] **Step 4: Wait for the deploy, then verify live**

Using the `url` from Step 3 (example `https://blindspot-api.onrender.com`):

```powershell
$base = "<url from step 3>"
for ($i = 1; $i -le 40; $i++) {
  try {
    $h = Invoke-RestMethod -Uri "$base/api/v1/health" -TimeoutSec 100
    if ($h.ok) { "health OK (attempt $i): v$($h.version)"; break }
  } catch { Start-Sleep -Seconds 15 }
}
Invoke-RestMethod -Uri "$base/api/v1/health/db" -TimeoutSec 100 | ConvertTo-Json -Compress
```
Expected: `health OK (attempt N): v0.1.0` (first attempt may take ~60 s — Render cold start; that is normal, keep retrying), then `{"db":"ok"}`. If `/health/db` returns 503, check the Render deploy logs for asyncpg errors and fix the database layer (the sanitizer should already force `prepared_statement_cache_size=0`), then `tools.render.trigger_deploy` and re-verify.

- [ ] **Step 5: Wire the real URL into app + docs**

If the live URL differs from `https://blindspot-api.onrender.com`: update `app/lib/services/api_client.dart` `defaultBaseUrl`. Then re-run `flutter analyze` + `flutter test` (in `app/`), commit and push:

```powershell
git -C "C:\Users\Piolo Mangilog\Dev\BlindSpot\app" add -A
git -C "C:\Users\Piolo Mangilog\Dev\BlindSpot\app" commit -m "chore: point API base URL at deployed blindspot-api service"
git -C "C:\Users\Piolo Mangilog\Dev\BlindSpot\app" push -u origin main
```

Update workspace `README.md` (add `Live API: <url>` line) and spec §17 (add "Render service `blindspot-api` live at `<url>` — free plan, Singapore"), then:

```powershell
git -C "C:\Users\Piolo Mangilog\Dev\BlindSpot" add -A
git -C "C:\Users\Piolo Mangilog\Dev\BlindSpot" commit -m "docs: record live API URL"
git -C "C:\Users\Piolo Mangilog\Dev\BlindSpot" push -u origin main
```

- [ ] **Step 6: Final verification pass**

```powershell
Invoke-RestMethod -Uri "$base/api/v1/health" | ConvertTo-Json -Compress
Invoke-RestMethod -Uri "$base/api/v1/health/db" | ConvertTo-Json -Compress
gh repo view Piyolo/blindspot-api --json "name,visibility,defaultBranchRef"
```
Expected: `{"ok":true,"version":"0.1.0"}`; `{"db":"ok"}`; repo public on `main`.

---

## Phase 0 Exit Criteria

- [x] `flutter analyze` clean and `flutter test` green in `app/` (assets + smoke + ApiClient tests).
- [x] `flutter build apk --debug` succeeds; APK at `app/build/app/outputs/flutter-apk/app-debug.apk` (on-device install is user-verified when sideloading later — no device on adb in this environment).
- [x] `pytest` green in `backend/` (7 tests) and the live Neon ping printed `neon ping ok`.
- [x] `https://<service>.onrender.com/api/v1/health` → `{"ok":true,"version":"0.1.0"}` and `/api/v1/health/db` → `{"db":"ok"}` (verified after cold start).
- [x] Three public GitHub repos pushed: `blindspot-app`, `blindspot-api`, `blindspot-workspace`; Render `autoDeploy` on `main`.
- [x] No secret value anywhere in any repo, commit, or command output; `.gitignore` files committed in all repos.
- [x] mas-reviewer audit of the full Phase 0 diff across all three repos completed with no open critical findings.

**Phase 0 completed 2026-10-18.** Evidence: `.superpowers/sdd/progress.md`, per-task reports/verifications (`task-1..6-*`), review `.superpowers/sdd/phase0-review.md` (APPROVE_WITH_NOTES — 0 critical/high/medium). Live API: https://blindspot-api-dkch.onrender.com.
