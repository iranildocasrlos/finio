# Security Policy — Finio

> `br.com.localoeste.finio` · Last updated: April 2026

---

## Overview

Finio handles financial data and integrates with a real BaaS (Unit.co). Security is enforced at every layer — network, storage, UI, and CI/CD.

---

## What is never in the repository

```
.env                             # Unit.co token, Firebase project ID
google-services.json             # Firebase Android config
GoogleService-Info.plist         # Firebase iOS config
*.keystore / *.jks               # Android signing key
*.p8 / *.p12                     # iOS push certificate
lib/core/network/secrets.dart    # No hardcoded keys anywhere
```

All listed in `.gitignore` and enforced via pre-commit hook (gitleaks scan).

---

## Secret management

| Secret | Storage | Lifecycle |
|---|---|---|
| Unit.co API token | `flutter_secure_storage` | Injected by Dio interceptor per request |
| Firebase ID token | Firebase Auth SDK | Auto-refresh; 1h expiry |
| Firebase refresh token | Firebase Auth SDK | Stored by SDK in Keychain/Keystore |
| Card full number | In-memory only | 30s display; cleared on background |
| Card CVV | In-memory only | 30s display; cleared on background |
| Environment config | `.env` file (not in repo) | Injected at build time |

---

## Authentication layers

### Firebase Auth
- Email/password with client-side validation before any network call
- Google Sign-in via Firebase Google provider
- ID token lifespan: 1 hour (Firebase default)
- Auto-refresh: Dio interceptor catches 401 → refreshes → retries original request

### Biometric auth (`local_auth`)
Required for:
- App unlock after background (configurable — default: after 5 minutes)
- All payment confirmations (send money)
- Card detail reveal (full number + CVV)
- Card freeze/unfreeze

```dart
final confirmed = await _localAuth.authenticate(
  localizedReason: 'Confirm your identity to view card details',
  options: const AuthenticationOptions(
    biometricOnly: true,   // card reveal: biometric only, no PIN fallback
    stickyAuth: true,
  ),
);
if (!confirmed) return const Left(AuthFailure('Biometric not confirmed'));
```

### Token auto-refresh flow

```
Request sent with Bearer token
       |
   Response 401
       |
DioInterceptor calls Firebase to get new ID token
       |
New token written to flutter_secure_storage
       |
Original request retried with new token
       |
If refresh fails -> SessionExpiredFailure -> force re-login
```

---

## Card security

### Reveal flow

```
User taps "Show card details"
       |
local_auth biometric prompt (biometric only — no PIN for this action)
       |
On success: Dio -> Unit.co GET /cards/{id}/secure-data
       |
fullNumber and cvv held in CardState (BLoC state — in memory)
       |
UI displays with 30-second countdown timer
       |
On timeout OR app goes to background:
  CardBloc dispatches CardDetailsCleared
  State updated: fullNumber = null, cvv = null
  UI masks the number immediately
```

Card details are **never**:
- Written to Drift (SQLite)
- Written to Firestore
- Written to `flutter_secure_storage`
- Logged by `BlocObserver` (observer filters `CardRevealRequested` events)

### Screenshot protection

Card detail screens set `FLAG_SECURE` on Android (prevents screenshots and screen recording):

```dart
// In CardDetailPage.initState()
if (Platform.isAndroid) {
  await _channel.invokeMethod('setSecureFlag', true);
}

@override
void dispose() {
  if (Platform.isAndroid) {
    _channel.invokeMethod('setSecureFlag', false);
  }
  super.dispose();
}
```

---

## Network security

### TLS enforcement

**Android** (`android/app/src/main/res/xml/network_security_config.xml`):
```xml
<network-security-config>
  <base-config cleartextTrafficPermitted="false">
    <trust-anchors>
      <certificates src="system"/>
    </trust-anchors>
  </base-config>
</network-security-config>
```

**iOS** (`Info.plist`):
```xml
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsArbitraryLoads</key>
  <false/>
</dict>
```

### Certificate pinning (planned v2.0)

Production release will pin Unit.co and Firebase certificates using `dio_pinning` to protect against MiTM attacks on compromised networks.

---

## Data classification

| Data | Local (Drift) | Firestore | Memory | Encrypted |
|---|---|---|---|---|
| Transaction records | Yes | Yes (sync) | No | No (non-sensitive) |
| Category definitions | Yes | No | No | No |
| User profile | No | Yes | No | No |
| Card custom name | Yes | Yes | No | No |
| Account number (masked) | Last 4 digits only | No | No | — |
| Card full number | **Never** | **Never** | 30s only | — |
| Card CVV | **Never** | **Never** | 30s only | — |
| Unit.co API token | **Never** | **Never** | Via flutter_secure_storage | AES-256 |
| Firebase tokens | SDK managed | SDK managed | SDK managed | SDK managed |

---

## Sensitive UI

### Balance visibility toggle
Home screen balance can be hidden — tap to toggle. State persisted via `hydrated_bloc`. Useful in public spaces.

### Masked account number
Only the last 4 digits of the account number are ever displayed in the UI. The full number is only used for the "Share account details" feature (user copies it to send to a payer).

---

## CI security checks

- **gitleaks** runs on every push — blocks commits containing API keys or tokens
- **flutter analyze --fatal-infos** — static analysis blocks merge on any lint issue
- Dependency audit: `flutter pub outdated` checked monthly for security updates

---

## Compliance notes

Finio currently operates in **Unit.co sandbox mode** — no real money, no real accounts.

For production deployment:
- Unit.co production requires a registered legal entity (US/CA LLC or corporation)
- KYC/AML compliance provided by Unit.co via API (identity verification on account creation)
- Privacy policy required at `finio.app/privacy-policy` (LGPD for Brazil, PIPEDA for Canada)
- App Store financial app guidelines compliance review
- Play Store financial app declaration

---

## Reporting a security issue

This is a private application. If you have repository access and discover a vulnerability:

1. Do **not** create a GitHub issue
2. Contact Iranildo Silva directly via LinkedIn or email
3. Include description and reproduction steps
4. Response within 48 hours
