# BlindSpot Phase 1 — Auth + Offline Session Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Real accounts on the live backend — signup, login, profile edit, password change — with a cached session so the app stays usable offline after the first login.

**Architecture:** FastAPI gains `models/schemas/security/deps/routers(auth, me)` + in-memory rate limiting, tables created on startup via `create_all`. The Flutter app gains models, validation, copy constants, a secure store for token+user, three repositories (auth/profile/session), two providers (Auth/Profile), and the Welcome/Login/Signup/Permissions/Profile screens plus a minimal honest Home and session-aware Splash routing. Speech stays out of this phase (TalkBack-only); TTS arrives in Phase 2.

**Tech Stack:** FastAPI 0.143.3 · SQLAlchemy 2.0.55 async + asyncpg · argon2-cffi 25.1.0 · PyJWT 2.15.1 · pydantic 2.14.1 · pytest/httpx/aiosqlite (dev) — already installed. Flutter: `provider ^6.1.5+1`, `http`, `flutter_secure_storage ^11.2.0`, `permission_handler ^12.0.3` — all already pinned in `app/pubspec.yaml`.

**Spec:** `docs/superpowers/specs/2026-10-17-blindspot-rebuild-design.md` (§4 screens, §7 backend, §8 integration contract, §10 constants, §11 copy deck, §12 pins, §14 verification, §15 Phase 1). Executors read both this plan and the spec.

## Phase-1 decisions made with the user (2026-10-18)

1. **No app speech in Phase 1.** Screens are TalkBack-usable: correct semantics labels + copy-deck strings as constants. `TtsService` lands in Phase 2 per the locked phase order.
2. **Minimal honest Home.** Home shows the signed-in name + a working Profile tile; the other designed tiles appear as their phases land (no dead ends for blind users).
3. Status-bar theming fix folded in (from the on-device Phase 0 screenshot): the system status bar rendered light against the dark app.

## Global Constraints

- Version floors / pins: `permission_handler ^12.0.3` (13.x fails under Flutter 3.41.5) · `flutter_secure_storage ^11.2.0` · `shared_preferences ^2.5.6` (prefer `SharedPreferencesAsync`) · `provider ^6.1.5+1` (only state-management lib allowed) · `http` · backend pins exactly as `backend/requirements.txt` (do not change).
- Backend code must run on local Python **3.10.11** and Render **Python 3.13.5**; no 3.11+ syntax (`X | None` in *annotations* is fine via `from __future__ import annotations` if needed; avoid `match`, `Self`, `ExceptionGroup`).
- JWT: HS256, `ACCESS_TTL_SECONDS = 2592000` (30 days), secret from `get_jwt_secret()` (env `JWT_SECRET`).
- `ApiClient`: base `/api/v1`, 25 s timeout, `ApiException(message, statusCode?)`, `Authorization: Bearer` when `authToken` set — unchanged from Phase 0. One retry on auth calls only, and only for network errors (`statusCode == null`).
- Error body shape is always `{"detail": "<string>"}` — including validation errors (see Task 1's handler). Never return pydantic's list-shaped 422 detail.
- Copy: every user-facing string comes from `lib/constants/copy.dart`, verbatim from spec §11 (EN + FIL). Screen code must not contain bare string literals for copy.
- Accessibility: every interactive element ≥48dp; every state change has a semantics announcement source (label + `SemanticsService.announce` where a screen-level message appears); no color-only signals.
- Theme tokens from `app/lib/theme/tokens.dart` only; no new colors.
- Touch/text: default body 18, tiles 56dp high, screen padding 20 (spec §5.3).
- Repos: `Piyolo/blindspot-app` (Flutter), `Piyolo/blindspot-api` (FastAPI), `Piyolo/blindspot-workspace` (docs). Legacy repos/services untouched.
- Live API base URL: `https://blindspot-api-dkch.onrender.com` (already wired into `ApiClient.defaultBaseUrl`).
- Single-writer rule: mas-developer implements tasks serially; only mas-tester (verify) and mas-reviewer (audit) read. No two agents edit one file.
- Secrets: never echo `DATABASE_URL`/`JWT_SECRET`; smoke tests use throwaway accounts; never print a password value.
- Per task: TDD order (failing test → implement → pass), commit at the end with the given message, then mas-tester verification before the next task starts.

## Review Focus

These are the input classes the spec implies but no single task's tests cover; each line has a test pinned in the owning task.

1. **Cold backend after idle** (Render free tier sleeps). The first auth call can take ~30-60 s; the user must see/hear "The server is waking up. One moment." and the call must be retried once rather than failing. Owner: Task 8 (`AuthProvider.retryingNetwork`) + Task 9 (login screen banner).
2. **Airplane relaunch with a cached session.** Launch with no network must land on Home (no network call, no spinner forever); logout must clear the cached token so a relaunch lands on Welcome. Owners: Task 6 (session store), Task 8 (restore/signOut), Task 10 (routing + smoke test).
3. **Server-side 401 on a cached session** (expired/rotated 30-day token). The app must not crash or loop: it signs out, clears the cache, shows 'Invalid email or password.'-class messaging and returns to Welcome. Owner: Task 8 (`AuthProvider.handleUnauthorized`) + Task 12 (Profile screen offline/401 notice).
4. **Contact-number formats.** `09xx xxx xxxx`, `+63 9xx xxx xxxx`, `0917-123-4567`, with spaces/dashes, and garbage. Both client and server must normalize to one canonical form (`+` + digits, or digits) and reject <7 or >15 digits with the copy-deck message. Owners: Task 1 (server validators), Task 6 (`normalizeContactNumber`), Task 9 (signup field error).
5. **Double-submit / duplicate signup.** Two rapid taps on Submit, or a signup with an existing email, must not create two accounts or two in-flight requests; 409 shows 'This email is already registered. Try logging in.'. Owners: Task 8 (`busy` guard), Task 9 (submit disabled while busy), Task 5 (live 409 evidence).

---

### Task 1: Backend data layer — models, schemas, test harness, string-shaped validation errors

**Files:**
- Create: `backend/app/models.py`, `backend/app/schemas.py`, `backend/tests/conftest.py`, `backend/tests/test_models.py`, `backend/tests/test_schemas.py`
- Modify: `backend/app/main.py` (create tables on startup; `RequestValidationError` → 400)

**Interfaces:**
- Consumes: `app.database.get_engine()`, `app.config.ACCESS_TTL_SECONDS` (Phase 0).
- Produces: `Base`, `User` (table `users`: `id:int`, `name:str(80)`, `email:str(254) unique`, `password_hash:str(255)`, `contact_number:str(20)`, `created_at:datetime(tz)`); pydantic `SignupRequest`, `LoginRequest`, `UserOut` (`from_attributes=True`), `TokenResponse`, `UpdateMeRequest`, `ChangePasswordRequest`, `OkResponse`; helper `normalize_contact_number(raw: str) -> str` and `is_valid_password(raw: str) -> bool`; test fixtures `db_engine`, `db_session`, `client`.

- [ ] **Step 1: Write the failing tests**

```python
# backend/tests/test_models.py
async def test_user_row_round_trips_with_timezone_created_at(db_session):
    user = User(name="Ana", email="ana@example.com", password_hash="x", contact_number="+639171234567")
    db_session.add(user); await db_session.commit(); await db_session.refresh(user)
    assert user.id is not None and user.created_at.tzinfo is not None

async def test_email_is_unique(db_session):  # IntegrityError on duplicate email
```
```python
# backend/tests/test_schemas.py
@pytest.mark.parametrize("raw,expected", [("0917 123 4567", "09171234567"), ("+63 917-123-4567", "+639171234567"), (" 0917-123-4567 ", "09171234567")])
def test_contact_normalization(raw, expected): assert normalize_contact_number(raw) == expected
@pytest.mark.parametrize("raw", ["123", "12345678901234567", "abcd", "+"])
def test_contact_rejects_bad(raw): ...  # ValueError from validator -> 400 detail string
@pytest.mark.parametrize("bad", ["short1", "allletters", "12345678"])
def test_signup_rejects_bad_password(bad): ...   # detail == "Password must be at least 8 characters with a letter and a number."
async def test_signup_missing_field_is_400_with_string_detail(client):
    r = await client.post("/api/v1/auth/signup", json={"email": "x@example.com"})
    assert r.status_code == 400 and isinstance(r.json()["detail"], str)
```

- [ ] **Step 2: Run tests, verify failure** — `cd backend; .\.venv\Scripts\python.exe -m pytest tests/test_models.py tests/test_schemas.py -q` → collection errors (missing modules/fixtures).

- [ ] **Step 3: Implement** `models.py` (DeclarativeBase, `datetime.now(timezone.utc)` default) · `schemas.py` (Field constraints + `field_validator`s raising `ValueError` with the copy-deck messages) · `conftest.py` (aiosqlite in-memory engine + `async_sessionmaker`, `dependency_overrides[get_db]`, `httpx.AsyncClient(transport=ASGITransport(app=app), base_url="http://test")`, fixture clearing rate limiters) · `main.py` lifespan running `Base.metadata.create_all` and an `@app.exception_handler(RequestValidationError)` returning `400 {"detail": first_message}`.

- [ ] **Step 4: Run tests, verify pass** — same command → `N passed`.

- [ ] **Step 5: Commit** — `feat(api): user model, pydantic schemas, and string-shaped 400 validation errors`

---

### Task 2: Backend security — argon2 hashing + JWT

**Files:** Create `backend/app/security.py`, `backend/tests/test_security.py`

**Interfaces:**
- Produces: `hash_password(password: str) -> str` · `verify_password(password: str, password_hash: str) -> bool` · `create_access_token(user_id: int, ttl_seconds: int = ACCESS_TTL_SECONDS) -> str` (claims `sub`=str(id), `iat`, `exp`) · `decode_access_token(token: str) -> int` (returns user id) · `class InvalidTokenError(Exception)`.

- [ ] **Step 1: Write failing tests** — hash differs from plaintext and verifies; wrong password returns False; token round-trips to the same id; token with `ttl_seconds=-1` raises `InvalidTokenError`; a tampered token raises `InvalidTokenError`; `decode_access_token("garbage")` raises `InvalidTokenError`.
- [ ] **Step 2: Run** `python -m pytest tests/test_security.py -q` → fails (module missing).
- [ ] **Step 3: Implement** — `PasswordHasher()` (argon2id defaults), `jwt.encode/decode(..., algorithms=["HS256"])`, wrap every `jwt` exception into `InvalidTokenError`.
- [ ] **Step 4: Run** → all pass.
- [ ] **Step 5: Commit** — `feat(api): argon2 password hashing and 30-day JWT helpers`

---

### Task 3: Backend auth router — signup, login, rate limiting

**Files:** Create `backend/app/deps.py`, `backend/app/rate_limit.py`, `backend/app/routers/auth.py`, `backend/tests/test_auth.py`; Modify `backend/app/main.py` (include router)

**Interfaces:**
- Consumes: Task 1 schemas/models, Task 2 security helpers.
- Produces: `get_db() -> AsyncIterator[AsyncSession]` · `get_current_user()` (HTTPBearer → `User`, 401 detail `"Invalid or expired token"`) · `class FailureLimiter: allowed(key, *, now=None) -> bool`, `record(key, *, now=None) -> None`, `reset(key) -> None` · module instances `login_limiter = FailureLimiter(10, 600)`, `signup_limiter = FailureLimiter(5, 3600)` · `client_ip(request) -> str` (first `X-Forwarded-For` entry else `request.client.host`) · `POST /api/v1/auth/signup` → 201 `TokenResponse`, `POST /api/v1/auth/login` → 200 `TokenResponse`.

- [ ] **Step 1: Write failing tests** (all via the Task 1 `client` fixture)

```python
async def test_signup_returns_token_and_user(client)            # 201; token decodes to user id; user.contact_number normalized
async def test_signup_duplicate_email_is_409(client)            # detail == "This email is already registered. Try logging in."
async def test_login_ok_and_wrong_password(client)             # 200; then 401 detail == "Invalid email or password."
async def test_login_unknown_email_is_401_generic(client)      # same detail as wrong password (no user enumeration)
async def test_login_rate_limited_after_10_failures(client)    # 11th attempt -> 429, detail "Too many attempts. Try again later."
async def test_login_success_resets_limiter(client)            # 9 failures + 1 success -> next failure still 401, not 429
async def test_signup_rate_limited_after_5_per_hour(client)    # 6th signup from same IP -> 429
async def test_me_requires_bearer(client)                      # GET /api/v1/me without token -> 401
```

- [ ] **Step 2: Run** `python -m pytest tests/test_auth.py -q` → fails.
- [ ] **Step 3: Implement** — signup: `signup_limiter` check → normalize/validate → 409 on existing email (case-insensitive) → insert → token. Login: `login_limiter.allowed` else 429 → look up by lowercased email → `verify_password` → `record()` failure + 401 generic, or `reset()` + 200. All DB access through `Depends(get_db)`.
- [ ] **Step 4: Run** → all pass; then full suite `python -m pytest -q` → previous 7 + new tests green.
- [ ] **Step 5: Commit** — `feat(api): signup and login with in-memory rate limiting`

---

### Task 4: Backend me router — read, update, change password

**Files:** Create `backend/app/routers/me.py`, `backend/tests/test_me.py`; Modify `backend/app/main.py`

**Interfaces:**
- Consumes: Task 1 schemas, Task 3 `get_db`/`get_current_user`.
- Produces: `GET /api/v1/me` → `{"user": UserOut}` · `PUT /api/v1/me` → `{"user": UserOut}` (updates `name` and/or `contact_number`, partial, ignores nulls) · `PUT /api/v1/me/password` → `{"ok": true}`.

- [ ] **Step 1: Write failing tests**

```python
async def test_me_returns_current_user(client, auth_headers)      # 200, email matches signup
async def test_put_me_updates_contact_and_persists(client, auth_headers)  # re-GET shows the new canonical value
async def test_put_me_rejects_bad_contact(client, auth_headers)   # 400 with string detail
async def test_put_me_without_token_is_401(client)                # 401
async def test_change_password_happy_path(client, auth_headers)   # 200 {"ok":true}; old password now 401, new password 200
async def test_change_password_wrong_current_is_401(client, auth_headers)  # detail "Current password is incorrect."
async def test_change_password_enforces_rules(client, auth_headers)        # 400 password rule detail
```

- [ ] **Step 2: Run** `python -m pytest tests/test_me.py -q` → fails.
- [ ] **Step 3: Implement** the three handlers; `auth_headers` fixture (in `conftest.py`) does a signup and returns `{"Authorization": f"Bearer {token}"}`.
- [ ] **Step 4: Run** `python -m pytest -q` → full suite green (expect ≥ 30 tests).
- [ ] **Step 5: Commit** — `feat(api): /me read, profile update, and password change`

---

### Task 5: Deploy backend and smoke-test the live API

**Files:** none (operations + evidence); Append evidence to `.superpowers/sdd/p1-task-5-report.md`

**Interfaces:**
- Consumes: Tasks 3-4 routers, Render autoDeploy on `main`.
- Produces: live evidence that every path works against `https://blindspot-api-dkch.onrender.com`.

- [ ] **Step 1: Push** — `git -C backend push origin main` (autoDeploy triggers; confirm with `tools.render.list_deploys` until `live`).
- [ ] **Step 2: Wait for deploy** — poll deploys until `status: live` for the pushed commit.
- [ ] **Step 3: Live smoke (throwaway account; never print the password)** — generate a random password into a PowerShell variable; then:

```powershell
$b = "https://blindspot-api-dkch.onrender.com/api/v1"
# signup 201 -> login 200 -> me 200 -> put me 200 -> password change 200 -> old login 401 -> new login 200
# duplicate signup -> 409 ; wrong password -> 401 ; 11 bad logins -> 429 ; no-token /me -> 401
```
Print only status codes and `detail` strings. Expected: `201 200 200 200 200 401 200 409 401 429 401`.
- [ ] **Step 4: Write evidence** — statuses + timings (cold-start seconds for the first call) into `.superpowers/sdd/p1-task-5-report.md`.
- [ ] **Step 5: Commit (workspace)** — not needed; report lives in gitignored `.superpowers/`. If any endpoint needed a fix, that fix is a new backend commit + re-run of Step 3.

---

### Task 6: App foundation — models, validation, copy constants, secure store, session repository

**Files:**
- Create: `app/lib/models/user.dart`, `app/lib/models/auth_result.dart`, `app/lib/services/validation.dart`, `app/lib/services/secure_store.dart`, `app/lib/services/token_store.dart`, `app/lib/repositories/session_repository.dart`, `app/lib/constants/copy.dart`, `app/lib/constants/api_paths.dart`
- Test: `app/test/models_test.dart`, `app/test/validation_test.dart`, `app/test/session_repository_test.dart`

**Interfaces:**
- Produces:
  - `class User { final int id; final String name; final String email; final String contactNumber; final DateTime createdAt; const User({...}); factory User.fromJson(Map<String, dynamic>); Map<String, dynamic> toJson(); }`
  - `class AuthResult { final String accessToken; final User user; factory AuthResult.fromJson(Map<String, dynamic>); }`
  - `String normalizeContactNumber(String raw)` (keep leading `+` if present, strip everything else) · `bool isValidContactNumber(String raw)` (7-15 digits) · `bool isValidPassword(String raw)` (≥8, ≥1 letter, ≥1 digit) · `bool isValidEmail(String raw)`
  - `abstract class SecureStore { Future<String?> read(String key); Future<void> write(String key, String value); Future<void> delete(String key); }` + `FlutterSecureStore` (uses `FlutterSecureStorage`) + `InMemorySecureStore` (test double)
  - `class TokenStore { TokenStore(SecureStore store); Future<String?> read(); Future<void> write(String token); Future<void> clear(); }` (key `blindspot_token`)
  - `class CachedSession { final String token; final User user; }` · `class SessionRepository { SessionRepository(SecureStore store); Future<CachedSession?> load(); Future<void> save({required String token, required User user}); Future<void> clear(); }` (user JSON key `blindspot_user`)
  - `class ApiPaths { static const signup = '/auth/signup'; static const login = '/auth/login'; static const me = '/me'; static const mePassword = '/me/password'; }`
  - `class Copy` — **the complete Phase-1 string set**; no screen may contain a hard-coded user-facing string, and no string outside this list may be rendered (additions require a plan amendment).
    - Verbatim from spec §11 (EN/FIL pairs): `boot`, `welcome`, `loginField`, `passwordField`, `passwordRule`, `emergencyNumberField`, `micPermission`, `cameraPermission`, `phonePermission`, `home`, `loginError`, `emailExists`, `offline`, `serverWaking`, `logoutConfirm`, `passwordChanged`, `wrongCurrentPassword`.
    - Phase-1 additions (pin these exact EN/FIL; they get folded into spec §11 during the Phase 5 copy-deck review):
      | Key | EN | FIL |
      |---|---|---|
      | `logIn` | Log in | Mag-log in |
      | `signUp` | Sign up | Mag-sign up |
      | `profileTitle` | Profile | Profile |
      | `permissionsTitle` | Permissions | Mga pahintulot |
      | `save` | Save | I-save |
      | `logOut` | Log out | Mag-log out |
      | `yes` / `no` | Yes / No | Oo / Hindi |
      | `continueLabel` | Continue | Magpatuloy |
      | `allSet` | All set. Continue. | Ayos na lahat. Magpatuloy. |
      | `limitedMode` | Continuing without some permissions. Some features will not work. | Magpapatuloy nang walang ilang pahintulot. Hindi gagana ang ilang feature. |
      | `needInternet` | You need internet to change this. | Kailangan mo ng internet para baguhin ito. |
      | `rateLimited` | Too many attempts. Try again later. | Masyadong maraming pagsubok. Subukan ulit mamaya. |
      | `nameLabel` | Your name | Pangalan mo |
      | `emailLabel` | Email address | Email address |
      | `contactNumberLabel` | Emergency contact number | Emergency contact number |
      | `currentPasswordLabel` | Current password | Kasalukuyang password |
      | `newPasswordLabel` | New password | Bagong password |
      | `changePasswordLabel` | Change password | Palitan ang password |
      | `savedContact` | Contact number saved. | Nai-save ang contact number. |
      | `signedInAs(name)` | Signed in as {name}. | Naka-log in bilang {name}. |
      | `allow` | Allow | Payagan |

- [ ] **Step 1: Write failing tests** — `User.fromJson/toJson` round-trip; `AuthResult.fromJson` parses `access_token`+`user`; `normalizeContactNumber` table (`"0917 123 4567"→"09171234567"`, `"+63 917-123-4567"→"+639171234567"`, `"0917-123-4567"→"09171234567"`); `isValidContactNumber` false for `"123"`/`"12345678901234567"`; `isValidPassword` false for `"short1"`, `"allletters"`, `"12345678"`; `SessionRepository` save→load returns token+user, `clear()` then `load()` returns null, corrupt stored JSON → `load()` returns null and deletes the bad entry.
- [ ] **Step 2: Run** `cd app; flutter test test/models_test.dart test/validation_test.dart test/session_repository_test.dart` → fails.
- [ ] **Step 3: Implement** the files above (copy constants typed verbatim from spec §11; no logic in `copy.dart`).
- [ ] **Step 4: Run** the three test files → pass; `flutter analyze` → clean.
- [ ] **Step 5: Commit** — `feat(app): user models, validation, copy constants, and secure session storage`

---

### Task 7: App repositories — auth and profile

**Files:** Create `app/lib/repositories/auth_repository.dart`, `app/lib/repositories/profile_repository.dart`; Test `app/test/auth_repository_test.dart`, `app/test/profile_repository_test.dart`

**Interfaces:**
- Consumes: `ApiClient` (Phase 0), `ApiPaths`, `AuthResult`, `User`, `ApiException`.
- Produces:
  - `abstract class AuthRepository { Future<AuthResult> signup({required String name, required String email, required String password, required String contactNumber}); Future<AuthResult> login({required String email, required String password}); }` · `class ApiAuthRepository implements AuthRepository` (takes `ApiClient`) · `class MockAuthRepository implements AuthRepository` (configurable success/failure for widget tests)
  - `abstract class ProfileRepository { Future<User> updateProfile({String? name, String? contactNumber}); Future<void> changePassword({required String currentPassword, required String newPassword}); }` · `ApiProfileRepository` · `MockProfileRepository`
  - Retry rule: both API implementations retry **once** when `ApiException.statusCode == null` (network/cold start); 4xx/5xx never retried.

- [ ] **Step 1: Write failing tests** (`MockClient` from `http/testing.dart`)
```dart
test('signup posts /api/v1/auth/signup with normalized contact and parses AuthResult')
test('signup maps 409 to ApiException with the detail message')
test('login maps 401 to ApiException(loginError)')
test('login retries once on network error then succeeds')   // first call throws ClientException, second returns 200
test('login does not retry on 401')                          // exactly one request seen
test('updateProfile PUTs /api/v1/me and parses the returned user')
test('changePassword PUTs /api/v1/me/password and resolves on {"ok":true}')
```
- [ ] **Step 2: Run** `flutter test test/auth_repository_test.dart test/profile_repository_test.dart` → fails.
- [ ] **Step 3: Implement** both repositories (single retry helper `Future<T> _withNetworkRetry<T>(Future<T> Function() call)`).
- [ ] **Step 4: Run** → pass; `flutter analyze` clean.
- [ ] **Step 5: Commit** — `feat(app): auth and profile repositories with single network retry`

---

### Task 8: App providers — AuthProvider and ProfileProvider

**Files:** Create `app/lib/providers/auth_provider.dart`, `app/lib/providers/profile_provider.dart`; Test `app/test/auth_provider_test.dart`, `app/test/profile_provider_test.dart`

**Interfaces:**
- Consumes: Task 6 session storage, Task 7 repositories, `ApiClient.authToken`.
- Produces:
  - `enum AuthStatus { unknown, signedOut, signedIn }`
  - `class AuthProvider extends ChangeNotifier { AuthProvider({required AuthRepository auth, required SessionRepository session, required ApiClient client}); AuthStatus get status; User? get user; String? get lastError; bool get busy; bool get retryingNetwork; Future<void> restoreSession(); Future<bool> login({required String email, required String password}); Future<bool> signup({required String name, required String email, required String password, required String contactNumber}); Future<void> signOut(); Future<void> replaceUser(User user); bool handleUnauthorized(int? statusCode); }`
  - `class ProfileProvider extends ChangeNotifier { ProfileProvider({required ProfileRepository repository, required AuthProvider auth}); bool get busy; String? get lastError; Future<bool> updateContact(String contactNumber); Future<bool> changeName(String name); Future<bool> changePassword({required String currentPassword, required String newPassword}); }`
- Ownership split (no duplicated logic): **AuthProvider owns the session and the cached user** (login/signup/logout/restore, token handling, persistence); **ProfileProvider solely owns profile edits** and on success calls `auth.replaceUser(updated)`; on a 401 from any profile call it calls `auth.handleUnauthorized(401)` and returns false.
- Behaviour pins: `restoreSession()` = load cached session → set `client.authToken` + `user`, `signedIn`; no cache → `signedOut`; never throws (offline-safe). `busy` guards re-entry (double-submit). `retryingNetwork` is true only while the repository's single retry runs. `replaceUser(user)` persists the new user into the session store and notifies listeners. `handleUnauthorized(401)` = clear store, clear client token, `signedOut`, return true. `signOut()` clears cache + token, `signedOut`.

- [ ] **Step 1: Write failing tests** (fakes: `InMemorySecureStore`, `MockAuthRepository`, `MockProfileRepository`)
```dart
test('restoreSession with cached session -> signedIn and client token set')
test('restoreSession without cache -> signedOut, no network call')
test('login success persists token+user and sets client authToken')
test('login 401 -> signedOut with loginError, nothing persisted')
test('login network error -> lastError is the offline copy, retryingNetwork true during the retry')
test('signup success -> signedIn; signup 409 -> emailExists copy')
test('second submit while busy is ignored (single request)')
test('signOut clears storage and client token -> signedOut')
test('handleUnauthorized(401) clears the cached session -> signedOut')
test('replaceUser persists the updated user into the session store')
test('profile updateContact success -> repository called once and auth user refreshed (ProfileProvider)')
test('profile changePassword wrong current -> wrongCurrentPassword copy, stays signedIn')
test('profile 401 -> handleUnauthorized cleared the cached session')
```
- [ ] **Step 2: Run** `flutter test test/auth_provider_test.dart test/profile_provider_test.dart` → fails.
- [ ] **Step 3: Implement** both providers (map `ApiException.statusCode`: 401→loginError (login) / `handleUnauthorized` (profile), 409→emailExists, 429→rateLimited copy, null→offline copy).
- [ ] **Step 4: Run** → pass; `flutter analyze` clean.
- [ ] **Step 5: Commit** — `feat(app): auth and profile providers with offline-safe session handling`

---

### Task 9: App screens — Welcome, Login, Signup (+ shared widgets)

**Files:**
- Create: `app/lib/widgets/big_action_tile.dart`, `app/lib/widgets/voice_field.dart`, `app/lib/screens/welcome_screen.dart`, `app/lib/screens/login_screen.dart`, `app/lib/screens/signup_screen.dart`
- Test: `app/test/screens_auth_test.dart`

**Interfaces:**
- Consumes: `AuthProvider`, `Copy`, theme tokens.
- Produces:
  - `class BigActionTile extends StatelessWidget { const BigActionTile({required String label, required IconData icon, required VoidCallback? onTap, String? semanticsHint}); }` (56dp+ high, 48dp min target, `Semantics(button: true, label: ...)`).
  - `class VoiceField extends StatelessWidget { const VoiceField({required String label, required TextEditingController controller, required String semanticsLabel, bool obscure = false, TextInputType keyboardType = TextInputType.text, String? errorText}); }` (uses `TextField` + `Semantics(label: semanticsLabel, textField: true)`; no speech input in Phase 1 — label says "tap to type").
  - `WelcomeScreen` (two tiles: Log in / Sign up + trusted-person note), `LoginScreen` (email+password VoiceFields, submit, signup link, waking banner when `auth.retryingNetwork`, error text from `auth.lastError`), `SignupScreen` (name, email, password, confirm, contact number; submit disabled while `auth.busy`; validation errors from `validation.dart` using copy-deck strings; 409 error text).
- Behaviour pins: submit disabled when `busy`; errors rendered as `Text` **and** announced via `SemanticsService.announce`; password fields `obscure: true`; confirm-password mismatch uses the password-rule copy; field labels come from `Copy` (`nameLabel`, `emailLabel`, `passwordField`, `contactNumberLabel`) — never literals. **On success, Login and Signup call `Navigator.pushReplacementNamed('/permissions')`** (the spec flow: Login/Signup → Permissions (if missing) → Home).

- [ ] **Step 1: Write failing widget tests** — Welcome shows both tiles with semantics labels; Login: entering email/password and tapping submit calls `MockAuthRepository.login` once, shows the waking copy while retrying, shows loginError after 401; Signup: bad password shows the password-rule copy and does not call the repository; contact field normalizes before submit; two rapid taps produce exactly one call.
- [ ] **Step 2: Run** `flutter test test/screens_auth_test.dart` → fails.
- [ ] **Step 3: Implement** the widgets and screens using `Copy` constants only.
- [ ] **Step 4: Run** → pass; `flutter analyze` clean.
- [ ] **Step 5: Commit** — `feat(app): welcome, login, and signup screens with accessible fields`

---

### Task 10: App routing — splash session check, minimal honest Home, dark status bar

**Files:** Modify `app/lib/screens/splash_screen.dart`, `app/lib/screens/home_screen.dart`, `app/lib/main.dart`, `app/lib/theme/app_theme.dart` (status-bar overlay style); Test `app/test/app_smoke_test.dart` (updated), `app/test/home_screen_test.dart`

**Interfaces:**
- Consumes: `AuthProvider.restoreSession()`, `ProfileScreen` (Task 12).
- Produces: routes `/welcome`, `/login`, `/signup`, `/permissions`, `/home`, `/profile` registered in `main.dart` with providers above the navigator; Splash awaits `restoreSession()` + minimum 800 ms, then replaces with `/home` when `status == signedIn` else `/welcome`; Home shows a status strip (`Copy.signedInAs(name)`) and a Profile `BigActionTile` (the only working tile in Phase 1) plus the existing placeholder line for future features.

- [ ] **Step 1: Write failing tests** — smoke test updated: with `MockAuthRepository` + empty store → splash → Welcome; with a pre-seeded `InMemorySecureStore` session → splash → Home showing the user's name; Home tap Profile → Profile route.
- [ ] **Step 2: Run** `flutter test test/app_smoke_test.dart test/home_screen_test.dart` → fails (existing smoke test expects splash → home placeholder).
- [ ] **Step 3: Implement** routing + Home + `SystemUiOverlayStyle` (light icons on the dark surface) on the app theme/AppBar.
- [ ] **Step 4: Run** `flutter test` (whole suite) → pass; `flutter analyze` clean.
- [ ] **Step 5: Commit** — `feat(app): session-aware splash, minimal home, matching status bar`

---

### Task 11: App Permissions screen

**Files:** Create `app/lib/services/permissions_service.dart`, `app/lib/widgets/permission_card.dart`, `app/lib/screens/permissions_screen.dart`; Test `app/test/permissions_screen_test.dart`

**Interfaces:**
- Produces: `enum AppPermission { microphone, camera, phone }` · `abstract class PermissionsService { Future<bool> isGranted(AppPermission p); Future<bool> request(AppPermission p); }` + `PermissionHandlerService` (maps to `Permission.microphone/camera/phone`) + `FakePermissionsService` (test double) · `class PermissionCard extends StatelessWidget { const PermissionCard({required String title, required String explanation, required bool granted, required Future<void> Function() onRequest}); }` · `PermissionsScreen` shows the three cards (mic/camera/phone copy from §11), a Continue button (`Copy.continueLabel`) and a limited-mode continue that announces `Copy.limitedMode`.
- Behaviour pins: entering the screen never auto-requests; each card's request button (labelled `Copy.allow`) updates that card's state without leaving the screen; when every permission is already granted the screen shows `Copy.allSet` instead of warnings; Continue (labelled `Copy.continueLabel`) always routes to `/home`, announcing `Copy.limitedMode` first when any permission is missing.

- [ ] **Step 1: Write failing tests** — three cards render with their §11 copy; tapping a card's request calls the fake service once and flips its state; Continue with all granted → `/home` with no warning; Continue with one denied → `/home` and the limited-mode announcement is emitted.
- [ ] **Step 2: Run** `flutter test test/permissions_screen_test.dart` → fails.
- [ ] **Step 3: Implement** the service, card, and screen.
- [ ] **Step 4: Run** → pass; `flutter analyze` clean.
- [ ] **Step 5: Commit** — `feat(app): permissions screen with per-permission cards and limited mode`

---

### Task 12: App Profile screen + offline/401 UX, then end-to-end verification

**Files:** Create `app/lib/screens/profile_screen.dart`; Test `app/test/profile_screen_test.dart`

**Interfaces:**
- Consumes: `AuthProvider`, `ProfileProvider`, `VoiceField`, `BigActionTile`, `Copy`.
- Produces: `ProfileScreen` — name/email readout, editable contact number (VoiceField + Save → `ProfileProvider.updateContact`), change password (current/new/confirm → `changePassword`), logout tile with a confirm dialog (`Copy.logoutConfirm`) that calls `AuthProvider.signOut()` then routes to `/welcome`.
- Behaviour pins: while offline (network error) edit attempts show `Copy.needInternet` and keep the cached value; a 401 on any profile call triggers `AuthProvider.handleUnauthorized` → routed to `/welcome` with `Copy.loginError` shown once; success shows `Copy.passwordChanged` / `Copy.savedContact`.

- [ ] **Step 1: Write failing tests** — readout shows name+email; saving a normalized contact calls the provider once and shows the saved copy; offline save shows `needInternet` and the displayed value is unchanged; logout asks for confirmation, cancels on "no", and on "yes" clears the session and routes to `/welcome`; a 401 from the profile repo routes to `/welcome`.
- [ ] **Step 2: Run** `flutter test test/profile_screen_test.dart` → fails.
- [ ] **Step 3: Implement** the screen + wire the offline/401 paths.
- [ ] **Step 4: Run** `flutter test` (full suite) + `flutter analyze` → green/clean.
- [ ] **Step 5: Device + live end-to-end checklist (evidence)** — build the **release** APK, publish it to the `blindspot-app` GitHub release (same pattern as Phase 0), then run the manual checklist on the user's phone and record results:
  - signup with a real email → lands on Home showing the name;
  - kill and relaunch in airplane mode → still signed in (Home, no spinner);
  - log out → relaunch → Welcome;
  - login with wrong password → speaking-free error text 'Invalid email or password.';
  - edit contact number (online) → re-login on another device shows the new number;
  - change password → old password fails, new password works;
  - first login attempt after ~15 min idle shows 'The server is waking up. One moment.'
  Record the checklist with pass/fail + timings in `.superpowers/sdd/p1-task-12-report.md`.
- [ ] **Step 6: Commit** — `feat(app): profile screen with contact edit, password change, and logout`

---

## Phase 1 Exit Criteria

- [ ] Backend suite green locally (`python -m pytest -q`), including rate-limit, 409, and 401 paths.
- [ ] Live smoke on Render prints `201 200 200 200 200 401 200 409 401 429 401` (Task 5 Step 3).
- [ ] App suite green (`flutter test`) and `flutter analyze` clean.
- [ ] Release APK rebuilt, published, and installed by the user.
- [ ] Task 12 checklist passes on the user's phone, including the airplane relaunch.
- [ ] No secret value in any repo, commit, log, or report; `.secrets/` still ignored.
- [ ] mas-reviewer audit of the Phase 1 diff (both repos) with no open critical/high findings.
