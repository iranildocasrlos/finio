# Feature Specification — Finio

> Detailed description of all Finio features, user flows, and technical notes.
> `br.com.localoeste.finio`

---

## 1. Finance control

The core feature — migrated from the original Java Android app.

### 1.1 Transaction logging

Users log any financial event manually:

- **Amount** — positive number, formatted as currency (intl package)
- **Type** — income or expense
- **Category** — selected from a customisable list with icon and colour
- **Description** — free text, optional
- **Date** — defaults to today, editable

Transactions are saved instantly to Drift (SQLite) — no network required.
Firestore sync happens in background when online.

### 1.2 Custom categories

Users can create, edit, and delete categories:

- Name (free text)
- Icon (selected from Material Icons grid)
- Colour (selected from a palette — maps to AppTheme colour tokens)
- Type (income or expense — a category belongs to one type)

Default categories are seeded on first launch:
`Food`, `Transport`, `Housing`, `Health`, `Entertainment`, `Salary`, `Freelance`, `Other`

### 1.3 Balance calculation

```dart
// CalculateBalanceUseCase — pure, no dependencies, directly testable
static double fromList(List<Transaction> transactions) =>
    transactions.fold(0.0, (acc, t) => acc + t.signedAmount);

// signedAmount is a computed property on Transaction:
double get signedAmount => isIncome ? amount : -amount;
```

Balance is shown for:
- All time (account total)
- Current month (home screen default)
- Custom range (filter screen)

### 1.4 Filters and search

- Filter by: date range, type (income/expense), category
- Search by description text (Drift full-text search)
- Sort by: date (default), amount

### 1.5 Offline-first

All finance control features work without internet.
Data is written to Drift first. Firestore sync is opportunistic — no UI blocking.

---

## 2. Digital payments

Integrated payment features powered by Unit.co BaaS.

### 2.1 Account setup

On first use of payment features, Finio creates a Unit.co deposit account for the user:

```
POST /accounts
{
  "data": {
    "type": "depositAccount",
    "attributes": {
      "depositProduct": "checking",
      "tags": { "finioUserId": "firebase-uid-here" }
    }
  }
}
```

The user receives a real account number and routing number.
These are displayed on the Account screen (masked by default, full on biometric confirm).

### 2.2 Send money

**Flow:**

1. User opens "Send" — enters amount, finds recipient by phone or Finio username
2. Adds optional note ("Dinner split", "Rent")
3. Reviews summary screen — amount, recipient, available balance
4. Biometric confirmation required
5. Payment dispatched via Unit.co:
   - Same Finio user → `POST /payments/book` (instant)
   - External account → `POST /payments/ach/debit` (1-3 business days)
6. Success: transaction added to local list, balance updated
7. Recipient receives push notification (Firebase Messaging)

**BLoC states:**
`idle → validating → awaitingConfirmation → processing → success / failure`

**Typed failures:**
- `InsufficientFundsFailure` — shows current balance + "Top up" CTA
- `RecipientNotFoundFailure` — suggests checking the number/username
- `PaymentLimitExceeded` — shows the limit and when it resets
- `NetworkFailure` — offers retry with cached payment data

### 2.3 Receive money

Users can receive money two ways:

**Passive receipt** (another Finio user sends to them):
- Unit.co books the transaction — balance updates automatically on next sync
- Push notification delivered via Firebase Messaging
- Transaction appears in history with `source: bankSync`

**Request payment:**
- User generates a payment request link or QR code with a pre-filled amount
- Recipient opens link → directed to Finio or a web payment page (v2 feature)

### 2.4 Transaction history

Transactions from two sources are merged into a single timeline:

| Source | Type | Example |
|---|---|---|
| Manual | Income/expense logged by user | "Grocery — $45.00" |
| Bank sync | Unit.co API transactions | "Payment received — $120.00" |

Merge key: `bankTransactionId` — if a manual entry matches a bank transaction,
the bank version is authoritative (amount, date) and the manual description is preserved.

---

## 3. Named virtual card

Finio's most distinctive feature.

### 3.1 Card creation

On first access to the Wallet screen, if the user has no card:

1. Prompt: "Create your virtual card"
2. User types a custom name — any text up to 20 characters
   Examples: `Travel Fund`, `Emergency`, `Side Hustle`, `Groceries`
3. Card is created via Unit.co Issuing API
4. Card face renders immediately with the chosen name

```
POST /cards
{
  "data": {
    "type": "individualVirtualDebitCard",
    "attributes": {
      "depositAccountId": "unit-account-id",
      "tags": { "customName": "Travel Fund", "finioUserId": "firebase-uid" }
    }
  }
}
```

### 3.2 Card face — custom Flutter widget

The card is not a static image. It is a Flutter widget that:
- Renders the custom name as live text on the card face
- Shows masked number `•••• •••• •••• 4242`
- Displays expiry month/year
- Uses Finio brand colours (from AppTheme)
- Supports a card flip animation to reveal the back (CVV side)

```dart
class CardFaceWidget extends StatelessWidget {
  final VirtualCard card;
  final bool isFlipped;

  @override
  Widget build(BuildContext context) {
    return AnimatedSwitcher(
      duration: const Duration(milliseconds: 400),
      child: isFlipped ? _CardBack(card: card) : _CardFront(card: card),
    );
  }
}

class _CardFront extends StatelessWidget {
  @override
  Widget build(BuildContext context) => Container(
    decoration: BoxDecoration(
      color: context.colors.primary,
      borderRadius: BorderRadius.circular(16),
    ),
    child: Column(
      children: [
        Text(card.customName, style: ...),  // user-defined name
        Text(card.maskedNumber, style: ...),
        Text('${card.expiryMonth}/${card.expiryYear}', style: ...),
      ],
    ),
  );
}
```

### 3.3 Rename card

Users can rename their card at any time:

1. Card settings → "Rename card"
2. Type new name (same 20 char limit)
3. CardNameUpdated event dispatched
4. Local Drift update (instant UI)
5. Firestore update (cross-device sync)
6. Unit.co PATCH tags.customName (async)

### 3.4 Reveal card number

Full card number and CVV are sensitive — never stored locally.

**Flow:**
1. User taps "Show number"
2. `CardNumberRevealed` event dispatched
3. BLoC requests biometric auth
4. On success: `GET /cards/{id}/secure-data` from Unit.co
5. Full number and CVV displayed for 30 seconds
6. Auto-hidden on timeout or when app goes to background

### 3.5 Lock / unlock card

- Tap "Freeze card" → `POST /cards/{id}/freeze` → card status: frozen
- Tap "Unfreeze card" → `POST /cards/{id}/unfreeze` → card status: active
- Frozen card: all charges declined, card face shows "Frozen" badge
- Lock/unlock does not require biometric (visibility, not money movement)

### 3.6 Multiple cards (v2.1)

In a future version, users can create multiple named cards:
- Each card has its own custom name and balance allocation
- Use case: separate cards for different spending categories
- Unit.co supports multiple cards per account

---

## 4. Analytics

### 4.1 Monthly overview

Home screen shows:
- Total balance (income − expense, current month)
- Income total for the month
- Expense total for the month
- Quick bar chart — last 6 months spending

### 4.2 Spending by category

Pie chart and ranked list:
- Each slice = one category, coloured with category colour
- Tap a slice → drill down to that category's transactions

### 4.3 Budget goals

Users set a monthly budget per category:
- Visual progress bar: spent / budget
- Alert at 80% (push notification via Firebase)
- Over-budget categories highlighted in red

### 4.4 PDF export (v2.1)

Generate a monthly or custom-range PDF report:
- Finio header with logo
- Summary: balance, income, expenses
- Transaction list
- Category breakdown chart (fl_chart rendered to image)

---

## 5. Settings and profile

- Edit display name and profile photo
- Change card custom name
- Toggle balance visibility (hide/show all amounts)
- Biometric authentication toggle (on by default)
- Push notification preferences
- Export data as CSV (transactions)
- Delete account (with confirmation)
- Sign out
