# Named Card System — Finio

> Feature specification for the custom card naming system.
> This is Finio's signature UX differentiator.

---

## Concept

Every Finio user has one virtual Visa card. Unlike most digital wallet apps that display a generic card number, Finio lets the user name their card — and that name is displayed on the card face.

A user might name their card:
- "My Finio Card" (default)
- "Travel Budget"
- "Daily Expenses"
- "Emergency Fund"
- "Sarah's Card"

The name is personal, meaningful, and makes the wallet feel like it belongs to that person.

---

## Rules

| Rule | Value |
|---|---|
| Max length | 20 characters |
| Min length | 1 character (after trim) |
| Allowed characters | Letters, numbers, spaces, hyphens, apostrophes |
| Default (no name set) | "My Finio Card" |
| Editable | Yes — user can rename at any time |
| Synced across devices | Yes — via Firestore real-time listener |
| Displayed on card face | Yes — main label on the card widget |

---

## Domain entity

```dart
class FinioCard extends Equatable {
  final String id;
  final String customName;      // user's label — stored in Firestore
  final String maskedNumber;    // "•••• •••• •••• 4321" — always visible
  final String? fullNumber;     // in memory only after biometric reveal
  final String expiry;          // "MM/YY"
  final String? cvv;            // in memory only after biometric reveal
  final CardStatus status;      // active | frozen | closed
  final DateTime createdAt;

  // Always returns something — never shows empty string on card face
  String get displayName =>
      customName.trim().isNotEmpty ? customName.trim() : 'My Finio Card';

  bool get isPersonalized => customName.trim().isNotEmpty;
  bool get isRevealed => fullNumber != null;
  bool get isFrozen => status == CardStatus.frozen;
}
```

---

## BLoC events

```dart
// User submits a new card name
final class CardNameUpdated extends CardEvent {
  final String cardId;
  final String name;           // raw input — UseCase validates and trims
  const CardNameUpdated({required this.cardId, required this.name});
}

// User clears the custom name (resets to default)
final class CardNameCleared extends CardEvent {
  final String cardId;
  const CardNameCleared(this.cardId);
}
```

---

## UseCase

```dart
class UpdateCardNameParams extends Equatable {
  final String cardId;
  final String name;
  const UpdateCardNameParams({required this.cardId, required this.name});
}

@injectable
class UpdateCardNameUseCase {
  final CardRepository _repository;
  const UpdateCardNameUseCase(this._repository);

  Future<Either<Failure, FinioCard>> call(UpdateCardNameParams params) async {
    final trimmed = params.name.trim();

    if (trimmed.isEmpty) {
      return const Left(ValidationFailure('Card name cannot be empty'));
    }
    if (trimmed.length > 20) {
      return const Left(ValidationFailure('Card name must be 20 characters or less'));
    }
    if (!_isValidName(trimmed)) {
      return const Left(ValidationFailure('Card name contains invalid characters'));
    }

    return _repository.updateCardName(params.cardId, trimmed);
  }

  bool _isValidName(String name) {
    // Letters, numbers, spaces, hyphens, apostrophes only
    return RegExp(r"^[a-zA-Z0-9\s\-\']+$").hasMatch(name);
  }
}
```

---

## Storage

The custom name lives in **two places**:

### Firestore (source of truth)

```
users/{userId}/cardPreferences/{cardId}
  └── customName: "Travel Budget"
      updatedAt: Timestamp
```

A real-time listener in `CardRepositoryImpl` watches this document. When the name changes on one device, all other devices update instantly via Firestore `snapshots()`.

### Drift (local cache)

```dart
class CardPreferencesTable extends Table {
  TextColumn get cardId     => text()();
  TextColumn get customName => text().withDefault(Constant(''))();
  DateTimeColumn get updatedAt => dateTime()();

  @override
  Set<Column> get primaryKey => {cardId};
}
```

Drift stores the name locally for offline access. Firestore is the source of truth — Drift is updated on every sync.

---

## Card naming flow — step by step

```
1. User opens Cards screen
   → CardBloc dispatches CardLoadRequested
   → CardRepositoryImpl reads from Drift (immediate)
   → Firestore listener active (real-time updates)
   → UI shows card with current name

2. User taps "Name your card" (or pencil icon on card face)
   → Bottom sheet opens with text field
   → Pre-filled with current customName (if set)
   → Character counter shows remaining chars (20 - current length)

3. User types name and taps "Save"
   → CardBloc dispatches CardNameUpdated(cardId, name)
   → UpdateCardNameUseCase validates input
   → On validation pass: CardRepositoryImpl writes to Firestore
   → Drift updated locally
   → BLoC emits CardState with updated card
   → Bottom sheet closes
   → Card face updates with new name (animated transition)

4. Other devices
   → Firestore listener triggers on name change
   → CardBloc receives CardNameUpdated from stream
   → UI updates without user action
```

---

## Card face UI

The card widget displays the custom name as the primary label:

```
┌─────────────────────────────────┐
│  finio                          │
│                                 │
│  •••• •••• •••• 4321           │
│                                 │
│  Travel Budget          12/27   │
│                           VISA  │
└─────────────────────────────────┘
```

When revealed (biometric confirmed):

```
┌─────────────────────────────────┐
│  finio                          │
│                                 │
│  4111 1111 1111 4321           │
│                                 │
│  Travel Budget          12/27   │
│  CVV: 123               VISA   │
└─────────────────────────────────┘
       [30s countdown bar]
```

---

## Name change — edge cases

| Scenario | Behaviour |
|---|---|
| User clears the name field and saves | Rejected by UseCase — name cannot be empty. Field reverts to previous value. |
| User submits same name as current | UseCase writes to Firestore anyway (idempotent). No visible change. |
| Firestore update fails (offline) | Local Drift write succeeds. Firestore retries automatically when online. Drift is shown in UI. |
| Two devices update simultaneously | Last write wins (Firestore default). No conflict resolution needed — only one user, one card. |
| Card is frozen | Name can still be updated. Freeze only affects payment processing, not preferences. |

---

## Future — multiple named cards (v3.0 consideration)

The current model is one card per user. A future version could allow multiple cards with different names for different purposes:

```
"Daily Card"     — linked to main balance
"Travel Budget"  — linked to a separate savings bucket
"Kids Card"      — spending limit set by parent
```

The naming system is designed with this in mind: `customName` is per-card (not per-user), and `CardPreferencesTable` uses `cardId` as the primary key. Adding multiple cards would not require changing the naming architecture.
