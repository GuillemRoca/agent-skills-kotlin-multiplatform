---
name: security-and-hardening
description: >-
  Use when handling sensitive data, authentication, or network communication in
  a Kotlin Multiplatform app, or before shipping a release to any target
  (Android, iOS, desktop, web). Three-tier framework (Always Do, Ask First,
  Never Do) with per-platform secret storage behind a common interface, Ktor
  TLS and certificate pinning, and target-aware hardening.
---

# Security and Hardening

## Overview

Security is a development constraint, not an afterthought. This skill provides a three-tier framework — Always Do, Ask First, Never Do — adapted for a codebase that ships one `commonMain` to Android, iOS, desktop, and the browser. The core discipline: secrets and trust decisions are platform concerns implemented in platform source sets, exposed to common code only through narrow interfaces. The full checklist lives in `references/security-checklist.md`.

## When to Use

- Handling user credentials, tokens, or personal data
- Implementing network communication (Ktor client, auth flows)
- Before shipping a release to any target
- Reviewing code that touches authentication or authorization
- Adding a new target (each target changes the attack surface)

**Skip when:**
- Changes are purely cosmetic with no data or network impact
- You need a general quality review — see `code-review-and-quality` (its Security axis links back here)

## Core Process

### Step 1: Always Do — Secrets at Rest Behind a Common Interface

1. **Define secret storage in commonMain as an interface** — never as expect/actual constants and never as plain DataStore values:

```kotlin
// commonMain — the only secret API common code ever sees
interface SecretStorage {
    suspend fun put(key: String, value: String)
    suspend fun get(key: String): String?
    suspend fun remove(key: String)
}
```

2. **Implement per platform and wire with Metro** from each platform source set:

```kotlin
// androidMain — AES key in the Android Keystore, ciphertext in DataStore
@ContributesBinding(AppScope::class)
@SingleIn(AppScope::class)
@Inject
class KeystoreSecretStorage(
    private val dataStore: DataStore<Preferences>,
) : SecretStorage {
    private val masterKey: SecretKey by lazy {
        val ks = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
        ks.getKey(KEY_ALIAS, null) as? SecretKey
            ?: KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
                .apply {
                    init(
                        KeyGenParameterSpec.Builder(
                            KEY_ALIAS,
                            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT,
                        )
                            .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                            .build()
                    )
                }
                .generateKey()
    }

    override suspend fun put(key: String, value: String) {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
            .apply { init(Cipher.ENCRYPT_MODE, masterKey) }
        val blob = cipher.iv + cipher.doFinal(value.encodeToByteArray())
        dataStore.edit { it[stringPreferencesKey(key)] = Base64.getEncoder().encodeToString(blob) }
    }
    // get() decrypts iv + ciphertext; remove() clears the preference

    private companion object { const val KEY_ALIAS = "secret_storage_key" }
}
```

Jetpack Security Crypto (`EncryptedSharedPreferences`) is deprecated and unmaintained — do not add it to new code.

```kotlin
// iosMain — Keychain Services; device-bound accessibility class
@OptIn(ExperimentalForeignApi::class)
class KeychainSecretStorage : SecretStorage {
    override suspend fun put(key: String, value: String) {
        remove(key) // replace-on-write
        val status = SecItemAdd(
            cfDictionaryOf(
                kSecClass to kSecClassGenericPassword,
                kSecAttrService to SERVICE,
                kSecAttrAccount to key,
                kSecValueData to value.encodeToByteArray().toCFData(),
                kSecAttrAccessible to kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
            ),
            null,
        )
        check(status == errSecSuccess) { "Keychain write failed: $status" }
    }
    // get() via SecItemCopyMatching, remove() via SecItemDelete;
    // cfDictionaryOf/toCFData are small cinterop helpers in iosMain

    private companion object { const val SERVICE = "com.example.app" }
}
```

3. **Desktop (`jvmMain`):** use the OS credential store — Windows Credential Manager, macOS Keychain, libsecret on Linux — via a credential-store library. Fall back to an AES-GCM-encrypted file only when no store is available, with the key derived from an OS-protected source. Plain files and `java.util.prefs.Preferences` are not acceptable for tokens.

4. **Web (`jsMain`/`wasmJsMain`): never store secrets client-side.** `localStorage`, `sessionStorage`, and non-HttpOnly cookies are readable by any injected script. Keep session tokens in memory only and rely on short-lived server sessions plus HttpOnly cookies:

```kotlin
// jsMain / wasmJsMain — session-scoped memory; gone on refresh, by design
class InMemorySecretStorage : SecretStorage {
    private val values = mutableMapOf<String, String>()
    override suspend fun put(key: String, value: String) { values[key] = value }
    override suspend fun get(key: String): String? = values[key]
    override suspend fun remove(key: String) { values.remove(key) }
}
```

### Step 2: Always Do — Network Security with Ktor

5. **TLS by default.** Every Ktor engine verifies certificates out of the box — never turn that off. All URLs are `https://`; treat any `http://` literal outside tests as a defect.

6. **Certificate pinning is per engine**, so it lives in platform source sets. Common code depends on `HttpClient` through DI; each platform provides its configured engine:

```kotlin
// androidMain — pin via the OkHttp engine
fun androidHttpClient(): HttpClient = HttpClient(OkHttp) {
    engine {
        config {
            certificatePinner(
                CertificatePinner.Builder()
                    .add("api.example.com", "sha256/PrimaryPinBase64=")
                    .add("api.example.com", "sha256/BackupPinBase64=") // always ship a backup
                    .build()
            )
        }
    }
}
```

```kotlin
// iosMain — pin via the Darwin engine's challenge handler
fun iosHttpClient(): HttpClient = HttpClient(Darwin) {
    engine {
        handleChallenge { _, _, challenge, completion ->
            val serverTrust = challenge.protectionSpace.serverTrust
            if (serverTrust != null && matchesPinnedPublicKey(serverTrust)) {
                completion(
                    NSURLSessionAuthChallengeDisposition.NSURLSessionAuthChallengeUseCredential,
                    NSURLCredential.credentialForTrust(serverTrust),
                )
            } else {
                completion(
                    NSURLSessionAuthChallengeDisposition
                        .NSURLSessionAuthChallengeCancelAuthenticationChallenge,
                    null,
                )
            }
        }
    }
}
```

Desktop (CIO/Java engine) pins through a custom `TrustManager`. Browser engines cannot pin — the web target relies on the browser's TLS plus server-side controls (HSTS, short-lived tokens).

7. **Validate input at every boundary** — deep links, query parameters, API responses — in `commonMain` so every target benefits:

```kotlin
fun parseTaskId(raw: String?): TaskId? =
    raw?.takeIf { it.matches(Regex("^[a-zA-Z0-9-]{1,36}$")) }?.let(::TaskId)
```

8. **Keep secrets out of the build.** No API keys or tokens as `commonMain` constants — commonMain compiles into every artifact, and the js/wasm bundle is plain text anyone can read in browser DevTools. Keys the client genuinely needs are injected at build time per target, kept out of git (`local.properties`, CI secrets), and treated as public identifiers, not credentials. Anything that must stay secret stays server-side behind your own API.

9. **Strip sensitive logging from release builds.** With Kermit, raise the minimum severity for release and never log token values at any severity:

```kotlin
Logger.setMinSeverity(if (isReleaseBuild) Severity.Warn else Severity.Verbose)
```

### Step 3: Ask First

10. **Changes that require human review before implementing:**
    - Modifying authentication or authorization logic
    - Changing certificate pinning configuration or any trust decision
    - Adding permissions (AndroidManifest entries, iOS Info.plist usage descriptions)
    - Integrating new third-party SDKs (they run in-process on every target you add them to)
    - Persisting a new category of personal data
    - Embedding web content (WebView / WKWebView) — see the scoped note in Step 5
    - Changing R8/ProGuard rules or release build configuration
    - Exposing new deep links or URL schemes

### Step 4: Never Do

11. **Absolute prohibitions:**

| Never | Why |
|-------|-----|
| Commit secrets to git | Secrets in git history are permanent, even after removal |
| Put secrets in `commonMain` constants | They ship in every binary — the js/wasm bundle is fully inspectable in DevTools |
| Store tokens in web `localStorage` or readable cookies | Any injected script exfiltrates them; use memory plus HttpOnly cookies |
| Log sensitive data (tokens, passwords) | Logs leak via logcat, crash reports, and the browser console |
| Disable TLS certificate verification | Man-in-the-middle attacks; no legitimate production reason exists |
| Store passwords or tokens in plain DataStore or files | Use `SecretStorage` (Keystore / Keychain / OS credential store) |
| Trust client-side validation alone | All client validation can be bypassed; the server must re-validate |
| Ship a debuggable or unminified release | Debug flags and readable code aid reverse engineering on every target |

### Step 5: Harden Per Target Before Release

12. **Run the target-aware checklist** (full version in `references/security-checklist.md`):

| Risk (OWASP mobile style) | Mitigation across targets |
|---|---|
| Improper credential usage | `SecretStorage` per platform; nothing persisted on web |
| Supply chain | Version catalog pinning, dependency verification, review every new SDK |
| Insecure authentication | OAuth2 with PKCE; BiometricPrompt (androidMain) / LocalAuthentication (iosMain) for re-auth |
| Input/output validation | Common validators at boundaries; server re-validates everything |
| Insecure communication | Ktor TLS everywhere; per-engine pinning; HSTS for web |
| Privacy controls | Minimize collection; per-platform permission prompts with rationale |
| Binary protections | R8 for androidApp; stripped release binaries for iOS/desktop; no source maps in web production |
| Misconfiguration | No debuggable release; review manifest, Info.plist, and web CSP headers |
| Insecure data storage | Encrypted at rest via `SecretStorage`; nothing sensitive in web storage |
| Insufficient cryptography | Platform key stores manage keys; never hand-rolled crypto in commonMain |

13. **Embedded web content (scoped note).** Shared Compose UI has no WebView; embedded web content appears only through `platform-interop` in `androidMain` (WebView) or the iOS side (WKWebView). If you must embed one: restrict navigation to an allowlist of trusted hosts, disable file and content access, and never bridge tokens or native objects into JavaScript.

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "It's just a debug key, I'll change it later" | Keys committed to git stay in history forever. |
| "Our API already validates input" | Client validation improves UX; server validation prevents exploits. Both needed. |
| "We don't need cert pinning for v1" | v1 is when MITM attacks are most damaging — no monitoring exists to detect them. |
| "The wasm target is just a demo" | The demo bundle contains your commonMain — including any constant you put there. |
| "localStorage is fine, everyone does it" | One XSS anywhere on the page and every stored token is stolen. Memory plus HttpOnly cookies. |
| "The Keychain code is verbose, DataStore is easier" | Plain DataStore is unencrypted at rest. The verbosity is one class, written once, behind `SecretStorage`. |

## Red Flags

- Secrets (API keys, tokens) anywhere in `shared/src/commonMain`
- Tokens written to DataStore, files, or web storage without the `SecretStorage` boundary
- `http://` URLs or disabled certificate verification in any engine configuration
- Certificate pinning with a single pin (no backup — one rotation bricks the app)
- Kermit or console logging of tokens, credentials, or personal data
- Auth logic that differs per platform (drifted copies instead of common code)
- New third-party SDK added without review
- Release builds shipping debug flags, web source maps, or unminified Android artifacts

## Verification

- [ ] No secrets in common code: `grep -rniE "api[_-]?key|secret|password" shared/src/commonMain` returns nothing sensitive
- [ ] `SecretStorage` implementations exist for every shipped target; web is memory-only
- [ ] All Ktor clients use HTTPS; pinning configured in androidMain (OkHttp) and iosMain (Darwin)
- [ ] Backup pin shipped alongside the primary pin
- [ ] Input validated at all boundaries (deep links, API responses) in commonMain
- [ ] Release logging severity raised; no token values logged at any severity
- [ ] R8 enabled for androidApp release; no source maps in the web production bundle
- [ ] `references/security-checklist.md` reviewed for every target this release ships
- [ ] `./gradlew :shared:allTests` passes after security changes
