# NsLoyalty Module - Updated Planning

## Research Summary (context gathered)
- Core rewards (unchanged, `nexopos.*`) = group-based rules → coupons. Tables: `nexopos_rewards_system`, `nexopos_rewards_system_rules`, `nexopos_customers_rewards`. Trigger: `OrderAfterCheckPerformedEvent` → `ProcessCustomerOwedAndRewardsJob`. Permissions: `nexopos.*.rewards`. Routes in `routes/web/customers.php` + `routes/api/rewards.php`. Menu under Customers.
- Order events available: `OrderAfterPaymentStatusChangedEvent` (payment=paid), `OrderVoidedEvent`, `OrderAfterRefundedEvent`.
- Earning pattern precedent: NsCommissions listens `OrderAfterPaymentStatusChangedEvent` → `AwardLoyaltyPointsJob` equivalent.
- User CRUD has `type: 'switch'` fields (see `app/Crud/UserCrud.php`).
- Options service = global-only; per-store config must use a settings table with `store_id` scoping or prefixed option keys.
- Module convention precedent: NsCommissions, NsAppointments (Routes/web.php, api.php, multistore.php, Providers/ModuleServiceProvider.php, Settings/*, Events/Listeners/, Console/Kernel.php).

## Updated Module Design (per user requirements — payment/wallet system)
- Namespace `NsLoyalty`, module `modules/NsLoyalty/`
- **Points are a wallet**: earn from POS transactions → redeem for order discounts or store credit (currency-like)
- **Earn**: for every X money spent, earn Y points (configurable rules)
- **Exchange rate**: X points → $Y (points-to-money conversion)
- **Default on**: all registered customers EXCEPT individual "deny/eligible" switch in User/Customer CRUD
- **Multistore**: global default config + per-store overrides; wallet balance global; POS checkout resolves to transaction store
- **Config page**: dashboard settings hook + POS checkout UI
- Independent of core rewards (no shared tables/FKs; a customer can have both)

## Updated Schema
1. `nexopos_loyalty_rules` — earning rules: id, store_id (NULL = network default), spend_threshold, points_awarded, label
2. `nexopos_loyalty_exchange_rates` — id, store_id (NULL = network default), currency, points_per_currency_unit (e.g., 100 pts per $1)
3. `nexopos_loyalty_balances` — per-customer wallet: customer_id FK, total_points, lifetime_spend, is_eligible, updated_at
4. `nexopos_loyalty_transactions` — audit ledger: id, customer_id, type (earned/redemption/store-credit-issued/expired/adjustment/refund-chargeback), points_delta (signed), balance_after, reference_type/id, order_id, note, created_at
5. `nexopos_loyalty_redemptions` — redemption: id, customer_id, points_used, money_value, money_currency, reference_type/id, order_id, reason, created_at

## Earning / Redemption Mechanics
- **Trigger**: `OrderAfterPaymentStatusChangedEvent` (payment = paid) → `AwardLoyaltyPointsJob` → find applicable rule(s) for the transaction store → award points (dedupe: mark once per order, e.g., check ledger for prior award or set flag)
- **Clawback**: `OrderVoidedEvent` + `OrderAfterRefundedEvent` listeners → reverse earned points, log `refund-chargeback` transaction (critical — prevents ordering/voiding point farming)
- **Redemption options** (configurable per store):
  a. **Discount at checkout** (primary): points → cart/order discount via existing discount system
  b. **Store credit**: points → `CustomerAccountHistory` credit (currency-like)
  c. **External cash-out** (advanced): points → PayPal/bank payout, admin-approved, accounting entries
- **POS integration**: loyalty balance shown at payment step; points applied as order discount before final payment; partial payments allowed (points cover part, remainder via normal payment types)
- **Earn base**: subtotal by default (exclude tax/shipping); discounts reduce earn base; optional min-order threshold

## User/Customer CRUD switch
- Global: `ns_loyalty_enabled` (yes/no) — module settings
- Per-user: "Loyalty Eligible" Yes/No switch in `CustomerCrud` (users with storecustomer role) — overrides global; stored via a `loyalty_eligibility` table keyed by user_id (cleaner than user-table column)

## Multistore config
- **Table-based settings** (not string-prefixed options): `loyalty_rules`/`exchange_rates` with `store_id NULL` = network default, `store_id N` = store override
- Resolution hierarchy at POS checkout: transaction store config → network default → module default constants
- **Wallet scope**: global balance per customer (one wallet), store-level earn rates and exchange rates per store
- Currency handling: earn rate normalized per currency; multistore with multiple currencies needs currency conversion in earn calculation

## Settings page
- Hook `ns.settings` → `LoyaltySettings` (pattern: NsCommissions `ns.commissions-settings`)
- Tabs: General (enable module), Earning (rates, thresholds, earn-base exclusions, POS behavior), Exchange (points-to-money), Redemption (discount vs store credit vs cash-out, per-order caps), Reporting/Accounting notes

## Events (extensibility + accounting hooks, mirroring NsCommissions)
- `LoyaltyPointsEarnedEvent`, `LoyaltyPointsRedeemedEvent`, `LoyaltyBalanceUpdatedEvent` (expiries as needed later)
- Accounting hooks: `ns-accounting-events` + `ns-accounting-actions` (like NsCommissions) for deferred-revenue: loyalty accrual (credit liability), redemption (debit liability / revenue recognition), store-credit transfer, cash-out payout

## Phases
- Phase 1: Rules/exchange/balance/ledger tables, earning job + clawback, settings, customer CRUD switch
- Phase 2: POS checkout UI (show balance, redeem), admin transaction view, permissions
- Phase 3: Store credit + external cash-out, accounting hooks, reporting
- Phase 4: Expiry job + notifications, welcome bonus, optional promo multipliers / tiers

## Payment-system integration proposal (req 1 — my recommendation)
**Hybrid approach:**
1. Primary = **discount integration** at POS/cart (points → order discount). Familiar to customers, simplest, works with all payment types, refunds handle discount reversal naturally.
2. Secondary = **store credit conversion** (points → account credit via CustomerAccountHistory) — gives the "currency" feel without external payouts.
3. Advanced = **external cash-out** (PayPal/bank) gated behind admin approval + accounting entries.
4. Add **accounting hooks** for deferred-revenue tracking: unredeemed points = liability; redemption = revenue recognition.

Rationale: flexible per store; discount path is simplest/most common; store credit adds currency feel; accounting hooks make finance-safe; cash-out optional. Alternatives: pure discount (no accounting) vs pure accounting (complex, forced currency model).

## Overlooked items surfaced by user (req 7 — additions to original list)
1. **Clawback on void/refund** (critical anti-fraud)
2. **Min-order threshold** for earning
3. **Earn-base exclusions** (tax/shipping/discounts)
4. **POS checkout UI** — balance display, redeem, partial-payment with points
5. **Customer view** — balance + transaction history (POS + account page)
6. **Admin point adjustments** — staff grant/withdraw points with reason (logged in ledger)
7. **Fraud prevention** — usability wait period before points redeemable; void-loop prevention; per-order redemption caps; dedupe-once flag
8. **Refund logic** — proportional vs all-or-nothing points deduction
9. **External cash-out workflow** — admin approval, accounting entries, compliance
10. **Currency conversion** in multistore (different currencies per store)
11. **Notifications** — points earned, redeemed, expiring
12. **Permissions** — `ns.loyalty.read/manage/redeem/adjust`
13. **Reporting** — points issued/redeemed/liability balance
14. **Welcome bonus** points on first order (optional)
15. **Tiers / promo multipliers** — optional Phase 4+ extension (not in current scope)
16. **Test strategy** — queue jobs, events, multi-store configs, currency, clawback
