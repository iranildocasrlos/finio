# Roadmap — Finio

> `br.com.localoeste.finio`
> Living document — updated as milestones are completed.

---

## Version history and plan

```
v1.0  Finance control MVP         ← in development
v1.1  Cloud sync + payments       ← planned
v2.0  Named card wallet           ← planned
v2.1  Analytics + goals           ← planned
v3.0  Multi-card + marketplace    ← future
```

---

## v1.0 — Finance control MVP

**Target:** 12 weeks from project start
**Goal:** Fully functional expense tracker with Clean Architecture, BLoC, CI, and tests.
Published to Play Store and App Store (internal testing track).

### Infrastructure
- [x] Flutter project scaffold — `br.com.localoeste.finio`
- [x] Clean Architecture folder structure (6 features)
- [x] Failure sealed class hierarchy
- [x] Dio client with token interceptor and auto-refresh
- [x] GetIt + injectable DI configuration
- [x] Material 3 theme with Finio colour tokens
- [x] GitHub Actions CI — lint + test + build
- [x] README, ARCHITECTURE, PORTFOLIO, SECURITY, FEATURES documentation

### Transactions feature
- [ ] Drift database — TransactionTable, CategoryTable
- [ ] TransactionDao and CategoryDao
- [ ] TransactionRepositoryImpl (local only for v1.0)
- [ ] All UseCases — Get, Add, Update, Delete, CalculateBalance
- [ ] TransactionBloc — all events and states
- [ ] Transaction list page with shimmer loading
- [ ] Add/edit transaction form with category picker
- [ ] Balance summary header widget
- [ ] Filter bottom sheet — date range, type, category

### Auth feature
- [ ] Firebase Auth setup (Android + iOS)
- [ ] Sign in with email/password
- [ ] Google Sign-in
- [ ] Biometric unlock (local_auth)
- [ ] AuthBloc with session management
- [ ] Onboarding flow (first launch only)

### App shell
- [ ] Bottom navigation — Home, Transactions, Payments, Wallet
- [ ] GoRouter navigation with auth guards
- [ ] Splash screen with Finio logo

### Quality
- [ ] Unit tests — TransactionBloc (12 cases), AuthBloc (6 cases)
- [ ] Widget tests — TransactionCard, BalanceSummary
- [ ] Coverage ≥ 70% enforced in CI

**Definition of done:** App installs, user can log transactions, view balance,
filter by month, sign in/out with biometrics. CI green. Published to internal testing.

---

## v1.1 — Cloud sync + payments

**Target:** 8 weeks after v1.0
**Goal:** Firestore sync working. Users can send and receive payments.

### Cloud sync
- [ ] Firestore — transactions collection with security rules
- [ ] Offline-first merge: local first, remote in background
- [ ] Conflict resolution — bank tx wins on amount/date, manual wins on description
- [ ] Firebase Cloud Messaging — push token management

### Payments feature
- [ ] Unit.co account creation on payment feature first access
- [ ] Account screen — masked account + routing number
- [ ] Send payment flow — recipient search, amount, note
- [ ] PaymentBloc — full state machine with biometric gate
- [ ] Receive payment — passive (webhook → Firestore → push notification)
- [ ] Payment history — merged with manual transactions
- [ ] ACH transfer (external, 1-3 day settlement)
- [ ] Unit.co webhook handler (via Firebase Cloud Functions)

### Quality
- [ ] PaymentBloc tests (10 cases)
- [ ] Integration test — send payment flow end to end
- [ ] Coverage maintained ≥ 70%

**Definition of done:** User can create account, send money to another Finio user,
receive a push notification when payment arrives, see updated balance.

---

## v2.0 — Named card wallet

**Target:** 10 weeks after v1.1
**Goal:** Virtual card with user-defined name live in the wallet.

### Named card feature
- [ ] Unit.co card issuance — `POST /cards`
- [ ] VirtualCard entity with `customName` field
- [ ] CardBloc — load, rename, lock/unlock, reveal number
- [ ] Card face widget — custom Flutter widget (no image assets)
- [ ] Card flip animation — front (name + masked number) / back (CVV)
- [ ] Reveal flow — biometric gate + 30s auto-hide + background hide
- [ ] Rename card — Drift + Firestore + Unit.co tags sync
- [ ] Lock / unlock card — Unit.co freeze/unfreeze API
- [ ] Screenshot prevention — FLAG_SECURE on wallet screen (Android)

### Wallet screen
- [ ] Wallet home — card face widget + balance + quick actions
- [ ] Card settings — rename, freeze, reveal number, report lost
- [ ] Smooth animated transitions between card states

### Quality
- [ ] CardBloc tests (8 cases)
- [ ] Widget test — CardFaceWidget renders customName correctly
- [ ] Screenshot prevention verified on device

**Definition of done:** User creates a card, names it "Travel Fund",
sees "Travel Fund" on the card face, can rename it, reveal the number
with biometrics, and freeze it.

---

## v2.1 — Analytics, goals, and export

**Target:** 6 weeks after v2.0
**Goal:** Full analytics suite. PDF export. Budget goals with alerts.

### Analytics feature
- [ ] Monthly overview — balance trend, 6-month bar chart (fl_chart)
- [ ] Spending by category — pie chart with drill-down
- [ ] GetTotalsByCategoryUseCase
- [ ] AnalyticsBloc

### Budget goals
- [ ] Budget entity — category + monthly limit
- [ ] BudgetBloc — track progress, trigger alert at 80%
- [ ] Push notification at 80% and 100% of budget
- [ ] Over-budget visual state in category list

### PDF export
- [ ] Monthly report — summary + transaction list + category chart
- [ ] printing package integration
- [ ] Share sheet (share_plus)

### Quality
- [ ] AnalyticsBloc tests
- [ ] PDF output validated on device

---

## v3.0 — Multi-card + future features

**Target:** Post-v2.1, timing TBD

### Multi-card
- [ ] Create up to 3 named virtual cards per account
- [ ] Each card has independent name and spending tracking
- [ ] Card selector in wallet — horizontal scroll

### Marketplace integrations (research phase)
- [ ] Bill payment via partner APIs
- [ ] Subscription tracker — detect recurring charges automatically
- [ ] Savings goals — dedicated sub-account per goal

### International
- [ ] Localisation — Portuguese (pt-BR) + English (en-CA)
- [ ] Currency support — BRL + CAD + USD
- [ ] Play Store and App Store — Brazil + Canada regions

---

## Milestone tracking

| Milestone | Target | Status |
|---|---|---|
| v1.0 infrastructure complete | Week 2 | Done |
| v1.0 transactions feature | Week 6 | In progress |
| v1.0 auth + app shell | Week 10 | Planned |
| v1.0 published (internal) | Week 12 | Planned |
| v1.1 payments live | Week 20 | Planned |
| v2.0 named card wallet | Week 30 | Planned |
| v2.1 analytics + export | Week 36 | Planned |
