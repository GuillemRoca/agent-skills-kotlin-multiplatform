# Security Checklist — Kotlin Multiplatform Reference

## Data Storage

- [ ] **Secrets behind a common interface** — shared code sees only a `commonMain` abstraction; each platform implements it with its native secure store (Android Keystore-backed encryption, iOS Keychain, desktop OS credential store: Windows Credential Manager / macOS Keychain / libsecret). On web, do not persist secrets client-side — `localStorage` is readable by any injected script

```kotlin
// commonMain — the only secret API shared code sees
interface SecretStore {
    suspend fun put(key: String, value: String)
    suspend fun get(key: String): String?
    suspend fun remove(key: String)
}
// androidMain: Keystore-encrypted storage · iosMain: Keychain
// jvmMain: OS credential store · web: in-memory only, never persisted
```

- [ ] **No secrets as `commonMain` constants** — `commonMain` compiles into EVERY target, including the js/wasm bundle shipped to browsers, where any string literal is world-readable
- [ ] **No plain-text passwords** — use secure hashing or token-based auth
- [ ] **Room KMP database** in app-private storage on each platform (the default); never external or shared storage
- [ ] **Android backup rules** (`androidApp`) configured — exclude sensitive data from auto-backup:

```xml
<!-- androidApp/src/main/res/xml/backup_rules.xml -->
<data-extraction-rules>
    <cloud-backup>
        <exclude domain="sharedpref" path="secure_prefs.xml"/>
        <exclude domain="database" path="sensitive.db"/>
    </cloud-backup>
</data-extraction-rules>
```

## Network Security

- [ ] **HTTPS-only Ktor clients** — no `http://` base URLs anywhere in `commonMain`
- [ ] **Cleartext traffic disabled per platform** — `androidApp`: network security config with `cleartextTrafficPermitted="false"`; `iosApp`: App Transport Security left enabled (no `NSAllowsArbitraryLoads`)
- [ ] **Certificate pinning per Ktor engine** for production API endpoints — OkHttp `CertificatePinner` in `androidMain`/`jvmMain`, Darwin `handleChallenge` in `iosMain`; browser engines (js/wasm) cannot pin — the browser owns TLS
- [ ] **Backup pins** configured (in case primary pin rotates)
- [ ] **Pin expiration** set with rotation plan
- [ ] **No disabled certificate verification on any engine** — no trust-all `TrustManager` (Android/JVM), no blanket `handleChallenge` acceptance (Darwin)

## Authentication & Authorization

- [ ] **OAuth2 with PKCE** for third-party auth (no implicit grant)
- [ ] **Biometric/local auth behind a common interface** for sensitive operations:

```kotlin
// commonMain
interface AuthGate {
    suspend fun requireUserPresence(reason: String): Boolean
}
// androidMain: BiometricPrompt · iosMain: LocalAuthentication (LAContext)
// jvmMain: OS re-authentication where available · web: re-enter credentials
```

- [ ] **Session tokens** refreshed regularly, stored via the `SecretStore` interface (never in plain DataStore/preferences)
- [ ] **Token expiration** handled gracefully (redirect to login)

## Input Validation

- [ ] **Deep links / universal links / custom URL schemes** validated (scheme, host, path, parameters) on every platform that registers them
- [ ] **Intent extras** validated for exported components (`androidApp`)
- [ ] **Parameterized queries only** — Room KMP / SQLDelight generate them; never string-concatenate SQL
- [ ] **File paths** validated to prevent path traversal
- [ ] **Web interop** — never inject untrusted strings into DOM APIs from js/wasm interop code

## Platform Entry Points

- [ ] **Minimize exported components** (`androidApp`) — only export what's necessary
- [ ] **Permission-protect** exported Activities, Services, Receivers (`androidApp`):

```xml
<activity
    android:name=".DeepLinkActivity"
    android:exported="true"
    android:permission="com.example.DEEP_LINK">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <data android:scheme="example" android:host="task" />
    </intent-filter>
</activity>
```

- [ ] **iOS URL scheme payloads** validated in `onOpenURL` / scene delegate before routing into shared navigation
- [ ] **Web `postMessage`** — validate `event.origin`; never use a wildcard `targetOrigin` when sending

## Secrets Management

- [ ] **API keys never in `commonMain` source** — generate a build-time config object from gitignored `local.properties`/CI env (BuildKonfig-style codegen); remember js/wasm output is world-readable, so any key shipped to the browser must be public-safe
- [ ] **Signing material NOT in repository** — Android keystore, Apple signing certificates and provisioning profiles
- [ ] **CI secrets** in GitHub Secrets or equivalent vault
- [ ] **No hardcoded secrets** in source code (grep for patterns):

```bash
grep -rn "api_key\|apiKey\|secret\|password\|token" \
    --include="*.kt" --include="*.swift" --include="*.properties" \
    | grep -v "local.properties" | grep -v "test" | grep -v "build/"
```

## Code Shrinking & Log Stripping

- [ ] **R8 enabled** for `androidApp` release builds (`isMinifyEnabled = true`, `isShrinkResources = true`)
- [ ] **Logs gated in release on every target** — set Kermit minimum severity per build configuration in shared code; additionally strip `android.util.Log` in `androidApp`:

```proguard
-assumenosideeffects class android.util.Log {
    public static int d(...);
    public static int v(...);
    public static int i(...);
}
```

- [ ] **Keep rules** for serialized classes (kotlinx.serialization models, Room entities):

```proguard
-keep class com.example.shared.data.model.** { *; }
```

- [ ] **Mapping file** uploaded to Play Console for crash deobfuscation (`androidApp`)
- [ ] **js/wasm distribution treated as public source** — minification is not obfuscation-grade security; nothing sensitive relies on it

## Embedded Web Content

- [ ] **JavaScript** disabled unless required (Android `WebView`, `androidMain` interop)
- [ ] **File access** disabled (`allowFileAccess = false`)
- [ ] **URL validation** before navigation — whitelist trusted domains (`shouldOverrideUrlLoading` on Android, `decidePolicyFor navigationAction` on iOS `WKWebView`)
- [ ] **No JavaScript bridges** exposing sensitive operations (`addJavascriptInterface` / `WKScriptMessageHandler`)
- [ ] **Content loaded via HTTPS** only

## Dependency Management

- [ ] **Dependency verification** enabled in `gradle/verification-metadata.xml`
- [ ] **Version pinning** — no dynamic versions (`implementation("lib:+")`); versions centralized in `gradle/libs.versions.toml`
- [ ] **Vulnerability scanning** in CI (OWASP dependency-check or similar)
- [ ] **Unused dependencies** removed regularly
- [ ] **License compliance** verified for all dependencies

## Release Configuration

- [ ] **`android:debuggable`** not set in release (`androidApp`, defaults to false)
- [ ] **`android:allowBackup`** reviewed — sensitive data excluded (`androidApp`)
- [ ] **iOS release archive** uses the distribution configuration — no debug entitlements (`get-task-allow`), ATS not disabled
- [ ] **Desktop distribution** (`:desktopApp:packageDistributionForCurrentOS`) contains no debug flags or dev-server endpoints
- [ ] **Web production deploy** — source maps not published (or accepted as public)
- [ ] **Test code not shipped** — test dependencies only in `*Test` source sets, never `commonMain`
