# Architecture Decision Records — Finio

> `br.com.localoeste.finio`
> Each ADR documents a significant decision, its context, and trade-offs.

---

## ADR-001 · Clean Architecture

**Decision:** 3-layer separation per feature — `domain`, `data`, `presentation`.

**Rules:**
- `domain/` — zero imports from Flutter, Drift, Firebase, or Dio. Pure Dart.
- `data/` — depends on `domain/`. Owns all external integrations.
- `presentation/` — depends on `domain/` (UseCases only). Owns Flutter UI.

**Trade-off:** More files per feature. Accepted because each layer tests in isolation and swapping providers (e.g., Unit.co → Synctera) only touches `data/`.

---

## ADR-002 · BLoC over Riverpod

**Decision:** `flutter_bloc` with Dart 3 sealed classes across all features.

**Why BLoC for Finio:**
- Explicit event→state flow is auditable — essential for financial operations
- `BlocObserver` provides free global logging of all state transitions
- `bloc_test` gives deterministic, readable test syntax
- Sealed classes make every event handled — compiler-enforced, no silent gaps
- Canadian fintech companies (RBC, TD Bank Flutter teams) predominantly use BLoC

**BlocObserver in Finio:**
```dart
class FinioBlocObserver extends BlocObserver {
  @override
  void onEvent(BlocBase bloc, Object? event) {
    // Every financial action is logged — useful for support and debugging
    _logger.info('[EVENT] ${bloc.runtimeType} → ${event.runtimeType}');
  }
  @override
  void onError(BlocBase bloc, Object error, StackTrace stack) {
    _logger.error('[ERROR] ${bloc.runtimeType}: $error');
  }
}
```

---

## ADR-003 · Either<Failure, T> for all async operations

**Decision:** Use `dartz` Either. No raw exceptions propagate beyond repositories.

**Failure hierarchy:**
```dart
sealed class Failure { final String message; }

// Network
final class NetworkFailure           extends Failure {}
final class TimeoutFailure           extends Failure {}
final class ServerFailure            extends Failure { final int? statusCode; }

// Auth
final class AuthFailure              extends Failure {}
final class SessionExpiredFailure    extends Failure {}
final class InvalidCredentialsFailure extends Failure {}

// Financial
final class InsufficientFundsFailure extends Failure {}
final class AccountNotFoundFailure   extends Failure {}
final class TransactionFailure       extends Failure {}
final class CardFailure              extends Failure {}

// Local
final class CacheFailure             extends Failure {}
final class ValidationFailure        extends Failure {}
```

**Why:** Dart function signatures don't communicate exceptions. `Either` makes failure visible at every call site. The compiler prevents ignoring it. In a financial app, silent failures are unacceptable.

---

## ADR-004 · Named virtual card as core feature

**Decision:** Every Finio user gets one virtual card they can name themselves.

**Context:** Most digital wallet apps show generic card numbers. Finio makes the card personal — the user names it, and that name is displayed on the card face.

**Implementation decisions:**

| Decision | Rationale |
|---|---|
| Name stored in Firestore, not Unit.co | Unit.co card objects don't have a custom label field. Firestore is the right place for user preferences. |
| Name synced across devices via Firestore listener | User changes card name on phone; tablet sees it in real time. |
| Max 20 characters | Fits on card face at any font size. |
| Default to "My Finio Card" | Always shows something meaningful even before user personalizes. |
| Full card number in memory only | Never written to SQLite or Firestore. Cleared after 30s or on background. |
| Biometric required for reveal | One-way gate — once you authenticate, you see details. No PIN fallback for card reveal (biometric only). |

**Card entity design:**
```dart
class FinioCard extends Equatable {
  final String customName;   // Firestore — user's label
  final String? fullNumber;  // memory only — null when not revealed
  final String? cvv;         // memory only — null when not revealed
  ...
  String get displayName => customName.isNotEmpty ? customName : 'My Finio Card';
  bool get isRevealed => fullNumber != null;
}
```

---

## ADR-005 · Unit.co as BaaS provider

**Decision:** Unit.co for virtual accounts, card issuing, and payment processing.

**Evaluated options:**

| Provider | Sandbox | Canada focus | Dev-friendly | Decision |
|---|---|---|---|---|
| Unit.co | Free, complete | Strong | High | **Chosen** |
| Stripe Treasury | Free | US only | High | Rejected — no Canada |
| Synctera | On request | US/CA | Medium | Rejected — no free sandbox |
| Galileo | On request | US | Low | Rejected |

**Unit.co chosen because:**
- Free sandbox with real accounts, cards, and ACH — no sales call
- Clean JSON:API REST spec maps well to Dart models
- Strong US/Canada focus matches Finio's target market
- Excellent documentation for solo developers

**Integration boundary:** Unit.co is only accessed from `data/datasources/`. The `domain` layer knows nothing about Unit.co. If Finio migrates to Synctera, only `UnitAccountDataSource` and `UnitCardDataSource` are replaced.

---

## ADR-006 · Drift for local storage

**Decision:** Drift (type-safe SQLite) over Hive or raw sqflite.

**Why not Hive:** Financial data needs relational queries — balance aggregations across date ranges, category joins, sorted transaction history. Key-value stores can't do this efficiently.

**Why not raw sqflite:** String SQL queries break at runtime. Drift generates type-safe Dart from table definitions — same model as Android Room, familiar from the original Java app.

**Offline-first strategy:**
1. Drift returns data immediately (always fast, works offline)
2. Unit.co sync runs in background on `SyncBankTransactionsUseCase`
3. Firestore listener updates user preferences in real time
4. Conflict resolution: bank transactions are read-only (Unit.co is source of truth); manual transactions are local-first (Drift is source of truth)

---

## ADR-007 · Send & receive architecture

**Decision:** Two payment paths — internal (book transfer) and external (ACH).

**Internal transfer (Finio to Finio):**
```
Unit.co POST /payments/book
  No clearing delay — instant
  Both users see the transaction in real time
  Firebase Cloud Function updates both Firestore docs
```

**External ACH:**
```
Unit.co POST /payments/ach/debit
  1-3 business day clearing (ACH standard)
  Status tracked via Unit.co webhook -> Firebase -> push notification
  Drift record created immediately with "pending" status
  Updated to "settled" on webhook confirmation
```

**Why not support ACH credit (push to external) in v2.0:**
Unit.co ACH credit requires additional compliance review. Debit (pull from external) is simpler to enable in sandbox. Credit is planned for v2.1 after production onboarding.

---

## ADR-008 · Private repository

**Decision:** Source code is proprietary, not open source.

**Reasons:**
1. BaaS API token patterns and integration details must not be public
2. Firebase security rules are project-sensitive
3. Card reveal and biometric implementation details are a security surface
4. The card naming system is a proprietary UX differentiator

**Portfolio strategy:**
- `README.md` — complete technical documentation (public on request)
- `docs/PORTFOLIO.md` — recruiter-facing summary (this file)
- Live code walkthrough during technical interviews
- Isolated code challenge covering same patterns (available on request)
- TestFlight + Play Store internal testing (available on request)

---

## ADR-009 · GetIt + injectable for DI

**Decision:** Service locator pattern with code-generated wiring.

**Scope strategy:**
- `@singleton` — `DioClient`, `AppDatabase` (one per app lifecycle)
- `@lazySingleton` — all Repository implementations (created on first use)
- `@injectable` — all UseCases and BLoCs (new instance per injection point)

**Generated injection graph (key nodes):**
```
AppDatabase (singleton)
  -> TransactionDao
  -> CategoryDao
  -> CardPreferencesDao

DioClient (singleton)
  -> unitDio    (Dio configured for Unit.co)
  -> authDio    (Dio configured for Firebase Auth REST)

TransactionRepositoryImpl (lazySingleton)
  -> TransactionLocalDataSource  <- AppDatabase
  -> TransactionRemoteDataSource <- DioClient.unitDio

CardRepositoryImpl (lazySingleton)
  -> UnitCardDataSource     <- DioClient.unitDio
  -> CardPreferencesSource  <- Firestore + AppDatabase

CardBloc (injectable)
  -> GetCardUseCase
  -> UpdateCardNameUseCase
  -> RevealCardUseCase
  -> FreezeCardUseCase
```
